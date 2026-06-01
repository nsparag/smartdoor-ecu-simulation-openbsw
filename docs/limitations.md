# Smart Door ECU on OpenBSW — Limitations

## 1. Purpose of This Document

This document lists the main limitations of the **Smart Door ECU Simulation implemented on OpenBSW**.

The objective of this PoC is to demonstrate **application behavior, platform integration, persistence, CAN interaction, and timeout-driven logic** on top of the OpenBSW POSIX environment. It is not intended to represent a production-ready automotive ECU. OpenBSW itself is positioned as an open basic software stack for automotive microcontroller applications and explicitly provides a POSIX environment for development and learning without real automotive hardware. [1](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/3rdparty/lwip/src/api/api_msg.c)[2](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/io/examples/BufferedWriterExample.cpp)[3](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/storage/src/storage)

---

## 2. Simulation-Only Scope

This project runs on the **OpenBSW POSIX platform** using:

- Linux user-space execution
- virtual CAN through `SocketCAN`
- file-backed persistent storage
- a software-simulated runtime and task model

Therefore, the current implementation should be understood as a **simulation and learning PoC**, not as a hardware-validated ECU. OpenBSW’s POSIX platform is specifically intended to allow development without automotive-specific hardware, including CAN access through SocketCAN where supported. [2](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/io/examples/BufferedWriterExample.cpp)[4](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/doip/include/doip/server/DoIpServerTransportLayer.h)[5](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/3rdparty/lwip/contrib/ports/win32/include/arch/cc.h)

### Practical implication
The project demonstrates **design intent and software structure**, but not final embedded-hardware behavior.

---

## 3. No Real Hardware Validation Yet

The current Smart Door PoC has **not been validated on a physical microcontroller target**.

### Not yet covered
- real MCU startup and reset behavior
- real CAN peripheral driver timing
- real EEPROM / flash write constraints
- real GPIO door/lamp actuator behavior
- board-level power/reset interactions

Although OpenBSW supports embedded targets in addition to POSIX, this project currently uses the POSIX target only. As a result, hardware-level validation remains out of scope for the present version. [4](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/doip/include/doip/server/DoIpServerTransportLayer.h)[6](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/storage/include/storage/StorageJob.h)[3](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/storage/src/storage)

---

## 4. Not an AUTOSAR Implementation

This project should **not** be interpreted as an AUTOSAR ECU implementation.

OpenBSW is an automotive basic software stack, but the project scope explicitly states that **AUTOSAR BSW components and the AUTOSAR RTE are out of scope**. Therefore, while this PoC is useful for understanding platform-based ECU software design, it should not be presented as AUTOSAR-compliant software. [1](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/3rdparty/lwip/src/api/api_msg.c)

### Practical implication
The architecture is **AUTOSAR-like in layering and intent**, but **not AUTOSAR-compliant**.

---

## 5. Timing Is Functional, Not Real-Time Certified

Lamp timeout behavior is implemented using the OpenBSW async scheduling mechanism and demonstrated successfully on POSIX. However, the current timeout behavior should be treated as **functional behavior**, not as a real-time guarantee.

### Why
On POSIX:
- scheduling depends on host OS behavior
- task timing depends on the simulation environment
- timing jitter is acceptable for learning/demo purposes but not representative of deterministic ECU timing

