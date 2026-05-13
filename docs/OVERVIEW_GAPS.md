# Overview Gap Notes

This file records the exact source-backed information used to fill the active `\anton{}` placeholders in `amazon26/overview.tex`, together with source locations and short explanatory notes.

## Scope

- Target file edited: `amazon26/overview.tex`
- Active placeholders addressed:
  - `amazon26/overview.tex:158`
  - `amazon26/overview.tex:330`
- User instruction on framing:
  - mix DPDK and Linux for the kernel-integration example
  - do not broaden the edit beyond replacing the placeholders

## Final Insertions In `overview.tex`

### 1. Kernel-integration example

Inserted text in `amazon26/overview.tex:157-166`:

```tex
2)~integration with the kernel -- kernel communicates with the driver via a
collection of hierarchical data strucrues, e.g., in Linux ICE via \texttt{net\_device}, \texttt{napi\_struct}, and \texttt{sk\_buff} objects connected to driver-owned \texttt{ice\_vsi}, \texttt{ice\_q\_vector}, and \texttt{ice\_rx\_ring}/\texttt{ice\_tx\_ring} state and APIs such as \texttt{register\_netdev}, \texttt{netif\_napi\_add\_config}, \texttt{napi\_gro\_receive}, and \texttt{dma\_map\_single}, while hardware communication uses MMIO register accessors like \texttt{rd32}/\texttt{wr32} and firmware Admin Queue commands like \texttt{ice\_aq\_send\_cmd}; in DPDK, via \texttt{rte\_eth\_dev}/\texttt{rte\_eth\_dev\_data}, per-port \texttt{rx\_queues}/\texttt{tx\_queues}, and \texttt{rte\_mempool} objects connected to \texttt{ice\_pf}, \texttt{main\_vsi}, and \texttt{ci\_rx\_queue}/\texttt{ci\_tx\_queue} state and APIs such as \texttt{eth\_dev\_ops}, \texttt{rte\_eth\_dma\_zone\_reserve}, \texttt{rte\_mbuf\_raw\_alloc}, interrupt hooks, BAR-mapped MMIO via \texttt{rte\_read32}/\texttt{rte\_write32}, and firmware commands via \texttt{ice\_aq\_send\_cmd}.
```

### 2. Device-action examples

Inserted text in `amazon26/overview.tex:328-330`:

```tex
We develop an agentic system that mimics the development process of the driver,
i.e., it 1)~identifies ``actions'', i.e., possible interactions with the device
like, on E810, to bring up the firmware Admin Queue the driver first allocates ATQ/ARQ memory and posts ARQ buffers, then clears \texttt{ATQH}/\texttt{ATQT} and \texttt{ARQH}/\texttt{ARQT}, programs \texttt{ATQBA[LH]}/\texttt{ATQLEN} and \texttt{ARQBA[LH]}/\texttt{ARQLEN}, issues \texttt{Get Version (0x0001)}, and only after a compatible reply sends \texttt{Driver Version (0x0002)}, etc.,
```

## Evidence For The Kernel-Integration Example

The revised sentence now names the specific interacting structure families and concrete kernel/runtime and hardware communication APIs on both sides of the Linux ICE and DPDK ICE driver boundaries.

### Linux networking core to Linux ICE driver

#### Kernel-owned networking structures

1. `struct net_device`

Source:

- Code: `linux-7.0.3/include/linux/netdevice.h:2109-2126`

Relevant facts:

- `struct net_device` contains `const struct net_device_ops *netdev_ops` and queue-related state such as `struct netdev_queue *_tx` and `real_num_tx_queues`.

Why it matters:

- This is the top-level networking object through which the Linux kernel invokes the driver.

2. `struct napi_struct`

Source:

- Code: `linux-7.0.3/include/linux/netdevice.h:381-404`

Relevant facts:

- `struct napi_struct` contains a `poll` callback, a backpointer `struct net_device *dev`, and a `struct sk_buff *skb` field.

Why it matters:

- This is the kernel polling object tied to driver interrupt and receive processing.

3. `struct sk_buff`

Source:

- Code: `linux-7.0.3/include/linux/skbuff.h:885-904`

Relevant facts:

- `struct sk_buff` carries a `struct net_device *dev` association.

Why it matters:

