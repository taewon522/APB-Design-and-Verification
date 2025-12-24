# APB Master (RTL) — Design Spec & Verification Spec

<p align="center">
  <img alt="APB" src="https://img.shields.io/badge/bus-APB-blue">
  <img alt="RTL" src="https://img.shields.io/badge/language-Verilog-orange">
  <img alt="FSM" src="https://img.shields.io/badge/FSM-IDLE%2FSETUP%2FACCESS-green">
</p>

> **What this is**  
> 내부 request interface(transfer/write/addr/wdata)를 입력으로 받아 APB SETUP/ACCESS 트랜잭션을 생성하고, address decoding을 통해 5개 slave 중 1개를 선택하는 APB Master + Decoder/Mux 구조입니다.

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
- [7. RTL](#7-rtl-code)
- [8. DOC](#8-doc)

---

## 1. Overview

**Module name:** `APB_Master`  
**목적(Purpose):**
- internal request를 입력으로 받아 transaction 시작
- `addr/write/wdata`를 내부 register에 **latch**
- APB style **SETUP → ACCESS** 프로토콜로 변환
- address decoding으로 **5개 slave 중 1개 선택**
- `ready`와 (read인 경우) `rdata`를 mux를 통해 반환

**Slaves supported:** `SEL0 .. SEL4`  
**FSM states:** `IDLE`, `SETUP`, `ACCESS`  
**Transaction 완료 조건:** `ready == 1` (선택된 slave의 `pReadyX`가 mux되어 들어옴)

---

## 2. Interfaces

### 2.1 Global
- `pclk` : clock
- `pReset` : synchronous active-high reset (`posedge pclk`에서 샘플링)

### 2.2 Internal Request Interface
| Signal | Dir | Width | Description |
|---|---:|---:|---|
| `transfer` | in | 1 | Request valid (IDLE에서만 샘플링) |
| `write` | in | 1 | 1=write, 0=read |
| `addr` | in | 32 | Address |
| `wdata` | in | 32 | Write data |
| `ready` | out | 1 | Transaction 완료 신호(선택된 slave ready를 mux) |
| `rdata` | out | 32 | Read data(선택된 slave rdata를 mux) |

**Request acceptance rule:**
- request는 **IDLE 상태에서만** `transfer==1`일 때 accept 됨
- accept 시점에 `addr/write/wdata`를 내부 register에 **latch**

### 2.3 APB-side Interface
| Signal | Dir | Width | Description |
|---|---:|---:|---|
| `pSel0..pSel4` | out | 1 each | one-hot slave select (decoder output) |
| `pEnable` | out | 1 | APB enable phase |
| `pWrite` | out | 1 | 1=write, 0=read |
| `pAddr` | out | 32 | Address (latched) |
| `pWdata` | out | 32 | Write data (latched) |
| `pRdata0..4` | in | 32 | Slave read data |
| `pReady0..4` | in | 1 | Slave ready |

---

## 3. Address Map & Decode Rules

Address decode는 `decoder_en==1`일 때, latched된 `temp_addr_reg` 기준으로 동작합니다.

### 3.1 Slave selection
| Address Pattern | Selected Slave | `pselx` | `mux_sel` |
|---|---:|---|---:|
| `32'h1000_0xxx` | 0 | `5'b00001` | `3'd0` |
| `32'h1000_1xxx` | 1 | `5'b00010` | `3'd1` |
| `32'h1000_2xxx` | 2 | `5'b00100` | `3'd2` |
| `32'h1000_3xxx` | 3 | `5'b01000` | `3'd3` |
| `32'h1000_4xxx` | 4 | `5'b10000` | `3'd4` |

### 3.2 Decode enable
- `decoder_en=1` : `SETUP`, `ACCESS`
- `decoder_en=0` : `IDLE`

### 3.3 Mux behavior
- `ready` = `pReady[mux_sel]`
- `rdata` = `pRdata[mux_sel]`

---

## 4. FSM & Protocol Behavior

### 4.1 States
- **IDLE**: `transfer` 대기
- **SETUP**: `pSelX=1`, `pEnable=0` (setup phase)
- **ACCESS**: `pEnable=1` 후 `ready`를 기다림 (wait-state 지원)

### 4.2 Reset behavior
`pReset==1`이면(posedge):
- `state <= IDLE`
- latched request register들 0으로 초기화

### 4.3 State transitions
- `IDLE → SETUP` : `transfer==1` (request latch)
- `SETUP → ACCESS` : 다음 cycle에 항상 진입 (setup은 1-cycle)
- `ACCESS → IDLE` : `ready==1`이면 종료

### 4.4 Output behavior per state (high level)
| State | `decoder_en` | `pEnable` | `pSelX` | Notes |
|---|---:|---:|---:|---|
| IDLE | 0 | 0 | 0 | request는 여기서만 accept |
| SETUP | 1 | 0 | one-hot | addr/write/wdata는 이미 latch |
| ACCESS | 1 | 1 | one-hot | `ready`가 1 될 때까지 유지 |

---

## 5. Key Assumptions / Limitations
- `transfer`는 **IDLE에서만 의미 있음** (SETUP/ACCESS에서 변화해도 무시)
- address는 decode 범위(`1000_0xxx` ~ `1000_4xxx`) 안에 있다고 가정
- slave는 선택되었을 때 `pReadyX`와 `pRdataX`를 적절히 제공해야 함

---

## 6. Verification Spec

### 6.1 Requirements (Shall)

| Req ID | Requirement (Shall) | Suggested Check Method |
|---|---|---|
| FR-001 | `pReset` 시 DUT는 `IDLE`로 진입하고 latch 값들을 초기화해야 함. | Directed test + SVA |
| FR-010 | `IDLE`에서 `transfer==1`이면 `addr/write/wdata`를 latch하고 `SETUP`으로 전이해야 함. | Scoreboard + SVA |
| FR-020 | `SETUP`에서 `pEnable==0`을 유지해야 함. | SVA |
| FR-021 | `ACCESS`에서 `pEnable==1`을 assert 해야 함. | SVA |
| FR-022 | `SETUP`은 정확히 1 cycle 이어야 하며 다음 cycle에 `ACCESS`로 전이해야 함. | SVA |
| FR-030 | decode 영역 별로 `decoder_en==1`일 때 `pSelX`는 one-hot이어야 함. | SVA + Coverage |
| FR-040 | `ACCESS && !ready`이면 `ACCESS`에 머물러 wait-state를 처리해야 함. | Random ready delay + SVA |
| FR-041 | `ACCESS && ready`이면 다음 상태는 `IDLE`이어야 함. | SVA |
| FR-050 | read 완료 시점(`ready==1`)에 `rdata`는 선택된 slave의 `pRdataX`와 같아야 함. | Scoreboard |
| FR-060 | write인 경우 `pWrite==1` 및 `pWdata==latched wdata`를 SETUP/ACCESS 동안 유지해야 함. | SVA + Scoreboard |
| FR-070 | back-to-back request를 지원해야 함. | Random sequence + Coverage |
| FR-080 | SETUP/ACCESS 중 `transfer` 변화는 현재 transaction에 영향을 주면 안 됨. | Directed glitch test + SVA |

---

### 6.2 Assertions (SVA) Checklist
추천 assertion 목록:

- **A1**: `SETUP`은 1 cycle 후 `ACCESS`로 전이
- **A2**: `pEnable==0` in `SETUP`, `pEnable==1` in `ACCESS`
- **A3**: `decoder_en==1`이면 `pSel`은 `$onehot0` 만족
- **A4**: wait-state(`ACCESS && !ready`) 동안 `pAddr/pWrite/pWdata/pSel` 안정(stable)
- **A5**: `ACCESS && ready`이면 next state는 `IDLE`

> Tip: `$onehot0({pSel4,pSel3,pSel2,pSel1,pSel0})`

---

### 6.3 Functional Coverage
최소 coverage 포인트:

- **C1**: slave select hit: SEL0~SEL4
- **C2**: Read/Write × slave cross coverage
- **C3**: wait-state 길이 binning: 0-cycle, 1~N cycles
- **C4**: back-to-back 길이 binning: 1,2,3,4+ consecutive transfers

---

### 6.4 Test Plan
추천 test 항목:

- **T1**: reset sanity
- **T2**: slave 별 기본 read/write (no wait-state)
- **T3**: random wait-state read/write
- **T4**: address sweep (모든 decode 영역)
- **T5**: random back-to-back transfers (stress)
- **T6**: SETUP/ACCESS 중 `transfer` assert (무시되어야 함)
- **T7**: illegal address behavior (정책에 따라: assert fail / no select)

---
## 7 RTL CODE

<details>
  <summary><b>Click to expand RTL (APB_Master / APB_Decoder / APB_Mux)</b></summary>

```verilog
// ==============================
// APB_Master
// ==============================
`timescale 1ns / 1ps

module APB_Master (
    // global signals
    input              pclk,
    input              pReset,
    // APB Interface Signals
    output reg  [31:0] pAddr,
    output reg         pWrite,
    output reg         pEnable,
    output reg  [31:0] pWdata,
    output wire        pSel0,
    output wire        pSel1,
    output wire        pSel2,
    output wire        pSel3,
    output wire        pSel4,
    input       [31:0] pRdata0,
    input       [31:0] pRdata1,
    input       [31:0] pRdata2,
    input       [31:0] pRdata3,
    input       [31:0] pRdata4,
    input              pReady0,
    input              pReady1,
    input              pReady2,
    input              pReady3,
    input              pReady4,
    // Internal Interface Signals
    input              transfer,
    output wire        ready,
    input              write,
    input       [31:0] addr,
    input       [31:0] wdata,
    output wire [31:0] rdata
);
    reg [4:0] pselx;
    reg [2:0] mux_sel;
    reg decoder_en;
    reg [31:0] temp_addr_reg, temp_addr_next, temp_wdata_reg, temp_wdata_next;
    reg temp_write_reg, temp_write_next;

    assign pSel0 = pselx[0];
    assign pSel1 = pselx[1];
    assign pSel2 = pselx[2];
    assign pSel3 = pselx[3];
    assign pSel4 = pselx[4];

    localparam [1:0] IDLE   = 2'b00;
    localparam [1:0] SETUP  = 2'b01;
    localparam [1:0] ACCESS = 2'b10;

    reg [1:0] state, state_next;

    always @(posedge pclk) begin
        if (pReset) begin
            state          <= IDLE;
            temp_addr_reg  <= 0;
            temp_wdata_reg <= 0;
            temp_write_reg <= 0;
        end else begin
            state          <= state_next;
            temp_addr_reg  <= temp_addr_next;
            temp_wdata_reg <= temp_wdata_next;
            temp_write_reg <= temp_write_next;
        end
    end

    always @(*) begin
        state_next      = state;
        decoder_en      = 1'b0;
        pEnable         = 1'b0;
        temp_addr_next  = temp_addr_reg;
        temp_wdata_next = temp_wdata_reg;
        temp_write_next = temp_write_reg;
        pAddr           = temp_addr_reg;
        pWrite          = temp_write_reg;
        pWdata          = temp_wdata_reg;

        case (state)
            IDLE: begin
                decoder_en = 1'b0;
                if (transfer) begin
                    state_next      = SETUP;
                    temp_addr_next  = addr;   // latching
                    temp_wdata_next = wdata;
                    temp_write_next = write;
                end
            end

            SETUP: begin
                decoder_en = 1'b1;
                pEnable    = 1'b0;
                pAddr      = temp_addr_reg;
                pWrite     = temp_write_reg;
                state_next = ACCESS;
                if (temp_write_reg) begin
                    pWdata = temp_wdata_reg;
                end
            end

            ACCESS: begin
                decoder_en = 1'b1;
                pEnable    = 1'b1;
                if (ready) begin
                    state_next = IDLE;
                end
            end
        endcase
    end

    APB_Decoder U_APB_Decoder (
        .en     (decoder_en),
        .sel    (temp_addr_reg),
        .y      (pselx),
        .mux_sel(mux_sel)
    );

    APB_Mux U_APB_Mux (
        .sel   (mux_sel),
        .rdata0(pRdata0),
        .rdata1(pRdata1),
        .rdata2(pRdata2),
        .rdata3(pRdata3),
        .rdata4(pRdata4),
        .ready0(pReady0),
        .ready1(pReady1),
        .ready2(pReady2),
        .ready3(pReady3),
        .ready4(pReady4),
        .rdata (rdata),
        .ready (ready)
    );

endmodule


// ==============================
// APB_Decoder
// ==============================
module APB_Decoder (
    input             en,
    input      [31:0] sel,
    output reg [ 4:0] y,
    output reg [ 2:0] mux_sel
);
    always @(*) begin
        y = 5'b00000;
        if (en) begin
            casex (sel)
                32'h1000_0xxx: y = 5'b00001;
                32'h1000_1xxx: y = 5'b00010;
                32'h1000_2xxx: y = 5'b00100;
                32'h1000_3xxx: y = 5'b01000;
                32'h1000_4xxx: y = 5'b10000;
            endcase
        end
    end

    always @(*) begin
        mux_sel = 3'd0; // avoid X in synthesizable default values
        if (en) begin
            casex (sel)
                32'h1000_0xxx: mux_sel = 3'd0;
                32'h1000_1xxx: mux_sel = 3'd1;
                32'h1000_2xxx: mux_sel = 3'd2;
                32'h1000_3xxx: mux_sel = 3'd3;
                32'h1000_4xxx: mux_sel = 3'd4;
            endcase
        end
    end
endmodule


// ==============================
// APB_Mux
// ==============================
module APB_Mux (
    input      [ 2:0] sel,
    input      [31:0] rdata0,
    input      [31:0] rdata1,
    input      [31:0] rdata2,
    input      [31:0] rdata3,
    input      [31:0] rdata4,
    input             ready0,
    input             ready1,
    input             ready2,
    input             ready3,
    input             ready4,
    output reg [31:0] rdata,
    output reg        ready
);
    always @(*) begin
        rdata = 32'b0;
        case (sel)
            3'd0: rdata = rdata0;
            3'd1: rdata = rdata1;
            3'd2: rdata = rdata2;
            3'd3: rdata = rdata3;
            3'd4: rdata = rdata4;
        endcase
    end

    always @(*) begin
        ready = 1'b0;
        case (sel)
            3'd0: ready = ready0;
            3'd1: ready = ready1;
            3'd2: ready = ready2;
            3'd3: ready = ready3;
            3'd4: ready = ready4;
        endcase
    end
endmodule
```
</details>

---

## 8 DOC

- [APB SPEC PDF 보기 (GitHub)](https://github.com/taewon522/APB-Design-and-Verification/blob/main/doc/IHI0024E_amba_apb_architecture_spec.pdf)
- [APB SPEC PDF 다운로드 (RAW)](https://raw.githubusercontent.com/taewon522/APB-Design-and-Verification/main/doc/IHI0024E_amba_apb_architecture_spec.pdf)
