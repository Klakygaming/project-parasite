---
tags: #parasite #phase-07 #hardware #prototype #pcb #bring-up
version: 1.1
status: In Progress
author: princetheprogrammer
---

# Phase 07: Hardware Prototype — Fabrication, Assembly & Bring-up

## Document Metadata
| Field | Value |
|-------|-------|
| Phase | 07 |
| Title | Hardware Prototype |
| Version | 1.1 |
| Last Updated | 2025-12-16 |
| Author | princetheprogrammer |
| Dependencies | [[05_Circuit_Design]], [[04_Bill_of_Materials]] |

---

## 1. Executive Summary

This document outlines the comprehensive plan for the fabrication, assembly, and initial testing (bring-up) of the PARASITE hardware prototype. The successful execution of this phase is a critical milestone, as it provides the physical platform upon which all subsequent firmware development, integration, and testing will occur. A robust and reliable hardware prototype is essential for validating the project's core assumptions and demonstrating its end-to-end functionality.

The plan encompasses three main stages. First, **Fabrication**, where the Printed Circuit Board (PCB) will be manufactured according to the specifications derived from the Phase 05 circuit design. We have specified a 4-layer board to ensure signal integrity and provide excellent power distribution for this initial prototype. Second, **Assembly**, where the components specified in the Phase 04 BOM will be professionally soldered onto the fabricated PCBs. For the prototype run of 10 units, we will utilize a professional assembly service to ensure high quality and reliability.

The final and most critical stage is the **Bring-up Plan**. This is a systematic, step-by-step procedure designed to safely power on and validate the functionality of the assembled boards. The plan begins with bare-board checks and progresses through power supply verification, clock signal measurement, and establishing communication with the microcontroller and secure element. Each step is designed to verify a specific part of the circuit before proceeding to the next, minimizing the risk of damage and simplifying troubleshooting. This phase will conclude with the delivery of fully validated hardware prototypes to the firmware development team, ready for the bootloader and PARASITE core to be loaded and tested.

---

## 2. PCB Fabrication Specifications

The PCB is the physical foundation of the prototype. The following specifications will be provided to the selected fabrication house.

| Specification        | Value                                    | Rationale                                                                                                          |     |        |                                              |
| -------------------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | --- | ------ | -------------------------------------------- |
| **Board Material**   | FR-4                                     | Industry standard, good balance of cost and performance.                                                           |     |        |                                              |
| **Layer Count**      | 4                                        | Provides dedicated GND and VCC planes for superior signal integrity and EMI performance, critical for a prototype. |     |        |                                              |
| **Layer Stackup**    | Signal                                   | GND                                                                                                                | VCC | Signal | Standard stackup for good impedance control. |
| **Dimensions**       | 80mm x 60mm                              | Compact size suitable for an embedded device prototype.                                                            |     |        |                                              |
| **Copper Weight**    | 1 oz (35 µm)                             | Standard for most digital logic applications.                                                                      |     |        |                                              |
| **Min. Trace/Space** | 5 mil / 5 mil                            | Achievable by most fabrication houses, allows for dense routing.                                                   |     |        |                                              |
| **Surface Finish**   | ENIG (Electroless Nickel Immersion Gold) | Provides a flat surface, ideal for fine-pitch components (MCU), and is RoHS compliant.                             |     |        |                                              |
| **Solder Mask**      | Green                                    | Standard, provides good contrast for inspection.                                                                   |     |        |                                              |
| **Silkscreen**       | White                                    | For component designators, test point labels, and version info.                                                    |     |        |                                              |
| **Quantity**         | 12                                       | 10 for assembly, 2 as bare-board spares.                                                                           |     |        |                                              |

---

## 3. Assembly Plan

### 3.1 Sourcing & Kitting
-   **Component Sourcing:** All components from the Phase 04 BOM will be ordered from the authorized distributors (Arrow, Avnet, Mouser) and shipped to the selected assembly house.
-   **Kitting:** The components will be kitted according to the BOM and pick-and-place file generated by our EDA tool (KiCad). Each component type will be clearly labeled in its own bag or reel.

### 3.2 Assembly Process
-   **Assembly House:** A professional PCB assembly (PCBA) service in India will be contracted for the prototype run. This is chosen over in-house assembly to ensure high-quality solder joints, especially for the fine-pitch BGA/QFP packages of the MCU and Ethernet PHY.
-   **Process:** Standard SMT (Surface Mount Technology) reflow soldering process.
-   **Inspection:** After assembly, all boards will undergo Automated Optical Inspection (AOI) to check for soldering defects like shorts, opens, or incorrect component placement. A single board will undergo X-ray inspection to verify the solder joints on the BGA package of the MCU.

