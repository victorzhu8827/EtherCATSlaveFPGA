# AGENTS.md

Guidelines for AI agents working on this Verilog RTL project.

This document defines development rules, coding standards, simulation flow, and expectations for automated code generation tools (e.g., Codex, AI agents).

AI agents MUST read this file before modifying or generating RTL code.

---

# 1. Project Overview

This repository contains synthesizable Verilog RTL modules with testbenches and simulation infrastructure.

Development flow:

RTL Design
→ Testbench Development
→ Simulation Verification
→ Lint Checking
→ Synthesis

All AI-generated code must follow this workflow.

---

# 2. Repository Structure

```
rtl/        Synthesizable RTL modules
tb/         Testbenches
sim/        Simulation scripts
scripts/    Build utilities
docs/       Architecture and design documentation
```

Key files:

```
Makefile        Build entry point
AGENTS.md       AI development rules
ARCHITECTURE.md System architecture
DESIGN.md       Module design descriptions
```

---

# 3. Build and Simulation

Run simulation:

```
make sim
```

Run lint:

```
make lint
```

Run synthesis:

```
make synth
```

Run coverage:

```
make coverage
```

All RTL changes must pass simulation and lint.

---

# 4. Coding Standard

Language: Verilog-2001

RTL must be synthesizable.

Avoid SystemVerilog unless explicitly required.

---

# 5. Clock and Reset Rules

Clock name:

```
clk
```

Reset name:

```
rst_n
```

Reset type:

Active-low synchronous reset preferred.

Example:

```
always @(posedge clk or negedge rst_n)
```

All registers must be reset.

---

# 6. Sequential vs Combinational Logic

Sequential logic:

```
always @(posedge clk)
```

Use **non-blocking assignment**

```
<=
```

Combinational logic:

```
always @*
```

Use **blocking assignment**

```
=
```

---

# 7. Naming Convention

| Type       | Convention |
| ---------- | ---------- |
| Clock      | clk        |
| Reset      | rst_n      |
| Register   | *_r        |
| Next state | *_n        |
| Wire       | *_w        |
| Input      | *_i        |
| Output     | *_o        |
| Parameter  | ALL_CAPS   |

---

# 8. FSM Design Rules

Use two-process FSM structure.

State register:

```
always @(posedge clk)
```

Next state logic:

```
always @*
```

Default encoding:

one-hot

---

# 9. Latch Prevention

Always assign default values in combinational blocks.

Example:

```
always @* begin
    next_state = state;
end
```

---

# 10. Parameterization

Modules should support parameter configuration.

Example:

```
parameter DATA_WIDTH = 32
```

---

# 11. Testbench Requirements

Each module must have a testbench.

Naming rule:

```
tb_<module>.v
```

Testbench must include:

Clock generator
Reset logic
Stimulus generation
Output checking
Simulation termination

---

# 12. Simulation Requirements

Simulation must:

Compile without errors
Run without runtime errors
Produce expected behavior

---

# 13. Lint Requirements

Example lint command:

```
verilator --lint-only
```

Common errors to avoid:

implicit wires
width mismatches
unused signals
blocking assignments in sequential logic

---

# 14. CDC Guidelines

For multi-clock systems:

Use double-flip-flop synchronizers.

Example:

```
sync1 <= async_signal;
sync2 <= sync1;
```

---

# 15. Synthesis Restrictions

Do NOT use:

initial blocks in RTL
delay statements (#10)
force/release

---

# 16. AI Agent Workflow

When performing a task:

1. Read AGENTS.md
2. Read ARCHITECTURE.md
3. Inspect existing RTL
4. Create implementation plan
5. Implement minimal changes
6. Create/modify testbench
7. Run simulation
8. Run lint
9. Report results

---

# 17. Output Format for AI Tasks

All AI implementations must report:

Task summary
Files modified
New modules added
Testbench added
Simulation result
Lint result
Known risks

---

# 18. Definition of Done

A task is complete when:

RTL compiles successfully
Simulation passes
Lint passes
Coding rules are followed
Testbench verifies functionality
