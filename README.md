# Microprocessor Visualisation — Interactive Tomasulo's Algorithm Simulator

An interactive, cycle-by-cycle web simulator for **Tomasulo's algorithm** — the dynamic instruction-scheduling technique that lets a CPU execute floating-point operations out-of-order while preserving in-order results. You configure a small MIPS-style floating-point program, the initial register file, the contents of memory, and the latency of each functional unit, then step through the pipeline one clock cycle at a time and watch issue, execute, and write-back propagate across the reservation stations and load/store buffers in real time.

Built jointly by **Anas ElNemr** and **Ahmed Eltawel** for the **Microprocessors** course at the **German University in Cairo (GUC)**, ~January 2022.

---

## Features

- **Interactive instruction editor** — add or remove instructions on the fly with `+` / `−` controls, pick the opcode from a dropdown, and fill in destination + source register names (`F0` … `F31`).
- **Memory editor** — pre-populate any number of `(address, value)` pairs that loads / stores will read from and write to.
- **Register file editor** — set initial values for all 32 floating-point registers (`F0` … `F31`) in a 4-column grid.
- **Configurable latencies** — independently tune the cycle count of `ADD`, `SUB`, `MUL`, `DIV`, `LD`, `SD` (defaults: 2 / 2 / 10 / 40 / 2 / 2).
- **Cycle-by-cycle stepping** — a single **Next** button advances the simulator one clock cycle. Execution ends with an "Execution Complete!" alert when every reservation station, load buffer, and store buffer is empty *and* there are no instructions left to issue.
- **Live state inspection** — every cycle, the UI re-renders:
  - **Instruction status table** — `Issue`, `Execution Start`, `Execution End`, `Write Back` cycle numbers per instruction.
  - **Add/Sub reservation stations** — 3 slots (`A1 / A2 / A3`) with `Busy`, `Op`, `Vj`, `Vk`, `Qj`, `Qk`, `Cycles Remaining`.
  - **Mul/Div reservation stations** — 2 slots (`M1 / M2`) with the same fields.
  - **Load buffer** — 3 slots (`L1 / L2 / L3`) with `Busy`, `Address`, `Cycles Remaining`.
  - **Store buffer** — 3 slots (`S1 / S2 / S3`) with `Busy`, `Address`, `V` (value), `Q` (producing tag), `Cycles Remaining`.
  - **Register file** — final / current value (or producing tag) of every `F0`–`F31` register.

---

## ISA / Instructions Supported

The simulator implements a **6-instruction subset of the MIPS floating-point ISA**, scheduled through Tomasulo's algorithm with separate Add/Sub and Mul/Div functional unit pools plus dedicated load and store buffers.

| Mnemonic | Operation | Reservation Pool | Default Latency |
|----------|-----------|------------------|-----------------|
| `L.D`    | Load double from memory: `dst ← MEM[addr]` | Load Buffer (3 slots) | 2 cycles |
| `S.D`    | Store double to memory: `MEM[addr] ← src`  | Store Buffer (3 slots) | 2 cycles |
| `ADD.D`  | Floating-point add: `dst ← src1 + src2`    | Add/Sub RS (3 slots)  | 2 cycles |
| `SUB.D`  | Floating-point sub: `dst ← src1 − src2`    | Add/Sub RS (3 slots)  | 2 cycles |
| `MUL.D`  | Floating-point mul: `dst ← src1 * src2`    | Mul/Div RS (2 slots)  | 10 cycles |
| `DIV.D`  | Floating-point div: `dst ← src1 / src2`    | Mul/Div RS (2 slots)  | 40 cycles |

Registers are 32 floating-point registers `F0`–`F31`. Memory is an unbounded sparse map keyed by the address strings the user supplies. Branches, integer ALU ops, and re-order-buffer / speculation are intentionally out of scope for this coursework simulator.

### How a tick maps to Tomasulo's stages

Each call to `Main.tick()` does, in order:

1. **Issue** — pull the next instruction from the program; place it into a free slot of the matching reservation station / load / store buffer; copy operand values from the register file (`Vj` / `Vk`) or, if a producer is still in flight, the producing station's tag (`Qj` / `Qk`); mark the destination register as "produced by this station ID".
2. **Execute** — for every busy station whose operands are *both* available (or whose load address is set), decrement `cyclesRemaining`. The first cycle it becomes ready becomes `Execution Start`; the cycle it hits zero becomes `Execution End`.
3. **Write back (CDB broadcast)** — the cycle after `Execution End` (`cyclesNeeded === -1`), publish the result on the common data bus; every other reservation station / store buffer waiting on that tag (`Qj` / `Qk` / `Q`) snaps to the value; the destination register is updated *only* if it's still pointing at the producing station ID (preserving WAW correctness); the station is freed.
4. **Termination** — when there are no instructions left to issue *and* every Add, Mul, Load, and Store station is idle, `End = true` and further clicks alert "Execution Complete!".

