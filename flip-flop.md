
> A **flip-flop** is a **fundamental 1-bit storage element** in digital electronics that **captures and holds a value based on a clock signal**. It’s the basic building block of **registers, state machines, and sequential logic**.

# 🔹 Core Idea

```
Flip-flop = 1-bit memory element
```

- Stores either **0 or 1**
- Changes state only at specific **clock events**

# 🔹 How It Works

For a typical **D flip-flop**:

```
On clock edge:  Q(next) = D
```

- **D** = input (data)
- **Q** = stored output

👉 Between clock edges, Q remains stable.


# 🔹 Timing Behavior

```
Clock:   ↑     ↑     ↑
D:       1     0     1
Q:       1     0     1   (updated only on clock edge)
```

# 🔹 Types of Flip-Flops

## ✔ D Flip-Flop (most common)

- Stores input directly
- Used in registers and pipelines

## ✔ SR Flip-Flop (Set/Reset)

```
S=1 → set Q=1
R=1 → reset Q=0
```

## ✔ JK Flip-Flop

- Generalized SR (avoids invalid state)

## ✔ T Flip-Flop (Toggle)

```
T=1 → toggle output
```

# 🔹 Key Signals

## ✔ Clock (CLK)

- Controls when data is captured

## ✔ Reset

- Forces output to known state (usually 0)

## ✔ Enable (optional)

- Controls whether update occurs


# 🔹 Flip-Flop vs Latch

|Feature|Flip-Flop|Latch|
|---|---|---|
|Trigger|Clock edge|Level (transparent)|
|Timing|Synchronous|Asynchronous|
|Usage|Most digital designs|Special cases|

# 🔹 Role in Digital Design

Flip-flops are used to:

- Build **registers**
- Store **state** in FSMs
- Create **pipelines**
- Synchronize signals

# 🔹 Example RTL

```
always @(posedge clk) begin
  q <= d;
end
```

👉 This describes a D flip-flop


# 🔹 Physical Implementation

Internally built from:

- Logic gates (NAND/NOR)
- Feedback loops to hold state

---

# 🔹 Key Insight

> Flip-flops introduce **memory and time** into digital systems—they are what make circuits sequential instead of purely combinational.

---

# 🔹 One-Line Summary

**A flip-flop is a clocked 1-bit memory element that captures input data on a clock edge and holds it until the next clock event.**


If you want, I can:

- show **gate-level implementation of a D flip-flop**
- or explain **setup/hold time and timing violations (critical for FPGA/ASIC design)**