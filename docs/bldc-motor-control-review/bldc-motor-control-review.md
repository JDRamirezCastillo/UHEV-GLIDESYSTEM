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

Batch 2 adds 4 more papers:

5. A full Shell Eco-marathon Urban Concept powertrain built around a BLDC motor — the closest
   direct comparandum to the Herons EV in this whole review.
6. A much larger survey (~240 references) that places Batch 1's torque-ripple findings inside a
   7-family taxonomy of control strategies, with quantified comparison tables for each.
7. A from-scratch, minimal-hardware commutation-logic design (digital logic + cheap MCU).
8. A technique for generating six-step PWM from general I/O pins on an MCU with only one on-chip
   timer, avoiding the cost of dedicated PWM peripherals.

2 more papers may be added in a future batch. Each section below records what was found, the key
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

## 6. A Complete Vehicle Case Study — Shell Eco-marathon Urban Concept Powertrain

**Hazizi, Erateb, Delli Carri, Jones, Leung, Sam & Yau, "Design, Construction, and
Simulation-Based Validation of a High-Efficiency Electric Powertrain for a Shell Eco-marathon
Urban Concept Vehicle," Designs (MDPI) 9(5) 113 (2025), Coventry University (BA Momentum team).**

The closest direct comparandum in this whole review: a full, documented, competition-built BLDC
powertrain for the **same Shell Eco-marathon Urban Concept category and voltage class** as the
Herons EV, with open-access CAD, bill of materials, and a validated Simulink model — essentially
a reference design the Herons EV's own drivetrain choices can be checked against line by line.

**Motor and drivetrain selection.** A structured decision matrix scored brushed DC, induction,
reluctance, BLDC, and PMSM motors on efficiency, power density, control complexity, cost, and
maintenance (their Figure 1); BLDC won as "the optimal compromise between efficiency, cost, and
simplicity" (>90% efficiency, moderate control complexity) over PMSM/axial-flux alternatives that
score higher on performance but cost and manufacturing complexity more. A single central BLDC
motor (vs. dual independent or in-wheel motors) was chosen for simplicity/cost despite dual-motor
options offering independent-wheel torque control — the same "regulatory allows up to 2 motors,
but 1 is simpler and cheaper" tradeoff the Herons EV would face. Selected motor: **1500 W, 48 V,
39.06 A rated, 4.78 Nm @ 3000 rpm, 85% efficiency, 5.5 kg** (Golden Motor HPM-1500B) — sized from
first-principles traction-force calculations (rolling + aerodynamic drag + acceleration force,
Section 2.1 of the paper) against a 295 kg loaded mass and a 16 km / 40 min race format.

**Transmission.** A literature-grounded choice of **two-stage chain drive** over gear or belt
drives: chain achieves ~98% efficiency (vs. >99% for gears, which are costly/hard to prototype)
and is more efficient than belt drives, whose frictional losses can run 34.6% higher than chain at
equivalent preload (cited from Friction Facts, 2012). Two gear ratios (12:1 and 8:1, single
sprocket swap) were sized to put the motor at its efficiency-optimal operating point at both 25
km/h and 40 km/h target speeds — directly reusable methodology for sizing a Herons EV drivetrain
around its own two competition-speed targets, if it has them.

**Battery and controller.** Custom **12S8P Li-ion pack, 96× Molicel P28A 18650 cells, 43.2 V
nominal, 967.68 Wh**, within SEM's 60 V / 1000 Wh limits, with an integrated BMS (cell balancing,
over/under-voltage, over-current, automatic isolation) — a directly comparable spec sheet if the
Herons EV's own pack needs benchmarking. Motor control is delegated to an **off-the-shelf FOC
controller** (Sabvoton SVMC72150, 48–72 V) rather than a custom design, with a separate
STM32-based Vehicle Control Unit (VCU) handling throttle/brake/BMS inputs and issuing the torque
command — i.e., in this design the FOC-vs-trapezoidal question from Section 5 above is resolved by
buying a COTS FOC controller rather than building one, which may be a relevant cost/effort
comparison point if the Herons EV is deciding whether to build vs. buy.

