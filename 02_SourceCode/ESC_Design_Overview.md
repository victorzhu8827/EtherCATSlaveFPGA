# EtherCAT Slave Controller (ESC) Verilog Software Design Overview

## 1. Document Purpose

This document defines the software and RTL design overview for implementing a complete EtherCAT Slave Controller (ESC) in Verilog. The goal is to establish a clear engineering baseline before detailed architecture, register design, interface timing, and RTL coding begin.

The design basis is derived from the documents in [01_Docs](D:/CodexWorkspace/EtherCATSlave/01_Docs):

- `ETG2200_V3i1i1_G_R_SlaveImplementationGuide.pdf`
- `ethercat_esc_datasheet_sec1_technology_v2.5.pdf`
- `ethercat_esc_datasheet_sec2_registers_v3.3.pdf`
- `ethercat_et1100_datasheet_v2i1.pdf`

## 2. Design Scope and Boundary

### 2.1 Scope

This project targets a hardware ESC core implemented in FPGA/ASIC style RTL, including:

- EtherCAT frame receive, parse, process, and forward
- ESC register space and process data RAM space
- EtherCAT Application Layer (AL) state machine related hardware support
- FMMU address translation
- SyncManager mailbox mode and 3-buffer process data mode
- Distributed Clocks (DC)
- SII EEPROM interface
- Interrupt and watchdog logic
- PDI interface toward external application controller or local logic
- PHY/MII side interface management needed by the ESC

### 2.2 Out of Scope

The following functions are not part of the pure ESC hardware core and should be treated as external software or upper-layer firmware unless explicitly included in a later phase:

- Full application protocol stack implementation: CoE, FoE, EoE, SoE, AoE
- Object Dictionary semantic processing
- Device profile implementation such as CiA402 or MDP object model
- User application logic
- EtherCAT master-side ENI/ESI tooling

Note: The ESC must still provide the mailbox transport mechanism and the hardware resources required for these protocols, but the protocol semantics should remain outside the base ESC RTL.

## 3. Requirements Definition

### 3.1 Functional Requirements

The ESC shall support the following base capabilities:

1. It shall support EtherCAT slave communication over at least 2 Ethernet ports, with architecture extensible to 4 ports.
2. It shall process EtherCAT datagrams on the fly and forward frames with minimum latency.
3. It shall provide 64 KB logical ESC address space, including:
   - register space `0x0000` to `0x0FFF`
   - process data RAM starting at `0x1000`
4. It shall support configured station address and alias address handling.
5. It shall implement AL Control, AL Status, and AL Status Code registers.
6. It shall support EtherCAT state transitions: `INIT`, `PRE-OP`, `SAFE-OP`, `OP`, and optional `BOOT`.
7. It shall support SyncManager resources for:
   - mailbox out
   - mailbox in
   - process data outputs
   - process data inputs
8. It shall support FMMU-based logical-to-physical mapping for process data access.
9. It shall support mailbox mode and buffered mode operation.
10. It shall support process data consistency control by SyncManager.
11. It shall support AL event generation and ECAT event generation.
12. It shall support process data watchdog and PDI watchdog.
13. It shall support SII EEPROM read, write, and reload control.
14. It shall support Distributed Clocks registers, local time base, and Sync/Latch signaling.
15. It shall provide a PDI-side access path for host controller or local user logic.
16. It shall support error counters, link status, and port status reporting.
17. It shall support PHY management interface registers if external PHY management is included in the selected architecture.

### 3.2 Performance Requirements

1. The frame processing pipeline shall support line-rate 100 Mbps EtherCAT operation.
2. The forwarding path shall be streaming-based; full-frame store-and-forward shall be avoided except where explicitly required by protocol handling.
3. The internal RAM and arbitration design shall guarantee deterministic behavior for EtherCAT, PDI, and DC events.
4. The DC subsystem shall provide a stable local time base and support precise synchronization better than microsecond level as required by EtherCAT DC concept.

### 3.3 Configurability Requirements

The RTL shall be parameterizable for:

- number of physical ports: 2/3/4
- number of SyncManagers
- number of FMMUs
- size of process data RAM
- PDI type
- DC enable/disable
- EEPROM emulation enable/disable
- mailbox support enable/disable

