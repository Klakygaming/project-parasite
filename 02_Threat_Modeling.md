---
tags: #parasite #phase-02 #threat-modeling #stride #attack-surface
version: 1.1
status: In Progress
author: princetheprogrammer
---

# Phase 02: Threat Modeling — STRIDE Analysis & Attack Surface Mapping

## Document Metadata
| Field | Value |
|-------|-------|
| Phase | 02 |
| Title | Threat Modeling |
| Version | 1.1 |
| Last Updated | 2025-12-16 |
| Author | princetheprogrammer |
| Dependencies | [[01_Research_and_Validation]] |

---

## 1. Executive Summary

This document presents a systematic and comprehensive threat model for the PARASITE project. Following the research and validation conducted in Phase 01, this phase proactively identifies and analyzes potential security threats to the system's integrity, confidentiality, and availability. The primary goal is to inform the system architecture (Phase 03) and subsequent development phases with a concrete set of security requirements derived from a structured understanding of the risks.

Our methodology is rooted in the industry-standard STRIDE framework, which categorizes threats into Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, and Elevation of Privilege. Each component of the PARASITE ecosystem—from the on-device firmware agent to the backend verifier—is decomposed and analyzed through this lens. We utilize Data Flow Diagrams (DFDs) to map the flow of information and identify trust boundaries, which are critical for pinpointing potential weaknesses. Threats are then prioritized using the DREAD scoring model (Damage, Reproducibility, Exploitability, Affected Users, Discoverability) to focus mitigation efforts on the most significant risks.

The analysis reveals several high-priority threat areas. For the on-device agent, these include: **Tampering** with the PARASITE binary in flash, **Bypassing** the Sentinel's detection logic through low-entropy implants, and **Denial of Service** attacks targeting the Guardian's containment mechanism. For the backend, threats include **Spoofing** of device identities and **Information Disclosure** of aggregated threat intelligence.

For each identified threat, a specific mitigation strategy is proposed. These strategies are not abstract concepts but concrete architectural and implementation requirements. For example, the threat of tampering is mitigated by a secure boot chain and runtime memory protection. The threat of detection bypass is mitigated by a multi-layered detection strategy that combines entropy analysis with other heuristics. This document concludes with a prioritized list of security requirements that will be formally tracked and implemented throughout the project lifecycle, ensuring that PARASITE is not only functional but secure by design.

---

## 2. Threat Modeling Methodology

### 2.1 STRIDE Framework Overview
STRIDE provides a structured mnemonic for brainstorming threats. We apply it to every component and data flow within the PARASITE system.

```ascii
┌─────────────────────────────────────────────────────────────────┐
│ STRIDE CATEGORIES APPLIED TO PARASITE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ [S]POOFING         - An attacker fakes a device's identity to send false reports.
│ [T]AMPERING        - An attacker modifies PARASITE's code on-device to disable it.
│ [R]EPUDIATION      - A compromised device's owner denies that a valid alert originated from it.
│ [I]NFORMATION DISC - An attacker intercepts telemetry to map national threat landscape.
│ [D]ENIAL OF SERVICE - An attacker triggers many false alerts to drain device battery.
│ [E]LEVATION OF PRIV - An unprivileged process on-device gains control over the Guardian.
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Attack Tree Methodology
For high-impact threats, we use attack trees to decompose an attacker's goal into specific, achievable steps. This helps in identifying the most effective points for mitigation. The root of the tree is the adversary's goal, and the leaves are the concrete actions they might take.

### 2.3 DREAD Risk Scoring
To prioritize threats, we use the DREAD model. Each threat is scored from 1 (low) to 5 (high) across five categories. The sum provides a risk rating.

- **D**amage: How severe is the impact if the attack succeeds?
- **R**eproducibility: How easily can the attack be repeated?
- **E**xploitability: How much effort, skill, and resources are needed?
- **A**ffected Users: How many devices or users would be impacted?
- **D**iscoverability: How likely is it that this vulnerability will be found?

A score of **20-25** is Critical, **15-19** is High, **10-14** is Medium, and **<10** is Low.

---

## 3. System Decomposition

### 3.1 Data Flow Diagram (DFD) - Level 0
This DFD shows the entire PARASITE ecosystem and its interaction with external entities.

```ascii
┌─────────────────────────────────────────────────────────────────┐
│ PARASITE DFD - LEVEL 0                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ ┌─────────┐       Malicious Implant       ┌─────────────────┐   │
│ │ ATTACKER│ ───────────────────────────────►│ PROTECTED DEVICE  │   │
│ │ (Threat │                                 │(Running PARASITE) │   │
│ │ Actor)  │◄─────────────────────────────── │(e.g., Smart Meter)│   │
│ └─────────┘       Side-Channel Leakage      └────────┬────────┘   │
│                                                      │            │
│                                     Anonymized IOCs  │            │
│                                     (TLS 1.3)        │            │
│                                                      ▼            │
│ ┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐   │
│ │  DEVICE OWNER   │◄──────┤  PARASITE CLOUD │◄──────┤ CERT-In / NTRO  │   │
│ │ (Accesses Dash) │       │    (VERIFIER)   │       │ (Auth. Agency)  │   │
│ └─────────────────┘       └─────────────────┘       └─────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Trust Boundaries
Trust boundaries are the borders between trusted and untrusted components. Crossing these boundaries is where threats often materialize.

