# BLDC Motor Control — Literature Review / State of the Art

Unisabana Herons EV — Shell Eco-marathon Prototype

Status: living document, built in batches. This is a **separate review from
`docs/literature-review/literature-review.md`** (the GLIDE driver-feedback review) — this one
covers control of the vehicle's brushless DC (BLDC) drive motor itself: commutation strategy,
sensing, torque ripple, and embedded implementation.

---

## 0. Scope and Method

Batch 1 covers 5 papers spanning four sub-topics in BLDC motor control:

1. Sensorless rotor-position sensing (reducing Hall-sensor/encoder hardware cost).
2. Commutation torque ripple: where it comes from, and circuit/control techniques to suppress it.
3. Embedded speed-loop implementation on a real microcontroller with Hall sensors.
4. Six-step trapezoidal commutation vs. field-oriented control (FOC), compared experimentally on
   the same motor.

6 more papers are planned for a second batch. Each section below records what was found, the key
equations/results, and how it bears on the Herons EV's BLDC drive.

---

## 1. Sensorless Rotor-Position Sensing

**Su & McKeever, "Low Cost Sensorless Control of Brushless DC Motors with Improved Speed Range,"
IEEE 2002, Oak Ridge National Laboratory.**

Standard BLDC six-step commutation needs rotor position at only the six commutation instants per
electrical cycle. Hall sensors give this directly but add cost/parts and a failure point; a
well-known cheaper alternative is to infer position by sensing the back-EMF (BEMF) on the
currently-unexcited ("floating") phase, since only two of three phases conduct at a time in
trapezoidal excitation.

**Traditional approach:** sense all three terminal voltages, integrate each (ideal integrator
gives exactly π/2 phase shift from the BEMF zero-crossing, independent of speed), and detect
zero-crossings of the integrator outputs to get commutation timing. Problem: true integrators
drift/offset in practice, so real circuits use low-pass filters instead — but a filter's phase
delay is **speed-dependent** and always < π/2, so timing error grows as speed drops.