**Simulink validation.** A full closed-loop model (drive-cycle source → PID speed loop → torque
demand → Electric Drive Unit → vehicle dynamics ↔ battery SoC/voltage, their Figure 3) was run
against a custom 293 s SEM-track drive cycle. Results: **20.95 Wh/lap (45.8 Wh/km)**, SoC dropping
only 100%→97.8% per lap, confirming multi-lap range within the 1000 Wh budget. Benchmarked against
other 2025 SEM Urban Concept teams: top performers (SZEnergy, TIM UPS-INSA) reach below 5 Wh/km
(<~4 Wh/km), while this design's ~45.8 Wh/km sits in the "competitive mid-range," explicitly
framed by the authors as a cost/reproducibility-vs-peak-efficiency tradeoff appropriate for a
resource-constrained student team — a useful calibration point for where the Herons EV's own
energy-consumption target should realistically sit. The EDU (motor) block itself was simplified to
a **fixed operating point** (rated 48 V/1500 W/4.78 Nm/3000 rpm) rather than a full
torque-speed-current map, flagged explicitly as a limitation; regenerative braking, detailed
nonlinear motor maps, and thermal/wear effects were all excluded from the model and left as future
work requiring hardware-in-the-loop validation.

**Relevance:** this paper is less "a technique to adopt" and more "a sibling design to
cross-check against" — same competition category, same rough voltage class, comparable vehicle
mass, published BOM/costs (total budget GBP 13,500) and a validated 45.8 Wh/km baseline. If the
Herons EV's own drivetrain is a single-motor BLDC + chain drive, this paper's component choices,
gear-ratio-sizing method, and Simulink modeling structure (drive cycle → PID → EDU → vehicle
dynamics → battery, Section 4 of this paper's Figure 3) are a near-direct template; where the two
designs diverge (motor supplier/rating, chain vs. other transmission, custom vs. COTS controller)
is exactly where a comparative cost/efficiency case is easiest to make.

---

## 7. A Broader Taxonomy of Torque-Ripple Control Strategies

**Prabhu, Thirumalaivasan & Ashok, "Critical Review on Torque Ripple Sources and Mitigation
Control Strategies of BLDC Motors in Electric Vehicle Applications," IEEE Access 11 (2023),
Vellore Institute of Technology.**

A large (~240-reference) survey that reframes and substantially broadens Batch 1's Sections 2, 4,
and 5. Where Batch 1 covered one hardware approach (DC-bus boosting, Section 2), one
software/MPC approach (Section 4), and one head-to-head comparison (trapezoidal vs. FOC, Section
5), this paper places all of that inside a **7-family taxonomy** of BLDC torque-ripple control,
each with its own comparison table of dozens of individually cited techniques (their Tables 1–7,
covering roughly 150 of the paper's 240 references):

- **Field-oriented control (FOC)** — Table 1, ~25 technique variants (novel flux estimation,
  fault-tolerant FOC, dither-signal injection for backlash, wavelet-controlled FOC, FOC+ANN,
  etc.), reporting torque-ripple figures from sub-1% THD up to several percent depending on
  technique and operating point.
- **Direct torque control (DTC)** — Table 2, ~24 variants (hysteresis-based, SVPWM-combined,
  sensorless via flux observers, firefly-algorithm-tuned PID, adaptive neuro-fuzzy, etc.).
- **Intelligent control** (fuzzy, ANN, ANFIS, PSO/BAT-optimized gains) — Table 3, ~20 variants.
- **Controlling input voltage (CIV)** — Table 4, ~10 variants, the same DC-bus-boost family as
  Batch 1 Section 2, generalized: Cuk converters, PR compensators, back-EMF wave shaping, varying
  input voltage via buck-converter duty cycle.
- **Current shaping techniques** — Table 5, ~15 variants built around current hysteresis control
  (CHC) and duty-cycle modification (PWM-ON-PWM, three-segment modulation, adaptive soft-start),
  directly overlapping with Batch 1 Section 4's duty-cycle half of its FCS-MPC method.
- **Model predictive control (MPC)** — Table 6, ~22 variants, including finite-control-set MPC
  (the same family as Batch 1 Section 4), deadbeat current control, nonlinear MPC, and four-quadrant
  MPC; this table is the most direct extension of Batch 1 Section 4, situating that paper's
  specific technique among ~20 published alternatives with the same underlying approach.
- **Sliding mode control (SMC)** — Table 7, ~9 variants (adaptive SMC, fuzzy-SMC, SMC-SMO,
  SMC-MPC hybrids), a control family not otherwise covered in Batch 1.

**Cross-family comparison (their Section VII):** FOC needs accurate rotor-orientation
transformation and is parameter-sensitive (stator/rotor resistance drifts with temperature); DTC
avoids the coordinate transform and has a better transient torque response (faster settling, less
overshoot) at the cost of slightly higher steady-state ripple than FOC; **MPC is reported as
generally outperforming FOC and DTC** on torque ripple, current/torque pulsations, and THD, at the
cost of higher computational burden per control cycle — which the authors note is increasingly
offset by deep-learning/ANN-assisted implementations; SMC is robust to parameter uncertainty and
external disturbance but its chattering effect is a known drawback, addressed in the literature by
fuzzy-SMC and SMC+MPC hybrids. The paper's Figure 13 gives a compact pros/cons wheel diagram for
all 7 families.

**Section VIII (a genuine novelty of this paper)** sketches **cloud-based torque control**: motor
sensor data (currents, terminal voltages, vibration, temperature) streamed via IoT to a
cloud/AI-based condition-monitoring system for real-time defect detection and predictive
maintenance — speculative and not something a competition vehicle would implement, but flagged
here since it's a direction the authors present as where BLDC EV control is heading.

**Relevance:** this paper doesn't replace Batch 1 Sections 2/4/5, it contextualizes them — Batch
1's specific FCS-MPC paper (Section 4) is one entry in this review's ~22-entry MPC table, and
Batch 1's DC-bus-boost topologies (Section 2) are a subset of this review's 10-entry CIV table.
The main new decision-relevant input for the Herons EV is the **cross-family verdict** (Section
VII, above): if the drive controller design gets past the FOC-vs-trapezoidal question (Batch 1
Section 5) and needs a specific ripple-suppression technique, this review's ranking — MPC best on
ripple/THD but heaviest computationally, DTC best on transient response, SMC most robust to
parameter uncertainty but prone to chattering — is a reasonable starting shortlist, to be weighed
against whatever compute headroom the target MCU has (echoing the STM32-based off-the-shelf FOC
controller Section 6 above ended up using rather than a from-scratch design).

---

## 8. Minimal-Hardware Commutation Logic — a From-Scratch Two-Phase Design

**Hazari, Jahan, Siraj, Khan & Saleque, "Design of a Brushless DC (BLDC) Motor Controller,"
ICEEICT 2014, American International University-Bangladesh.**

Where Batch 1's papers all assume six-step commutation logic as a given, this paper derives it
from first principles for a **two-phase** BLDC motor (phases A and B rather than the usual
three-phase A/B/C) and implements the result two ways: as pure digital logic gates, and on a cheap
general-purpose microcontroller (Atmega32).

**Derivation.** Starting from the physical picture — each phase has two terminals (F/start, S/end)
and current direction through a phase sets where a stator pole forms — the paper walks through 4
rotor positions (0°/45°/90°/135°, the pattern then repeating with polarity reversed for
135°–360°) showing which of each phase's 4 switches (S1–S4) must be open/closed to advance the
rotor by 45° at a time, tabulated fully in their Tables I–III and reduced to a compact 8-row truth
table (their Table IV) over 4 Hall-effect-style sensors (C1–C4) and a user-set CW/CCW direction
bit, producing 4 output drive signals (A14, A23, B14, B23).

**Two implementations of the same truth table:**
1. **Pure combinational logic** (their Figure 12): sum-of-products expressions derived directly
   from Table IV and built from AND/OR/NOT gates in DSCH-2 — no processor at all.
2. **Microcontroller** (Atmega32, simulated in Proteus): the same truth table implemented in
   firmware, with LEDs standing in for the stator windings in simulation (sensors were driven by
   manual switches / a second microcontroller generating synthetic Hall signals, due to hardware
   availability limits) — simulated output waveforms (their Figure 14) matched the theoretical
   switching waveform derived in Section III.

**Cost breakdown** (their Table V): 8× MOSFET ($1.5), 4× Hall sensor ($3), 1× Atmega32 ($2.90), 1×
PCB ($1.25) = **$8.62 total**, vs. $14–18 for commercial BLDC controllers at the time — roughly
50% cheaper, attributed entirely to using a general-purpose MCU plus discrete logic instead of a
purpose-built motor-control IC.

**Relevance:** the two-phase case is not what the Herons EV would use (three-phase is standard and
is what every other paper in this review assumes), but the **method** — walk through every rotor
position by hand, derive the exact switch states needed, reduce to a truth table, then implement
that table either in gates or in firmware — is a legitimate from-scratch design path for a
three-phase equivalent if the team ever needs to build commutation logic without a
dedicated motor-driver IC (e.g., a backup/bring-up path, or a teaching exercise before adopting a
COTS controller as Section 6's Coventry team did). The cost table is also a useful sanity check on
how cheap a bare-minimum commutation implementation can be, as a lower bound against whatever the
Herons EV's actual controller costs.

---

## 9. PWM Generation Without Dedicated PWM Hardware

**Kim, Toliyat, Panahi & Kim, "BLDC Motor Control Algorithm for Low-Cost Industrial Applications,"
IEEE 2007, Texas A&M University / University of Texas at Dallas / Yeungnam College.**

A specific, reusable trick for driving a three-phase BLDC motor from a microcontroller that has
**only one on-chip timer and no dedicated multi-channel PWM peripheral** — relevant to any
low-cost MCU choice where a full motor-control-specific chip (with 6 built-in PWM channels) isn't
available or is judged not worth its cost premium.

**The key enabling fact:** in standard 120°-conduction six-step BLDC commutation, only 2 of 6
switches are ever active at once (one high-side, one low-side, per Section III of Batch 1 Section
1's discussion) — and critically, **the high-side and low-side switches of the same inverter leg
are never both on at the same PWM cycle** by construction of the six-step pattern. This means,
unlike a general three-phase inverter, **BLDC commutation needs no dead-time** between switching a
leg's high and low side — which is what makes it safe to generate the PWM in software via general
I/O toggling rather than dedicated complementary-PWM hardware (which exists specifically to
enforce dead-time).

**Implementation (MSP430F123, "asymmetric PWM" strategy):** Timer_A is configured in up-mode with
a fixed-period register (CCR0 = 0x200, i.e. constant switching frequency) and a duty-ratio register
(CCR1) updated every cycle by software. Two interrupts drive the scheme: the Timer_A **overflow**
interrupt marks the start of each PWM period and turns the active P3.x output pin on; the
**CCR1 compare** interrupt fires mid-period at the commanded duty point and turns it off — i.e.
the "PWM channel" is synthesized entirely from two interrupt handlers toggling a plain digital
output pin, with the six-step commutation pattern itself (which of P3.0–P3.5 is the "active" PWM
pin at any given moment) selected by a separate Hall-sensor-driven state machine. A third interrupt
(ADC10 end-of-conversion) samples DC-link current for a proportional current controller that
adjusts the duty cycle every cycle to track a current reference set by the user (push-button
increase/decrease, toggle-switch direction).

**Resource footprint and cost:** the entire algorithm (commutation state machine + PWM generation
+ current control) fit in **422 bytes of flash**, needing no external timer or memory devices. The
paper's Table 1 compares the MSP430F123 (no on-chip PWM, $2.30/1k units) against three
motor-control-oriented parts with on-chip PWM generation (TMS320F2401A $3.50, ST7MC1K2 $3.30,
56F8013 $3.15) — **using the general-purpose part with this software-PWM technique cuts MCU cost
by about 37%** relative to the cheapest PWM-capable alternative, with no other component
differences. Oscilloscope captures (their Figs. 8–9) confirm clean, correctly-timed phase current
and PWM waveforms with no shoot-through, validating the no-dead-time claim experimentally.

**Relevance:** directly actionable if the Herons EV's drive controller is (or could be) built
around a cheap general-purpose MCU rather than a motor-control-specific part — the "BLDC needs no
dead-time, so PWM can be synthesized from general I/O plus one timer's overflow/compare
interrupts" trick is exactly the kind of BOM-cost reduction Section 8 above pursued through a
different route (discrete logic instead of software PWM). It's also a useful complement to Section
3's (Batch 1) root-locus PI speed loop, which was implemented on a part (LPC1549) that *does* have
a hardware six-step commutation state machine — this paper shows what the same control loop looks
like on hardware one tier cheaper, at the cost of more interrupt-handler complexity in software.

---

## 10. Synthesis (Batches 1–2)

Across all nine papers, three threads recur:

- **The 4×-BEMF commutation condition** (Batch 1 Sections 2 and 4) is the unifying quantitative
  fact behind commutation torque ripple in six-step trapezoidal BLDC drives; Batch 2 Section 7
  shows this is just one branch (the "controlling input voltage" family) of a much larger
  published taxonomy of ripple-suppression techniques, with MPC (Batch 1 Section 4's family)
  reported as the strongest performer on ripple/THD at the highest computational cost, and DTC and
  SMC as the leading alternatives on transient response and parameter robustness respectively.
- **Simplicity vs. smoothness/cost is the recurring tradeoff**, and Batch 2 adds two concrete data
  points at the *cheap* end of that spectrum that Batch 1 didn't have: a from-scratch commutation
  design costing $8.62 in parts (Section 8), and a software-PWM technique cutting MCU cost ~37%
  by avoiding a dedicated PWM peripheral entirely (Section 9). Both are ways of buying back some
  of trapezoidal commutation's cost advantage over FOC (Batch 1 Section 5) at the hardware layer,
  as opposed to Batch 1 Section 4 and Batch 2 Section 7's software/control-algorithm approaches to
  buying back some of trapezoidal's *smoothness* disadvantage.
- **Build vs. buy is now a visible fork**, thanks to Section 6: a real competition team in the same
  category as the Herons EV, facing the same FOC-vs-trapezoidal and custom-vs-COTS questions this
  review has been assembling literature for, resolved it by buying an off-the-shelf FOC controller
  (Sabvoton SVMC72150) rather than building either a trapezoidal or FOC drive from scratch — which
  reframes Sections 1–5, 7–9 of this review as most useful either for *evaluating* whether a COTS
  controller is doing something reasonable, or for a from-scratch build if the team decides
  against the COTS route.

**Open questions carried forward:** what commutation/control architecture (if any) the Herons EV's
actual drive electronics currently use, and specifically whether it's a COTS controller (as in
Section 6) or a custom design; if custom, whether it would be better served by Batch 1 Section 3's
simple PI blueprint, Batch 1 Section 5's smoother-but-heavier FOC, or one of Section 7's ranked
alternatives (MPC for best ripple performance, DTC for best transient response), given the
platform's MCU resources; and how the Herons EV's own energy-consumption figures compare to
Section 6's 45.8 Wh/km baseline and the wider SEM Urban Concept field it cites (<5 Wh/km to ~20
Wh/km).