```ascii
┌─────────────────────────────────────────────────────────────────┐
│ TRUST BOUNDARIES                                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EXTERNAL UNTRUSTED NETWORK                                     │
│ ═══════════════════════════════════════════════════════════════ │ TB4
│      │                                                          │
│      │ (TLS 1.3 Termination Point)                              │
│      ▼                                                          │
│  PARASITE CLOUD BACKEND (Trusted)                               │
│ ═══════════════════════════════════════════════════════════════ │
│      │                                                          │
│      │ (Physical Device Perimeter)                              │
│      ▼                                                          │
│  PROTECTED DEVICE                                               │
│  ├───────────────────────────────────────────────────────────┐  │
│  │ APPLICATION FIRMWARE (Untrusted by PARASITE Core)         │  │
│  │ ═════════════════════════════════════════════════════════ │ TB3
│  │    │                                                      │  │
│  │    │ (MPU/PMP Boundary)                                   │  │
│  │    ▼                                                      │  │
│  │ PARASITE CORE (Trusted within Device)                     │  │
│  │ ═════════════════════════════════════════════════════════ │ TB2
│  │    │                                                      │  │
│  │    │ (Bootloader Verification)                            │  │
│  │    ▼                                                      │  │
│  │ SECURE BOOTLOADER (Root of Trust for Software)            │  │
│  └───────────────────────────────────────────────────────────┘  │
│ ═══════════════════════════════════════════════════════════════ │ TB1
│  HARDWARE (Root of Trust)                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. STRIDE Analysis by Component

### 4.1 SENTINEL (Detection Engine) Threats

| Threat ID | Category | Description | DREAD (D+R+E+A+D) | Mitigation Strategy |
|-----------|----------|-------------|-------------------|---------------------|
| **S-SEN-01** | **T**ampering | An attacker with code execution modifies Sentinel's entropy thresholds in SRAM to disable detection. | 18 (4+4+3+5+2) | **Architectural:** Store critical configuration in read-only memory (flash). **Implementation:** The `sentinel_init` function must configure the MPU to make its own configuration struct read-only after initialization. |
| **S-SEN-02** | **D**enial of Service | An attacker crafts legitimate application data (e.g., a large compressed file) that triggers a false positive, causing the Guardian to quarantine valid code. | 16 (3+4+3+4+2) | **Architectural:** Implement a two-stage detection. An initial entropy alert is correlated with other heuristics (e.g., is the memory region marked as executable?). Implement a baselining mode on first boot to profile the legitimate firmware. |
| **S-SEN-03** | **I**nformation Disclosure | An attacker uses a timing side-channel to infer Sentinel's scanning progress and predict when it is safe to execute a short-burst malicious payload. | 14 (3+2+4+3+2) | **Implementation:** Introduce random jitter into the scan interval. The scanning operation itself should be designed to be as close to constant-time as possible. |
| **S-SEN-04** | **T**ampering | An attacker develops a "low-entropy" implant, using techniques like code-shuffling or embedding shellcode within legitimate-looking string data to evade the Chi-squared test. | 19 (5+3+4+5+2) | **Architectural:** Do not rely on entropy alone. Sentinel's detection score must be a weighted average of multiple heuristics: 1. Entropy analysis. 2. Check for executable code in data segments. 3. Monitor for unexpected MPU fault escalations. |

### 4.2 GUARDIAN (Containment Engine) Threats

| Threat ID | Category | Description | DREAD (D+R+E+A+D) | Mitigation Strategy |
|-----------|----------|-------------|-------------------|---------------------|
| **G-GUA-01** | **E**levation of Privilege | A compromised but un-contained process exploits a bug in the PARASITE API to call `guardian_release()` and un-quarantine a malicious code block. | 22 (5+4+5+5+3) | **Architectural:** The PARASITE API must be strictly one-way. The application firmware should have no ability to call functions that would reduce the security state. **Implementation:** The API exposed to the application must not include any privileged functions. Use a hardware fault exception (e.g., `HardFault`) as the entry point into the Guardian's privileged context. |
| **G-GUA-02** | **D**enial of Service | An attacker triggers a large number of alerts, causing the Guardian to exhaust its available MPU regions and fail to contain a subsequent, truly malicious payload. | 17 (4+4+3+4+2) | **Architectural:** Design a "containment garbage collector." If all MPU regions are used, the Guardian should either escalate to a full device lockdown or attempt to merge adjacent quarantined regions to free up an MPU slot. |
| **G-GUA-03** | **T**ampering | An attacker with physical access uses a debugger (JTAG/SWD) to halt the CPU and manually reverse the MPU changes made by the Guardian. | 20 (5+5+2+5+3) | **Architectural:** The bootloader must permanently disable debug interfaces (e.g., by blowing an eFuse) as part of the final device provisioning process. PARASITE's attestation report must include the state of the debug interface fuses. |

### 4.3 REPORTER (Telemetry Engine) Threats

| Threat ID | Category | Description | DREAD (D+R+E+A+D) | Mitigation Strategy |
|-----------|----------|-------------|-------------------|---------------------|
| **R-REP-01** | **S**poofing | An attacker without a valid device identity spoofs reports to the backend, polluting the threat intelligence database (a "reporting DDoS"). | 18 (4+4+3+5+2) | **Architectural:** Implement a strong cryptographic device identity using a hardware-backed key (e.g., in a Secure Element or from a PUF). The backend must perform mutual TLS authentication (mTLS), verifying the client certificate against a provisioned list. |
| **R-REP-02** | **I**nformation Disclosure | An attacker performs a man-in-the-middle (MITM) attack on the telemetry channel to read the IOCs being reported from a specific device or region. | 19 (4+3+4+5+3) | **Implementation:** Strictly enforce TLS 1.3 with forward secrecy cipher suites (e.g., `TLS_AES_256_GCM_SHA384`). The device must pin the backend's public key or root CA to prevent spoofing of the server identity. |
| **R-REP-03** | **R**epudiation | An attacker claims a valid report from their device was a forgery, challenging the integrity of the national threat database. | 15 (3+2+3+4+3) | **Architectural:** Each report must be digitally signed by the device's unique private key. The backend must store this signature alongside the report as non-repudiable evidence. |

---

## 5. Attack Surface Analysis

### 5.1 Physical Attack Surface
This covers all threats requiring physical interaction with the device.
- **JTAG/SWD Debug Ports:** The most critical physical vector. Provides direct memory and CPU access. **Mitigation:** Permanently disable via eFuses during production.
- **UART/Console Access:** Can expose boot logs and provide shell access. **Mitigation:** Disable shell access in production firmware; use only for logging.
- **Flash Memory Probing:** Direct access to the flash chip to read/write firmware. **Mitigation:** Use MCUs with on-chip encrypted flash. If not available, use an epoxy blob to physically protect chips.
- **Side-Channel Analysis (SCA):** Power analysis or EM analysis to leak cryptographic keys. **Mitigation:** Use constant-time crypto implementations. Use hardware crypto accelerators with built-in SCA countermeasures.

### 5.2 Firmware Attack Surface
- **PARASITE API:** The API exposed to the main application. **Mitigation:** Minimize the API surface. Make it one-way (app can't de-escalate security). Use strong input validation for any data passed to PARASITE.
- **Secure Bootloader:** Vulnerabilities in the bootloader can compromise the entire chain of trust. **Mitigation:** Use a community-vetted bootloader (MCUBoot).
- **Cryptographic Primitives:** Weak or poorly implemented crypto. **Mitigation:** Use well-vetted, industry-standard libraries (e.g., the `p256` and `chacha20poly1305` Rust crates).

A conceptual Rust snippet for input validation on the PARASITE API:
```rust
/// A potential, limited API call that the main application could make.
/// Notice the strict validation of the input length.
pub enum AppRequest {
    /// Request PARASITE to scan an additional, dynamically loaded memory region.
    RequestScan { address: u32, length: u32 },
}

