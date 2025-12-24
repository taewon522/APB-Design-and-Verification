# APB Master (RTL) — Design Spec & Verification Spec

<p align="center">
  <img alt="APB" src="https://img.shields.io/badge/bus-APB-blue">
  <img alt="RTL" src="https://img.shields.io/badge/language-Verilog-orange">
  <img alt="FSM" src="https://img.shields.io/badge/FSM-IDLE%2FSETUP%2FACCESS-green">
</p>

> **What this is**  
> A simple APB-like master that converts an internal request interface (`transfer/write/addr/wdata`) into APB bus signals and selects **one of 5 slaves** via address decoding.

---

## Table of Contents
- [1. Overview](#1-overview)
- [2. Interfaces](#2-interfaces)
- [3. Address Map & Decode Rules](#3-address-map--decode-rules)
- [4. FSM & Protocol Behavior](#4-fsm--protocol-behavior)
- [5. Key Assumptions / Limitations](#5-key-assumptions--limitations)
- [6. Verification Spec](#6-verification-spec)
  - [6.1 Requirements (Shall)](#61-requirements-shall)
  - [6.2 Assertions (SVA) Checklist](#62-assertions-sva-checklist)
  - [6.3 Functional Coverage](#63-functional-coverage)
  - [6.4 Test Plan](#64-test-plan)
- [7. RTL](https://github.com/taewon522/APB-Design-and-Verification/tree/main/rtl)
- [8. DOC](#8-참고문서)

---

## 1. Overview

**Module name:** `APB_Master`  
**Purpose:**  
- Accept a request on the internal interface
- Latch request fields (`addr/write/wdata`)
- Perform an APB-style **SETUP → ACCESS** transaction
- Select one of **5 slaves** using address decoding
- Return `ready` and (for reads) `rdata` through a mux

**Slaves supported:** `SEL0 .. SEL4`  
**States:** `IDLE`, `SETUP`, `ACCESS`  
**Completion condition:** `ready == 1` (muxed from selected slave `pReadyX`)

---

## 2. Interfaces

### 2.1 Global
- `pclk` : clock
- `pReset` : synchronous active-high reset (sampled on `posedge pclk`)

### 2.2 Internal Request Interface
| Signal | Dir | Width | Description |
|---|---:|---:|---|
| `transfer` | in | 1 | Request valid (sampled in IDLE only) |
| `write` | in | 1 | 1=write, 0=read |
| `addr` | in | 32 | Address |
| `wdata` | in | 32 | Write data |
| `ready` | out | 1 | Transaction completion (muxed from slave ready) |
| `rdata` | out | 32 | Read data (muxed from slave rdata) |

**Request acceptance rule:**  
- The request is accepted **only in `IDLE`** when `transfer==1`.  
- On acceptance, `addr/write/wdata` are **latched** into internal registers.

### 2.3 APB-side Interface
| Signal | Dir | Width | Description |
|---|---:|---:|---|
| `pSel0..pSel4` | out | 1 each | One-hot slave select (from decoder) |
| `pEnable` | out | 1 | APB enable phase |
| `pWrite` | out | 1 | 1=write, 0=read |
| `pAddr` | out | 32 | Address (latched) |
| `pWdata` | out | 32 | Write data (latched) |
| `pRdata0..4` | in | 32 | Slave read data inputs |
| `pReady0..4` | in | 1 | Slave ready inputs |

---

## 3. Address Map & Decode Rules

Address decode uses `temp_addr_reg` (latched `addr`) while `decoder_en==1`.

### 3.1 Slave selection
| Address Pattern | Selected Slave | `pselx` | `mux_sel` |
|---|---:|---|---:|
| `32'h1000_0xxx` | 0 | `5'b00001` | `3'd0` |
| `32'h1000_1xxx` | 1 | `5'b00010` | `3'd1` |
| `32'h1000_2xxx` | 2 | `5'b00100` | `3'd2` |
| `32'h1000_3xxx` | 3 | `5'b01000` | `3'd3` |
| `32'h1000_4xxx` | 4 | `5'b10000` | `3'd4` |

### 3.2 Decode enable
- `decoder_en=1` in **SETUP** and **ACCESS**
- `decoder_en=0` in **IDLE**

### 3.3 Mux behavior
- `ready` is selected from `pReady[mux_sel]`
- `rdata` is selected from `pRdata[mux_sel]`

---

## 4. FSM & Protocol Behavior

### 4.1 States
- **IDLE**: waiting for `transfer`
- **SETUP**: select slave (`pSelX=1`) and hold `pEnable=0`
- **ACCESS**: assert `pEnable=1` and wait for `ready==1`

### 4.2 Reset behavior
On `pReset==1` (posedge):
- `state <= IDLE`
- latched request regs cleared to 0

### 4.3 State transitions
- `IDLE → SETUP` : if `transfer==1` (request accepted & latched)
- `SETUP → ACCESS` : always next cycle (single-cycle setup)
- `ACCESS → IDLE` : if `ready==1`

### 4.4 Output behavior per state (high level)
| State | `decoder_en` | `pEnable` | `pSelX` | Notes |
|---|---:|---:|---:|---|
| IDLE | 0 | 0 | 0 | Request accepted only here |
| SETUP | 1 | 0 | one-hot | Address/write/data already latched |
| ACCESS | 1 | 1 | one-hot | Wait-state supported via `ready` |

---

## 5. Key Assumptions / Limitations
- `transfer` is meaningful **only in IDLE**; changes in SETUP/ACCESS are ignored.
- Address is expected to be within the defined decode ranges (`1000_0xxx` ~ `1000_4xxx`) for a valid slave select.
- Slave is responsible for asserting its `pReadyX` and providing `pRdataX` when selected.

---

## 6. Verification Spec

### 6.1 Requirements (Shall)

| Req ID | Requirement (Shall) | Suggested Check Method |
|---|---|---|
| FR-001 | On `pReset`, DUT shall enter `IDLE` and clear latched fields. | Directed test + SVA |
| FR-010 | In `IDLE`, if `transfer==1`, DUT shall latch `addr/write/wdata` and transition to `SETUP`. | Scoreboard + SVA |
| FR-020 | In `SETUP`, DUT shall keep `pEnable==0`. | SVA |
| FR-021 | In `ACCESS`, DUT shall assert `pEnable==1`. | SVA |
| FR-022 | `SETUP` shall be exactly 1 cycle and transition to `ACCESS` next cycle. | SVA |
| FR-030 | For each address region, DUT shall assert exactly one `pSelX` (one-hot) when `decoder_en==1`. | SVA + Coverage |
| FR-040 | In `ACCESS`, if `ready==0`, DUT shall remain in `ACCESS` (wait-state handling). | Random ready delay + SVA |
| FR-041 | In `ACCESS`, when `ready==1`, DUT shall transition to `IDLE`. | SVA |
| FR-050 | For reads, on completion (`ready==1`), `rdata` shall match selected slave `pRdataX`. | Scoreboard |
| FR-060 | For writes, DUT shall drive `pWrite==1` and `pWdata==latched wdata` during SETUP/ACCESS. | SVA + Scoreboard |
| FR-070 | DUT shall support back-to-back requests (multiple transfers), completing each independently. | Random sequence + Coverage |
| FR-080 | `transfer` changes during SETUP/ACCESS shall not affect the current transaction. | Directed glitch test + SVA |

---

### 6.2 Assertions (SVA) Checklist
Recommended assertions to implement in the verification environment:

- **A1**: `SETUP` lasts 1 cycle then goes to `ACCESS`
- **A2**: `pEnable==0` in `SETUP`, `pEnable==1` in `ACCESS`
- **A3**: When `decoder_en==1`, `pSel` is `onehot0` (`0 or exactly 1 bit set`)
- **A4**: During wait-states (`ACCESS && !ready`), `pAddr/pWrite/pWdata/pSel` remain stable
- **A5**: `ready==1` in `ACCESS` causes next state to be `IDLE`

> Tip: `onehot0` can be checked using `$onehot0({pSel4,pSel3,pSel2,pSel1,pSel0})`.

---

### 6.3 Functional Coverage
Minimum coverage points:

- **C1**: Slave select hit: SEL0~SEL4
- **C2**: Read/Write cross coverage per slave
- **C3**: Wait-state length bins: 0-cycle, 1~N cycles
- **C4**: Back-to-back length bins: 1,2,3,4+ consecutive transfers

---

### 6.4 Test Plan
Suggested tests:

- **T1**: Reset sanity
- **T2**: Basic read/write per slave (no wait-state)
- **T3**: Random wait-state read/write per slave
- **T4**: Address sweep over all regions
- **T5**: Random back-to-back transfers (stress)
- **T6**: `transfer` asserted during SETUP/ACCESS (must be ignored)
- **T7**: Illegal address behavior (policy-based: assertion fail / no select)

---
### 8 참고문서

- [ABP SPEC PDF 열기](https://raw.githubusercontent.com/taewon522/APB-Design-and-Verification/main/doc/IHI0024E_amba_apb_architecture_spec.pdf)
- [ABP SPEC PDF 다운로드](https://raw.githubusercontent.com/taewon522/APB-Design-and-Verification/main/doc/IHI0024E_amba_apb_architecture_spec.pdf)