### 3.4 Verification and Compliance Requirements

1. The implementation shall align with EtherCAT ESC technology and register model described in the referenced documents.
2. Register behavior, reset values, access permissions, and side effects shall be traceable to the register specification.
3. The design shall be structured for later conformance validation using EtherCAT Conformance Test Tool (CTT).
4. The implementation shall support ESI/EEPROM-configured initialization expected by standard EtherCAT masters.

## 4. Functional Decomposition

Based on the ESC technology documentation, the complete ESC hardware is decomposed into the following function domains.

### 4.1 Port and PHY Domain

Responsibilities:

- RX/TX interface to external PHYs through MII or equivalent adaptation
- link detection and port status collection
- loop control and port open/close logic
- forwarding path selection between ports
- support for special handling of port 0 and unused ports

### 4.2 EtherCAT Frame Processing Domain

Responsibilities:

- EtherCAT frame detection
- Ethernet frame header and EtherCAT datagram parsing
- address mode handling
- command decode
- working counter update
- read/write/readwrite command execution
- frame forwarding after local modification
- circulating frame protection and frame order control

This domain is the real-time core of the ESC and must be implemented as a low-latency pipeline.

### 4.3 Register and CSR Domain

Responsibilities:

- register map implementation from `0x0000` to `0x0FFF`
- read/write decode
- reset defaults
- side-effect handling
- event and status reflection
- write protection and reset control

This block is the control plane anchor of the whole design.

### 4.4 Process Data Memory Domain

Responsibilities:

- dual-ported or arbitrated RAM for ESC and PDI access
- separation of register space and process data RAM
- mailbox and PDO buffer allocation
- byte-enable and alignment handling
- arbitration between EtherCAT access, PDI access, and DC timestamp/event access

### 4.5 FMMU Domain

Responsibilities:

- logical start address and length matching
- logical bit to physical byte/bit translation
- direction control
- activation control
- support for separate input/output mappings

This domain converts EtherCAT logical memory operations into local process memory operations.

### 4.6 SyncManager Domain

Responsibilities:

- SyncManager register set
- mailbox mode state control
- buffered mode and 3-buffer mode handling
- producer/consumer ownership handshaking
- buffer full/empty/available status
- PDI acknowledge by write support if selected
- interrupt and watchdog trigger generation

SyncManager is the consistency manager between network side and application side memory accesses.

### 4.7 AL State Machine Domain

Responsibilities:

- AL control request reception
- state transition validation
- AL status update
- AL status code generation
- `INIT`, `PRE-OP`, `SAFE-OP`, `OP`, `BOOT` transition support
- interaction with mailbox readiness, SyncManager validity, FMMU validity, and output safety gating

Important boundary:

- The ESC hardware shall provide the AL hardware state machine and transition checks related to ESC resources.
- Device-specific application validation may be delegated to external firmware through PDI handshake.

### 4.8 Interrupt and Event Domain

Responsibilities:

- ECAT event mask and request registers
- AL event mask and request registers
- event aggregation from SyncManager, EEPROM, DC, watchdog, and state machine
- interrupt output generation toward PDI or host controller

### 4.9 Watchdog Domain

Responsibilities:

- watchdog divider
- process data watchdog
- PDI watchdog
- timeout status and counters
- fail-safe output indication interface

This block is safety-critical for output validity in `SAFE-OP` and `OP` behavior.

### 4.10 SII EEPROM Domain

Responsibilities:

- I2C-based EEPROM access
- ECAT/PDI ownership control
- read/write/reload command handling
- EEPROM-loaded indication
- optional EEPROM emulation path

The EEPROM domain is required because standard EtherCAT startup relies on SII data such as vendor ID, product code, revision, mailbox defaults, and SyncManager/FMMU defaults.

### 4.11 Distributed Clocks Domain

Responsibilities:

- local 64-bit system time counter
- receive timestamp capture for ports
- offset and delay compensation registers
- synchronization control loop support
- Sync0/Sync1 pulse generation
- latch input timestamp capture
- SyncManager event time capture

DC support is mandatory if the target product needs tight synchronization with drives or synchronized I/O.

### 4.12 PDI Domain

Responsibilities:

