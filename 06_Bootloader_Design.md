---
tags: #parasite #phase-06 #bootloader #security #firmware
version: 1.1
status: In Progress
author: princetheprogrammer
---

# Phase 06: Bootloader Design — Secure Boot & Chain of Trust

## Document Metadata
| Field | Value |
|-------|-------|
| Phase | 06 |
| Title | Bootloader Design |
| Version | 1.1 |
| Last Updated | 2025-12-16 |
| Author | princetheprogrammer |
| Dependencies | [[03_System_Architecture]], [[02_Threat_Modeling]] |

---

## 1. Executive Summary

This document specifies the design of the secure bootloader for the PARASITE ecosystem. The bootloader is the first piece of mutable software to execute after the device powers on and is therefore the root of trust for the entire software stack. Its primary responsibility is to cryptographically verify the authenticity and integrity of the main application firmware (which includes the PARASITE agent) before execution is transferred. A failure in the bootloader's security invalidates all subsequent security measures.

Our design is based on the principles of a measured, immutable boot chain. We will leverage the open-source, community-vetted **MCUBoot** project as the foundation, customizing it to meet the specific security and architectural requirements of PARASITE. The key features of this design are:
-   **Chain of Trust:** A cryptographic chain starting from the immutable MCU Boot ROM, which verifies the MCUBoot binary, which in turn verifies the main application firmware.
-   **Signed Firmware Images:** All application firmware updates will be packaged into a secure image format, digitally signed using ECDSA (P-256) with a key held by the OEM.
-   **Anti-Rollback Protection:** The bootloader will enforce a strict, monotonic versioning scheme, preventing an attacker from downgrading the device to an older, vulnerable firmware version. This is a critical mitigation against a common attack vector.
-   **Dual-Bank Update Mechanism:** The design supports two separate memory banks for the application firmware. This allows for a new firmware image to be downloaded and verified in the secondary bank without interrupting the currently running application. If verification is successful, the bootloader swaps the active bank on the next boot, ensuring a fail-safe update process.
-   **Recovery Mode:** A minimal, fail-safe recovery mode is included, which can be triggered if both firmware banks are found to be invalid. This mode allows for device recovery via a secure physical interface.

This document details the boot sequence, firmware image format, cryptographic processes, and the integration of the bootloader with the PARASITE core. The resulting design provides a robust, secure foundation, directly mitigating the threats of firmware tampering and unauthorized modification identified in the Phase 02 threat model.

---

## 2. Boot Sequence & Chain of Trust

The boot process is a sequence of verification steps, where each stage cryptographically verifies the next before passing control.

### 2.1 Boot Flowchart

```ascii
┌───────────────────┐
│    POWER ON       │
└─────────┬─────────┘
          │
┌─────────▼─────────┐
│ MCU Boot ROM (HW) │ Checks signature of MCUBoot in flash
└─────────┬─────────┘
          │
          ├─[FAIL]──► Enters DFU/Serial Bootloader Mode
          │
┌─────────▼─────────┐
│ MCUBoot Execution │ (Privileged, Immutable)
└─────────┬─────────┘
          │
┌─────────▼─────────┐
│ Find Active Bank  │ Reads metadata to find primary slot
└─────────┬─────────┘
          │
┌─────────▼─────────┐
│ Verify Signature  │ ECDSA-P256 check on firmware image hash
└─────────┬─────────┘
          │
          ├─[FAIL]──► Invalidate Bank, Try Secondary Bank
          │
┌─────────▼─────────┐
│ Check Anti-Rollback │ Compare image version with security counter
└─────────┬─────────┘
          │
          ├─[FAIL]──► Invalidate Bank, Try Secondary Bank
          │
┌─────────▼─────────┐
│  Jump to App FW   │ Transfers execution to verified image
└─────────┬─────────┘
          │
┌─────────▼─────────┐
│ App FW + PARASITE │ (PARASITE starts in privileged mode)
└───────────────────┘
```

### 2.2 Chain of Trust Explained
1.  **Hardware Root of Trust:** The process begins in the MCU's immutable Boot ROM. On STM32L5 devices, this ROM can verify the signature of the code in the main flash memory. This ensures that the bootloader itself (MCUBoot) has not been tampered with.
2.  **MCUBoot (First-Stage Bootloader):** Once verified, MCUBoot executes. Its sole purpose is to verify and launch the main application. It is the software root of trust.
3.  **Application Firmware (includes PARASITE):** MCUBoot performs the signature and version checks on the application image. Only if these checks pass does it transfer control. This ensures the integrity and authenticity of the code that PARASITE will monitor.

---

## 3. Firmware Image Format

The application firmware is not a raw binary but is encapsulated in a structured format that includes metadata and cryptographic signatures.

### 3.1 Image Layout

```ascii
+----------------------------------+
| Image Header (struct image_header) |
+----------------------------------+
| TLV Block 1 (e.g., Image Hash)   |
+----------------------------------+
| TLV Block 2 (e.g., Version)      |
+----------------------------------+
| ...                              |
+----------------------------------+
| TLV Block N                      |
+----------------------------------+
| (Padding to align)               |
+----------------------------------+
|                                  |
|      Firmware Binary (.bin)      |
|                                  |
+----------------------------------+
| Image Trailer (struct image_trailer)|
+----------------------------------+
| TLV Block 1 (e.g., Signature)    |
+----------------------------------+
| ...                              |
+----------------------------------+
```

