# Requirements traceability matrix — AutoGate

**Document ID:** AG-TRACE-001  
**Version:** 0.1.0 (stub — populated in Phase 5)  
**Status:** In progress

---

## How to read this matrix

Each row links one requirement from `SRS.md` to the test(s) that verify it. A requirement is considered verified when all linked tests pass in CI.

| Status | Meaning |
|---|---|
| PENDING | Test not yet written |
| PASS | Test written and passing in CI |
| FAIL | Test written but failing |
| N/A | Verified by inspection or integration test, not unit test |

---

## Traceability table

| REQ ID | Requirement summary | Test ID | Test file | Status |
|---|---|---|---|---|
| REQ-001 | HC-SR04 sampled every 100 ms | TEST-001 | `test_hc_sr04.c` | PENDING |
| REQ-002 | Distance range 2–400 cm | TEST-002 | `test_hc_sr04.c` | PENDING |
| REQ-003 | Accuracy ±1 cm over 2–50 cm | TEST-003 | `test_hc_sr04.c` | PENDING |
| REQ-004 | 3 consecutive failures → error log | TEST-004 | `test_hc_sr04.c` | PENDING |
| REQ-005 | Gate opens at ≤20 cm | TEST-005 | `test_gate_ctrl.c` | PENDING |
| REQ-006 | Open = servo 90° | TEST-006 | `test_sg90.c` | PENDING |
| REQ-007 | Closed = servo 0° | TEST-007 | `test_sg90.c` | PENDING |
| REQ-008 | Auto-close after 3 s | TEST-008 | `test_gate_ctrl.c` | PENDING |
| REQ-009 | Transition ≤500 ms | N/A | Integration test on HW | N/A |
| REQ-010 | Threshold configurable at compile time | TEST-010 | `test_gate_ctrl.c` | PENDING |
| REQ-011 | DHT11 sampled every 2 s | TEST-011 | `test_dht11.c` | PENDING |
| REQ-012 | Temperature and humidity range | TEST-012 | `test_dht11.c` | PENDING |
| REQ-013 | Failed read → `-- ERR` on display | TEST-013 | `test_dht11.c` | PENDING |
| REQ-014 | 3 read retries | TEST-014 | `test_dht11.c` | PENDING |
| REQ-015 | LCD shows distance and temp/humidity | N/A | Visual inspection | N/A |
| REQ-016 | Line 1 format | TEST-016 | `test_lcd1602.c` | PENDING |
| REQ-017 | Line 2 format | TEST-017 | `test_lcd1602.c` | PENDING |
| REQ-018 | Display refreshes every 500 ms | N/A | Integration test on HW | N/A |
| REQ-019 | LCD protected by mutex | N/A | Code review | N/A |
| REQ-020 | UART logging at 115200 baud | N/A | Integration test on HW | N/A |
| REQ-021 | Log includes timestamp and task name | TEST-021 | `test_log_format.c` | PENDING |
| REQ-022 | Log format `[XXXXXX ms][Task] msg` | TEST-022 | `test_log_format.c` | PENDING |
| REQ-023 | IWDG 5-second timeout | N/A | Integration test on HW | N/A |
| REQ-024 | Each task feeds watchdog | N/A | Code review | N/A |
| REQ-025 | Log reset cause after restart | N/A | Integration test on HW | N/A |

---

## Non-functional requirement verification

| NFR ID | Requirement | Verification method | Status |
|---|---|---|---|
| NFR-001 | Drivers free of HAL dependencies | CI build with host GCC (no HAL headers) | PENDING |
| NFR-002 | Stack ≤80% used per task | `uxTaskGetStackHighWaterMark()` logged on startup | PENDING |
| NFR-003 | CI builds and tests without hardware | GitHub Actions green badge | PENDING |
| NFR-004 | Doxygen comments on all public functions | Doxygen build in CI (Phase 6) | PENDING |
| NFR-005 | No warnings with `-Wall -Wextra -Werror` | Compiler flags in Makefile | PENDING |
| NFR-006 | MISRA-C:2012 advisory compliance | cppcheck `--addon=misra` in CI | PENDING |

---

## Stack high watermark log (populated during Phase 4)

Record the minimum remaining stack (in words) for each task after 60 seconds of normal operation.

| Task | Allocated (words) | Min remaining (words) | Usage % |
|---|---|---|---|
| UltrasonicTask | 256 | TBD | TBD |
| TempTask | 256 | TBD | TBD |
| GateCtrlTask | 256 | TBD | TBD |
| DisplayTask | 384 | TBD | TBD |
| LogTask | 512 | TBD | TBD |