- host-side register and RAM access interface
- PDI timing adaptation
- read/write transaction handling
- byte ordering and access size conversion
- interrupt handshaking
- optional support for SPI or asynchronous/synchronous microcontroller interface variants

For an FPGA-based first version, an internal memory-mapped bus PDI is recommended because it simplifies integration and verification. External SPI or microcontroller bus compatibility can be added later as wrappers.

## 5. Recommended Top-Level RTL Architecture

The recommended top-level architecture is shown logically below:

1. `esc_port_if`
2. `esc_frame_engine`
3. `esc_datagram_executor`
4. `esc_reg_bank`
5. `esc_pd_ram`
6. `esc_fmmu_array`
7. `esc_sm_array`
8. `esc_al_fsm`
9. `esc_event_irq`
10. `esc_watchdog`
11. `esc_eeprom_ctrl`
12. `esc_dc_unit`
13. `esc_pdi_if`
14. `esc_reset_clock`

Suggested data flow:

- PHY RX -> port interface -> frame engine -> datagram executor
- datagram executor -> register bank / FMMU / SyncManager / process RAM
- results -> working counter update -> frame rewrite -> forwarding path -> PHY TX
- PDI -> register bank / process RAM / AL handshake / interrupt status
- DC, watchdog, EEPROM, SyncManager -> event/interrupt fabric

## 6. Module Partitioning Proposal

### 6.1 Top Level

`esc_top`

Responsibilities:

- clock/reset distribution
- instantiation of all submodules
- global parameter binding
- top-level bus and event interconnect

### 6.2 Real-Time Data Plane Modules

`esc_port_rx`

- MII RX adaptation
- preamble/SFD/frame boundary detection
- RX error reporting

`esc_port_tx`

- MII TX adaptation
- transmit arbitration
- IFG handling if needed by wrapper

`esc_forward_switch`

- port-to-port routing
- loop and forwarding control

`esc_frame_parser`

- EtherType and EtherCAT frame recognition
- datagram boundary parsing

`esc_datagram_executor`

- EtherCAT command execution
- address decode
- local data insertion/extraction
- working counter processing

### 6.3 Control Plane Modules

`esc_reg_bank`

- full register decode and storage
- access permission checks

`esc_reset_ctrl`

- ECAT reset
- PDI reset
- local soft reset sequencing

`esc_status_ctrl`

- link, error, LED, and feature status composition

### 6.4 Memory and Mapping Modules

`esc_pdpram`

- process data and mailbox RAM
- dual-port or multi-master arbitration

`esc_fmmu`

- one FMMU mapping element

`esc_fmmu_array`

- multiple FMMU entries and match selection

`esc_syncmanager`

- one SyncManager channel

`esc_sm_array`

- multiple SyncManager channels
- mailbox/process-data role binding

### 6.5 AL and Application Interface Modules

`esc_al_fsm`

- AL transition control
- state code generation

`esc_pdi_if`

- host application access interface
- register and RAM access bridge

`esc_output_safe_ctrl`

- output validity gating in `SAFE-OP`, watchdog timeout, or error state

### 6.6 Service Modules

`esc_irq_event`

- event mask/request handling
- interrupt output generation

`esc_watchdog`

- watchdog timers and timeout events

`esc_eeprom_ctrl`

- I2C master for SII EEPROM
- reload sequence

`esc_phy_mgmt`

- MDIO/MI management if implemented

`esc_dc_core`

- system time, offset, delay, timestamp capture

`esc_sync_latch`

- Sync0/1 generation and latch input capture

## 7. External Interfaces Definition

### 7.1 EtherCAT Port Interface

Minimum first-version recommendation:

- 2 x MII/RMII-compatible logical wrappers
- link status inputs
- PHY reset outputs
- optional MDIO management interface

### 7.2 PDI Interface

Recommended first-version internal interface:

- simple synchronous memory-mapped slave bus
- address, write data, read data, byte enable, read/write strobes, ready, IRQ

Recommended later compatibility wrappers:

- SPI slave PDI
- asynchronous 8/16-bit microcontroller bus
- synchronous 8/16-bit microcontroller bus

### 7.3 EEPROM Interface

- I2C master signals: `SCL`, `SDA`

### 7.4 DC Interface

- `SYNC0`, `SYNC1`
- `LATCH0`, `LATCH1`

## 8. Address Map Planning