**Proposed scheme:** sense only *one* terminal voltage (66% reduction in sensing components).
Each terminal voltage's zero-crossing gives 2 of the 6 commutation instants per cycle; the other
4 are interpolated by measuring the elapsed time $T_k$ between the two sensed instants and
firing at $T_k/3$ and $2T_k/3$ after the last sensed instant (works well only when speed doesn't
change rapidly between cycles). To fix the low-pass filter's speed-dependent phase error, a
look-up table (built offline from the filter's known phase-delay-vs-frequency curve) converts
measured frequency $f_m = 1/(2T_k)$ into a correction delay $\tau_k$, applied via a second timer
before firing the next commutation.

**Why the phase error matters:** insufficient phase delay at low speed causes (a) lower
torque-per-amp because current isn't sustained through the full flat-top of the BEMF, (b) torque
ripple, and (c) a **circulating current** through a freewheeling diode + the opposite low-side
switch during commutations between lower switches — extra copper loss, worse at low speed.

**Experimental results** (2.2 kW / 14 Nm / 4-pole PM motor, band-pass-filter variant sensing one
phase against the negative DC rail): with phase-delay correction, operating range extended from
"inoperable below ~300 rpm" (uncorrected) down to 100 rpm, at essentially the same efficiency as
full three-phase Hall-based sensing at the 1500 rpm rated speed. Stable closed-loop response to
step load-torque changes was also demonstrated at 750 rpm.

**Relevance:** the Herons EV likely already uses Hall sensors (cheap, reliable, appropriate for a
one-off competition vehicle) rather than needing to remove a BOM cost — but the underlying
mechanism (BEMF only appears on the floating phase; filter-introduced phase delay is
speed-dependent and must be compensated) is the same physics behind any *fallback* sensorless
mode (e.g., graceful degradation if a Hall sensor fails) and behind understanding why cheap Hall
sensor drives still show speed-dependent commutation error if timing isn't corrected.

---

## 2. Commutation Torque Ripple: Sources and Reduction Topologies

**Karthika & Nisha, "Review on Torque Ripple Reduction Techniques of BLDC Motor," IEEE ICICT
2020.**

BLDC torque ripple during commutation arises because the incoming phase current can't rise
exactly as fast as the outgoing phase current falls (their rates depend on different circuit
conditions), so the non-commutating phase's current — and hence torque, which for BLDC is
$T_d = \frac{2Ei_c}{\omega}$ under the standard "phase A incoming, phase B outgoing, phase C
non-commutating" analysis — transiently deviates from its steady value.

**Key derived result:** analyzing the commutation-interval circuit (switch S1 off, S3 on, freewheel
diode D4 conducting, S2 still on) and imposing the condition for the non-commutating phase current
to stay exactly constant during commutation gives

$$\frac{di_c}{dt} = \frac{V_{dc} - 4E}{3L_a} = 0 \quad\Longrightarrow\quad V_{dc} = 4E$$

i.e. **the DC bus voltage must be raised to (at least) four times the instantaneous back-EMF
during the commutation interval** to eliminate commutation-torque ripple. This 4× relationship
recurs as the theoretical basis for every topology reviewed.

**Topologies surveyed, all of which boost the DC-link voltage only during commutation:**

- **Cuk converter** (boost+buck DC-DC stage ahead of the inverter): during normal conduction
  the converter free-wheels through a diode; during commutation a switch $T_c$ engages the
  converter in boost mode, raising $V_{c2} = d_c V_{c1}$. Demonstrated on a 70 W, 24 V, 4 A,
  3000 rpm, 0.23 N·m motor.
- **Charged-capacitor method:** a capacitor charges through a diode during normal conduction;
  at commutation the diode reverse-biases and the capacitor's stored voltage adds to $V_{dc}$
  (condition: $V_{dc} = V_s + U_e \geq 4E$). No extra chopper circuit needed. Demonstrated on a
  3.7 kW, 300 V, 1500 rpm, 24 N·m motor — at $V_{dc}=300$ V, reported 23.4% non-commutated-phase
  current ripple and 20.4% torque ripple.
- **DVM (voltage-modulator) scheme:** two series/parallel-switchable capacitors — parallel
  (normal DC voltage) during conduction, series (double DC voltage) during commutation — feeding
  the inverter from an AC-supply/diode-bridge front end. Demonstrated on a 1 hp, 300 V, 2500 rpm,
  2 N·m motor; used for high-speed operation with PWM handling low speed separately.

A comparison table (their Table 1) also lists an auxiliary step-up circuit, plain PWM, and a
three-level NPC inverter as alternative approaches with their own component counts and switching
frequencies (typically 20 kHz, 80 kHz for the NPC case).

**Relevance:** this gives the Herons EV a concrete, quantified design criterion — if commutation
torque ripple is a problem worth solving in the drive controller, "boost the effective bus voltage
to ≥4× BEMF only during the ~π/3 commutation window" is the common thread across every hardware
technique surveyed, and is worth checking against whatever the drive's actual DC bus headroom is
before reaching for one of these converter topologies.

---

## 3. Embedded Hall-Sensor Speed Control on a Real Microcontroller

**Rad, Şimonca, Frătean & Dobra, "Embedded speed control of BLDC motors using LPC1549
microcontroller," IEEE 2016, Technical University of Cluj-Napoca.**

Practical worked example of a **sensored** (Hall-sensor-based) BLDC speed loop implemented on a
motor-control-oriented microcontroller (NXP LPC1549), useful as a template for how much of the
control chain can run without CPU intervention.

**Modeling:** full three-phase electrical model reduces (since only two phases conduct at a time)
to a single-phase-equivalent first-order electrical model $v_s = Ri(t) + L\,di(t)/dt + K_e\omega_r(t)$,
combined with the mechanical equation $K_T i(t) = T_m(t) + J\,d\omega_r/dt + B\omega_r(t)$. Because
the motor's electrical time constant is close to its mechanical time constant, the open-loop
transfer function is well-approximated as first order, $H_m(s) = K_m/(T_m s + 1)$, rather than the
full second-order form. This was **identified experimentally** via a PWM duty-cycle step
(40%→60%) and recording the RPM step response, giving $T_m = 103$ ms, $K_m = 18.72$ for their test
motor (Wantai 42BLF01, 24 V, up to 6000 rpm) — a directly reusable identification procedure.

**Controller design:** standard PI regulator, $H_R(s) = K_p \frac{T_i s + 1}{T_i s}$, tuned via
root locus on the combined open-loop transfer function $H_{ol}(s) = \frac{K_m K_p (T_i s+1)}{T_i s
(T_m s + 1)}$. Stability requires the PI zero to sit below the plant's pole ($T_i < T_m$); they
chose $T_i = T_m/20$ for a faster response and picked $K_p$ from the point where a tangent line
from the origin touches the root locus (a standard "most robust" gain-selection heuristic). Final
values: $K_p = 0.4$, $T_i = 5.2$ ms, running the digital PI at 1 kHz.

**Implementation detail worth reusing:** the microcontroller's State Configurable Timer (SCT) runs
the entire six-step commutation state machine (6 states, Hall-transition-triggered, driving 2 of 6
PWM outputs active per state) **entirely in hardware**, with zero CPU intervention per commutation
event — the CPU only handles the outer speed loop. They also found and fixed a real hardware
defect on a cheap motor: Hall sensors not perfectly at 120°/not concentric with the shaft produced
hundreds-of-RPM peak-to-peak speed noise; a software calibration (average the last 24
Hall-transition intervals per revolution, build a per-transition correction lookup table) cut that
to 10 RPM peak-to-peak.

**Experimental results:** no-load 2000→3000 rpm step: 210 ms settling time, 33.5% overshoot.
Loaded step (same PI gains): settling time dropped to 80 ms but overshoot rose to 50.5% — i.e. the
same PI tuning is not robust to load changes, flagged by the authors as future work (they suggest
predictive control to cut the low-speed overshoot).

**Relevance:** this is close to a direct blueprint for a from-scratch BLDC speed controller on a
cheap Hall-sensored motor: (1) identify the motor as a first-order plant from a duty-cycle step
response rather than trusting datasheet parameters, (2) root-locus-tune a discrete PI from that
identified model, (3) push the six-step commutation logic into hardware timer/state-machine
peripherals if the target MCU has them, freeing the CPU for the speed loop, and (4) budget for a
Hall-sensor calibration step if using low-cost motors — a real, previously-unbudgeted failure mode
worth checking against whatever motor the Herons EV drive uses. The load-vs-no-load overshoot gap
is also a concrete cautionary data point for PID gain selection generally (echoes the "reactive
vs. anticipatory control" framing already in the GLIDE review's Section 3).

---

## 4. Model Predictive Control for Commutation Torque Ripple

**Li, Fan, Kong, Liu & Zhang, "Torque Ripple Suppression of BLDCM With Optimal Duty Cycle and
Switch State by FCS-MPC," IEEE Open Journal of Power Electronics, 2024.**

A software-only (no extra converter hardware) alternative to Section 2's boost topologies:
replaces the traditional PI current loop with finite-control-set model predictive control
(FCS-MPC) to keep the non-commutating phase current constant through commutation, which — per the
same derivation as Section 2 — is equivalent to holding the effective bus voltage at 4× BEMF
($T_e = \frac{2E(U_{dc}-4E)}{3L\omega_m}$, zero ripple iff $U_{dc}=4E$).

**Two-part control scheme:**

1. **Optimal duty cycle** (`PWM_ON_PWM` bilateral modulation): discretizes the phase current
   dynamics via forward Euler ($i_x(k{+}1)$ predicted from present current, terminal voltages, and
   BEMFs) and solves algebraically for the duty cycle $d$ that drives the non-commutating phase
   current to a reference value at the next sample — replacing PI tuning with a closed-form
   calculation from the (known/estimated) $R$, $L$, and BEMF.
2. **Switch-state insertion compensation** (for high speed, where PWM duty cycle alone can't act
   fast enough): during the commutation interval, insert one of three intermediate conduction
   states — single-phase (A+), two-phase (A+C-), or three-phase (A+B-C-) — each producing a
   different $di_a/dt$ sign/magnitude (derived analytically for each case), and pick whichever
   state minimizes a cost function $g = ||i^*(k{+}1)| - |\hat i_p(k{+}1)||$ over the three
   candidates every control cycle.

Feedback correction ($\hat i_p(k{+}1) = \hat i(k{+}1) - c\,e(k)$, error $e(k) = \hat i(k) - i(k)$)
compensates for inductance mismatch between the controller's model and the real motor; tested
robust for $L_r$ between $0.5L$ and $2L$.

**Quantitative results** (100 W, 24 V, 3000 rpm rated, 0.32 N·m rated BLDCM; simulation +
STM32F407 hardware): comparing traditional PI_PWM_ON, PI_PWM_ON_PWM, MPC_PWM_ON (duty-cycle-only
MPC), and their full proposed method (duty cycle + switch-state insertion) —

| Speed | PI_PWM_ON | PI_PWM_ON_PWM | MPC_PWM_ON | Proposed |
|---|---|---|---|---|
| 750 rpm | 8.1% | 6.3% | 5.3% | 2.6% |
| 1500 rpm | 10.7% | 8.1% | 6.4% | 4.2% |
| 3000 rpm | 27.9% | 23.7% | 21% | 8.6% |

The gap between MPC-duty-cycle-only and the full method widens sharply at high speed — consistent
with their claim that duty-cycle prediction alone runs out of headroom once the available PWM
cycles per commutation interval shrink, which is exactly where the discrete switch-state insertion
picks up the slack.

**Relevance:** offers a middle ground between Section 2's ripple-reduction converters (extra
hardware, fixed 4× boost) and Section 3's plain PI (simple but load- and speed-dependent overshoot
per Rad et al.'s own results): MPC-based duty-cycle/switch-state selection needs no extra power
hardware, replaces PI gain-tuning with a model-based calculation from $R$/$L$/BEMF, and its
reported robustness to 0.5×–2× inductance mismatch is a relevant number if motor parameters are
only roughly known (as is common for off-the-shelf competition motors). The tradeoff to weigh
against Section 3's simpler PI approach is implementation complexity (per-cycle discrete
optimization over three candidate states, in addition to the duty-cycle solve) on whatever MCU the
Herons EV drive uses.

