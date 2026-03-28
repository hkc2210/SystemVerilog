# Design Plan

## 1. Module Design Plan

This design includes three modules: `fpga_calculator`, `alu`, and `storage`.

The top-level module `fpga_calculator` is responsible for connecting the storage and arithmetic blocks and updating the final output.

The `alu` module performs four basic operations: addition, subtraction, multiplication, and division.

The `storage` module stores operands in an array-based structure so that input data can be reused during calculation.

---

## 2. Data Type Plan

The design uses `logic` for control signals such as `clk`, `reset`, `load_en`, and `opcode`.

Arithmetic data is represented by `logic [WIDTH-1:0]`, which makes the bit-width easy to scale through parameters.

The storage array is defined with `logic [WIDTH-1:0] mem [DEPTH]` to keep multiple operands.

In this project, `WIDTH = 16` is a reasonable choice for a basic calculator design.

---

## 3. Array and Loop Plan

An array is used in the storage module to hold input operands.

This structure makes the design more scalable if more operands are added later.

Loops are not heavily used in the current skeleton code, but they can be applied for initialization or repeated processing in future extensions.

Depending on the design purpose, these loops may be placed in either `always_comb` or `always_ff`.

---

## 4. Integration Plan

First, an input value is written into the storage module when `load_en` is enabled.

Next, the stored operands are sent to the `alu`.

The `alu` uses `op_select` to choose the required arithmetic operation.

The calculation result is then registered into `final_result` on the clock edge.

The `reset` signal clears the output when needed.

---

## 5. Skeleton Code

### ALU Module

```systemverilog
module alu #(parameter WIDTH = 16) (
    input  logic [WIDTH-1:0] a,
    input  logic [WIDTH-1:0] b,
    input  logic [1:0] opcode,
    output logic [WIDTH-1:0] result,
    output logic error
);

always_comb begin
    result = '0;
    error  = 1'b0;

    case (opcode)
        2'b00: result = a + b;
        2'b01: result = a - b;
        2'b10: result = a * b;
        2'b11: begin
            if (b != 0)
                result = a / b;
            else
                error = 1'b1;
        end
        default: result = '0;
    endcase
end

endmodule
```

### Storage Module

```systemverilog
module storage #(parameter WIDTH = 16, DEPTH = 4) (
    input  logic clk,
    input  logic we,
    input  logic [$clog2(DEPTH)-1:0] addr,
    input  logic [WIDTH-1:0] data_in,
    output logic [WIDTH-1:0] data_out [DEPTH]
);

logic [WIDTH-1:0] mem [DEPTH];

always_ff @(posedge clk) begin
    if (we)
        mem[addr] <= data_in;
end

assign data_out = mem;

endmodule
```

### Top-Level Module

```systemverilog
module fpga_calculator #(parameter WIDTH = 16, DEPTH = 2) (
    input  logic clk,
    input  logic reset,
    input  logic [WIDTH-1:0] input_val,
    input  logic [1:0] op_select,
    input  logic load_en,
    output logic [WIDTH-1:0] final_result
);

logic [WIDTH-1:0] operands [DEPTH];
logic [WIDTH-1:0] calc_res;
logic div_error;

storage #(.WIDTH(WIDTH), .DEPTH(DEPTH)) input_buffer (
    .clk(clk),
    .we(load_en),
    .addr(0),
    .data_in(input_val),
    .data_out(operands)
);

alu #(.WIDTH(WIDTH)) core_alu (
    .a(operands[0]),
    .b(operands[1]),
    .opcode(op_select),
    .result(calc_res),
    .error(div_error)
);

always_ff @(posedge clk or posedge reset) begin
    if (reset)
        final_result <= '0;
    else
        final_result <= calc_res;
end

endmodule
```