The RTL shall reserve the standard EtherCAT register areas:

- `0x0000` to `0x001F`: ESC information and station address
- `0x0020` to `0x0041`: write protection and reset
- `0x0100` to `0x0139`: DL and AL control/status
- `0x0140` to `0x019D`: PDI configuration
- `0x0200` to `0x0223`: interrupt/event
- `0x0300` to `0x0342`: error counters
- `0x0400` to `0x0448`: watchdog
- `0x0500` to `0x051B`: EEPROM and PHY management
- `0x0600` to `0x06FF`: FMMU
- `0x0800` to `0x087F`: SyncManager
- `0x0900` to `0x09FF`: Distributed Clocks
- `0x0F80` to `0x0FFF`: user RAM / ESC-specific area if adopted
- `0x1000` and above: process data RAM

The detailed register behavior shall be defined in the next stage as a register specification document.

## 9. Recommended Development Strategy

To control implementation risk, development should be divided into phases.

### Phase 1: ESC Minimum Communication Core

Target:

- 2-port communication
- register access
- station addressing
- basic datagram execution
- process RAM access
- minimal FMMU
- minimal SyncManager
- AL state transitions up to `PRE-OP` / `SAFE-OP`

Goal:

- master can discover slave
- EEPROM data can be read
- basic mailbox and process RAM path works

### Phase 2: Process Data and State Completeness

Target:

- complete SyncManager behavior
- FMMU completeness
- watchdog
- interrupt/event
- `OP` state support
- output safety gating

Goal:

- cyclic PDO exchange stable under standard master

### Phase 3: DC and Advanced Interfaces

Target:

- distributed clocks
- Sync/Latch
- precise timestamping
- extended PDI wrappers
- PHY management

Goal:

- synchronized I/O or drive-class timing support

### Phase 4: Conformance Hardening

Target:

- corner-case register behavior
- reset side effects
- mailbox edge conditions
- error injection handling
- CTT-oriented fixes

Goal:

- ready for formal conformance testing

## 10. Key Engineering Decisions Recommended at This Stage

1. Use a streaming frame-processing architecture instead of store-and-forward.
2. Use parameterized arrays for FMMU and SyncManager instances.
3. Separate data plane and control plane strictly to reduce timing risk.
4. Implement process RAM as a dedicated memory subsystem with explicit arbitration.
5. Treat mailbox protocol semantics as software above the ESC unless later required in hardware.
6. Implement a simple internal PDI bus first, then wrap external bus protocols later.
7. Keep DC as an independent timing subsystem so that it can be disabled for low-end variants.
8. Maintain register compatibility with Beckhoff-style ESC address map as the primary compatibility target.

## 11. Risks and Technical Difficulties

The main technical risks are:

- on-the-fly EtherCAT frame modification without breaking timing
- exact SyncManager mailbox and buffered mode semantics
- AL state transition corner cases and error reporting
- concurrent access arbitration among ECAT, PDI, EEPROM, and DC units
- watchdog behavior and safe output gating
- DC synchronization precision and timestamp correctness
- compatibility with standard master initialization sequences

These risks justify designing the RTL around independent, testable modules with a strict register specification and a directed verification plan.

## 12. Deliverables for the Next Stage

After this overview, the next documents and design outputs should be produced in order:

1. ESC system architecture specification
2. detailed register specification
3. memory map and DPRAM allocation specification
4. PDI interface specification
5. AL state machine specification
6. FMMU and SyncManager detailed behavior specification
7. DC detailed design
8. RTL module interface definition
9. verification plan and testbench architecture

## 13. Summary

The complete Verilog ESC shall be built as a standard-compatible hardware communication core centered on:

- frame processing
- register compatibility
- FMMU mapping
- SyncManager consistency control
- AL state control
- watchdog and interrupt handling
- EEPROM startup support
- optional Distributed Clocks precision timing
- clean PDI interface toward upper application logic

From an engineering perspective, the correct boundary is:

- ESC hardware implements EtherCAT communication infrastructure and resource control
- mailbox transport exists in hardware
- mailbox protocol semantics and device profile behavior stay in upper-layer firmware/software

This boundary is the most practical way to achieve a complete, reusable, and verifiable EtherCAT slave controller in Verilog.