### 3.2 Header & TLV (Tag-Length-Value)
-   **Image Header:** A fixed-size structure at the start of the image containing a magic number, total image size, and offsets to other sections.
-   **TLVs:** A flexible way to add metadata. Key tags will include:
    -   `IMAGE_TLV_SHA256`: The SHA256 hash of the firmware binary. This is what is actually signed.
    -   `IMAGE_TLV_VERSION`: The semantic version number (e.g., 1.2.3).
    -   `IMAGE_TLV_SEC_COUNTER`: The security version number for anti-rollback.
    -   `IMAGE_TLV_SIGNATURE`: The ECDSA signature of the SHA256 hash.

A conceptual Rust representation of the header, which the bootloader will use to parse the image:
```rust
// This code would be part of the bootloader's image parsing logic.
// It demonstrates how the image header is defined and read.

#[repr(C)]
pub struct ImageHeader {
    magic: u32,
    // Total size of the image, including header and trailer.
    image_size: u32, 
    // Offset to the start of the TLV block in the header.
    tlv_offset: u16,
    // Number of TLVs in the header.
    num_tlvs: u16,
}

impl ImageHeader {
    pub const MAGIC: u32 = 0x96f3b83d;

    /// Reads the header from a specific memory address (e.g., start of a flash bank).
    ///
    /// # Safety
    /// This function is unsafe because it dereferences a raw pointer. The caller
    /// must ensure the address is valid and points to a correctly formatted header.
    pub unsafe fn from_address(address: usize) -> &'static Self {
        &*(address as *const Self)
    }

    /// Validates the magic number to quickly check if this is a valid image.
    pub fn is_valid(&self) -> bool {
        self.magic == Self::MAGIC
    }
}
```

---

## 4. Cryptographic Operations

### 4.1 Signature Verification
The core security function of the bootloader.
1.  **Parse Image:** The bootloader parses the image header and TLVs to find the firmware hash (`IMAGE_TLV_SHA256`) and the signature (`IMAGE_TLV_SIGNATURE`).
2.  **Calculate Hash:** The bootloader calculates its own SHA256 hash of the firmware binary portion of the image.
3.  **Compare Hashes:** It compares its calculated hash with the hash found in the TLV block. If they do not match, verification fails immediately. This prevents an attacker from signing one payload but packaging another.
4.  **Verify Signature:** If the hashes match, the bootloader uses a public key (baked into the bootloader binary itself) to verify the ECDSA-P256 signature against the hash.
5.  **Result:** If the signature is valid, the image is considered authentic.

### 4.2 Anti-Rollback Protection
This mechanism prevents an attacker from installing an older, signed, but vulnerable version of the firmware.
1.  **Security Counter:** The device will store a "security counter" in a write-once or monotonically increasing memory location (e.g., an eFuse, or a specific region of flash managed by the bootloader).
2.  **Version in Image:** Each firmware image contains a security version number (`IMAGE_TLV_SEC_COUNTER`). This number must be incremented for any update that contains a security fix.
3.  **Verification Step:** During boot, after verifying the signature, the bootloader compares the security counter from the image with the one stored on the device.
    -   If `image_counter >= device_counter`, the check passes. The bootloader then updates the `device_counter` to match the `image_counter`.
    -   If `image_counter < device_counter`, the check fails, and the bootloader rejects the image, even if the signature is valid.

---

## 5. Dual-Bank Update & Recovery

### 5.1 Update Process
The use of two memory banks for the application provides a robust, fail-safe update mechanism.
1.  **Normal Operation:** The device boots from the primary bank (Slot 0).
2.  **Download:** The application receives a new firmware update and writes it to the secondary, inactive bank (Slot 1).
3.  **Staging:** The application requests that MCUBoot "stage" the new image for the next boot. MCUBoot marks Slot 1 as pending verification.
4.  **Reboot:** The device reboots.
5.  **Verification:** MCUBoot detects the pending update in Slot 1. It performs a full verification (signature, anti-rollback) on the image in Slot 1.
6.  **Swap or Revert:**
    -   If verification succeeds, MCUBoot "swaps" the banks, making Slot 1 the new primary. It then proceeds to boot from Slot 1.
    -   If verification fails, MCUBoot discards the update, reverts the pending flag, and boots normally from the original Slot 0. The device remains fully functional on the old firmware.

### 5.2 Recovery Mode
In the catastrophic event that both Slot 0 and Slot 1 are found to be invalid (e.g., due to corruption or a failed update), the bootloader will enter a minimal recovery mode.
-   **Functionality:** The recovery mode will not run the main application. It will provide a single, secure interface (e.g., encrypted Y-modem over UART) to receive a new, valid firmware image.
-   **Security:** This mode is highly restricted and will only accept an image signed with a special "recovery key," which may be different from the standard OEM update key. This prevents abuse of the recovery mechanism.

---

## 6. Integration with PARASITE

The bootloader is a critical enabler for PARASITE's security model.
-   **MPU Configuration:** Before jumping to the application, the bootloader is responsible for setting up the initial MPU regions. It will configure the memory regions for the application code and PARASITE code with the correct permissions (e.g., making PARASITE's region privileged and the application's unprivileged), as defined in the System Architecture.
-   **Passing Control:** The bootloader jumps to the application's entry point. The application's startup code (`crt0`) is then responsible for initializing its own C runtime environment and then explicitly calling the PARASITE initialization function (`parasite_init()`) before entering its main loop. This ensures PARASITE is running before any significant application logic executes.
-   **Shared Data:** The bootloader can place critical information (e.g., the result of the boot verification, the current firmware version) at a well-known memory address for PARASITE to read upon startup. This information can be included in PARASITE's attestation reports.
