---
title: 8-Bit ALU Implementation
---

# Lab Assignment 1: 8-Bit ALU Implementation

Welcome to the first hardware programming assignment for **CSC 397/595 Hardware Design and Programming using FPGAs**. In this lab, you will design and implement a simple 8-bit Arithmetic Logic Unit (ALU) using structural Verilog (logic gates).

## 1. Overview

An Arithmetic Logic Unit (ALU) is a combinational digital circuit that performs arithmetic and bitwise operations on integer binary numbers. It is a fundamental building block of many types of computing circuits, including the central processing units (CPUs) of modern computers.

For this assignment, you will construct an 8-bit ALU that supports eight basic operations and generates standard status flags. You are required to implement the combinational logic strictly using Verilog's built-in logic gate primitives (e.g., `and`, `or`, `not`, `xor`, `nand`, `nor`).

## 2. Downloading the assignment

To start the lab, you must first accept the assignment in GitHub Classroom. Accepting the assignment on GitHub Classroom is a 2-step process:

- **Step 1:** click on the invitation link from the assignment description from D2L.
- **Step 2:** check the inbox of the email address that you used when you created your GitHub account (it will probably be your personal email address), and look for an invitation email from GitHub. Click accept.

Now you can clone your repository in your home directory on the **matrix.cdm.depaul.edu** machine (replace *USER* with your GitHub account name):

```shell
$ cd ~
$ git clone https://github.com/transcendental-software/csc-397-595-lab1-USER.git
```

This command creates a directory named `csc-397-595-lab1-USER` containing all the necessary files for the assignment. You can then navigate into it using `cd csc-397-595-lab1`. *You only need to modify* the `rtl/alu.v` file. Use the `make` command to compile your code and the `make test` to run all the tests.

## 3. Basics of Verilog

Before diving into the ALU specifications, let's briefly review some fundamental Verilog concepts that you will use in this assignment.

### 3.1. Defining Modules

In Verilog, a `module` is the basic building block of a design. It encapsulates the functionality and defines the input and output ports. A module is declared using the `module` and `endmodule` keywords.

```verilog
module my_module (
    input wire a,
    input wire b,
    output wire out
);
    // Module implementation goes here
endmodule
```

### 3.2. Wires and Vectors

A `wire` represents a physical electrical connection between components. By default, a wire is a single bit (a scalar). To represent multiple bits, you can use vectors. Vectors are declared by specifying a range `[msb:lsb]`.

```verilog
wire single_bit;        // 1-bit wire
wire [7:0] eight_bits;  // 8-bit wire vector (bits 7 down to 0)

wire msb = data[7];        // Accesses the 8th bit (Most Significant Bit)
wire lsb = data[0];        // Accesses the 1st bit (Least Significant Bit)
wire bit_three = data[3];  // Accesses the 4th bit

wire [3:0] lower_nibble = data[3:0]; // Extracts bits 3 down to 0 (4 bits total)
wire [3:0] upper_nibble = data[7:4]; // Extracts bits 7 down to 4 (4 bits total)
wire [1:0] middle_bits = data[4:3];  // Extracts bits 4 and 3 (2 bits total)
```

### 3.3. Basic Logic Gates

Verilog provides built-in primitives for basic logic gates like `and`, `or`, `not`, `xor`, `nand`, and `nor`. When instantiating a gate primitive, the output port is always listed first, followed by the input ports. You can also provide an optional instance name.

```verilog
wire w_out, w_in1, w_in2;

// AND gate
and my_and (w_out, w_in1, w_in2);

// OR gate
or my_or (w_out, w_in1, w_in2);

// NOT gate (inverter)
not my_not (w_out, w_in1);
```

