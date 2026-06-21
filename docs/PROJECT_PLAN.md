# Embedded Linux Driver & Systems Project — Scope & Plan (v4)

**Owner:** [you] — currently Firmware Engineer III, Varian (med-device, regulated)
**Platform:** Renesas RZ/V2L — 2× Cortex-A55 @ 1.2 GHz (Linux) / 1× Cortex-M33 @ 200 MHz (RTOS/bare-metal) / 64 KB SRAM (M33) / DRP-AI (unused)
**Primary diagnostic instrument:** Saleae Logic Pro 8 (8-ch digital logic analyzer; digital timing + bus decode)
**Status:** v4 scope. Frozen after this revision. Implementation not yet started.

---

## 0. Target and strategy (read first)

**Target pipeline:** senior embedded roles centered on **embedded Linux application + driver development** in regulated/industrial/medical hardware (representative employers: Honeywell, Stryker, Emerson, Thermo Fisher, GE, ZEISS-class). Google/kernel-driver-at-scale is a stretch target — *not* optimized for, but largely covered as a side effect of building this well. No tears if it misses.

**Intersection these roles actually reward (build to this):**
1. Embedded Linux + **Yocto/BSP** + **device driver** + board bring-up. *(highest frequency across the pipeline)*
2. **Modern C++ (C++14)**, OOP, design patterns, under a **named test framework (GTest/GMock) + CI**. *(often "Required")*
3. **Python** for test automation. *(recurring)*
4. Board bring-up + **scope/logic-analyzer** HW/SW debugging.
5. **RCA / regulated rigor / IEC 62304.** *(owner's genuine edge)*

**Owner skill calibration (drives sequencing):**
- **Yocto: experienced** → safe anchor, demonstration not learning. Lead with it.
- **Kernel drivers: NEW** (bare-metal/userspace peripheral experience only) → growth artifact, must be de-risked with a learning ramp, not treated as polish.
- **Dual-core A55↔M33 IPC: NEW** → optional capstone only; not a blocking dependency.
- **RCA/CAPA: strong** (4 CAPA investigations, regulated med-device) → lead credential; content confidential, method/role fully disclosable.

**Governing rule:** two unfamiliar, schedule-eating items exist (kernel driver, IPC). **Never run both as blocking dependencies.** Kernel driver is the priority growth item (higher pipeline value); IPC is optional.

---

## 1. Center of gravity (what changed from earlier versions)

Earlier versions centered on an M33 bounded-queue/watchdog safety subsystem. That fit one outlier target (a real-time-safety controller role) and is a *poor* fit for this pipeline, which is overwhelmingly **embedded-Linux application + driver** work. The center of gravity therefore moves **from the M33 bare-metal side to the A55 Linux side.** The M33 safety work survives as a supporting pillar (§4), not the centerpiece.

---

## 2. Hero artifact — Linux device driver (gated, growth item)

A **real Linux device driver** for one on-board sensor (I2C or SPI — both named across the pipeline), written from scratch, with correct kernel-driver discipline. This is the artifact that converts the owner's profile from "bare-metal peripheral engineer" to "kernel driver engineer" — the single biggest paper gap for this pipeline.

### 2.1 Learning ramp (because kernel drivers are new — de-risk first)
Time-boxed climb up the framework curve *before* the real driver:
- A throwaway "hello-world" loadable kernel module (build, insmod/rmmod, dmesg).
- A simple GPIO + sysfs character driver.
- *Then* the real sensor driver.
Treat the ramp like the IPC spike: if the framework curve is steeper than expected, that's learned cheaply here, not three weeks into a dependent build.

### 2.2 The real driver — what "done right" means
- Proper driver-model lifecycle: probe / remove.
- **Devicetree** binding + overlay for the sensor.
- Correct bus integration (I2C client or SPI device).
- Interrupt handling with appropriate locking (if the sensor supports it), or documented polling rationale.
- A clean userspace interface: **sysfs** attributes or, ideally, an **IIO** driver if the sensor fits (IIO is the "right" framework for sensors and signals senior intent).
- Loadable, unloadable, no leaks, no oopses; documented.

### 2.3 Bring-up debugging narrative (this is interview gold)
Document the hard parts: devicetree not matching, device not probing, IRQ not firing, bus silent. **Saleae captures of the bus transaction driven from kernel space** are the evidence. The *debugging story* is exactly what driver interviews probe.

---

## 3. Linux platform + application layer (safe anchor + Required-list coverage)

### 3.1 Yocto image / BSP (safe anchor — lead with it)
Custom Yocto image for the RZ/V2L: kernel + the driver layer (§2) as a recipe, U-Boot, root filesystem. Owner is experienced here → this is demonstration, build it early and solid. Covers the highest-frequency requirement in the pipeline.

### 3.2 Modern C++14 application + tests + CI (clears "Required" lines)
- Userspace application in **modern C++14** consuming the driver via a clean abstraction; justify any design pattern used (don't pattern for pattern's sake).
- **GTest/GMock** unit tests; host-compiled.
- **CI pipeline** (GitHub Actions acceptable) running the build + tests on push.
- This trio directly clears Honeywell/medical-UI "Required" C++/TDD/CI lines.

### 3.3 Python test automation (recurring preferred line)
A Python script that exercises the application/driver end-to-end (drives inputs, checks outputs, reports). Covers the Python-for-test-automation line that recurs across the pipeline.

---

## 4. Supporting pillar — M33 real-time / safety + RCA (demoted, not deleted)

Retained to demonstrate RTOS/determinism (named in several JDs) and to anchor the RCA story. No longer the centerpiece.

- **Bounded priority queue + liveness watchdog + state machine + latched safe-shutdown** on the M33, with fault injection that reproduces a queue-saturation→watchdog-timeout chain. (Real-time + functional-safety evidence.)
- **Hardware-backed watchdog** (not a software task alone).
- **RCA write-up on this subsystem** — the un-redacted, fully-detailed companion to the confidential CAPA record. Inject fault → symptom → ranked hypotheses → instrumentation → containment → durable fix → verification.
- Built single-core / host-testable; does **not** depend on IPC.

---

## 5. Measured evidence (Saleae-driven; "professional-grade" proof)

- Driver-driven **bus transaction captures** (I2C/SPI) — including a fault-and-recovery case, not just happy path.
- Control-loop **WCET distribution** over many iterations (M33 side), vs. deadline.
- Saturation→timeout chain as **one annotated GPIO timeline** (M33 side).
- **RAM/stack footprint** from linker map vs. 64 KB M33 budget.
- Saleae is digital-first; 8-channel budget planned before capture; not used for analog signal-integrity work.

---

## 6. Build sequence (ordered by risk + pipeline value)

**Rule:** never let two consecutive sessions pass without something executing.

| # | Step | Owner familiarity | Blocks others? | Ends in |
|---|---|---|---|---|
| 1 | RZ/V2L Linux + Yocto image bring-up | Experienced | Anchor | Board boots a custom image |
| 2 | **Kernel-driver learning ramp** (§2.1), time-boxed | **New — de-risk** | No | hello-world + GPIO/sysfs module load |
| 3 | **Real sensor driver** (§2.2–2.3) — HERO | New (ramped) | No | Driver probes, exposes sysfs/IIO, captured on Saleae |
| 4 | C++14 app + GTest + CI (§3.2) | Mixed | No | App consumes driver; tests green in CI |
| 5 | Python test automation (§3.3) | Mixed | No | End-to-end automated test run |
| 6 | M33 real-time/safety pillar + RCA write-up (§4) | Experienced | No | Subsystem runs; RCA documented |
| 7 | Saleae evidence pass (§5) | Experienced | No | WCET dist + bus captures + timeline |
| 8 | **OPTIONAL:** dual-core A55↔M33 IPC | **New** | No | Integer across boundary, then latest-valid IPC |

**Risk rule enforced:** steps 3 (driver) and 8 (IPC) are both new/unfamiliar. Step 3 is prioritized; step 8 is optional and only attempted if step 3 landed and time remains. They never block each other.

---

## 7. Explicitly OUT of scope

| Cut / deferred | Why |
|---|---|
| Camera/DRP-AI, CCS811, LED ring, dot-matrix, rich HMI | Don't serve the target rubric. |
| Multiple drivers / many protocols | One *well-built* driver beats three shallow ones. Add a second only if the first is solid and time remains. |
| Substation protocols (IEC 61850/DNP3), DDS, QT/QML, BLE, PCIe driver | Divergent per-employer extras — résumé/interview keywords, not build targets. Building them all = the old breadth mistake. |
| Google/kernel-at-scale-specific prep, DSA grind | Stretch target; not optimized for. (DSA practice, if pursued, is a separate workstream, not this project.) |
| Dual-core IPC (conditionally) | Optional capstone only. |

---

## 8. How interview evidence assembles

| Requirement area | Evidence |
|---|---|
| Embedded Linux + Yocto/BSP | §3.1 custom image |
| **Device drivers** (the pipeline's core gap) | §2 real sensor driver + bring-up debugging narrative |
| Modern C++14 + TDD + CI | §3.2 |
| Python test automation | §3.3 |
| Board bring-up + scope/analyzer debug | §2.3 + §5 Saleae captures |
| RTOS / real-time / determinism | §4 M33 subsystem + §5 WCET |
| RCA / regulated rigor / IEC 62304 | "Led RCA on 4 CAPA investigations" (method concrete, content generic) + §4 board RCA write-up as un-redacted companion |
| Multi-core IPC (a few JDs) | §6 step 8, if it lands |

Lead credential everywhere: **regulated med-device rigor + CAPA/RCA.** Biggest growth artifact: **the kernel driver.**

---

## 9. Risks

| Risk | Severity | Mitigation |
|---|---|---|
| **No deadline → project dies.** Open-ended timelines produced the original empty-skeleton state. | **High** | Set a hard date for §2–§4 (driver + app + tests) done. Tell someone. "Something runs every two sessions." |
| **Kernel driver is new → learning curve eats weeks silently.** | **High** | §2.1 time-boxed learning ramp before the real driver. Treat like a spike. |
| **Two unfamiliar bring-ups (driver + IPC) compounding.** | **High** | Driver prioritized; IPC strictly optional. Never both blocking. |
| **Breadth creep** ("show all my skills" / "the tool can also…" / "as detailed as possible"). Recurring behavioral pattern across this project's history. | **High** | §7. Every addition must map to a *high-frequency* requirement in §0. One good driver > three shallow ones. |
| Plan-polishing as procrastination | Medium | Doc frozen at v4. Next artifact = code/capture, not a doc edit. |
| Vague RCA narration to protect IP | Medium | Rehearse concrete-method / generic-content version. Vague = junior signal. |

---

## 10. The one discipline that decides this

The spec is frozen. **This plan is frozen at v4.** The next thing produced must be a booting Yocto image, a loaded kernel module, or a green CI run — not a better document. Further plan edits are avoidance.

**Set the deadline. Build the Yocto image. Climb the driver ramp. Write the driver.**
