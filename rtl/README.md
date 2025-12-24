## RTL CODE

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