---

## 5. Trapezoidal (Six-Step) vs. Field-Oriented Control — Head-to-Head Comparison

**Nurtriartono, Mukhlisin, Yuniarto & Rijanto, "Performance Comparison of BLDC Motor Controllers
Designed Based on Trapezoidal Commutation and FOC," AIP Conf. Proc. 2187 (2019), ITS Surabaya /
LIPI Indonesia.**

Directly relevant control-architecture question: for an EV BLDC drive, is six-step trapezoidal
commutation (simple, but inherently produces commutation torque ripple — Sections 1–2, 4 above)
or field-oriented control (FOC, sinusoidal current control via Clarke/Park transforms into a
synchronously-rotating d-q frame, decoupling flux ($i_{sd}$) and torque ($i_{sq}$) control) the
better choice? Rather than simulating, the authors built and tested **both controller types on the
same physical 30 kW / 100 V / 240 A / 7000 rpm BLDC motor** on a chassis dynamometer (power capped
to 6 kW for safety during testing).

**Two experiments:**

1. **Unloaded ramp-up then loaded to stall**, recording speed/torque/power vs. time. Trapezoidal
   reached a higher peak speed (1800 rpm vs. lower for FOC in the same window) and peak power
   (5330 W at 1074 rpm vs. FOC's 5530 W at 862 rpm — FOC's peak power was actually marginally
   *higher* but reached later), but trapezoidal's peak torque was lower (67 N·m at 300 rpm vs.
   FOC's 78 N·m at 324 rpm). FOC's speed trace was visibly smoother; trapezoidal's was faster to
   respond but noisier.
