# AutoGate — STM32F407 FreeRTOS Barrier Gate

![Build](https://github.com/<your-username>/autogate-stm32/actions/workflows/build.yml/badge.svg)
![Tests](https://github.com/<your-username>/autogate-stm32/actions/workflows/test.yml/badge.svg)

A proximity-activated barrier gate controller running FreeRTOS on an STM32F407. Built as a portfolio project demonstrating end-to-end embedded development: requirements, HAL-decoupled driver design, concurrent RTOS tasks, host-side unit testing, and CI/CD with GitHub Actions.

---

## What it does

An HC-SR04 ultrasonic sensor watches for approaching objects. When something gets within 20 cm, a FreeRTOS queue message triggers the servo to open the gate. A DHT11 sensor monitors ambient temperature and humidity. A 16×2 LCD shows live readings. All activity is logged over UART at 115200 baud.

```
Object detected within 20 cm → [dist_q] → GateCtrl → Servo opens
DHT11 reads every 2 s        → [temp_q] → Display  → LCD updates
All tasks                    → [log_q]  → LogTask  → UART
```

---

## Hardware

| Component | Interface | Notes |
|---|---|---|
| STM32F407VGT6 | — | 168 MHz Cortex-M4 |
| HC-SR04 | GPIO + TIM5 input capture | Voltage divider on ECHO (5V→3.3V) |
| DHT11 | GPIO bit-bang | 4.7kΩ pull-up on DATA |
| SG90 servo | TIM2 CH1 PWM, 50 Hz | Separate 5V 2A supply |
| LCD 16×2 + PCF8574 | I²C1 (PB6/PB7) | Address 0x27 |
| USB-TTL adapter | USART2 (PA2/PA3) | 115200 baud |

Full pin mapping: [`docs/architecture.md`](docs/architecture.md)

---

## Project structure

```
autogate-stm32/
├── docs/               # SRS, architecture, traceability matrix
├── Core/               # STM32CubeMX-generated HAL init
├── drivers/            # Peripheral drivers — HAL-free logic + HAL binding
│   ├── hc_sr04/
│   ├── dht11/
│   ├── sg90/
│   └── lcd1602/
├── app/                # FreeRTOS tasks and IPC primitives
│   ├── app_queues.c/h
│   ├── app_types.h
│   └── tasks/
├── test/               # Unity unit tests — run on host PC, no hardware needed
└── .github/workflows/  # CI: build, test, release
```

---

## Build

### Prerequisites

```bash
# Ubuntu / Debian
sudo apt install gcc-arm-none-eabi make

# macOS
brew install --cask gcc-arm-embedded
brew install make
```

### Firmware build

```bash
make all
# Output: build/autogate.bin, build/autogate.elf
```

### Flash to board

```bash
make flash
# Requires OpenOCD and ST-Link v2
```

### Unit tests (no hardware required)

```bash
make test
# Runs Unity tests with host GCC — same tests that run in CI
```

### Clean

```bash
make clean
```

---

## Architecture highlights

**Why the driver split matters:** Every peripheral has a `driver.c` (pure logic, no HAL) and `driver_hal.c` (HAL binding). This means all calculation and validation logic — pulse-to-cm conversion, DHT11 checksum, servo angle clamping, LCD string formatting — is unit-testable on a normal PC with no STM32 board attached.

**FreeRTOS task priorities:**

| Task | Priority | Why |
|---|---|---|
| GateCtrlTask | 4 (highest) | Deterministic response to proximity events |
| UltrasonicTask | 3 | Consistent 100 ms sensor sampling |
| TempTask | 2 | 2-second period; missing one is harmless |
| DisplayTask | 2 | 500 ms refresh; visual update can slip |
| LogTask | 1 (lowest) | Best-effort UART drain; yields to everything |

**IPC design:** `dist_q` and `temp_q` are depth-1 queues using `xQueueOverwrite`. Sensor data is always the current state — there's no value in buffering old readings. `lcd_mutex` prevents GateCtrlTask and DisplayTask from corrupting a partially-written LCD line.

Full design rationale: [`docs/architecture.md`](docs/architecture.md)

---

## Documentation

| Document | Description |
|---|---|
| [`docs/SRS.md`](docs/SRS.md) | Software requirements specification (25 functional, 6 non-functional) |
| [`docs/architecture.md`](docs/architecture.md) | Task design, pin mapping, memory budget, driver contract |
| [`docs/traceability.md`](docs/traceability.md) | REQ-xx → TEST-xx traceability matrix |
| [`docs/CHANGELOG.md`](docs/CHANGELOG.md) | Version history |

---

## Demo

> Video link — add after Phase 4 hardware integration

---

## License

MIT — see [LICENSE](LICENSE)