- This is the packet object exchanged across the kernel/driver networking boundary.

4. Linux networking entrypoints actually used by ICE

Source:

- Code: `linux-7.0.3/include/linux/netdevice.h:1088-1097`
- Code: `linux-7.0.3/include/linux/netdevice.h:2823-2835`
- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_main.c:9830-9849`

Relevant facts:

- The networking core defines `ndo_start_xmit(struct sk_buff *skb, struct net_device *dev)`.
- The networking core defines `netif_napi_add(struct net_device *dev, struct napi_struct *napi, ...)`.
- Linux ICE installs `.ndo_start_xmit = ice_start_xmit` in `ice_netdev_ops`.

Why it matters:

- These are the concrete invocation points connecting `net_device` and `sk_buff` to the ICE driver.

5. Linux kernel and hardware communication APIs actually used by ICE

Source:

- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_main.c:4663-4670`
- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_lib.c:2853-2857`
- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_txrx.c:1425-1433`
- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_txrx_lib.c:255-261`
- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_osdep.h:22-25`
- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_common.c:1890-1894`

Relevant facts:

- ICE registers its networking interface with `register_netdev(vsi->netdev)`.
- ICE attaches NAPI polling with `netif_napi_add_config(vsi->netdev, &vsi->q_vectors[v_idx]->napi, ice_napi_poll, v_idx)`.
- ICE maps transmit buffers for DMA with `dma_map_single(...)`.
- ICE hands received packets back to the kernel with `napi_gro_receive(&rx_ring->q_vector->napi, skb)`.
- ICE talks to hardware registers through `rd32`/`wr32`, which expand to `readl`/`writel` on `hw_addr`.
- ICE sends firmware Admin Queue commands through `ice_aq_send_cmd(...)`.

Why it matters:

- These APIs make the kernel/driver/hardware boundary explicit: Linux netdev and NAPI APIs are the host-side control surface, while DMA, MMIO, and Admin Queue commands are the NIC-facing control surface.

#### Linux ICE driver-owned structures

1. `struct ice_vsi`

Source:

- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice.h:333-340,399-415`

Relevant facts:

- `struct ice_vsi` contains `struct net_device *netdev`, arrays of `struct ice_rx_ring **rx_rings`, `struct ice_tx_ring **tx_rings`, and `struct ice_q_vector **q_vectors`.
- It also contains queue-count and queue-mapping state.

Why it matters:

- This is the central ICE driver object tying the kernel-visible netdev to driver-owned queue and interrupt state.

2. `struct ice_q_vector`

Source:

- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice.h:467-493`

Relevant facts:

- `struct ice_q_vector` contains `struct ice_vsi *vsi` and an embedded `struct napi_struct napi`.

Why it matters:

- This is the exact intersection point between Linux NAPI state and ICE queue-processing state.

3. `struct ice_rx_ring` and `struct ice_tx_ring`

Source:

- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_txrx.h:269-339`
- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_txrx.h:341-391`

Relevant facts:

- `struct ice_rx_ring` contains `struct net_device *netdev`, `struct ice_q_vector *q_vector`, and `struct ice_vsi *vsi`.
- `struct ice_tx_ring` contains `struct ice_q_vector *q_vector`, `struct net_device *netdev`, and `struct ice_vsi *vsi`.

Why it matters:

- These are the driver-owned queue structures that sit directly underneath the kernel-facing networking objects.

#### Linux core/ICE intersection points

1. ICE binds `net_device` to NAPI and q-vectors.

Source:

- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_lib.c:2850-2857`

Relevant facts:

- `netif_napi_add_config(vsi->netdev, &vsi->q_vectors[v_idx]->napi, ice_napi_poll, v_idx)` connects the kernel `net_device` and `napi_struct` to ICE `ice_q_vector` state.

2. ICE binds rings back to the owning `vsi` and `netdev`.

Source:

- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_lib.c:1421-1440`

Relevant facts:

- During Rx ring allocation ICE sets `ring->vsi = vsi` and `ring->netdev = vsi->netdev` before storing the ring in `vsi->rx_rings[i]`.

3. Receive path carries `sk_buff` through `ice_rx_ring` into NAPI.

Source:

- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_txrx_lib.c:255-262`

Relevant facts:

