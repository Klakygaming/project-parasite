---
tags: #parasite #phase-05 #schematic #circuit-design #hardware
version: 1.1
status: In Progress
author: princetheprogrammer
---

# Phase 05: Circuit Design — Schematic & Power Analysis

## Document Metadata
| Field | Value |
|-------|-------|
| Phase | 05 |
| Title | Circuit Design (Schematic) |
| Version | 1.1 |
| Last Updated | 2025-12-16 |
| Author | princetheprogrammer |
| Dependencies | [[04_Bill_of_Materials]] |

---

## 1. Executive Summary

This document details the electronic circuit design for the PARASITE hardware prototype. It provides schematics, design rationale, and analysis for each functional block of the system, translating the component choices from the Phase 04 Bill of Materials into a complete and robust hardware design. The design prioritizes security, stability, and testability, ensuring a solid foundation for the firmware development and system integration phases.

The core of the design is the **STM32L562QE microcontroller**, integrated with the **Infineon OPTIGA™ Trust M Secure Element**. The schematic includes several key blocks:
-   **Power Supply:** A multi-stage power supply unit that accepts a wide input voltage range (5-12V) and provides clean, stable 3.3V power to all components, with extensive filtering and ESD protection.
-   **MCU Core:** The STM32L562QE circuit, including all necessary decoupling capacitors, boot mode configuration, and connections for programming and debug.
-   **Secure Element Interface:** A dedicated I2C bus connecting the MCU to the OPTIGA™ Trust M, with appropriate pull-up resistors and layout considerations for signal integrity.
-   **Connectivity:** An Ethernet interface using the LAN8720A PHY and a USB-to-serial bridge for console access, providing flexible options for telemetry and debugging.
-   **Debug & Provisioning Interface:** A secure debug interface that allows for initial programming via SWD but is designed to be permanently disabled in a production scenario.

A detailed power budget analysis estimates a peak power consumption of approximately 350mW during full operation (MCU + Ethernet active) and an ultra-low power consumption of under 1mW in standby mode. This document also outlines critical PCB layout considerations to ensure signal integrity, power stability, and RF compliance. The resulting design is a secure, reliable, and manufacturable hardware platform ready for prototyping.

---

## 2. Design Philosophy

-   **Defense-in-Depth:** The hardware design incorporates multiple layers of protection. This includes electrical protection (fuses, TVS diodes), signal integrity measures (controlled impedance, termination), and physical security considerations (secure debug access).
-   **Power Stability:** A clean and stable power supply is critical for the reliable operation of the MCU and cryptographic components. The design uses a combination of LDOs, ferrite beads, and a carefully planned decoupling capacitor strategy to minimize noise.
-   **Testability:** All key signals are exposed via accessible test points or headers on the prototype board. This is crucial for debugging and validating both hardware and firmware during development.
-   **Manufacturability (DFM):** Component footprints, placements, and trace routing are chosen to align with standard PCB manufacturing processes, ensuring a smooth transition from prototype to production.

---

## 3. Power Supply Unit (PSU)

The PSU is designed to be robust, accepting a common DC barrel jack input and providing clean 3.3V power.

### 3.1 PSU Schematic

```ascii
                                    +3V3_Digital
                                          │
                                          │
             ┌──────────┐   ┌───┐   ┌─────┴─────┐
VIN (5-12V)──┤ FUSE 1A  ├───┤D1 ├─┬─┤ LDO 3.3V  │
             └──────────┘   │TVS│ │ │ (AP2112K) │
                            └───┘ │ │           │
                                  │ │ VIN  VOUT ├─┬───► +3V3
                                  │ │           │ │
                                  │ │ GND  EN   │ │
                                  │ └─────┬─────┘ │
                                  │       │       │
                                  C1      C2      FB1 (Ferrite Bead)
                                10uF    0.1uF     │
                                  │       │       │
GND ──────────────────────────────┴───────┴───────┴─┬───► GND
                                                    │
                                                    │
                                              ┌─────┴─────┐
                                              │ LDO 3.3V  │
                                              │ (AP2112K) │
                                              │           │
                                              │ VIN  VOUT ├────► +3V3_Analog
                                              │           │
                                              │ GND  EN   │
                                              └─────┬─────┘
                                                    │
                                                    │
                                                   GND
```

