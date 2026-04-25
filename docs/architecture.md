# Architecture — AutoGate

**Document ID:** AG-ARCH-001  
**Version:** 1.0.0  
**Status:** Draft

---

## 1. Layered architecture

The codebase is split into three strict layers. No layer may import from a layer above it.

```
┌─────────────────────────────────────────┐
│              app/tasks/                 │  FreeRTOS tasks — orchestration only
├─────────────────────────────────────────┤
│              drivers/                   │  Peripheral logic — HAL-free modules
│              drivers/*_hal.c            │  HAL binding — not unit-tested on PC
├─────────────────────────────────────────┤
│         STM32 HAL / FreeRTOS            │  Third-party, not modified
└─────────────────────────────────────────┘
```

The key rule: `drivers/hc_sr04.c` contains `pulse_to_cm()` with no HAL calls. `drivers/hc_sr04_hal.c` contains the GPIO/TIM code that actually talks to hardware. Unit tests import only the first file.

---

## 2. FreeRTOS task design

### 2.1 Task table

| Task | Function | Priority | Period | Stack (words) | Notes |
|---|---|---|---|---|---|
| UltrasonicTask | `Ultrasonic_Task()` | 3 | 100 ms | 256 | Highest sensor priority |
| TempTask | `Temp_Task()` | 2 | 2 000 ms | 256 | Float math, DHT11 retries |
| GateCtrlTask | `GateCtrl_Task()` | 4 | Event-driven | 256 | Blocks on `dist_q` |
| DisplayTask | `Display_Task()` | 2 | 500 ms | 384 | Holds `lcd_mutex` briefly |
| LogTask | `Log_Task()` | 1 | Event-driven | 512 | Blocks on `log_q`; large stack for sprintf |

Priority rationale: GateCtrl at P4 ensures deterministic response when an object is detected. Sensor tasks at P2–P3 ensure consistent sampling. LogTask at P1 yields to everything — acceptable because UART logging is best-effort.

### 2.2 IPC primitives

| Name | Type | Depth | Producer | Consumer | Notes |
|---|---|---|---|---|---|
| `dist_q` | Queue | 1 | UltrasonicTask | GateCtrlTask | `xQueueOverwrite` — always freshest value |
| `temp_q` | Queue | 1 | TempTask | DisplayTask | `xQueueOverwrite` — sensor data is stateless |
| `lcd_mutex` | Mutex | — | — | DisplayTask, GateCtrlTask | Protects I²C LCD from concurrent writes |
| `log_q` | Queue | 16 | All tasks | LogTask | Depth 16 to absorb bursts; LogTask drains to UART |

### 2.3 Task interaction diagram

```
UltrasonicTask ──dist_q──► GateCtrlTask ──PWM──► SG90 Servo
                                │
                           lcd_mutex
                                │
TempTask ────temp_q──────► DisplayTask ──I²C──► LCD 16×2

All tasks ────log_q──────► LogTask ──────UART──► Debug PC
```

---

## 3. Pin mapping — STM32F407

### 3.1 HC-SR04 (ultrasonic)

| Signal | STM32 Pin | Mode | Timer |
|---|---|---|---|
| TRIG | PA0 | GPIO Output | — |
| ECHO | PA1 | Alternate (TIM5 CH2) | TIM5, Input Capture |

Note: HC-SR04 ECHO is 5V; use 1kΩ/2kΩ voltage divider before PA1.

### 3.2 DHT11 (temperature/humidity)

| Signal | STM32 Pin | Mode |
|---|---|---|
| DATA | PB0 | GPIO Input/Output (bit-bang) |

Add 4.7kΩ pull-up to 3.3V on DATA line.

### 3.3 SG90 (servo)

| Signal | STM32 Pin | Mode | Timer |
|---|---|---|---|
| PWM | PA5 | Alternate (TIM2 CH1) | TIM2, PWM Out, 50 Hz |

Power servo from separate 5V rail to avoid brownout on the MCU 3.3V supply.

### 3.4 LCD 16×2 via PCF8574 I²C backpack

| Signal | STM32 Pin | Mode |
|---|---|---|
| SDA | PB7 | I²C1 SDA |
| SCL | PB6 | I²C1 SCL |

I²C address: `0x27` (default PCF8574). Add 4.7kΩ pull-ups to 3.3V on SDA and SCL.

### 3.5 UART2 (debug logging)

| Signal | STM32 Pin | Mode |
|---|---|---|
| TX | PA2 | USART2 TX |
| RX | PA3 | USART2 RX |

Baud rate: 115200, 8N1. Connect USB-TTL adapter for PC logging.

---

## 4. Memory budget

| Region | Size | Allocated | Remaining |
|---|---|---|---|
| Flash | 1024 KB | ~64 KB (estimate) | ~960 KB |
| SRAM | 192 KB | ~24 KB tasks + heap | ~168 KB |

FreeRTOS heap: `configTOTAL_HEAP_SIZE = 32768` (32 KB), using `heap_4.c`.

Task stack allocations (words × 4 bytes):

| Task | Words | Bytes |
|---|---|---|
| UltrasonicTask | 256 | 1024 |
| TempTask | 256 | 1024 |
| GateCtrlTask | 256 | 1024 |
| DisplayTask | 384 | 1536 |
| LogTask | 512 | 2048 |
| **Total** | **1664** | **6656** |

---

## 5. FreeRTOS configuration highlights (`FreeRTOSConfig.h`)

```c
#define configUSE_PREEMPTION              1
#define configCPU_CLOCK_HZ                168000000UL
#define configTICK_RATE_HZ                1000          // 1 ms tick
#define configMAX_PRIORITIES              8
#define configMINIMAL_STACK_SIZE          128
#define configTOTAL_HEAP_SIZE             32768
#define configUSE_MUTEXES                 1
#define configUSE_RECURSIVE_MUTEXES       0
#define configQUEUE_REGISTRY_SIZE         10
#define configUSE_TRACE_FACILITY          1             // enables runtime stats
#define configCHECK_FOR_STACK_OVERFLOW    2             // hook on overflow
#define configUSE_MALLOC_FAILED_HOOK      1
#define INCLUDE_uxTaskGetStackHighWaterMark 1           // for stack measurement
```

---

## 6. Driver design contract

Every peripheral has two files:

- `drivers/<name>.c` — Pure logic. No HAL includes. Fully unit-testable on host GCC.
- `drivers/<name>_hal.c` — HAL binding. Calls `drivers/<name>.c` functions and provides the hardware I/O. Not unit-tested (integration-tested on hardware).

Example for HC-SR04:

```c
// hc_sr04.h — pure interface
float HC_SR04_PulseToCm(uint32_t pulse_us);
bool  HC_SR04_IsValidRange(float cm);

// hc_sr04_hal.h — hardware interface
HAL_StatusTypeDef HC_SR04_Trigger(void);
uint32_t          HC_SR04_ReadEchoPulse_us(void);
```

This boundary is the reason all unit tests can run in GitHub Actions CI without a board attached.
