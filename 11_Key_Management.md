---
tags: #parasite #phase-11 #key-management #cryptography #security
version: 1.1
status: In Progress
author: princetheprogrammer
---

# Phase 11: Key Management — Key Hierarchy, Provisioning & Lifecycle

## Document Metadata
| Field | Value |
|-------|-------|
| Phase | 11 |
| Title | Key Management |
| Version | 1.1 |
| Last Updated | 2025-12-16 |
| Author | princetheprogrammer |
| Dependencies | [[03_System_Architecture]], [[09_Attestation_Protocol]] |

---

## 1. Executive Summary

This document specifies the key management architecture for the PARASITE project. The security of all cryptographic operations, including device identity, secure reporting, attestation, and secure updates, depends entirely on the proper management of the underlying cryptographic keys. This document details the complete key lifecycle: their generation, secure provisioning into devices, storage, usage, and eventual revocation. A flawed key management strategy can render even the strongest cryptographic algorithms useless.

Our architecture is based on a hierarchical model that establishes a clear chain of trust, originating from a physically protected hardware root. The key principles are **key separation** (a single key should not be used for multiple, unrelated purposes) and **minimizing key exposure** (private keys should never be exportable or accessible by application software).

The key hierarchy consists of:
1.  **Hardware Root of Trust (HRoT):** An immutable, device-unique value, such as a PUF output or a fused key.
2.  **Device Identity Key (DIK):** An asymmetric private key (ECDSA P-256) derived from the HRoT and used exclusively for signing attestations and establishing secure sessions. This key is the long-term identity of the device.
3.  **Firmware Update Keys:** A set of asymmetric public keys embedded in the bootloader to verify OEM firmware signatures.
4.  **Ephemeral Session Keys:** Symmetric keys (ChaCha20) derived for each individual communication session, ensuring forward secrecy.

This document details the secure provisioning process, which must occur in a trusted factory environment. It specifies how keys are stored on-device, leveraging the Secure Element (OPTIGA™ Trust M) to protect the critical Device Identity Key. Finally, it outlines the server-side processes for managing the corresponding public keys and the procedure for revoking a device's trust in the event of a suspected compromise.

---

## 2. Key Hierarchy and Purpose

A hierarchical structure ensures that the compromise of a lower-level key (like a session key) does not impact the security of higher-level, more critical keys.

```ascii
┌─────────────────────────────────────────────────────────────────┐
│ PARASITE KEY HIERARCHY                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ [ROOT] ┌───────────────────┐   ┌───────────────────┐             │
│        │ HARDWARE ROOT KEY │   │ OEM ROOT OF TRUST │(Offline HSM) │
│        │ (HRK) - PUF/eFuse │   └─────────┬─────────┘             │
│        └─────────┬─────────┘             │                       │
│                  │ KDF (HKDF)            │ Signing Keys          │
│                  │                       │                       │
│ [IDENTITY]       ▼                       ▼                       │
│        ┌───────────────────┐   ┌───────────────────┐             │
│        │ DEVICE ID KEY     │   │ FIRMWARE SIGNING  │             │
│        │ (DIK) - ECDSA     │   │ KEY (FSK) - ECDSA │             │
│        └─────────┬─────────┘   └───────────────────┘             │
│                  │                                               │
│                  │ ECDH Key Exchange                             │
│                  │                                               │
│ [SESSION]        ▼                                               │
│        ┌───────────────────┐                                     │
│        │  SESSION KEY      │ (Ephemeral, ChaCha20)               │
│        └───────────────────┘                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

| Key Name | Type | Algorithm | Storage Location | Purpose |
|----------|------|-----------|------------------|---------|
| **Hardware Root Key (HRK)** | Symmetric | N/A (Entropy) | PUF / Fused OTP | The unique, immutable secret of the chip. Never used directly. |
| **Device Identity Key (DIK)** | Asymmetric | ECDSA P-256 | Secure Element | The long-term identity of the device. Signs attestations and initiates mTLS. |
| **Firmware Signing Key (FSK)** | Asymmetric | ECDSA P-256 | **Private:** Offline HSM. **Public:** Baked into bootloader. | To sign and verify firmware updates. |
| **Session Key** | Symmetric | ChaCha20 | Device RAM only | To encrypt a single communication session with the Verifier. |

---

## 3. Key Generation and Provisioning

Keys must be injected into the device in a secure, trusted environment. This process is known as provisioning and is a one-time event in the factory.

### 3.1 Provisioning Environment
-   The provisioning must happen in a physically secure room.
-   The provisioning station is a PC connected to a Hardware Security Module (HSM) and a device programmer/debugger.
-   The provisioning station must be air-gapped or on a highly restricted network.

### 3.2 Provisioning Steps
1.  **Firmware Loading:** The initial, verified firmware (including the bootloader and application) is flashed onto the device via the debug interface. This firmware includes the public part of the Firmware Signing Key (FSK).
2.  **HRK to DIK Derivation:** The provisioning firmware on the device initiates a command to the Secure Element (SE).
    -   The SE reads the unique entropy from the MCU's PUF or other HRK source.
    -   The SE uses an internal Key Derivation Function (KDF), like HKDF, with the HRK as the input keying material.
    -   The SE generates a new, internal ECDSA P-256 key pair. This private key is the **Device Identity Key (DIK)**. **Crucially, the DIK private key is never exportable and never leaves the Secure Element.**
3.  **Certificate Signing Request (CSR):** The SE uses the newly generated DIK to create and sign a Certificate Signing Request (CSR). The CSR contains the public part of the DIK.
4.  **Certificate Issuance:** The provisioning firmware reads the CSR from the SE and sends it to the provisioning station. The station's HSM, acting as a private Certificate Authority (CA), verifies the CSR and issues a signed Device Identity Certificate. This certificate binds the device's public key to a unique serial number.
5.  **Certificate Injection:** The provisioning station sends the new Device Identity Certificate back to the device, where it is stored in non-volatile memory (either in the SE or the MCU's flash).
6.  **Fuse Blowing:** The final, irreversible step. The provisioning firmware makes a call to permanently disable all debug interfaces (JTAG/SWD) by blowing the corresponding eFuses on the MCU.

After this process, the device is a unique, identifiable, and secure entity.

### 3.3 Key Derivation in Rust (Conceptual)
This snippet illustrates how a key could be derived using a KDF. In our architecture, this logic would be implemented inside the Secure Element, but the principle is the same.

```rust
// This conceptual code demonstrates using HKDF to derive a stable key
// from a high-entropy, but potentially non-uniform, hardware secret.

