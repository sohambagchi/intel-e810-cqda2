# E810 Specification For The CQDA2 Target

## Status

This document is a thorough, Termite-inspired specification outline for a minimal Intel E810 PF driver model, prepared from:

- `docs/termite.md`
- `docs/e810_datasheet.md`

It is not an executable Termite specification. It is structured to match the Termite paper's separation of concerns and state-machine style, but it remains a documentation artifact.

The requested hardware target is `e810-cqda2`. The provided datasheet, however, is a controller-family datasheet for Intel E810 devices and does not explicitly identify a board SKU named `E810-CQDA2` or `CQDA2`. This specification therefore covers:

- the E810 PF programming model that can be derived from the provided controller datasheet
- only those CQDA2-related claims that are directly supportable from that datasheet

## Derivation

Per the Termite paper, a driver specification should separate:

1. device-class behavior
2. OS interface behavior
3. device behavior

This document follows that structure.

The device specification here is derived from the provided datasheet, not from RTL and not from an existing driver. No model checking or hardware-faithfulness proof has been performed.

## Scope

This specification intentionally models a minimal PF-level NIC core.

Included:

- single PF driver load
- interrupt disable/enable points relevant to bring-up
- firmware Admin Queue bring-up
- `Get Version (0x0001)` gating
- `Driver Version (0x0002)` reporting
- `Get Link Status (0x0607)` and Link Status Event behavior
- receive queue enable flow at the register/state level
- transmit queue creation through `Add Tx LAN Queues (0x0C30)`
- AQ shutdown and driver unload behavior

Excluded:

- VF and mailbox semantics except where they constrain PF behavior
- RDMA / Protocol Engine behavior
- detailed HMC programming
- detailed MAC/PHY programming beyond link-status control
- detailed switch and scheduler topology programming
- DDP
- NVM update and recovery flows
- 1588 / SyncE
- NC-SI / PLDM / SMBus management protocols
- board-specific optics, retimers, LEDs, cage wiring, and topology netlist details

## Source Limits

Supported by the provided datasheet:

- Intel E810 controller-family behavior
- PF Admin Queue structure and initialization
- PF link-status control through AQ
- PF-level interrupt enabling model
- PF receive-queue enable flow
- PF transmit-queue add flow

Not supported by the provided datasheet:

- that the exact adapter SKU is named `E810-CQDA2`
- CQDA2-specific port count, cage type, PHY topology, retimer topology, or board reset wiring
- any board-level statement tying CQDA2 to `E810-CAM2`, `E810-CAM1`, or `E810-XXVAM2`

## Termite Alignment

The Termite paper requires these properties, which this document reflects explicitly:

- the `device-class` specification defines shared events used by both device and OS specifications
- the `OS` specification defines requests, callbacks, ordering constraints, and liveness obligations
- the `device` specification defines software-visible objects and state transitions caused by driver actions and device responses
- safety means the driver does not violate allowed ordering
- liveness means required completions eventually occur when the environment behaves within the specification

## Component Declaration

```text
component E810PfDriver
  E810PfClass class
  E810PfOS    os
  E810PfDev   dev
```

## Device-Class Specification

The device-class layer defines the abstract events shared between the OS-facing and device-facing specifications. In Termite terms, this is the shared vocabulary, not the register-level mechanism.

### Interface

```text
interface E810PfClass

types:
  enum init_status_t {
    INIT_OK,
    INIT_ERR_FW_NOT_READY,
    INIT_ERR_AQ_SETUP,
    INIT_ERR_AQ_VERSION
  }

  enum link_state_t {
    LINK_DOWN,
    LINK_UP
  }

  enum admin_status_t {
    AQ_OK,
    AQ_ERR,
    AQ_ERR_VERSION,
    AQ_ERR_TIMEOUT
  }

messages:
  internal adminQueueReady()
  internal deviceReady(init_status_t status)
  internal linkEventsEnabled()
  internal linkStatusObserved(link_state_t state)
  internal linkStatusChanged(link_state_t state)
  internal rxQueueEnabled(unsigned<32> queue_id)
  internal txQueueAdded(unsigned<32> queue_id)
  internal controlQueueShutdownComplete()
  internal deviceInactive()
```

### Meaning

