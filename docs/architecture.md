# Smart Door ECU on OpenBSW — Architecture

## 1. Purpose

This document describes the architecture of the **Smart Door ECU Simulation implemented on OpenBSW**.

The goal of this project is to demonstrate how a typical ECU feature can be built not only as a standalone logic prototype, but also as an **application integrated into a platform-oriented embedded software stack**. OpenBSW provides the lifecycle management, async execution, CAN integration, and storage abstraction used by this PoC, while the Smart Door application adds the business logic, rule handling, persistence behavior, and timeout-based lamp control. [4](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/docan/doc/user/index.rst)[1](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/storage/src/storage)[5](https://github.com/eclipse-openbsw/openbsw/blob/main/.cmake-format)

---

## 2. System Context

This Smart Door ECU runs on the **OpenBSW POSIX platform** using:

- **SocketCAN (`vcan0`)** for virtual CAN communication
- **OpenBSW lifecycle manager** for startup / run / shutdown transitions
- **OpenBSW async scheduling** for timeout-driven behavior
- **OpenBSW storage abstraction** for lock-state persistence

The POSIX target is intended specifically for development without hardware, allowing CAN-based applications to run through SocketCAN while using the same application model as embedded targets. [3](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/io/examples/BufferedWriterExample.cpp)[6](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/doip/include/doip/server/DoIpServerTransportLayer.h)[7](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/3rdparty/lwip/contrib/ports/win32/include/arch/cc.h)

---

## 3. High-Level Architecture

```text
+--------------------------------------------------------------+
|                    OpenBSW Reference Application             |
|                                                              |
|  +----------------------+    +----------------------------+  |
|  | Lifecycle Manager    |    | Async Scheduling          |  |
|  | - init/run/shutdown  |    | - periodic ticking        |  |
|  +----------+-----------+    +-------------+-------------+  |
|             |                                |               |
|             v                                v               |
|      +--------------------------------------------------+    |
|      |                SmartDoorSystem                   |    |
|      |--------------------------------------------------|    |
|      | - lifecycle integration                          |    |
|      | - persistence restore/save                       |    |
|      | - timeout supervision                            |    |
|      | - command processing orchestration               |    |
|      +-------------------+------------------------------+    |
|                          |                                   |
|                          v                                   |
|      +--------------------------------------------------+    |
|      |              SmartDoorController                 |    |
|      |--------------------------------------------------|    |
|      | - LOCK / UNLOCK                                  |    |
|      | - DOOR_OPEN / DOOR_CLOSE                         |    |
|      | - fault rules                                    |    |
|      | - lamp timeout state                             |    |
|      +-------------------+------------------------------+    |
|                          |                                   |
|             +------------+------------+                      |
|             |                         |                      |
|             v                         v                      |
|   +----------------------+   +---------------------------+   |
|   | SmartDoorStatusBuilder|   | SmartDoorPersistenceAdapter| |
|   | - 4-byte status CAN   |   | - storage block access    | |
|   |   payload generation  |   | - load / save lock state  | |
|   +----------------------+   +---------------------------+   |
|                                                              |
|  +----------------------+                                    |
|  | CanDemoListener      |                                    |
|  | - receives CAN 0x101 |                                    |
|  | - decodes command    |                                    |
|  | - sends status 0x201 |                                    |
|  +----------------------+                                    |
+--------------------------------------------------------------+
````

This architecture aligns with OpenBSW’s reference-app structure, where **application systems** participate in lifecycle management and platform services such as CAN and storage are provided by the stack. The POSIX runtime separates platform main/startup from application-level systems, which is why the Smart Door logic is integrated as an application system rather than as a custom standalone loop. [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/storage/src/storage), [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/blob/main/executables/referenceApp/application/include/systems/TransportSystem.h), [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/3rdparty/etl/examples/ArmTimerCallbacks%20-%20C%2B%2B/RTE/Device/STM32F401RETx/startup_stm32f401xe.s)

***

## 4. Runtime Flow

### 4.1 Startup Flow

```text
Application start
    ->
OpenBSW lifecycle initialization
    ->
Platform services initialized
    ->
Storage system initialized
    ->
SmartDoorSystem init()
    ->
Restore persisted lock state
    ->
SmartDoorSystem run()
    ->
Periodic tick scheduling starts
```

OpenBSW lifecycle components are initialized and run according to **run levels**, and each component must complete its transitions properly so the lifecycle manager can continue startup. The Smart Door system is registered as an application lifecycle component, and it restores lock state during its initialization phase. [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/3rdparty/etl/examples/ArmTimerCallbacks%20-%20C%2B%2B/RTE/Device/STM32F401RETx/startup_stm32f401xe.s), [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/storage/src/storage), [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/middleware/interfaces/include/middleware/os/TaskIdProvider.h)

***

### 4.2 Command Processing Flow

```text
CAN frame received on vcan0
    ->
CanDemoListener filters command CAN ID (0x101)
    ->
Command byte decoded
    ->
SmartDoorSystem::processCommand()
    ->
SmartDoorController updates internal state
    ->
Optional lock-state persistence save
    ->
SmartDoorStatusBuilder creates 4-byte status payload
    ->
CanDemoListener sends status CAN frame (0x201)
```

This flow uses the existing application-side CAN listener hook rather than a manually written polling loop, which keeps the design aligned with the OpenBSW reference application structure and its SocketCAN-based POSIX target. [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/io/examples/BufferedWriterExample.cpp), [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/io/examples), [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/storage/src/storage)

***

### 4.3 Timeout Flow

```text
DOOR_CLOSE command
    ->
Door becomes CLOSED
    ->
Lamp remains ON
    ->
SmartDoorSystem periodic tick runs
    ->
SmartDoorController::tick(elapsedMs)
    ->
Lamp timeout expires
    ->
Lamp switched OFF
    ->
Status payload rebuilt
    ->
Updated status CAN frame sent automatically
```

The periodic timeout behavior is driven using OpenBSW’s async scheduling support, which provides fixed-rate execution and is the correct framework-native replacement for a manual infinite loop with delays. [\[eclipse-op....github.io\]](https://eclipse-openbsw.github.io/openbsw/sphinx_docs/doc/dev/index.html), [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/3rdparty/etl/examples/ArmTimerCallbacks%20-%20C%2B%2B/RTE/Device/STM32F401RETx/startup_stm32f401xe.s)

***

## 5. Main Components

## 5.1 SmartDoorSystem

**Role:** lifecycle-managed integration layer

### Responsibilities

* acts as the application-facing Smart Door system
* participates in OpenBSW init / run / shutdown transitions
* restores persisted lock state on startup
* saves lock state when it changes
* schedules periodic ticking for timeout behavior
* supervises timeout-driven status publication

### Why it exists

This component separates **platform/runtime integration** from **business logic**, which keeps the application maintainable and closer to real ECU software design.

***

## 5.2 SmartDoorController

**Role:** business logic / state machine

### Responsibilities

* handles Smart Door commands
* enforces lock rule when door is open
* tracks lock, door, lamp, and fault states
* supports duplicate command ignore behavior
* maintains lamp timeout state
* exposes `tick()` for time-based state progression

### Why it exists

This component keeps the core ECU logic independent of:

* CAN API
* lifecycle API
* storage API

That makes it easier to test, understand, and later port to another target.

***

## 5.3 SmartDoorStatusBuilder

**Role:** status payload generation

### Responsibilities

* convert controller state into CAN payload bytes

### Status Payload Format

* Byte 0 -> lock state
* Byte 1 -> door state
* Byte 2 -> lamp state
* Byte 3 -> fault state

### Why it exists

This prevents CAN-payload formatting logic from being mixed into controller or runtime code.

***

## 5.4 SmartDoorPersistenceAdapter

**Role:** persistence bridge

### Responsibilities

* load persisted lock state from OpenBSW storage
* save lock state when it changes
* hide `StorageJob` details from higher-level Smart Door logic

### Persistence Data

* storage block ID: `0xA02`
* value `0` -> UNLOCKED
* value `1` -> LOCKED

OpenBSW storage uses `IStorage::process(StorageJob&)` and models read/write access through explicitly initialized jobs, which is why the adapter exists as a small abstraction layer. [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/middleware/interfaces/include/middleware/os/TaskIdProvider.h), [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/lifecycle/src/lifecycle/AsyncLifecycleComponent.cpp), [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/doip/src/doip/common/DoIpCommonLogger.cpp)

***

## 5.5 CanDemoListener

**Role:** CAN-facing command and status bridge

### Responsibilities

* filters and receives Smart Door command frames
* decodes command byte
* invokes Smart Door processing
* sends status response frames
* also supports timeout-driven status transmission

### CAN Contract

* command CAN ID: `0x101`
* status CAN ID: `0x201`

***

## 6. Command and Status Contract

## 6.1 Commands

| Command      | Byte 0 |
| ------------ | ------ |
| LOCK         | `0x01` |
| UNLOCK       | `0x02` |
| DOOR\_OPEN   | `0x03` |
| DOOR\_CLOSE  | `0x04` |
| RESET\_FAULT | `0x05` |
| READ\_STATUS | `0x06` |

## 6.2 Status Frame Layout

| Byte | Meaning     |
| ---- | ----------- |
| 0    | Lock state  |
| 1    | Door state  |
| 2    | Lamp state  |
| 3    | Fault state |

***

## 7. Scratch vs OpenBSW Design

## Scratch Version

The scratch version implemented the ECU with:

* custom main loop
* custom CAN handling
* custom persistence/file path logic
* direct orchestration

## OpenBSW Version

The OpenBSW version keeps the **same Smart Door behavior** but moves it into:

* lifecycle-managed application system
* async-scheduled periodic behavior
* platform-provided CAN path
* storage abstraction layer

### Key difference

The scratch version focuses on **raw ECU behavior implementation**, while the OpenBSW version focuses on **how the same ECU behavior is integrated into an automotive software platform**. OpenBSW is specifically intended to provide a modular basic software stack for automotive microcontroller applications, including lifecycle, CAN, storage, and POSIX-based prototyping support. [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/3rdparty/lwip/src/api/api_msg.c), [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/docan/doc/user/index.rst), [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/storage/src/storage)

***

## 8. Porting Perspective

This architecture was designed so that the **Smart Door logic remains mostly reusable** across platforms.

### Reusable mostly as-is

* `SmartDoorController`
* `SmartDoorStatusBuilder`
* command contract
* state rules
* timeout logic

### Platform-specific adaptation needed later

* CAN driver binding
* storage backend binding
* task / async context mapping
* target board/platform configuration

That means the OpenBSW version is already a good stepping stone toward a later MCU target such as STM32, because the application logic is now separated from the platform service layer. OpenBSW supports multiple targets through platform abstraction, and the same application model is intended to run across POSIX and embedded targets with different underlying services. [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/doip/include/doip/server/DoIpServerTransportLayer.h), [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/3rdparty/lwip/src/api/api_msg.c), [\[github.com\]](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/storage/src/storage)

***

## 9. Current Limitations

* The reference application still contains **other demo systems** that may generate unrelated logs or demo traffic.
* This PoC focuses on **lock-state persistence only**.
* The implementation is **simulation-focused**, not production-certified.
* Ethernet / UDS demo subsystems may still initialize because they are part of the reference application build profile.

These do not affect the Smart Door PoC logic itself, but they are relevant when interpreting the full runtime log on the POSIX reference application. OpenBSW’s reference app includes multiple subsystems such as CAN, Ethernet, transport, and UDS, which are initialized according to enabled platform features.

***

## 10. Summary

This architecture demonstrates how a **feature-level ECU function** can be implemented on top of OpenBSW in a way that is:

* modular
* lifecycle-aware
* async-driven
* storage-integrated
* CAN-integrated
* portable in design

The final result is a Smart Door PoC that preserves the behavior of the scratch implementation while moving the software structure closer to a real embedded software platform model. OpenBSW’s purpose is exactly to provide such a code-first automotive software platform for prototyping and learning on POSIX as well as embedded targets.

