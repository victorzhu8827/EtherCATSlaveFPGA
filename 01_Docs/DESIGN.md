# DESIGN.md

Detailed Module Design Specification

This document describes individual RTL modules and their interfaces.

---

# 1. Module Design Methodology

Each RTL module must follow a standard design pattern:

Interface Definition
Internal Logic
State Machines
Output Generation

---

# 2. Module Template

Standard module structure:

```
module module_name
#(
    parameter DATA_WIDTH = 32
)
(
    input  wire clk,
    input  wire rst_n,

    input  wire [DATA_WIDTH-1:0] data_i,
    output wire [DATA_WIDTH-1:0] data_o
);

endmodule
```

---

# 3. Interface Specification

Each module must clearly define:

inputs
outputs
parameters

Example interface description:

| Signal | Direction | Description    |
| ------ | --------- | -------------- |
| clk    | input     | system clock   |
| rst_n  | input     | reset          |
| data_i | input     | input data     |
| data_o | output    | processed data |

---

# 4. Internal Structure

Internal logic may include:

register stages
state machines
combinational logic

Registers should use suffix:

```
_r
```

Next state variables:

```
_n
```

---

# 5. State Machine Design

FSM design rules:

two-process structure
registered states
separate next-state logic

Example:

```
state_r
state_n
```

---

# 6. Data Path Design

Data path must avoid large combinational chains.

Recommended:

pipeline registers

Example:

```
stage1_r
stage2_r
stage3_r
```

---

# 7. Parameter Design

Parameters allow module reuse.

Example:

```
parameter FIFO_DEPTH = 16
```

---

# 8. Testbench Design

Each module must have a testbench.

Testbench components:

clock generator
reset logic
stimulus generator
result checker

---

# 9. Verification Strategy

Testbench should verify:

normal operation
boundary conditions
error conditions

Simulation must verify all FSM states.

---

# 10. Error Handling

Modules must define behavior for:

invalid inputs
overflow conditions
unexpected states

---

# 11. Documentation Requirements

Each module must include comments describing:

function
interface
timing assumptions

---

# 12. Example Design Flow

Step 1: Define module interface
Step 2: Write RTL skeleton
Step 3: Implement logic
Step 4: Write testbench
Step 5: Run simulation
Step 6: Fix bugs
Step 7: Run lint
Step 8: Update documentation