- `adminQueueReady()` denotes the documented point at which firmware AQ use is safe enough for `Get Version` to succeed.
- `deviceReady(INIT_OK)` denotes completion of the minimal PF initialization modeled here.
- `linkEventsEnabled()` denotes successful software arming of LSE through `Get Link Status`.
- `linkStatusObserved(...)` denotes an explicit software poll of link status through AQ.
- `linkStatusChanged(...)` denotes an asynchronous firmware-generated LSE delivered on ARQ.
- `rxQueueEnabled(...)` and `txQueueAdded(...)` abstract queue lifecycle milestones without exposing every queue-context field at the class level.
- `controlQueueShutdownComplete()` denotes completion of the documented AQ shutdown flow.

This device-class specification does not impose an ordering of its own. As in the Termite paper's example, the ordering is imposed by the OS and device specifications.

## OS Interface Specification

The OS specification defines the service that the driver provides upward. This is a simplified OS interface, not a kernel-specific ABI.

### Interface

```text
interface E810PfOS

types:
  enum os_status_t {
    OS_OK,
    OS_ERR_FW_NOT_READY,
    OS_ERR_AQ_SETUP,
    OS_ERR_AQ_VERSION,
    OS_ERR_LINK,
    OS_ERR_RX_QUEUE,
    OS_ERR_TX_QUEUE,
    OS_ERR_SHUTDOWN
  }

  struct rxq_request_t {
    unsigned<32> queue_id;
    unsigned<64> ring_base;
    unsigned<32> ring_len;
  }

  struct txq_request_t {
    unsigned<32> queue_id;
    unsigned<32> parent_teid;
  }

messages:
  in  probe()
  out probeComplete(os_status_t status)

  in  enableLinkEvents()
  out enableLinkEventsComplete(os_status_t status)

  in  pollLinkStatus()
  out linkStatusReport(bool link_up)

  in  setupRxQueue(rxq_request_t q)
  out setupRxQueueComplete(unsigned<32> queue_id, os_status_t status)

  in  setupTxQueue(txq_request_t q)
  out setupTxQueueComplete(unsigned<32> queue_id, os_status_t status)

  in  remove()
  out removeComplete(os_status_t status)
```

### Safety Requirements

1. `setupRxQueue` and `setupTxQueue` are not allowed before a successful `probeComplete(OS_OK)`.
2. `pollLinkStatus` and `enableLinkEvents` are not allowed before AQ initialization and AQ version verification succeed.
3. Once AQ shutdown has started, no further AQ-using OS requests are allowed.
4. The driver must not report success for queue operations before the corresponding device-class queue event occurs.

### Liveness Requirements

1. After `probe()`, the driver must eventually respond with `probeComplete(...)`.
2. After `enableLinkEvents()`, the driver must eventually either arm LSE or return an error.
3. After `pollLinkStatus()`, the driver must eventually return a link report.
4. After `setupRxQueue(...)`, the driver must eventually enable the queue or return an error.
5. After `setupTxQueue(...)`, the driver must eventually add the queue or return an error.
6. After `remove()`, the driver must eventually complete shutdown and return `removeComplete(...)`.

### State Machine

```text
transitions:

  probe;
  class adminQueueReady():timed;
  class deviceReady(INIT_OK):timed;
  probeComplete(OS_OK):timed;
  RUNNING

where

  process RUNNING
    enableLinkEvents;
    class linkEventsEnabled():timed;
    enableLinkEventsComplete(OS_OK):timed;
    RUNNING

  [] pollLinkStatus;
     class linkStatusObserved($state):timed;
     linkStatusReport($state == LINK_UP):timed;
     RUNNING

  [] setupRxQueue / m_rxqid = $q.queue_id;
     class rxQueueEnabled(m_rxqid):timed;
     setupRxQueueComplete(m_rxqid, OS_OK):timed;
     RUNNING

  [] setupTxQueue / m_txqid = $q.queue_id;
     class txQueueAdded(m_txqid):timed;
     setupTxQueueComplete(m_txqid, OS_OK):timed;
     RUNNING

  [] class linkStatusChanged(LINK_UP);
     linkStatusReport(true):timed;
     RUNNING

  [] class linkStatusChanged(LINK_DOWN);
     linkStatusReport(false):timed;
     RUNNING

  [] remove;
     class controlQueueShutdownComplete():timed;
     class deviceInactive():timed;
     removeComplete(OS_OK):timed;
     exit
```

### OS Simplifications