- `ice_receive_skb(struct ice_rx_ring *rx_ring, struct sk_buff *skb, ...)` ends with `napi_gro_receive(&rx_ring->q_vector->napi, skb)`.

4. Eswitch path explicitly uses `struct sk_buff`, `struct net_device`, and `struct ice_rx_ring` together.

Source:

- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_eswitch.h:26-31`
- Code: `linux-7.0.3/drivers/net/ethernet/intel/ice/ice_repr.c:267-283`

Relevant facts:

- `ice_eswitch_port_start_xmit(struct sk_buff *skb, struct net_device *netdev)` is the transmit entrypoint.
- `ice_eswitch_get_target(struct ice_rx_ring *rx_ring, ...)` maps receive-ring state back to a `net_device`.
- ICE representor `net_device_ops` uses `.ndo_start_xmit = ice_eswitch_port_start_xmit`.

Why it matters:

- This directly demonstrates the "kernel objects connected to driver-owned ICE state" wording in the overview sentence.

### DPDK ethdev to DPDK ICE PMD

#### DPDK framework-owned structures

1. `struct rte_eth_dev` and `struct rte_eth_dev_data`

Source:

- Code: `dpdk/lib/ethdev/ethdev_driver.h:91-100`
- Code: `dpdk/lib/ethdev/ethdev_driver.h:128-139`

Relevant facts:

- `struct rte_eth_dev` contains `struct rte_eth_dev_data *data`, `const struct eth_dev_ops *dev_ops`, and interrupt/device handles.
- `struct rte_eth_dev_data` contains `void **rx_queues`, `void **tx_queues`, `nb_rx_queues`, `nb_tx_queues`, and `dev_private`.

Why it matters:

- These are the framework-visible objects through which DPDK communicates queue, configuration, and device state to the PMD.

2. `struct eth_dev_ops`

Source:

- Code: `dpdk/lib/ethdev/ethdev_driver.h:1415-1434`

Relevant facts:

- `struct eth_dev_ops` exposes driver callbacks such as `dev_configure`, `dev_start`, `dev_stop`, and `link_update`.

Why it matters:

- This is the DPDK-side analogue of the kernel callback surface.

3. `struct rte_mempool`

Source:

- Code: `dpdk/drivers/net/intel/common/rx.h:39-47`
- Code: `dpdk/drivers/net/intel/ice/ice_rxtx.h:211-216`

Relevant facts:

- `struct ci_rx_queue` stores `struct rte_mempool *mp`.
- `ice_rx_queue_setup` takes `struct rte_mempool *mp` as an input.

Why it matters:

- This is the DPDK packet-buffer object that is directly wired into ICE receive queue state.

4. DPDK runtime and hardware communication APIs actually used by ICE

Source:

- Code: `dpdk/drivers/net/intel/ice/ice_ethdev.c:275-292`
- Code: `dpdk/drivers/net/intel/ice/ice_ethdev.c:2608-2614`
- Code: `dpdk/drivers/net/intel/ice/ice_ethdev.c:2735-2741`
- Code: `dpdk/drivers/net/intel/ice/ice_ethdev.c:2626-2633`
- Code: `dpdk/drivers/net/intel/ice/ice_rxtx.c:1376-1394`
- Code: `dpdk/drivers/net/intel/ice/ice_rxtx.c:462-475`
- Code: `dpdk/drivers/net/intel/ice/base/ice_osdep.h:96-105`
- Code: `dpdk/drivers/net/intel/ice/base/ice_common.c:2014-2018`

Relevant facts:

- ICE registers its DPDK callback surface in `ice_eth_dev_ops`, including `dev_start`, `rx_queue_setup`, and `tx_queue_setup`.
- ICE installs packet-path callbacks through `dev->rx_pkt_burst = ice_recv_pkts` and `dev->tx_pkt_burst = ice_xmit_pkts`.
- ICE registers and enables interrupts with `rte_intr_callback_register(...)` and `rte_intr_enable(...)`.
- ICE obtains BAR-mapped NIC access from `pci_dev->mem_resource[0].addr` via `hw->hw_addr`.
- ICE allocates DMA-visible descriptor rings with `rte_eth_dma_zone_reserve(...)`.
- ICE allocates receive buffers with `rte_mbuf_raw_alloc(...)` and converts them to DMA addresses with `rte_mbuf_data_iova_default(...)`.
- ICE performs MMIO through `rte_read32`/`rte_write32` wrapped by `readl`/`writel`.
- ICE sends firmware Admin Queue commands through `ice_aq_send_cmd(...)`.

Why it matters:

- These APIs show the DPDK analogue of the Linux integration problem: the driver must simultaneously satisfy host runtime callback and memory-management APIs and manipulate the device through PCI BAR MMIO and firmware command queues.

#### DPDK ICE driver-owned structures

1. `struct ice_pf` and `struct ice_vsi`

Source:

- Code: `dpdk/drivers/net/intel/ice/ice_ethdev.h:306-354`
- Code: `dpdk/drivers/net/intel/ice/ice_ethdev.h:564-612`

Relevant facts:

- `struct ice_pf` contains `struct ice_vsi *main_vsi` and `struct rte_eth_dev_data *dev_data`.
- `struct ice_vsi` contains queue-pair counts, base queue indices, and VSI identity/state.

Why it matters:

- These are the top-level PMD-owned structures named in the revised sentence.

2. `struct ci_rx_queue`

Source:

- Code: `dpdk/drivers/net/intel/common/rx.h:39-89`

Relevant facts:

- `struct ci_rx_queue` contains `struct rte_mempool *mp`, software/hardware ring pointers, queue identifiers, and a union field `struct ice_vsi *ice_vsi`.

Why it matters:

- This queue object explicitly links DPDK packet memory and queue arrays back to the ICE VSI.

3. `struct ci_tx_queue`

Source:

- Code: `dpdk/drivers/net/intel/common/tx.h:135-182`

Relevant facts:

- `struct ci_tx_queue` contains software/hardware Tx ring state, queue identifiers, fast-free mempool state, and a union field `struct ice_vsi *ice_vsi`.

Why it matters:

- This is the PMD-owned transmit queue object paired with DPDK queue arrays.

#### DPDK framework/ICE intersection points

1. Queue setup entrypoints connect `rte_eth_dev`, `rte_eth_dev_data`, `rte_mempool`, and ICE VSI state.

Source:

- Code: `dpdk/drivers/net/intel/ice/ice_rxtx.h:211-225`
- Code: `dpdk/drivers/net/intel/ice/ice_rxtx.c:1266-1429`
- Code: `dpdk/drivers/net/intel/ice/ice_rxtx.c:1487-1719`

Relevant facts:

- `ice_rx_queue_setup` takes `struct rte_eth_dev *dev` and `struct rte_mempool *mp`.
- It resolves `struct ice_pf *pf` from `dev->data->dev_private`, takes `struct ice_vsi *vsi = pf->main_vsi`, sets `rxq->mp = mp` or split-pool state, sets `rxq->ice_vsi = vsi`, and stores the queue in `dev->data->rx_queues[queue_idx]`.
- `ice_tx_queue_setup` similarly resolves `pf->main_vsi`, sets `txq->ice_vsi = vsi`, and stores the queue in `dev->data->tx_queues[queue_idx]`.

2. Queue start paths consume the queue objects from the DPDK queue arrays.

Source:

- Code: `dpdk/drivers/net/intel/ice/ice_rxtx.c:654-681`
- Code: `dpdk/drivers/net/intel/ice/ice_rxtx.c:786-807`

Relevant facts:

- `ice_rx_queue_start` retrieves `rxq = dev->data->rx_queues[rx_queue_id]`.
- `ice_tx_queue_start` retrieves `txq = dev->data->tx_queues[tx_queue_id]`.

3. Runtime datapath code follows the queue-to-VSI-to-adapter chain.

Source:

- Code: `dpdk/drivers/net/intel/ice/ice_rxtx.c:247-255`
- Code: `dpdk/drivers/net/intel/ice/ice_rxtx.c:752-823`
- Code: `dpdk/drivers/net/intel/ice/ice_rxtx_vec_avx2.c:41-43`

Relevant facts:

- The Rx path resolves `struct ice_vsi *vsi = rxq->ice_vsi` and `struct rte_eth_dev_data *dev_data = rxq->ice_vsi->adapter->pf.dev_data`.
- The Tx path similarly resolves `vsi = txq->ice_vsi`.
- Vectorized Rx code follows `rxq->ice_vsi->adapter`.

Why it matters:

- This is the concrete chain showing that DPDK framework objects are not just adjacent to PMD objects; they are linked and traversed together during operation.

### Why these are the right examples for the sentence

The sentence is about communication across a hierarchy of software objects, not just about callback names. The references above show exactly that hierarchy:

- Linux: `net_device` -> `ice_vsi` -> `ice_q_vector` -> `ice_rx_ring`/`ice_tx_ring`, with `sk_buff` and `napi_struct` moving across those boundaries.
- DPDK: `rte_eth_dev`/`rte_eth_dev_data` -> `ice_pf` -> `main_vsi` -> `ci_rx_queue`/`ci_tx_queue`, with `rte_mempool` and queue arrays tied into those objects.

### Termite rationale for this kind of example

Source:

- Markdown: `docs/knowledge/termite.md:84-110`
- Markdown: `docs/knowledge/termite.md:236-262`
- Markdown: `docs/knowledge/termite.md:357-357`
- PDF: `docs/knowledge/termite.pdf`
  - pp. 2-3 for the device-class / OS-interface discussion corresponding to `termite.md:84-110`
  - pp. 4-5 for the OS-interface state-machine discussion corresponding to `termite.md:236-262`

Relevant facts:

- Termite separates device and OS interfaces.
- The OS interface is "a collection of software entrypoints and callbacks."
- The OS interface is modeled as a state machine over valid driver invocations, callbacks, and device-class events.

Why it matters:

- This supports using framework-facing software objects and entrypoints as the kind of examples the placeholder needed.

## Evidence For The Device-Action Examples

The revised device-action sentence now uses one concrete, ordered E810 initialization example: firmware Admin Queue bring-up.

### TOC / navigation path used to locate the sequence

Source:

- Markdown: `docs/e810_datasheet.md:1359-1385`
- Markdown: `docs/e810_datasheet.md:1928-1950`

Relevant facts:

- `Chapter 4 Initialization` contains `4.4 Driver Load`.
- `Section 9.5 Control Queues` contains `9.5.2 Queue Structure`, `9.5.3 Initialization`, and `9.5.13 Generic Firmware Admin Commands`.

Why it matters:

- This is the methodical TOC-to-section path used to find the example.

### Recommended concrete ordered sequence: firmware Admin Queue bring-up

Placement in driver-load flow:

Source:

- Markdown: `docs/e810_datasheet.md:29114-29128`
- Datasheet section: `Section 4.4.1.1 Driver Load (non-Virtualized)`

Relevant facts:

- The major initialization steps include disabling interrupts, initializing the Admin Queue, initializing HMC, and initializing MAC/PHY.

Why it matters:

- This establishes AQ initialization as an explicit early bring-up phase in the driver-load narrative.

### Admin Queue / ARQ structure

1. The E810 uses Admin Queue pairs with an Admin Transmit Queue and an Admin Receive Queue.

Source:

- Markdown: `docs/e810_datasheet.md:3016-3040`
- Datasheet section: `Section 9.5`

Relevant facts:

- Admin Queue pairs map one ATQ and one ARQ.
- The host driver submits commands on the ATQ.
- The host driver posts empty buffers to the ARQ and the E810 fills them with events.

Why it matters:

- This directly supports the example "posting ARQ buffers."

2. The control-queue register names in the sentence are explicit datasheet objects.

Source:

- Markdown: `docs/e810_datasheet.md:103348-103381`
- Datasheet section: `Section 9.5.2 Queue Structure`, Table `9-25 Control Queue Registers`

Relevant facts:

- The datasheet names the CSRs `ATQBAH`, `ATQBAL`, `ATQLEN`, `ATQH`, `ATQT`, `ARQBAH`, `ARQBAL`, `ARQLEN`, `ARQH`, and `ARQT`.
- `ATQLEN` and `ARQLEN` use their MSB as the queue-enable bit.

Why it matters:

- This is the direct source for the exact register names used in the revised prose.

### Ordered initialization steps from `Section 9.5.3`

Source:

- Markdown: `docs/e810_datasheet.md:103405-103437`
- Datasheet section: `Section 9.5.3 Initialization`

Directly supported ordered steps:

1. Allocate host memory for the queues.
2. Post initialized buffers to the Receive Queue before using the Transmit Queue.
3. Clear the head and tail registers for each queue: `ATQH`, `ARQH`, `ATQT`, `ARQT`.
4. Program the base and length registers for each queue: `ATQBAL`, `ATQBAH`, `ATQLEN`, `ARQBAL`, `ARQBAH`, `ARQLEN`.
5. Set `length.enable` to `1`.
6. For a firmware Admin Queue, issue `Get Version (0x0001)` before using the queue for anything else.
7. Wait for the `Get Version` reply and check that the returned firmware/AQ API major version is compatible.
8. Only after successful version verification, send `Driver Version (0x0002)`.

Why it matters:

- This is the exact ordered command/CSR sequence summarized by the current `overview.tex` example.

2. The derived E810 spec in this repo makes that action explicit.

Source:

- `docs/E810_SPEC.md:337-353`
- `docs/E810_SPEC.md:378-389`

Relevant facts:

- Device messages include `postArqBuffer(buf_addr, buf_len)`.
- Control-queue initialization requires posting initialized receive buffers to ARQ before using ATQ.

Why it matters:

- This is a concise repository-local summary of the datasheet flow in exactly the state-machine style used by the paper draft.

### `Get Version (0x0001)` readiness gating

1. `Get Version (0x0001)` is explicitly the first-command gate for further AQ use.

Source:

- Markdown: `docs/e810_datasheet.md:104194-104200`
- Datasheet section: `Section 9.5.13.1 Get Version (0x0001)`

Relevant facts:

- `Get Version` must be the first command that the driver issues before using the queue for other purposes.
- The driver must inspect the reply and must not continue loading if the major version mismatches.

Why it matters:

- This is the direct source for the "only after a compatible reply" wording in `overview.tex`.

2. The datasheet states that firmware may not respond to `Get Version` until NVM load reaches the required point.

Source:

- Markdown: `docs/e810_datasheet.md:26909-26913`
- Datasheet section: reset/initialization flow immediately preceding `Section 9.5.3`; see also `docs/E810_SPEC.md:548-553`

Relevant facts:

- After NVM is loaded into hardware, the EMP might start responding to the `Get Version` Admin command.
- The command is blocked until that stage.
- Software should wait for a `Get Version` response or poll done bits.

Why it matters:

- This makes `Get Version` a strong example of a concrete device action with ordering constraints.

3. The repository’s derived E810 spec records this as the gating point for AQ use.

Source:

- `docs/E810_SPEC.md:363-365`
- `docs/E810_SPEC.md:385-389`

Relevant facts:

- AQ readiness plus a successful `Get Version` response are the proof points that driver initialization can start.
- `Get Version (0x0001)` must be issued before using the firmware AQ for anything else.

### `Driver Version (0x0002)`

Source:

- Markdown: `docs/e810_datasheet.md:104223-104229`
- Datasheet section: `Section 9.5.13.2 Driver Version (0x0002)`
- Summary corroboration: `docs/E810_SPEC.md:305-306`, `docs/E810_SPEC.md:388-389`

Relevant facts:

- The command is used by the driver to report the driver version.
- In the initialization sequence it is sent after successful version verification.

Note:

- The current overview sentence relies directly on `Section 9.5.3` and `Section 9.5.13.2`; `docs/E810_SPEC.md` is corroborating material only.

### Alternate concrete sequence considered but not chosen

Alternate candidate:

- Link bring-up via `Set PHY Config (0x0601)`, `Set MAC Config (0x0603)`, `Setup Link and Restart Auto-Negotiation (0x0605)`, and `Get Link Status (0x0607)`

Source:

- Markdown: `docs/e810_datasheet.md:11724-11730`
- Markdown: `docs/e810_datasheet.md:11764-11778`
- Markdown: `docs/e810_datasheet.md:11809-11820`
- Markdown: `docs/e810_datasheet.md:12132-12178`
- Markdown: `docs/e810_datasheet.md:12289-12293`
- Markdown: `docs/e810_datasheet.md:12931-12959`
- Datasheet sections: `3.2.4`, `3.2.4.1.1`, `3.2.4.1.2`, `3.2.4.1.3`, `3.2.4.1.5`

Why it was not chosen:

- It is a good second example, but the AQ bring-up sequence is cleaner and more foundational for illustrating exact ordered driver-to-hardware actions.

### Additional sharp supporting example: receive queue enable flow

Source:

- Markdown: `docs/e810_datasheet.md:110389-110463`
- Datasheet section: `Section 10.4.3.1.1 Receive Queue Enable Flow`

Relevant facts:

- The driver initializes the receive ring, programs queue context, clears and sets `QRX_TAIL[n]`, sets `QRX_CTRL[n].QENA_REQ`, and only after hardware sets `QRX_CTRL[n].QENA_STAT` may software start using the queue.

Why it matters:

- This is a second, sharper example of the kind of exact ordered hardware interaction sequence the paragraph is referring to, even though it is not the one used in the final overview sentence.

### `Get Link Status (0x0607)` and LSE

1. `Get Link Status (0x0607)` is explicitly documented as a driver command.

Source:

- Markdown: `docs/e810_datasheet.md:12931-12959`
- Datasheet section: `Section 3.2.4.1.5 Get Link Status (0x0607)`

Relevant facts:

- `Get Link Status (0x0607)` is used by the device driver to determine the current link status.
- In the E810 it is an indirect Admin Queue command.
- Command flags enable or disable LSE notification.

2. The datasheet says normal software should use AQ link-status mechanisms rather than hardware link interrupts.

Source:

- Markdown: `docs/e810_datasheet.md:13421-13425`
- Datasheet section: `Section 3.2.4.1.6 Link Status Event (LSE)`

Relevant facts:

- LSE is disabled by default.
- Software enables it through `Get Link Status`.
- Firmware disables LSE again after generating an event.
- Hardware link-status interrupts should be disabled for normal operation.
- Software should use AQ `Get Link Status` and LSE.

3. The DPDK ICE PMD mirrors this design choice.

Source:

- `dpdk/drivers/net/intel/ice/ice_ethdev.c:1392-1424`

Relevant facts:

- `ice_pf_enable_irq0` enables interrupt causes except link-state-change.
- The comment states that link-state change is delivered as an unsolicited AdminQ get-link-status notification.

Why it matters:

- This is a strong modern artifact showing that the datasheet mechanism is not just theoretical; the current ICE PMD actually follows it.

### RX queue enable via `QRX_CTRL[n].QENA_REQ`

1. The datasheet gives a concrete enable sequence using `QRX_TAIL[n]`, `QRX_CTRL[n].QENA_REQ`, and `QENA_STAT`.

Source:

- Markdown: `docs/e810_datasheet.md:110424-110461`
- Datasheet section: `Section 10.4.3.1.1`

Relevant facts:

- Program the receive queue context.
- Clear and then set `QRX_TAIL[n]` to the end of the descriptor ring.
- Set `QENA_REQ` in `QRX_CTRL[n]`.
- Hardware then sets `QENA_STAT`.
- Software may start using the queue once `QENA_STAT` is set.

2. The repository’s derived E810 spec captures the same flow.

Source:

- `docs/E810_SPEC.md:402-415`

Relevant facts:

- The modeled receive queue enable flow uses `QRX_TAIL[n]`, `QRX_CTRL[n].QENA_REQ`, and `QRX_CTRL[n].QENA_STAT`.
- The `QENA_STAT` transition is treated as the queue-enable completion event.

3. The ICE PMD contains the corresponding low-level register logic.

Source:

- `dpdk/drivers/net/intel/ice/ice_rxtx.c:548-576`

Relevant facts:

- The driver reads `QRX_CTRL(q_idx)`.
- It sets or clears `QRX_CTRL_QENA_REQ_M`.
- It checks for the matching `QRX_CTRL_QENA_STAT_M` state transition.

Why it matters:

- This is exactly the kind of fine-grained "action" and dependency the overview paragraph is trying to describe.

### "Configure a network send queue"

This phrase was already present in `overview.tex`. I left it unchanged and supplemented it with more concrete E810 actions.

Supporting evidence:

- `docs/E810_SPEC.md:416-426`
- `docs/e810_datasheet.md:97241-97247`
- `docs/e810_datasheet.md:97099-97128`
- Datasheet section: `Section 8.3.4.3.6 Admin Queue Commands` and transmit-queue handling material summarized in `docs/E810_SPEC.md:554-555`

Relevant facts:

- `Add Tx LAN Queues (0x0C30)` is an indirect AQ command.
- Firmware configures Tx queues and scheduler leaf-node associations.
- TEIDs are used to identify scheduling elements.

Why it matters:

- This supports the existing wording about send-queue configuration, even though the specific placeholder replacement focused on even more basic and clearly grounded actions.

## Verus Context Relevant To The Overview

Verus was not directly inserted into the two placeholder replacements, but it is part of the explanatory context around the methods section and helps justify the state-machine and transition language in the surrounding prose.

Source:

- `docs/knowledge/verus-sys.md:743-758`
- `docs/knowledge/verus-sys.md:999-1003`
- `docs/knowledge/verus-sys.md:1649-1649`

Relevant facts:

- VerusSync makes transitions the central object for reasoning.
- Developers specify transitions with enabling conditions and state updates.
- Ghost shards are connected to physical atomic memory cells via invariants.
- The paper characterizes VerusSync as a state-transition-based language for ghost state embedded in code via Rust ownership types.

Why it matters:

- This matches the surrounding `overview.tex` language about encoding fine-grained hardware states and transitions as ghost state in Verus.

## Termite Guidance Relevant To The Overview

Source:

- Markdown: `docs/knowledge/termite.md:84-110`
- Markdown: `docs/knowledge/termite.md:151-153`
- Markdown: `docs/knowledge/termite.md:236-262`
- Markdown: `docs/knowledge/termite.md:357-357`
- PDF: `docs/knowledge/termite.pdf`
  - pp. 2-3 for the device-class / OS-interface discussion corresponding to `termite.md:84-110`
  - pp. 3-4 for device-specification examples corresponding to `termite.md:151-153`
  - pp. 4-5 for the OS-interface state-machine discussion corresponding to `termite.md:236-262`

Relevant facts:

- Termite separates device interface and OS interface.
- The OS interface is a collection of software entrypoints and callbacks.
- Device specifications describe how register operations and device state produce higher-level device-class events.
- The OS interface is modeled as a state machine whose transitions correspond to OS invocations, driver callbacks, and device-class events.

Why it matters:

- This is the conceptual template for the kind of examples needed in both active placeholders.

## Caveats

1. Datasheet references in this file use Markdown line numbers plus section numbers.

- The Markdown conversion is the easiest way to cross-check exact content in this repository.
- Where PDF page numbers were straightforward, they are included; otherwise section numbers are given as the authoritative locator.

2. The inserted hardware actions were chosen to be specific and recent.

- They are more current than the older RTL8139-style examples in the Termite paper and align with the E810/ICE artifacts present in this repository.

## Additional Sourceable Gaps Fixed Outside `overview.tex` Action Examples

### Funding totals in `summary.tex`

Updated text:

- `amazon26/summary.tex:20` -> `\$79,964`
- `amazon26/summary.tex:21` -> `\$0`

Source:

- `amazon26/budget.tex:12`
- `amazon26/budget.tex:29-31`

Relevant facts:

- The detailed budget requests a total of `\$79,964`.
- The budget explicitly states that promotional credits are not requested because evaluation is planned on CloudLab.

### Hardware-spec scale in `overview.tex`

Updated text:

- `amazon26/overview.tex:169-170` now states that the Intel E810 datasheet is `2,750 pages`.

Source:

- PDF metadata: `docs/e810_datasheet.pdf`
- Verified with `pdfinfo`, which reports `Pages: 2750`

Relevant facts:

- This supports replacing the vague "1000 of pages" phrasing with a source-backed number.

### Linux ICE driver size example in `overview.tex`

Updated text:

- `amazon26/overview.tex:172-173` now states that Linux ICE in `linux-7.0.3` is `over 125,000 lines of code`.

Source:

- Source tree: `linux-7.0.3/drivers/net/ethernet/intel/ice/*`
- Source tree: `linux-7.0.3/drivers/net/ethernet/intel/ice/virt/*`
- Source tree: `linux-7.0.3/drivers/net/ethernet/intel/ice/devlink/*`

Relevant facts:

- A line count over those files reports `125398 total`.

Why it matters:

- This supports replacing `XX,XXX` with a concrete, repository-derived magnitude claim.