### 3.2 PSU Design Rationale
-   **Input Protection:** A 1A resettable fuse (F1) protects against overcurrent conditions. A Transient Voltage Suppressor (TVS) diode (D1) clamps voltage spikes from the external power supply, protecting the downstream components.
-   **Voltage Regulation:** The AnalogicTech AP2112K-3.3 is a low-dropout (LDO) regulator chosen for its low noise, high Power Supply Rejection Ratio (PSRR), and small footprint.
-   **Power Plane Isolation:** The main 3.3V rail is split into digital and analog planes (`+3V3_Digital`, `+3V3_Analog`) using a ferrite bead (FB1). This isolates the sensitive analog components (like the MCU's ADC and PLL) from the high-frequency noise generated by the digital logic and Ethernet PHY. A separate LDO is used for the analog rail for maximum isolation on the prototype.

---

## 4. MCU Core Circuit

This block contains the STM32L562QE and its essential support components.

### 4.1 MCU Schematic

```ascii
                +3V3
                  │
     ┌────────────┴────────────┐
     │                         │ 
   C3-C10 (0.1uF) per VDD pin  │
     │                         │
     │   ┌────────────────┐    │
     ├─--┤VDD1..VDD8      │    │
     │   │                │    │
     │   │   STM32L562QE  │    │
     │   │                │    │
     │   │VSS1..VSS8      ├----┴--GND
     │   └────────────────┘
     │
     │   ┌────────────────┐
     ├---|BOOT0           |
     │   │                |
     R1  │           NRST ├---► To Reset Circuit
    10k  │                |
     │   └────────────────┘
     GND
```

### 4.2 MCU Design Rationale
-   **Decoupling:** A 0.1µF ceramic capacitor is placed as close as possible to *every* VDD pin on the MCU. This is a critical requirement for high-speed stability. Additionally, larger bulk capacitors (e.g., 10µF) will be placed near the MCU package to handle lower-frequency current demands.
-   **Boot Mode:** The `BOOT0` pin is pulled low via resistor R1. This ensures that the MCU boots from its main flash memory by default. A header will be provided to allow pulling `BOOT0` high to enter the system bootloader for initial programming or recovery.
-   **Reset Circuit:** The `NRST` (reset) pin is connected to an external circuit containing a push-button for manual reset and an RC network to ensure a clean power-on reset sequence.

---

## 5. Secure Element & Interfaces

### 5.1 I2C Interface Schematic

```ascii
+3V3
  │
  ├─────────┬──────────┐
  │         │          │
  R2 (2.2k) R3 (2.2k)  │
  │         │          │
  │         │  ┌───────┴────────┐
  │         │  │ OPTIGA Trust M │
  │         └──┤ SCL            │
  │            │                │
  └────────────┤ SDA            │
               │                │
               │            VCC ├─► +3V3
               │                │
               │            GND ├─► GND
               └────────────────┘


To MCU I2C Pins (e.g., PB6, PB7)
  ▲         ▲
  │         │
SCL         SDA
```

### 5.2 Interface Design Rationale
-   **Bus Pull-ups:** Resistors R2 and R3 are required pull-ups for the I2C open-drain bus. The 2.2kΩ value is chosen for a 400kHz "Fast-mode" I2C clock, providing a good balance between signal rise time and power consumption.
-   **Layout:** The I2C traces between the MCU and the Secure Element will be kept as short as possible (<5cm) and routed away from noisy sources like the Ethernet PHY or crystal oscillator to ensure reliable communication.

---

## 6. Debug & Provisioning Interface

This interface is designed for development and must be securable for production.

### 6.1 SWD Schematic

```ascii
+3V3
  │
  │
  R4 (10k)
  │
  ├───────────► SWDIO (To MCU PA13)
  │
  R5 (10k)
  │
  ├───────────► SWCLK (To MCU PA14)
  │
  │
  └─► To 10-pin ARM Debug Connector (J1)
```

### 6.2 Secure Debug Rationale
-   **Standard Connector:** A standard 10-pin ARM Cortex debug connector provides an interface for a J-Link or ST-Link debugger.
-   **Pull-up/Pull-down:** The pull-up/pull-down resistors ensure the SWD lines are in a known state when the debugger is disconnected.
-   **Production Disablement:** The STM32L5's Readout Protection (RDP) level will be set to Level 2 (`RDP=0xCC`) during final production provisioning. This is an **irreversible** action that permanently disables the debug port, preventing any physical access to the MCU's memory or CPU state, as required by our threat model (G-GUA-03).

---

## 7. Power Budget Analysis

This analysis estimates power consumption in different operational states.

| State | MCU Mode | Ethernet | Crypto | Consumption (mA) | Power (mW @ 3.3V) |
|---------------|----------|----------|--------|------------------|-------------------|
| **Standby** | STOP2 | Down | Off | ~0.25 mA | ~0.8 mW |
| **Idle** | RUN | Link Up | Off | ~25 mA | ~82.5 mW |
| **Scanning** | RUN | Link Up | Off | ~45 mA | ~148.5 mW |
| **Reporting** | RUN | Active TX| Active | ~105 mA | ~346.5 mW |

-   **Conclusion:** The peak power consumption is well within the capabilities of a standard USB port or a 500mA wall adapter. The ultra-low power standby mode makes the design suitable for battery-powered applications, although the Ethernet PHY would need to be fully powered down in such a scenario.

---

## 8. PCB Layout Considerations

The physical layout of the Printed Circuit Board (PCB) is as critical as the schematic design for ensuring security and stability.

-   **Layer Stackup:** A 4-layer PCB is recommended for the prototype to allow for a dedicated ground plane and a dedicated power plane. This significantly reduces EMI and improves signal integrity. For production, a cost-optimized 2-layer board is feasible if layout is done carefully.
-   **Decoupling Capacitor Placement:** All decoupling capacitors must be placed on the same side of the board as the MCU and as physically close to the corresponding VDD/VSS pins as possible.
-   **High-Speed Signals:** Traces for the Ethernet (RMII interface) and the external crystal oscillator must have their impedance controlled (typically 50Ω). They should be routed as short, direct paths with no vias if possible.
-   **Grounding:** A solid, unbroken ground plane is the single most important factor for a stable design. Any splits in the ground plane should be carefully considered and bridged at a single point to avoid creating ground loops.
-   **Security:** The traces connecting to the JTAG/SWD port should be located on inner layers of the PCB in the production version, making them harder to physically probe.