/// Processes a request from the untrusted application.
/// This function would run in PARASITE's privileged context.
pub fn handle_app_request(request: AppRequest) -> Result<(), &'static str> {
    match request {
        AppRequest::RequestScan { address, length } => {
            // CRITICAL: Validate that the requested region does not overlap with
            // PARASITE's own memory or critical peripheral regions.
            const MAX_SCAN_LENGTH: u32 = 4096;
            if length > MAX_SCAN_LENGTH {
                return Err("Scan length exceeds maximum allowed limit.");
            }
            if is_protected_region(address, length) {
                return Err("Scan request overlaps with protected memory.");
            }
            // If validation passes, queue the scan.
            sentinel::queue_dynamic_scan(address, length);
            Ok(())
        }
    }
}

fn is_protected_region(address: u32, length: u32) -> bool {
    // Implementation would check against PARASITE's own memory map.
    true // Placeholder
}
```

### 5.3 Network Attack Surface
- **Telemetry Endpoint (Cloud):** The public-facing endpoint for receiving reports. **Mitigation:** Standard cloud security best practices: DDoS protection, WAF, strict firewall rules.
- **Device Network Stack:** Vulnerabilities in the device's own TCP/IP stack. **Mitigation:** PARASITE relies on the host firmware's network stack but protects itself by using its own TLS implementation within its protected memory region, preventing the host from tampering with its communication.

---

## 6. Attack Trees

### 6.1 Root Attack Tree: Disable PARASITE
This tree illustrates the primary goal of an attacker aiming to neutralize PARASITE.

```ascii
                      ┌───────────────────────────┐
                      │ GOAL: Disable PARASITE    │
                      │ on a Protected Device     │
                      └────────────┬──────────────┘
                                   │ [OR]
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
┌──────────▼──────────┐ ┌──────────▼──────────┐ ┌──────────▼──────────┐
│ Bypass Detection    │ │ Disable Containment │ │ Subvert Reporting   │
│ (Sentinel)          │ │ (Guardian)          │ │ (Reporter)          │
└──────────┬──────────┘ └──────────┬──────────┘ └──────────┬──────────┘
           │ [OR]                  │ [OR]                  │ [OR]
    ┌──────┴──────┐         ┌──────┴──────┐         ┌──────┴──────┐
    │             │         │             │         │             │