- The OS interface abstracts HMC, switch, and scheduler preparation that a production driver would manage internally.
- `setupTxQueue(...)` assumes any software-managed profile preparation needed before `Add Tx LAN Queues` has already been done by driver-internal logic.
- Queue interrupt-vector programming is not represented as a separate OS request in this first-pass model, because the reviewed datasheet material establishes the interrupt framework and later queue association steps, but not a compact OS-facing transaction boundary for them.

## Device Specification

This section models the software-visible E810 PF behavior relevant to the chosen scope.

## Device Overview

The reviewed datasheet material supports this high-level PF bring-up order for non-virtualized driver load in Section `4.4.1.1`:

1. probe resources
2. disable interrupts
3. initialize Admin Queue
4. initialize HMC
5. initialize MAC/PHY
6. initialize power management
7. initialize DCB
8. initialize switch and Tx-Scheduler
9. initialize statistics baseline
10. initialize Protocol Engine
11. initialize 1588
12. enable interrupts

After that point, VSIs, LAN queue pairs, and filters are initialized dynamically.

This specification models only the subset needed to get from reset to a minimally usable PF with AQ, link-status control, and basic queue bring-up.

## Software-Visible Objects

The following objects are directly established by the reviewed datasheet sections.

- Control Queue pair consisting of ATQ and ARQ
- 32-byte little-endian control descriptors
- AQ descriptor fields including `DD`, `CMP`, `ERR`, `BUF`, `RD`, `SI`, `EI`, `FE`, opcode, and indirect buffer pointers
- AQ CSR set including `{PFX}{TYP}ATQBAH`, `ATQBAL`, `ATQLEN`, `ATQH`, `ATQT`, `ARQBAH`, `ARQBAL`, `ARQLEN`, `ARQH`, `ARQT`
- queue-enable semantics through the MSB of the AQ length registers
- control-queue interrupts on ATQ completion and ARQ event posting
- `Get Version (0x0001)`
- `Driver Version (0x0002)`
- `Queue Shutdown (0x0003)`
- `Get Link Status (0x0607)` as an indirect AQ command
- Link Status Event on ARQ, using the same opcode family and response structure as `Get Link Status`
- receive queue control with `QRX_TAIL[n]`, `QRX_CTRL[n].QENA_REQ`, and `QRX_CTRL[n].QENA_STAT`
- transmit queue addition through `Add Tx LAN Queues (0x0C30)`

## Device State

```text
interface E810PfDev

variables:
  bool interrupts_enabled = false
  bool aq_rx_enabled = false
  bool aq_tx_enabled = false
  bool aq_heads_tails_cleared = false
  bool aq_receive_buffers_posted = false
  bool aq_version_verified = false
  bool driver_version_reported = false
  bool device_ready = false
  bool lse_enabled = false
  bool shutting_down = false
  bool link_up = false
  bool reported_link = false
  unsigned<32> last_rx_queue_id = 0
  unsigned<32> last_tx_queue_id = 0
```

## Device Messages

```text
messages:
  in  disableInterrupts()
  in  clearControlQueuePointers()
  in  programControlQueueRegisters()
  in  postArqBuffer(buf_addr, buf_len)
  in  adminGetVersion()
  in  adminDriverVersion(version_buf_addr, version_len)
  in  adminGetLinkStatus(enable_lse, resp_buf_addr, resp_len)
  in  programRxQueueContext(queue_id, ring_base, ring_len)
  in  setRxQueueTail(queue_id, tail)
  in  setRxQueueEnable(queue_id)
  in  adminAddTxLanQueues(queue_id, cmd_buf_addr, resp_buf_addr)
  in  enableInterrupts()
  in  adminQueueShutdown(driver_unloading)
  in  clearAtqEnable()
  in  functionReset()

  out arqCompletion(opcode, admin_status_t status)
  out arqLinkEvent(link_state_t state)
```

## Device Rules

### Reset And Readiness Gating

- Firmware may block `Get Version` until internal initialization has progressed far enough for the E810 to load required NVM content into hardware.
- The datasheet explicitly states that AQ readiness and a successful `Get Version` response are the proof points that software can start device-driver initialization.
- The driver must therefore not assume AQ usability immediately on PCI enumeration alone.

### Interrupt Model

- Interrupts are enabled at three levels in Section `9.1.1.1`:
  - PCIe-level interrupt enablement by OS configuration
  - driver-level enablement through interrupt control registers
  - cause-level enablement and moderation
- The documented driver-load sequence disables interrupts before AQ initialization and enables interrupts later in bring-up.
- Hardware link-status interrupts are not the normal control path for software link management. The datasheet states that software should use AQ `Get Link Status` and LSE instead.

