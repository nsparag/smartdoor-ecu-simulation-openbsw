# 🚗 Smart Door ECU Simulation on OpenBSW

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

## ⚙️ Build & Run

### ✅ Build

```bash
cmake --build --preset posix-freertos --parallel
```

***

### ✅ Run

```bash
./build/posix-freertos/executables/referenceApp/application/Release/app.referenceApp.elf
```

***

### ✅ Monitor CAN

```bash
candump vcan0
```

***

### ✅ Send Commands

```bash
cansend vcan0 101#0100000000000000   # LOCK
cansend vcan0 101#0200000000000000   # UNLOCK
cansend vcan0 101#0300000000000000   # DOOR_OPEN
cansend vcan0 101#0400000000000000   # DOOR_CLOSE
```

***

## 📡 Expected Behavior

| Action         | Status (`0x201`) |
| -------------- | ---------------- |
| LOCK           | `01 00 00 00`    |
| UNLOCK         | `00 00 00 00`    |
| DOOR\_OPEN     | `01 01 01 00`    |
| DOOR\_CLOSE    | `01 00 01 00`    |
| Timeout (\~3s) | `01 00 00 00`    |

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