┌───▼───┐     ┌───▼───┐ ┌───▼───┐     ┌───▼───┐ ┌───▼───┐     ┌───▼───┐
│Low-   │     │Timing │ │Exploit│     │Disable│ │Block  │     │Spoof  │
│Entropy│     │Attack │ │MPU    │     │Debug  │ │Network│     │Device │
│Implant│     │       │ │Fault  │     │Port   │ │Traffic│     │ID     │
└───────┘     └───────┘ └───────┘     └───────┘ └───────┘     └───────┘
```
**Mitigation Focus:** The tree shows that a robust defense requires multiple, independent layers. A failure in detection can be caught by containment. A failure in both can still be caught by attestation and reporting. Our "Defense in Depth" principle is a direct counter to this attack tree.

---

## 7. Security Requirements Derivation
This threat model leads to the following high-priority security requirements, which must be implemented and verified.

| Req. ID | Requirement Description | STRIDE Category | Source Threat(s) |
|---------|-------------------------|-----------------|------------------|
| **SR-01** | The secure bootloader must verify the integrity of the PARASITE binary before execution. | Tampering | G-GUA-03, S-SEN-01 |
| **SR-02** | Debug interfaces (JTAG, SWD) must be permanently disabled in production hardware. | Tampering, EOP | G-GUA-03 |
| **SR-03** | PARASITE's own code and configuration data must be located in a read-only memory region, enforced by the MPU. | Tampering | S-SEN-01 |
| **SR-04** | The application firmware must execute in an unprivileged state and be prevented by the MPU from accessing PARASITE's memory. | Elevation of Privilege | G-GUA-01 |
| **SR-05** | The Sentinel detection algorithm must use a combination of at least two heuristics (e.g., entropy + memory permissions). | Tampering | S-SEN-04 |
| **SR-06** | All communication with the backend verifier must use TLS 1.3 with mutual, certificate-based authentication. | Spoofing, Info. Disc. | R-REP-01, R-REP-02 |
| **SR-07** | Each device must possess a unique, hardware-backed private key for signing telemetry reports. | Repudiation | R-REP-03 |
| **SR-08** | The PARASITE API exposed to the application must not contain any functions that can disable or reduce security. | Elevation of Privilege | G-GUA-01 |
| **SR-09** | The system must be resilient to high-frequency, low-quality alerts (containment exhaustion / reporting DDoS). | Denial of Service | G-GUA-02, R-REP-01 |
| **SR-10** | All cryptographic operations must be implemented using vetted, constant-time libraries. | Information Disclosure | S-SEN-03 |