### Control Queue Initialization

The AQ rules from Section `9.5.2` and `9.5.3` are:

1. allocate host memory for ATQ and ARQ
2. post initialized receive buffers to ARQ before using ATQ
3. clear control-queue head and tail registers
4. program ATQ/ARQ base and length registers
5. set the queue-enable bit in the AQ length registers
6. issue `Get Version (0x0001)` before using a firmware AQ for anything else
7. verify queue/API and firmware major versions
8. fail driver load on major-version mismatch
9. send `Driver Version (0x0002)` after successful version verification

### Link Status

From Section `3.2.4.1.5` and `3.2.4.1.6`:

- `Get Link Status (0x0607)` is an indirect AQ command.
- It reports whether the link is up and ready for data communication.
- It also returns additional fault and operating information.
- The command flags can enable or disable Link Status Events.
- LSE is disabled by default.
- After firmware generates an LSE, firmware disables LSE immediately and does not queue further LSEs until software explicitly re-enables them via another `Get Link Status` command.
- LSE is delivered on ARQ.

### Receive Queue Enable Flow

From Section `10.4.3.1.1`:

1. allocate and populate the receive ring in host memory
2. program receive queue context in the required queue-context structures
3. clear `QRX_TAIL[n]`, then set it to the end of the ring
4. set `QRX_CTRL[n].QENA_REQ`
5. for No-Drop traffic classes, program the documented `CDS` and `CDE` behavior
6. wait until hardware reflects the enable by setting `QENA_STAT`
7. only after `QENA_STAT` is set may software use the queue

This spec treats the `QENA_STAT` transition as the queue-enable completion event.

### Transmit Queue Add Flow

From Section `10.5.5.8.1`:

- `Add Tx LAN Queues (0x0C30)` is an indirect AQ command.
- It configures Tx queues, allocates/configures Tx-scheduler leaf nodes, and associates queues with those nodes.
- Software supplies queue groups, parent TEIDs, queue IDs, queue contexts, and leaf-node configuration.
- Firmware returns per-queue TEIDs.
- Before association, software must already have initialized any Completion Queue used by the Tx queue.
- Before association, software must already have initialized any referenced quanta, cache, or packet-shaper profiles.
- Queue interrupt association is a later software step and is not part of `Add Tx LAN Queues` itself.

## Device State Machine

```text
transitions:

  RESET

where

  process RESET
    disableInterrupts / interrupts_enabled = false;
    AQ_SETUP

  process AQ_SETUP
    clearControlQueuePointers / aq_heads_tails_cleared = true;
    programControlQueueRegisters / { aq_tx_enabled = true; aq_rx_enabled = true; };
    postArqBuffer / aq_receive_buffers_posted = true;
    AQ_WAIT_READY

  process AQ_WAIT_READY
    class adminQueueReady():timed;
    AQ_VERIFY

  process AQ_VERIFY
    adminGetVersion [aq_tx_enabled && aq_rx_enabled && aq_heads_tails_cleared && aq_receive_buffers_posted];
    arqCompletion(0x0001, AQ_OK) / aq_version_verified = true :timed;
    adminDriverVersion($version_buf_addr, $version_len) [aq_version_verified];
    arqCompletion(0x0002, AQ_OK) / driver_version_reported = true :timed;
    class deviceReady(INIT_OK) / device_ready = true :timed;
    CORE_INIT

  process CORE_INIT
    enableInterrupts / interrupts_enabled = true;
    READY

  process READY
    adminGetLinkStatus(true, $resp_buf_addr, $resp_len);
    arqCompletion(0x0607, AQ_OK) / lse_enabled = true :timed;
    class linkEventsEnabled():timed;
    RUNNING

  process RUNNING
    adminGetLinkStatus($enable_lse, $resp_buf_addr, $resp_len);
    arqCompletion(0x0607, AQ_OK) / { lse_enabled = $enable_lse; link_up = $reported_link; } :timed;
    if[$reported_link] class linkStatusObserved(LINK_UP):timed [] else class linkStatusObserved(LINK_DOWN):timed;
    RUNNING

  [] arqLinkEvent(LINK_UP) [lse_enabled] / { link_up = true; lse_enabled = false; };
     class linkStatusChanged(LINK_UP):timed;
     RUNNING

  [] arqLinkEvent(LINK_DOWN) [lse_enabled] / { link_up = false; lse_enabled = false; };
     class linkStatusChanged(LINK_DOWN):timed;
     RUNNING

  [] programRxQueueContext($qid, $ring_base, $ring_len) [device_ready] / last_rx_queue_id = $qid;
     setRxQueueTail($qid, 0);
     setRxQueueTail($qid, ring_end);
     setRxQueueEnable($qid);
     await[rx_qena_stat[last_rx_queue_id] == 1]:timed;
     class rxQueueEnabled(last_rx_queue_id):timed;
     RUNNING

  [] adminAddTxLanQueues($qid, $cmd_buf_addr, $resp_buf_addr) [device_ready] / last_tx_queue_id = $qid;
     arqCompletion(0x0C30, AQ_OK):timed;
     class txQueueAdded(last_tx_queue_id):timed;
     RUNNING

  [] adminQueueShutdown($driver_unloading) / shutting_down = true;
     arqCompletion(0x0003, AQ_OK):timed;
     class controlQueueShutdownComplete():timed;
     SHUTDOWN

  process SHUTDOWN
    clearAtqEnable;
    if[$driver_unloading] functionReset;
    class deviceInactive():timed;
    INACTIVE

  process INACTIVE
    exit
```

