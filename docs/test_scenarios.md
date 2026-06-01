# Smart Door ECU on OpenBSW — Test Scenarios

## 1. Purpose

This document defines the **test scenarios used to validate the Smart Door ECU simulation** running on OpenBSW.

The goal is to verify:

- command → response behavior via CAN
- rule enforcement
- timeout behavior
- persistence behavior
- end-to-end system flow

---

## 2. Test Setup

### Prerequisites

- OpenBSW application built successfully
- `vcan0` interface available

### Enable virtual CAN (if not already)

```bash
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0
````

***

### Run application

```bash
./build/posix-freertos/executables/referenceApp/application/Release/app.referenceApp.elf
```

***

### Monitor CAN traffic

```bash
candump vcan0
```

***

### Send commands

```bash
cansend vcan0 <CAN_FRAME>
```

***

## 3. CAN Contract

### Command Frame

| Field     | Value   |
| --------- | ------- |
| CAN ID    | `0x101` |
| Byte 0    | Command |
| Bytes 1–7 | Ignored |

### Status Frame

| Field  | Value       |
| ------ | ----------- |
| CAN ID | `0x201`     |
| Byte 0 | Lock state  |
| Byte 1 | Door state  |
| Byte 2 | Lamp state  |
| Byte 3 | Fault state |

***

## 4. Test Scenarios

***

## ✅ Scenario 1 — LOCK

### Command

```bash
cansend vcan0 101#0100000000000000
```

### Expected CAN response

```
0x201 -> 01 00 00 00
```

### Expected behavior

* lock = LOCKED
* door = CLOSED
* lamp = OFF
* fault = NONE

***

## ✅ Scenario 2 — UNLOCK

### Command

```bash
cansend vcan0 101#0200000000000000
```

### Expected CAN response

```
0x201 -> 00 00 00 00
```

### Expected behavior

* lock = UNLOCKED
* no side effects

***

## ✅ Scenario 3 — DOOR\_OPEN

### Command

```bash
cansend vcan0 101#0300000000000000
```

### Expected CAN response

```
0x201 -> 01 01 01 00
```

### Expected behavior

* door = OPEN
* lamp = ON
* lock unchanged

***

## ✅ Scenario 4 — DOOR\_CLOSE (with timeout)

### Command

```bash
cansend vcan0 101#0400000000000000
```

### Immediate response

```
0x201 -> 01 00 01 00
```

### Expected behavior

* door = CLOSED
* lamp remains ON

### After \~3 seconds

Expected automatic response:

```
0x201 -> 01 00 00 00
```

### Explanation

* lamp timeout expires
* lamp is turned OFF
* status is pushed over CAN automatically

***

## ✅ Scenario 5 — LOCK when door is OPEN (fault case)

### Steps

```bash
cansend vcan0 101#0300000000000000   # DOOR_OPEN
cansend vcan0 101#0100000000000000   # LOCK
```

### Expected response

```
0x201 -> 01 01 01 01
```

### Expected behavior

* LOCK command rejected
* fault = LOCK\_DENIED\_DOOR\_OPEN
* state remains unchanged

***

## ✅ Scenario 6 — RESET\_FAULT

### Command

```bash
cansend vcan0 101#0500000000000000
```

### Expected response

```
0x201 -> 01 01 01 00
```

### Expected behavior

* fault cleared
* other states unchanged

***

## ✅ Scenario 7 — READ\_STATUS

### Command

```bash
cansend vcan0 101#0600000000000000
```

### Expected response

* current status (no state change)

Example:

```
0x201 -> 01 00 00 00
```

***

## ✅ Scenario 8 — Duplicate command handling

### Steps

```bash
cansend vcan0 101#0100000000000000   # LOCK
cansend vcan0 101#0100000000000000   # LOCK again
```

### Expected behavior

* second LOCK is ignored
* state remains unchanged

***

## ✅ Scenario 9 — Persistence (LOCK)

### Steps

```bash
cansend vcan0 101#0100000000000000
```

Then restart the app.

### Expected log

```
Restored persisted lock state: 1
```

***

## ✅ Scenario 10 — Persistence (UNLOCK)

### Steps

```bash
cansend vcan0 101#0200000000000000
```

Restart the app.

### Expected log

```
Restored persisted lock state: 0
```

***

## 5. Observability Checklist

During testing, verify:

### ✅ Application logs

* command processing messages
* status output logs
* persistence logs
* timeout logs

### ✅ CAN output

* responses on `0x201`
* timeout-triggered updates
* no missing frame after commands

***

## 6. Pass Criteria

The system is considered **correct** if:

* all commands generate expected CAN responses
* rule enforcement is correct
* timeout behavior works automatically
* persistence survives restart
* no unexpected crashes or assertion failures

***

## 7. Notes

* Demo logs (UDP/ETH/system) may still appear — they are unrelated
* Smart Door validation should focus on:
  * `[SMARTDOOR]` logs
  * `0x201` CAN frames

***

## 8. Summary

These test scenarios validate the Smart Door ECU across:

* command handling
* rule enforcement
* persistence
* timeout behavior
* platform integration

Together, they confirm that the Smart Door application is functioning correctly on top of the OpenBSW runtime.

````

---

# ✅ Where to place it

Save as:

```text
docs/test_scenarios.md
````

***