2. **Step response under pre-applied load** (torque loaded first, then speed commanded): FOC
   showed better torque stability with less overshoot, but a slower rise time; trapezoidal had a
   faster rise time but larger torque overshoot and higher steady-state torque.

**Authors' summary finding:** trapezoidal is **more responsive** (faster speed/torque rise) but
produces **higher torque overshoot and rougher delivery**; FOC gives **smoother speed and torque**
(smaller overshoot, smaller steady-state torque for the same command) at the cost of slower
transient response. This is presented as an empirical confirmation of the standard theoretical
tradeoff (trapezoidal = simple, high torque ripple; FOC = smooth, more complex) but is one of the
few papers in this batch that actually quantifies it on identical hardware rather than only
asserting it.

**Relevance:** this is the highest-level architectural question in this review — whether the
Herons EV's drive controller (if custom-built rather than an off-the-shelf ESC) should use
trapezoidal or FOC commutation — and this paper is direct evidence rather than theory. For a Shell
Eco-marathon efficiency-focused vehicle where **smooth, low-ripple torque delivery at
low/moderate, largely steady-state speed** matters more than fast transient response (unlike, say,
a drag-race or robotics application), the tradeoff this paper quantifies leans toward FOC — but
that has to be weighed against FOC's added computational/implementation complexity (Clarke/Park
transforms, current-loop tuning in the d-q frame) relative to trapezoidal's simplicity, especially
if the target MCU is resource-constrained. Sections 2 and 4 above are worth reading as "how to keep
trapezoidal's simplicity while clawing back most of FOC's smoothness" if full FOC turns out to be
too heavy for the platform.

---

## 6. Synthesis (Batch 1)

Across all five papers, two threads recur:

- **The 4×-BEMF commutation condition** (Section 2, re-derived independently in Section 4) is the
  unifying quantitative fact behind commutation torque ripple in six-step trapezoidal BLDC drives:
  every ripple-reduction technique in this batch — hardware voltage-boost (Section 2) or
  model-predictive current shaping (Section 4) — is, underneath, a different way of approximating
  that condition during the commutation window.
- **Simplicity vs. smoothness is the recurring tradeoff**, appearing in three different forms:
  filter-based sensorless position sensing needing phase-delay correction to stay accurate at low
  speed (Section 1), plain PI control being simple but load-and-speed-dependent in overshoot
  (Section 3), and trapezoidal commutation being fast/simple but rougher than FOC (Section 5).
  Section 4's MPC approach is the one candidate in this batch that targets smoothness without
  trapezoidal's simplicity cost being fully paid in hardware (Section 2) or in the full
  complexity of FOC (Section 5).

**Open questions carried into batch 2:** what commutation/control architecture (if any) the Herons
EV's actual drive electronics currently use; whether the drive's DC bus has headroom for a
Section-2-style voltage boost if commutation ripple turns out to matter at competition speeds; and
whether a from-scratch design would be better served by Section 3's simple PI blueprint or
Section 5's smoother-but-heavier FOC, given the platform's MCU resources.