## Safety Properties

1. The driver must not use a firmware AQ for normal commands before `Get Version` completes successfully.
2. The driver must not use ATQ before ARQ buffers are posted.
3. The driver must not treat the device as ready before AQ major-version verification succeeds.
4. The driver must not use hardware link-status interrupts as its normal link-management mechanism.
5. The driver must not treat an Rx queue as enabled before `QENA_STAT` reflects the enable request.
6. The driver must not add a Tx queue before the required software-managed queue dependencies are already prepared.
7. After `Queue Shutdown`, the driver must not issue further AQ commands on that queue until reset.
8. AQ re-enable after shutdown must clear AQ head and tail pointers before reuse.

## Liveness Properties

1. If firmware reaches the documented AQ-ready state, a valid `Get Version` command eventually completes.
2. If `Get Version` succeeds and the major version matches, `Driver Version` can eventually be posted and the device can reach `deviceReady(INIT_OK)`.
3. If software polls link status through AQ, firmware eventually returns a completion or an error.
4. If LSE is enabled and a link-state-changing condition occurs, firmware may emit one LSE and software must re-arm LSE for further asynchronous notifications.
5. If Rx queue enable is requested on a correctly configured queue, hardware eventually reflects the enable through `QENA_STAT`.
6. If `Add Tx LAN Queues` is issued with valid prerequisites, firmware eventually completes the command.
7. If `Queue Shutdown` is posted, firmware eventually completes the shutdown flow, after which software can clear ATQ enable and proceed to reset if unloading.

## Open Ambiguities

The following ambiguities remain and are documented rather than guessed:

1. The supplied datasheet does not identify a board SKU named `E810-CQDA2`.
2. The supplied datasheet does not establish CQDA2-specific topology, optics, PHY chain, retimer chain, or LED/reset wiring.
3. The reviewed material establishes the existence of dynamic queue interrupt association, but this first-pass spec does not formalize the exact per-queue interrupt-vector programming sequence.
4. The reviewed material establishes that `Get Link Status` returns many detailed fields, but this first-pass spec only models the link-up/link-down control result.
5. The full intermediate bring-up between AQ verification and datapath readiness is broader than this minimal spec and includes HMC, MAC/PHY, DCB, switch, scheduler, Protocol Engine, and 1588 initialization.

## References Used

This specification is grounded in these reviewed datasheet sections:

- `4.2.1.2` for firmware/AQ readiness gating
- `4.4.1.1` for non-virtualized PF driver-load ordering
- `9.1.1.1` and `9.1.1.2` for interrupt enablement structure
- `9.5.2`, `9.5.2.1`, and `9.5.2.2` for AQ structure, CSRs, and AQ interrupts
- `9.5.3` and `9.5.3.1` for AQ initialization and ARQ buffer initialization
- `9.5.4` and `9.5.13.3` for AQ shutdown behavior
- `9.5.13.1` and `9.5.13.2` for `Get Version` and `Driver Version`
- `3.2.4.1.5` and `3.2.4.1.6` for `Get Link Status` and LSE
- `10.4.3.1.1` for Rx queue enable flow
- `10.5.5.8.1` through `10.5.5.8.1.2` for `Add Tx LAN Queues`
