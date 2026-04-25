# Software Requirements Specification — AutoGate

**Project:** AutoGate — FreeRTOS Proximity-Activated Barrier Gate  
**Document ID:** AG-SRS-001  
**Version:** 1.0.0  
**Author:** \<your name\>  
**Date:** 2025-01  
**Status:** Draft

---

## 1. Introduction

### 1.1 Purpose

This document defines the functional and non-functional requirements for the AutoGate embedded system. It serves as the contractual baseline between design, implementation, and testing phases of the project.

### 1.2 Project scope

AutoGate is a proximity-activated barrier gate controller implemented on the STM32F407 microcontroller running FreeRTOS. The system detects approaching objects using an ultrasonic sensor, actuates a servo motor to open/close a gate, monitors ambient temperature, and displays live readings on a 16×2 LCD. All activity is logged over UART.

### 1.3 Definitions and acronyms

| Term | Definition |
|---|---|
| MCU | Microcontroller unit (STM32F407VGT6) |
| RTOS | Real-time operating system (FreeRTOS v10) |
| HAL | Hardware Abstraction Layer (STM32 HAL) |
| ISR | Interrupt service routine |
| PWM | Pulse-width modulation |
| SRS | Software requirements specification |
| REQ | Functional requirement identifier |
| NFR | Non-functional requirement identifier |

### 1.4 Document references

| Ref | Title |
|---|---|
| [1] | FreeRTOS Reference Manual v10.4 |
| [2] | STM32F407 Reference Manual (RM0090) |
| [3] | HC-SR04 Ultrasonic Sensor Datasheet |
| [4] | DHT11 Temperature/Humidity Sensor Datasheet |
| [5] | SG90 Servo Motor Datasheet |

---

## 2. Overall description

### 2.1 Product perspective

AutoGate is a standalone embedded system with no external network connectivity. All processing occurs on-device. The system is designed to demonstrate end-to-end embedded software development: requirements, architecture, HAL-decoupled drivers, FreeRTOS task design, unit testing, and CI/CD.

### 2.2 System context

```
[HC-SR04]──GPIO──┐
[DHT11]──GPIO────┤
                 │   STM32F407
                 ├── [FreeRTOS]──I²C──[LCD 16×2]
                 │              ──TIM2──[SG90 Servo]
[USB-TTL]──UART──┘              ──UART2──[Debug PC]
```

### 2.3 Assumptions and dependencies

- Power supply: 5V USB for MCU; separate 5V 2A rail for servo to avoid brownout
- HC-SR04 echo line is 5V tolerant via 1kΩ voltage divider to 3.3V GPIO
- DHT11 data line has a 4.7kΩ pull-up to 3.3V
- LCD uses PCF8574 I2C backpack module

---

## 3. Functional requirements

### 3.1 Proximity sensing

| ID | Requirement | Priority |
|---|---|---|
| REQ-001 | The system shall measure distance using the HC-SR04 ultrasonic sensor every 100 ms | Must |
| REQ-002 | Distance measurement range shall be 2 cm to 400 cm | Must |
| REQ-003 | Distance accuracy shall be within ±1 cm over the range 2–50 cm | Must |
| REQ-004 | If three consecutive readings fail, the system shall log an error and retain the last valid reading | Must |

### 3.2 Gate control

| ID | Requirement | Priority |
|---|---|---|
| REQ-005 | The system shall open the gate when a detected object is within a configurable proximity threshold (default: 20 cm) | Must |
| REQ-006 | The gate open position shall correspond to servo angle 90° | Must |
| REQ-007 | The gate closed position shall correspond to servo angle 0° | Must |
| REQ-008 | The system shall close the gate automatically 3 seconds after the detected object moves beyond the threshold | Must |
| REQ-009 | Gate open/close transitions shall complete within 500 ms | Must |
| REQ-010 | The proximity threshold shall be configurable at compile time via a `#define` in `app_config.h` | Should |

### 3.3 Temperature monitoring

| ID | Requirement | Priority |
|---|---|---|
| REQ-011 | The system shall read temperature and humidity from a DHT11 sensor every 2 seconds | Must |
| REQ-012 | Temperature range: −0°C to +50°C; humidity range: 20–80% RH | Must |
| REQ-013 | If the sensor read fails (checksum error or timeout), the display shall show `-- ERR` instead of stale data | Must |
| REQ-014 | Up to 3 read retries shall be attempted before declaring a failure | Should |

### 3.4 Display

| ID | Requirement | Priority |
|---|---|---|
| REQ-015 | The LCD shall display current distance on line 1 and temperature/humidity on line 2 | Must |
| REQ-016 | Line 1 format: `Dist: XXX cm  [OPEN/SHUT]` | Must |
| REQ-017 | Line 2 format: `T:XX.Xc H:XX%` | Must |
| REQ-018 | The display shall refresh every 500 ms | Must |
| REQ-019 | Access to the LCD shall be protected by a mutex to prevent concurrent write corruption | Must |

### 3.5 UART logging

| ID | Requirement | Priority |
|---|---|---|
| REQ-020 | The system shall log sensor readings, gate events, and errors over UART2 at 115200 baud | Must |
| REQ-021 | Each log line shall include a millisecond timestamp, task name, and message | Must |
| REQ-022 | Log format: `[XXXXXX ms][TaskName] message\r\n` | Must |

### 3.6 System reliability

| ID | Requirement | Priority |
|---|---|---|
| REQ-023 | The system shall use an independent watchdog (IWDG) with a 5-second timeout | Must |
| REQ-024 | Each task shall feed the watchdog on every cycle | Must |
| REQ-025 | On watchdog reset, the system shall log the cause over UART after restart | Should |

---

## 4. Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-001 | All driver logic (parsing, calculation, validation) shall be free of HAL dependencies to enable host-side unit testing |
| NFR-002 | Stack high watermark for each task shall be measured and documented; no task may use more than 80% of its allocated stack |
| NFR-003 | The CI pipeline shall build the firmware and run unit tests on every pull request without requiring physical hardware |
| NFR-004 | All public functions shall have Doxygen-compatible header comments |
| NFR-005 | No compiler warnings shall be permitted with flags `-Wall -Wextra -Werror` |
| NFR-006 | Code shall conform to MISRA-C:2012 advisory rules where feasible; deviations shall be documented |

---

## 5. Constraints

- MCU: STM32F407VGT6 (Cortex-M4, 168 MHz, 192 KB SRAM, 1 MB Flash)
- RTOS: FreeRTOS v10 via CMSIS-RTOS2 wrapper
- Build toolchain: arm-none-eabi-gcc ≥ 10.3
- IDE: STM32CubeIDE (for CubeMX pin configuration only; CLI build is authoritative)
- Language: C99

---

## 6. Acceptance criteria

The system is considered complete when:

1. All `Must` requirements pass their linked test cases in `traceability.md`
2. CI pipeline shows green on the `main` branch
3. A demo video shows gate opening within 1 second of hand approach and closing after hand withdrawal
4. Stack high watermark measurements are documented and within NFR-002 limits