use hkdf::Hkdf;
use sha2::Sha256;

/// Derives a 32-byte key suitable for use as a private key.
///
/// # Arguments
/// * `hardware_root_key` - The secret entropy read from the PUF/HRoT.
/// * `salt` - A fixed salt for the KDF process.
/// * `info` - Application-specific context (e.g., "PARASITE-DIK-v1").
///
/// # Returns
/// A 32-byte derived key.
pub fn derive_key_from_hrot(
    hardware_root_key: &[u8],
    salt: &[u8],
    info: &[u8]
) -> [u8; 32] {
    // HKDF-Extract: Create a strong pseudorandom key from the potentially non-uniform HRK.
    let (prk, hkdf) = Hkdf::extract(Some(salt), hardware_root_key);

    // HKDF-Expand: Generate the final output key of the desired length.
    let mut output_key = [0u8; 32];
    hkdf.expand(info, &mut output_key)
        .expect("Output key material length is valid");

    output_key
}
```

---

## 4. Key Storage and Protection

Where and how keys are stored is as important as their strength.

-   **Device Identity Key (Private):** Stored exclusively within the secure boundary of the OPTIGA™ Trust M Secure Element. It is marked as non-exportable. All signing operations are performed by sending data *to* the SE and receiving the signature back; the key itself is never exposed to the main MCU.
-   **Device Identity Certificate (Public):** Stored in the MCU's main flash memory. It is public information.
-   **Firmware Signing Key (Public):** Baked into the MCUBoot bootloader's binary image. It is read-only and protected by the secure boot process itself.
-   **Session Keys (Symmetric):** Generated on-the-fly for each communication session. They exist only in PARASITE's private RAM and are erased immediately after the session ends or on any device reset.

---

## 5. Key Usage Policy

Strict separation of duties is enforced.

-   The **DIK** is used *only* for:
    1.  Signing the Attestation Quote.
    2.  Authenticating the device during the mTLS handshake with the Verifier backend.
-   The **FSK** is used *only* by the bootloader to verify the signature on new firmware images.
-   **Session Keys** are used *only* to encrypt and decrypt the body of a single report being sent to the Verifier.

---

## 6. Key Rotation and Revocation

### 6.1 Key Rotation
-   **Session Keys:** Are rotated by definition, as a new one is generated for every session. This provides forward secrecy.
-   **Device Identity Key (DIK):** The DIK is a long-term identity and is not designed to be rotated in the field. Rotating it would be equivalent to creating a new device identity. The security of the DIK relies on its hardware protection.
-   **Firmware Signing Key (FSK):** The FSK can be rotated via a specific, highly secure firmware update. An update that contains a new FSK must be signed by the *old* FSK. Once the bootloader verifies and installs this update, it replaces the old public key with the new one. This is a high-risk operation reserved for responding to a catastrophic compromise of the OEM's offline HSM.

### 6.2 Key Revocation (Server-Side)
While a device's DIK cannot be changed, its *trustworthiness* can be revoked. The Verifier backend is responsible for this.
-   **Certificate Revocation List (CRL):** The Verifier backend will maintain a CRL. If a device is suspected of being compromised (e.g., it repeatedly fails attestation, or its physical hardware is reported stolen), its Device Identity Certificate serial number will be added to the CRL.
-   **Verification Check:** During every mTLS handshake, the Verifier will first check if the connecting device's certificate is on the CRL. If it is, the connection will be immediately rejected, effectively blocking the device from the PARASITE network. This prevents a compromised device from polluting the threat database or being used in further attacks.
