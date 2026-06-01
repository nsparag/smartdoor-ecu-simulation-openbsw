# 🚗 Smart Door ECU Simulation on OpenBSW

![Language](https://img.shields.io/badge/language-C++-blue)
![Platform](https://img.shields.io/badge/platform-OpenBSW-green)
![Use Case](https://img.shields.io/badge/usecase-Automotive%20ECU-orange)
![Status](https://img.shields.io/badge/status-Complete-brightgreen)

## 📌 Overview

This project demonstrates the implementation of a **Smart Door ECU** as an application running on top of the **OpenBSW platform**.

The system simulates real automotive ECU behavior by integrating:

* CAN-based command handling
* State-machine driven door/lock logic
* Persistent storage using platform services
* Time-based behavior (lamp timeout)
* Lifecycle-managed execution

***

## 🎯 What This Project Demonstrates

### ✅ Functional Capabilities

* Command-driven control via CAN (`0x101`)
* Real-time status response (`0x201`)
* Fault handling (lock denied if door is open)
* Persistent lock state across restart
* Lamp timeout (auto OFF after door close)
* Timeout-driven CAN status update

### ✅ Platform Concepts

* Lifecycle-managed execution
* Async scheduling instead of manual loops
* Storage abstraction via OpenBSW
* Clean separation between logic and platform

***

## 🏗️ Architecture Overview

👉 Full architecture details:  
📄 [Architecture](docs/architecture.md)

At a high level:

```text
CAN (vcan0)
   │
   ▼
CanDemoListener
   │
   ▼
SmartDoorSystem (lifecycle + integration)
   │
   ▼
SmartDoorController (core logic)
   │
 ┌───────┼────────┐
 ▼       ▼        ▼
Status  Timeout  Persistence
```

***

## 🔁 Scratch vs OpenBSW

| Aspect      | Scratch Version | OpenBSW Version     |
| ----------- | --------------- | ------------------- |
| Execution   | while-loop      | Lifecycle-managed   |
| CAN         | Custom handling | Platform CAN stack  |
| Timing      | Manual delay    | Async scheduler     |
| Persistence | File-based      | Storage abstraction |
| Structure   | Flat            | Layered             |

👉 **Key idea**

* Scratch → understand logic
* OpenBSW → understand system integration

***

## 📂 Project Structure

```text
executables/referenceApp/application/
│
├── include/
│   ├── smartdoor/
│   │   ├── SmartDoorController.h
│   │   ├── SmartDoorStatusBuilder.h
│   │   └── SmartDoorPersistenceAdapter.h
│   │
│   ├── systems/
│   │   └── SmartDoorSystem.h
│   │
│   └── app/
│       └── CanDemoListener.h
│
├── src/
│   ├── smartdoor/
│   │   ├── SmartDoorController.cpp
│   │   ├── SmartDoorStatusBuilder.cpp
│   │   └── SmartDoorPersistenceAdapter.cpp
│   │
│   ├── systems/
│   │   └── SmartDoorSystem.cpp
│   │
│   └── app/
│       └── CanDemoListener.cpp
│
├── CMakeLists.txt
└── app/
    └── app.cpp
```

***

## 🔍 Where to look for what

### 🔹 Core ECU Logic

```text
smartdoor/SmartDoorController.*
```

* state machine
* rules (lock restriction)
* timeout logic

***

### 🔹 CAN Interface

```text
app/CanDemoListener.*
```

* receives commands (`0x101`)
* sends responses (`0x201`)

***

### 🔹 System Integration

```text
systems/SmartDoorSystem.*
```

* lifecycle handling
* persistence integration
* periodic ticking

***

### 🔹 Status Format

```text
smartdoor/SmartDoorStatusBuilder.*
```

* builds CAN payload (4 bytes)

***

### 🔹 Persistence

```text
smartdoor/SmartDoorPersistenceAdapter.*
```

* load/store lock state via OpenBSW storage

***

### 🔹 Entry Point

```text
app/app.cpp
```

* application initialization
* lifecycle wiring

***

# 🚀 Setup from Scratch (Clone or ZIP)

This project demonstrates a **Smart Door ECU implementation integrated into OpenBSW**.

📌 Your Smart Door logic is implemented inside the OpenBSW application layer:

```

openbsw/executables/referenceApp/application/

````

Key Smart Door files:

- `smartdoor/SmartDoorController.*` → core ECU logic  
- `systems/SmartDoorSystem.*` → lifecycle & integration  
- `app/CanDemoListener.*` → CAN interface (0x101 / 0x201)  
- `smartdoor/SmartDoorPersistenceAdapter.*` → persistence  
- `smartdoor/SmartDoorStatusBuilder.*` → status payload  

---

## ✅ Option 1 — Recommended (Git Clone with Submodules)

```bash
git clone --recurse-submodules https://github.com/nsparag/smartdoor-ecu-openbsw.git
cd smartdoor-ecu-openbsw
````

***

## ✅ Option 2 — ZIP Download

```bash
unzip smartdoor-ecu-openbsw.zip
cd smartdoor-ecu-openbsw
git clone https://github.com/nsparag/openbsw.git openbsw
```

***

## ✅ Build and Run

### Step 1 — Go to OpenBSW

```bash
cd openbsw
```

***

### Step 2 — Build

```bash
cmake --preset posix-freertos
cmake --build --preset posix-freertos --parallel
```

***

### Step 3 — Run Smart Door ECU

```bash
./build/posix-freertos/executables/referenceApp/application/Release/app.referenceApp.elf
```

📌 This executable includes your **Smart Door System integrated into OpenBSW lifecycle**

***

## ✅ Setup Virtual CAN

```bash
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan 2>/dev/null || true
sudo ip link set up vcan0
```

***

## ✅ Test Smart Door Behavior

### Terminal 1

```bash
candump vcan0
```

***

### Terminal 2 — Send commands

```bash
cansend vcan0 101#0100000000000000   # LOCK
cansend vcan0 101#0200000000000000   # UNLOCK
cansend vcan0 101#0300000000000000   # DOOR_OPEN
cansend vcan0 101#0400000000000000   # DOOR_CLOSE
```

***

## ✅ Expected Smart Door Behavior

| Action         | Meaning      | Response (`0x201`) |
| -------------- | ------------ | ------------------ |
| LOCK           | Lock vehicle | `01 00 00 00`      |
| DOOR\_OPEN     | Door opened  | `01 01 01 00`      |
| DOOR\_CLOSE    | Door closed  | `01 00 01 00`      |
| Timeout (\~3s) | Lamp OFF     | `01 00 00 00`      |

***

## ✅ What is happening internally?

* CAN commands (`0x101`) → received by `CanDemoListener`
* Forwarded to → `SmartDoorSystem`
* Processed by → `SmartDoorController`
* Status built by → `SmartDoorStatusBuilder`
* Sent on CAN → `0x201`
* Lamp timeout handled by → async scheduler
* Lock state saved → OpenBSW storage

***

## ⚠️ Important Notes

* This is a **Smart Door application on top of OpenBSW**
* OpenBSW provides:
  * lifecycle management
  * CAN stack
  * storage
  * async execution
* Your contribution is the **Smart Door ECU logic integrated into it**

***

## ✅ Summary

```text
You are not just running OpenBSW.
You are running a Smart Door ECU built on top of OpenBSW.
```

***

## 🧪 Test Scenarios

👉 Full test coverage:  
📄 [Test Scenarios](docs/test_scenarios.md)

***

## ⚠️ Limitations

👉 Known limitations:  
📄 [Limitations](docs/limitations.md)

***

## 🚀 Future Extensions

* STM32 hardware port
* UDS diagnostics integration
* event logging
* configuration via external file
* AUTOSAR-style layering

***

## 🧠 Key Takeaway

> The same ECU logic can be **reused**, while only the **platform layer changes**.

This project demonstrates how to move from:

```
Scratch Logic → Platform-based ECU Design
```

***