You can find more resources here:
- [Verilog Numbers](https://www.asic-world.com/verilog/syntax1.html#Numbers_in_Verilog);
- [Verilog Gate Primitives](https://www.asic-world.com/verilog/gate1.html#Gate_Primitives);

## 4. ALU Specifications

### 4.1. Inputs and Outputs

Your top-level module should have the following ports:

- `a`: 8-bit input operand.
- `b`: 8-bit input operand.
- `op`: 3-bit operation selector.
- `out`: 8-bit output result.
- `cout`: 1-bit Carry Out flag (high if addition/subtraction generates a carry out).
- `zero`: 1-bit Zero flag (high if the 8-bit output result is strictly `0`).
- `neg`: 1-bit Negative flag (high if the most significant bit of the result is `1`).
- `overflow`: 1-bit Overflow flag (high if signed addition/subtraction produces an invalid sign bit).

### 4.2. Operations

The `op` selector will determine the output result based on the following mapping:

- `3'b000`: Bitwise AND (`out = a & b`)
- `3'b001`: Bitwise OR (`out = a | b`)
- `3'b010`: Bitwise XOR (`out = a ^ b`)
- `3'b011`: Addition (`out = a + b`)
- `3'b100`: Subtraction (`out = a - b`)
- `3'b101`: Signed Equal (`out = ($signed(a) == $signed(b)) ? 8'h01 : 8'h00`)
- `3'b110`: Signed Greater Than (`out = ($signed(a) > $signed(b)) ? 8'h01 : 8'h00`)
- `3'b111`: Signed Less Than (`out = ($signed(a) < $signed(b)) ? 8'h01 : 8'h00`)

## 5. Implementation Requirements

### 5.1. Wires and Logic Primitives

You must use structural gate-level modeling for your combinational logic implementation. In Verilog, a `wire` data structure represents a physical electrical connection between elements. Since you will strictly use built-in logic gate primitives (e.g., `and`, `or`, `not`, `xor`, `nand`, `nor`), you will need to declare various `wire` arrays (e.g., `wire [7:0] and_result;`) to route data between your gates and sub-modules. 

Do not use behavioral modeling constructs (like `always` blocks or `if-else` statements) or continuous assignments with built-in arithmetic operators (like `assign out = a + b`).

### 5.2. Bitwise Operations (AND, OR, XOR, NOT)

You will implement separate modules for each bitwise operation (`bitwise_and`, `bitwise_or`, `bitwise_xor`, `bitwise_not`). Inside each module, apply standard logic gates to each respective bit of the 8-bit operands `a` and/or `b`. You will instantiate 8 individual primitives for each operation, or use Verilog vector bits if preferred. Connect each bit of the inputs to these gates, and route the outputs appropriately.

**Flag Updates:** For bitwise operations, the `cout` and `overflow` flags should be `0`. The `zero` flag is `1` if all bits of the output result are `0`. The `neg` flag is the most significant bit (bit 7) of the output result.

### 5.3. Arithmetic Operations (Addition and Subtraction)

To implement the 8-bit addition, first implement the provided `full_adder` 1-bit module using only logic primitives. Once complete, implement an `adder_8bit` module by chaining 8 of these `full_adder` modules together to create an 8-bit Ripple Carry Adder.

**Flag Updates:** For addition, `cout` is the carry out from the final 1-bit full adder. The `overflow` flag is `1` if the carry into the most significant bit differs from the carry out of the most significant bit. The `zero` flag is `1` if the 8-bit sum is `0`, and the `neg` flag is the most significant bit of the sum.

For subtraction, implement a `subtractor_8bit` module. You should reuse your `adder_8bit` module using two's complement arithmetic (`a - b` is equivalent to `a + ~b + 1`). You can use your `bitwise_not` module to invert `b`, and set the initial carry-in of the adder to `1`. This module should also compute the subtraction flags (`cout`, `zero`, `neg`, `overflow`).

**Flag Updates:** For subtraction, the flags follow the same logic as addition, evaluated on the two's complement adder's difference output and carry signals.

### 5.4. Comparison Operations (Equal, Greater Than, Less Than)

Implement separate modules for these comparison operations (`signed_eq`, `signed_gt`, `signed_lt`). These modules should rely on the flags generated by the `subtractor_8bit` module rather than recalculating the subtraction:
- **Signed Equal:** The output is true (8'h01) if the `zero` flag is true.
- **Signed Less Than:** Determining if `a < b` using signed numbers requires checking the `neg` (N) and `overflow` (V) flags from the subtraction. `a < b` is true if `N ^ V == 1`.
- **Signed Greater Than:** `a > b` is true if `a` is not less than `b` and `a` is not equal to `b`.

**Flag Updates:** For comparison operations, the `cout` and `overflow` flags should be `0`. Since the output is strictly `8'h01` or `8'h00`, the `neg` flag will always be `0`, and the `zero` flag will be `1` when the comparison is false (i.e., the output is `8'h00`).

### 5.5. Output Multiplexer

In the top-level `alu` module, you will instantiate all the operation modules along with an output multiplexer. To select the correct 8-bit output result based on the 3-bit `op` code, implement the `mux8_1bit` module using logic primitives. Then, instantiate 8 of these multiplexers in your `alu` module to construct a full 8-bit multiplexer, routing the intermediate results of your operation modules to the final `out` port. You must also implement logic to route and output the correct `cout`, `zero`, `neg`, and `overflow` flags based on the selected operation (e.g., routing arithmetic flags during addition/subtraction, and evaluating global flags like `zero` and `neg` on the final ALU output).

### 5.6. File Structure

All your implementation will take place in the provided `alu.v` skeleton file. You will see the following module stubs to complete:
- `bitwise_and`, `bitwise_or`, `bitwise_xor`, `bitwise_not`: Modules for bitwise operations.
- `full_adder`: To calculate the sum and carry out for a single bit.
- `adder_8bit`, `subtractor_8bit`: Modules for arithmetic operations.
- `signed_eq`, `signed_gt`, `signed_lt`: Modules for comparison operations.
- `mux8_1bit`: A 1-bit 8-to-1 multiplexer to select outputs.
- `alu`: The top-level design integrating all the above modules and multiplexer network.

## 6. Running Tests Locally

Before submitting your code, it is highly recommended that you run the test suite locally to ensure your ALU implementation works correctly and passes all requirements. We have provided a Makefile to simplify this process.

From within your repository folder, run the following command to compile your Verilog code and execute the testbench:

```shell
$ make test
```

If there are any syntax errors or failing test cases, the output will help you identify which operation or flag is incorrect. The GitHub Classroom autograder will run this exact same test suite when you push your code.

## 7. Debugging with GTKWave

If your local tests are failing, you can use a waveform viewer like [**GTKWave**](https://gtkwave.github.io/gtkwave/index.html) to inspect the internal signals of your design and identify where the logic went wrong. When you run `make test`, the testbench automatically calls `vvp bin/alu.vvp` and passes an input to the simulation. Each `vpp` call generates a waveform dump file named `alu.vcd`.

To open the waveform file, run the following command in your terminal:

```shell
$ gtkwave alu.vcd
```

Tracing the signals visually makes it much easier to tell if a bug is caused by a flipped bit, a disconnected wire, or an incorrectly mapped primitive.

## 8. Submitting Your Code

Once you have completed your implementation and tested it locally, you need to push your code to your GitHub Classroom repository. Pushing your code will automatically trigger the autograding tests.

To submit your work, use the following `git` commands from within your repository folder:

```shell
$ git add rtl/alu.v
$ git commit -m "Completed ALU implementation"
$ git push
```

You can verify your submission and see if your tests passed by checking the "Actions" tab in your GitHub repository online. If any tests fail, you can make additional changes to your code and repeat the `git add`, `git commit`, and `git push` steps to update your submission.