---

## Tech Stack

- **React 17** (Create React App, `react-scripts` 4) — the simulator is a 2-screen SPA wired with React Router (`/` for setup, `/results` for cycle-by-cycle stepping).
- **Vanilla JavaScript classes** for the simulator core — `Main`, `ReservationStation`, `Instruction`, `Load`, `Store`, `Registers` are framework-agnostic and could be lifted out as a library.
- **Ant Design** (`antd`) for the opcode `Select` dropdowns; **react-loading** for the in-button spinner.
- **Bootstrap / styled-components / SCSS** are pulled in via `package.json` but most of the UI is written with inline styles (custom `Archivo` web fonts, lavender header `#7900FF`).
- DOM-rendered tables (no canvas, no animation library) — every cycle, `setrender(Math.random())` forces a re-render and the data tables redraw from the live `Main` instance.

---

## Project Structure

```
.
├── public/
│   └── index.html              CRA root template
├── src/
│   ├── App.js                  Router (/ → Tomasulo, /results → Results)
│   ├── index.js                React DOM entry
│   ├── App.css / index.css     Global styles + Archivo / Helvetica @font-face
│   ├── Utils.js                durationString helper
│   ├── components/
│   │   └── Button.js           Reusable purple action button (with hover + spinner)
│   ├── screens/
│   │   ├── Tomasulo.js         Setup screen — instructions, memory, registers, latencies
│   │   └── Results.js          Cycle stepper — instruction status + 4 buffer tables + register file
│   └── Main/
│       ├── Main.js             ★ The simulator engine — Issue / Execute / Write-back per tick
│       ├── Instruction.js      Instruction record (type, dest, src1, src2, issue/exec/wb cycles)
│       ├── ReservationStation.js   Add/Sub + Mul/Div RS slot (id, busy, op, Vj/Vk, Qj/Qk, A, …)
│       ├── Load.js             Load buffer slot (id, busy, A, cyclesNeeded, …)
│       ├── Store.js            Store buffer slot (id, busy, V, Q, A, cyclesNeeded, …)
│       └── Registers.js        Floating-point register file (F0 … F31)
└── package.json
```

---

## How to Run

Requires Node.js (16+ recommended) and npm.

```bash
git clone https://github.com/anaselnemr/Microprocessor-Visualisation.git
cd Microprocessor-Visualisation
npm install
npm start
```

The dev server opens at <http://localhost:3000>. Use `npm run build` to produce a minified production bundle in `build/`.

### A quick run

1. On the **Setup** screen, leave the default latencies (or tune them).
2. Add a few instructions — e.g.

   ```
   L.D   F6, 32        ; F6 ← MEM[32]
   L.D   F2, 44        ; F2 ← MEM[44]
   MUL.D F0, F2, F4    ; F0 ← F2 * F4
   SUB.D F8, F2, F6    ; F8 ← F2 − F6
   DIV.D F10, F0, F6   ; F10 ← F0 / F6
   ADD.D F6, F8, F2    ; F6 ← F8 + F2  (note WAW on F6)
   ```
3. Add memory entries `address=32 value=10` and `address=44 value=5`. Pre-load any registers you reference (e.g. `F4=3`).
4. Click **Execute**. You land on the **Results** screen at cycle 1.
5. Click **Next** repeatedly. Watch instructions fill reservation stations, watch tags propagate through `Qj` / `Qk`, watch the CDB broadcast resolve dependencies, and watch the destination register flip from a tag (`A1`, `M2`, …) back to a numeric value once its producer writes back.

---

## Coursework Context

Built for the **Microprocessors** undergraduate course at the **German University in Cairo (GUC)**, ~**January 2022**. The brief was to implement Tomasulo's algorithm — issue, execute, write-back, the common data bus, register-renaming via station tags, and WAW / RAW / WAR hazard handling — as an interactive teaching aid that lets you watch each stage tick rather than just stare at the textbook diagram in Hennessy & Patterson.

Some intentional scope limits (so the visualisation stays readable):

- Fixed reservation-station counts: **3 Add/Sub, 2 Mul/Div, 3 Load, 3 Store** (Hennessy & Patterson's classic example sizes).
- No branch handling, no re-order buffer, no speculation, no register aliasing table beyond the in-place tag overwrite on the register file.
- No memory hazard detection between loads and stores at the same address (the load buffer treats the address as ready immediately).

Within those limits, the engine matches the textbook semantics — operand forwarding through `Qj` / `Qk`, single-cycle write-back broadcast, late-binding of values to dependent stations, and correct WAW resolution by only updating the destination register if it still holds the producing station's tag.

---

## Authors

Anas ElNemr  ·  Ahmed Eltawel