---

## 4. Hardware Bring-up Plan

This is the most critical part of the hardware phase. It is a sequential process. **Do not proceed to the next step until the current one is fully verified.** A lab notebook will be maintained to record all measurements and observations for each of the 10 boards.

**Required Equipment:**
-   Digital Multimeter (DMM)
-   Bench Power Supply with current limiting
-   Oscilloscope (at least 200 MHz bandwidth)
-   J-Link/ST-Link Debugger
-   Soldering Iron and basic rework tools

### Step 1: Bare-Board Check (On un-assembled PCB)
-   **Action:** Visually inspect the bare PCB for any obvious manufacturing defects.
-   **Action:** Use the DMM in continuity mode to check for shorts between the main power rails (VCC and GND).
-   **Success:** No shorts are detected.

### Step 2: Visual Inspection (On assembled PCB, unpowered)
-   **Action:** Visually inspect the assembled board for any obvious soldering errors, incorrect component orientations (especially for diodes and ICs), or missing components.
-   **Success:** Board appears to be correctly assembled as per the design.

### Step 3: Power Supply Verification (Board powered, MCU held in reset)
-   **Action:** Set the bench power supply to 5V with a low current limit (e.g., 50mA).
-   **Action:** Connect the power supply to the board's input.
-   **Observation:** Monitor the current draw. It should be very low (<20mA). A high current draw indicates a short circuit. If so, power off immediately and debug.
-   **Action:** If current is normal, use the DMM to measure the output of the 3.3V LDOs.
-   **Success:** The `+3V3_Digital` and `+3V3_Analog` rails measure between 3.25V and 3.35V.

### Step 4: Clock Signal Verification
-   **Action:** With the board still powered, use the oscilloscope to probe the output of the main crystal oscillator (typically 24MHz or similar).
-   **Success:** A clean, stable clock signal of the correct frequency and amplitude is observed. A faulty clock is a common reason for an MCU to fail to start.

### Step 5: "Hello, World" - Establishing Debug Communication
-   **Action:** Power down the board. Connect the J-Link/ST-Link debugger to the SWD header (J1).
-   **Action:** Power up the board.
-   **Action:** Use the debugger software (e.g., `probe-rs-cli` or J-Link Commander) to attempt to connect to the MCU.
-   **Success:** The debugger successfully halts the CPU and reads its device ID. **This is a major milestone.** It confirms the MCU is powered, clocked, and its debug interface is functional.

### Step 6: Basic Firmware Test
-   **Action:** Write a minimal "blinky" program (toggles an LED) in Rust. Compile it.
-   **Action:** Use the debugger to flash this binary to the MCU's RAM.
-   **Action:** Run the program.
-   **Success:** The onboard LED blinks at the expected rate. This verifies the entire toolchain (compiler, debugger, flash loader) is working and the MCU can execute code.

### Step 7: Peripheral Verification
-   **Action:** Write and flash a series of small test programs to verify each key peripheral.
    -   **Test 1 (UART):** Send a "Hello, PARASITE!" string out of the UART. Verify it is received on a connected PC.
    -   **Test 2 (I2C):** Attempt to communicate with the OPTIGA Trust M Secure Element. Read its device ID or public key.
    -   **Test 3 (Ethernet):** Initialize the LAN8720A PHY. Check for a link light when connected to a network switch. Attempt to get a DHCP lease.
-   **Success:** All peripherals are initialized and respond as expected.

At the conclusion of these steps, a board is declared "validated" and can be handed over to the firmware team.

---

## 5. Test Jig Design

For testing the 10 prototype units efficiently, a simple test jig will be created.
-   **Hardware:** A pogo-pin fixture that makes contact with test points on the bottom of the PARASITE board. This avoids needing to solder headers to every board.
-   **Connections:** The jig will provide:
    -   Power (5V)
    -   SWD connection for programming
    -   UART connection for logging
    -   Ethernet connection
-   **Software:** A Python script will be written to automate the bring-up tests (Steps 6 and 7). The script will:
    1.  Flash the test firmware.
    2.  Listen for the "Hello" message on the UART.
    3.  Check the log for a successful I2C communication with the SE.
    4.  Check the log for a successful Ethernet link.
    5.  Report "PASS" or "FAIL" for the board under test.

This automated jig will allow for rapid, consistent validation of all prototype units before they enter the main development and integration cycle.