OpenBSW’s lifecycle and async execution model supports periodic execution behavior, but the POSIX runtime is primarily intended for learning, testing, and rapid prototyping rather than proving hard real-time guarantees. [7](https://eclipse-openbsw.github.io/openbsw/sphinx_docs/doc/dev/index.html)[2](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/io/examples/BufferedWriterExample.cpp)[3](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/storage/src/storage)

---

## 6. Persistence Scope Is Limited

The current persistence implementation stores **only lock state**.

### Persisted
- lock state (`LOCKED` / `UNLOCKED`)

### Not persisted
- door state
- lamp state
- fault state
- command history
- event history

This limitation is intentional, because the project currently focuses on demonstrating how application state can be restored using the existing OpenBSW storage abstraction, rather than building a full NVM strategy for all Smart Door states. OpenBSW’s storage path is based on `IStorage` and `StorageJob` abstractions and is suitable for this kind of incremental platform-based persistence integration. [8](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/middleware/interfaces/include/middleware/os/TaskIdProvider.h)[9](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/lifecycle/src/lifecycle/AsyncLifecycleComponent.cpp)[10](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/doip/src/doip/common/DoIpCommonLogger.cpp)

---

## 7. CAN Contract Is Project-Specific

The Smart Door CAN protocol used in this PoC is a **project-defined command/status contract**, not an industry-standard vehicle network message set.

### Current custom contract
- command CAN ID: `0x101`
- status CAN ID: `0x201`
- command byte mapping defined specifically for this project
- 4-byte status payload tailored to simulation needs

### Practical implication
The CAN interface is valid for this PoC and very useful for teaching and testing, but it would need to be adapted for any real vehicle network design, OEM communication matrix, or production CAN database integration.

---

## 8. Demo Components Still Exist in the Reference Application

This project is built on top of the **OpenBSW reference application**, which includes additional platform/demo systems such as:
- demo CAN activity
- Ethernet-related initialization
- UDS/transport-related initialization
- other framework services used by the reference app

As a result, some unrelated logs or example traffic may still appear in the runtime output even though the Smart Door feature itself is functionally isolated. This is a result of extending the existing reference app rather than creating a completely stripped-down dedicated executable. OpenBSW’s reference application is intentionally multi-featured and demonstrates several services such as CAN, transport, UDS, Ethernet, and storage. [11](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/cpp2can/src/can)[3](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/storage/src/storage)[12](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/io/examples)

### Practical implication
Runtime logs may include non-Smart-Door lines that are unrelated to Smart Door correctness.

---

## 9. Limited Safety Scope

Although OpenBSW includes safety-related modules in the wider platform and the reference application includes lifecycle/safety-related systems, this Smart Door PoC does **not** claim any functional safety compliance.

### Not covered
- ISO 26262 work products
- hazard analysis
- ASIL decomposition
- safety case
- diagnostic coverage metrics
- fault injection campaign for certification purposes

### Practical implication
This project is appropriate for **software design demonstration and learning**, but not for safety certification claims.

---

## 10. Limited Diagnostics Scope

The Smart Door ECU feature currently supports:
- command handling
- status reporting
- persistence
- timeout behavior

But it does **not** yet provide a full diagnostics strategy for the Smart Door application itself.

### Not yet included
- dedicated diagnostic identifiers for Smart Door internal states
- detailed event memory
- DTC-style error reporting
- diagnostic security / session handling specifically for Smart Door logic

The surrounding OpenBSW reference application includes transport and UDS support, but Smart Door-specific diagnostic integration has not been implemented in this PoC. OpenBSW includes CAN transport, UDS, and related application-level services, but those capabilities are broader than what is currently used by this project. [13](https://github.com/eclipse-openbsw/openbsw/blob/main/libs/bsw/docan/doc/user/index.rst)[14](https://github.com/eclipse-openbsw/openbsw/blob/main/.cmake-format)[3](https://github.com/eclipse-openbsw/openbsw/tree/main/libs/bsw/storage/src/storage)

---

## 11. Portability Is Architectural, Not Yet Proven on Target Hardware

The project was intentionally structured so that core Smart Door logic remains relatively reusable:
- controller logic
- status building
- command rules
- timeout logic

However, actual portability to a microcontroller target still requires additional integration work:
- target-specific CAN driver binding
- platform task/context setup
- storage backend adaptation
- board support and startup integration

OpenBSW’s architecture is explicitly designed to support multiple target environments through platform abstraction, including POSIX and embedded targets, but this project has only been exercised on POSIX so far.

---

## 12. Summary

This Smart Door ECU on OpenBSW is intentionally a **focused PoC** with clear educational and architectural value.

### What it is good for
- understanding how ECU application logic maps into a platform
- learning lifecycle-managed design
- understanding async scheduling in an embedded-style stack
- demonstrating persistence and timeout behavior on top of OpenBSW
- prototyping and teaching without hardware

### What it is not yet
- a hardware-validated ECU
- a production ECU
- an AUTOSAR implementation
- a safety-certified system
- a full diagnostic implementation

This limitation set is normal and acceptable for a platform-integration PoC and keeps the project honest, review-friendly, and professionally scoped. OpenBSW’s own scope emphasizes modular embedded software development, POSIX-based prototyping, and openness for experimentation rather than claiming a finalized production stack out of the box.