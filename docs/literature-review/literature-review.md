# GLIDE — Literature Review / State of the Art

**Guided Live Indicator for Driving Efficiency**
Unisabana Herons EV — Shell Eco-marathon Prototype, Battery Electric

Status: living document, built section by section. See `GLIDE_Handover.md` (Section 6) for the
original open items driving this review.

---

## 0. Scope and Method

GLIDE is a PID-driven, real-time driver feedback system that compares actual vs. reference energy
consumption and outputs a discrete LED instruction (coast/maintain/brake). This review positions
GLIDE against existing work in five areas:

1. Motorsport energy-delta feedback systems (Formula E, F1, sim racing) — comparanda already
   informally identified; need formal sourcing.
2. Pulse-and-glide / hypermiling eco-driving research — **priority area**, likely richest and most
   directly transferable literature.
3. PID / control-based real-time energy management in student competition vehicles (Shell
   Eco-marathon, SAE, Formula Student Electric).
4. Driver-coaching HMI design — discrete vs. continuous feedback, response time / error rate.
5. Regulatory positioning (not literature — tracked here for completeness, resolved via direct
   correspondence with Shell Eco-marathon, see handover Section 2).

Each section below records: what was searched, what was found (with citations), how it relates to
GLIDE's design choices, and open gaps.

---

## 1. Pulse-and-Glide / Hypermiling Eco-Driving Research

**Status:** complete (deep-dive on the 3 priority papers; survey-level on the rest)

Network note: `arxiv.org` and `iopscience.iop.org` became reachable mid-session once the
environment's outbound network policy was widened; `sciencedirect.com` and `researchgate.net`
remain blocked (403 at the proxy CONNECT level). The 3 papers below were downloaded directly via
`curl` and read in full (`WebFetch` itself still refuses these hosts under a separate, narrower
policy than the general egress proxy — worth flagging if deeper fetches are needed later).

### Definition and general savings

Pulse-and-glide (PnG) is a cyclic driving technique: a *pulse* phase (accelerate at a fixed power
level) followed by a *glide* phase (motor off/decoupled, vehicle coasts down to a lower speed
threshold, then repeats). General automotive literature reports savings up to ~5% vs.
constant-speed driving, conditional on the motor actually being deactivated during the glide phase
— Springer, *Int. J. Precision Eng. Manuf.-Green Tech.* (2023), ["Energy-Saving Strategy for Speed
Cruise Control Using Pulse and Glide
Driving"](https://link.springer.com/article/10.1007/s40684-023-00516-5); ScienceDirect, ["Pulse and
glide strategy analysis based on engine operating point during pulse
mode"](https://www.sciencedirect.com/science/article/abs/pii/S094735802200022X) (not fetched in
full — sciencedirect.com blocked).

### Deep-dive 1 — Whitehair, Denlinger & Fathy, *"Pulse-and-Glide Driving with Drivability
Constraints: A Pontryagin Approach"*, AVEC'18, Penn State ([arXiv:2012.11435](https://arxiv.org/pdf/2012.11435))

Treats PnG as a formal optimal-control problem rather than a heuristic. Vehicle model: states
$x_1$ = velocity, $x_2$ = propulsive force, input $u$ = jerk (rate of change of propulsive force):

$$\dot{x}_1 = \tfrac{1}{M}\left(x_2 - \tfrac12 \rho C_d A_f x_1^2 - \mu M g\right), \qquad \dot{x}_2 = u$$

Fuel rate is modeled as power × a quadratic brake-specific-fuel-consumption (BSFC) curve
$\beta(P) = \beta_0 + \tfrac{\gamma}{2}(P-P_0)^2$, $P = x_1 x_2$. The optimization objective
linearly trades off fuel consumption against average speed with a soft quadratic penalty $R$ on
jerk:

$$J(u,x,T) = \tfrac1T \int_0^T \left[\dot m_f(x) - C x_1 + \tfrac12 R u^2\right] dt$$

subject to periodicity ($x_i(0)=x_i(T)$). Applying the Pontryagin Minimum Principle and
linearizing around an equilibrium nominal speed gives a characteristic equation whose root locus
(parametric in $R$) shows: **for every nominal speed there is a critical jerk-penalty $R_{crit}$
below which the optimal trajectory is oscillatory (PnG) rather than steady-speed**, and **above a
critical nominal velocity (33.8 m/s ≈ 122 km/h for their test vehicle, a 1991 Dodge Caravan) PnG
can never beat constant-speed driving, regardless of $R$.** Nonlinear simulation (via `fmincon`,
approximating the optimal input as a 6-harmonic Fourier series) validates the linear analysis and
finds a PnG cost function $J=-0.3684$ vs. $J_{ss}=-0.2324$ for steady-speed at the same nominal
15 m/s (54 km/h) — i.e. PnG measurably outperforms constant speed in the combined
fuel/speed/jerk objective, not just raw fuel.

**Relevance to GLIDE:** the Herons EV's competition speed (~25 km/h per Section 4 of the handover)
is far below the paper's 122 km/h critical velocity — so PnG is very likely to remain
theoretically optimal at Eco-marathon speeds, i.e. GLIDE's control target is well inside the
regime where the technique should work. Also useful for a future PID gain-tuning discussion: the
critical-$R$ concept ("how big a jerk penalty before oscillation stops paying off") maps onto how
aggressively GLIDE's PID should react to error before it starts hurting drivability/comfort.

### Deep-dive 2 — Gechev & Punov, *"Driving strategy for minimal energy consumption of an
ultra-energy-efficient vehicle in Shell Eco-marathon competition"*, IOP Conf. Ser.: Mater. Sci.
Eng. 1002 (2020) 012018, Technical University of Sofia
([open access](https://iopscience.iop.org/article/10.1088/1757-899X/1002/1/012018), CC BY 3.0)

**Directly on point** — same competition (Shell Eco-marathon Urban Concept, hydrogen fuel cell,
1420 m track, SEM Europe 2019), same underlying vehicle-dynamics equations GLIDE will need
($F_{tr}=F_i+F_{roll}+F_{aero}$, motor torque/efficiency model, etc.). The paper proposes a
**standard driving strategy** decomposed into exactly the kind of segment-based structure GLIDE's
planned non-uniform reference profile (handover Section 4, item 4) will need:

1. **Initial Acceleration Phase (IAP)** from $V_{start}=0$ to $V_{max}$.
2. **Acceleration-Coasting Field (ACF)** — a repeating plateau: accelerate to $V_{max}$, coast down
   to a minimal no-load speed $V_{min}^{nl}$, repeat — this *is* pulse-and-glide, parameterized by
   the gap between $V_{max}$ and $V_{min}^{nl}$.
3. **Final phase** — coast early, brake only just before the finish line (shown better than braking
   alone, since it banks the ACF's accumulated kinetic energy).

36 strategies were simulated varying transmission ratio, motor count, and the $V_{max}/V_{min}^{nl}$
band. Quantitative findings: higher average plateau speed → lower energy (up to ~8% difference);
smaller $V_{max}-V_{min}^{nl}$ range → ~6.5% lower energy but more frequent
accelerations; 2-motor IAP reaches peak motor efficiency faster than 1-motor. Best strategy found:
19,224.69 J/lap (vs. 20,920 J/lap baseline, ~8.1% saving).

**The critical line for GLIDE's positioning (Section 6):** the authors themselves flag their own
strategy's weakness —

> "there are more frequent accelerations during ACF, which could make it more difficult to
> implement the strategy by the pilot in real conditions"

This is exactly the gap GLIDE is built to close: an optimal ACF/PnG strategy exists and is
quantified in the SEM literature, but executing its precise $V_{max}/V_{min}^{nl}$ switching points
by feel is acknowledged by this paper as hard for a human driver. GLIDE's LED instruction is a
direct, purpose-built solution to that stated limitation.

### Deep-dive 3 — Tian, Liu & Shi, *"A Pulse-and-Glide-driven Adaptive Cruise Control System for
Electric Vehicle"*, Wayne State University ([arXiv:2205.08682](https://arxiv.org/pdf/2205.08682))

Embeds PnG as a mode parallel to cruise control inside a full ACC system (PGACCS) for a simulated
D-class EV (CarSim + Simulink). Two results matter most for GLIDE:

1. **Speed control is done with a plain PI controller** — precedent directly supporting GLIDE's
   own PID choice for the analogous problem (tracking a target against a live measurement):
   $$e(t) = V_{des}(t) - V_{real}(t), \qquad u(t) = K_p e(t) + \tfrac{1}{T_i}\int_0^\tau e(t)\,dt$$
   The paper explicitly justifies PI/PID as the pragmatic choice over MPC/fuzzy/sliding-mode for
   "intelligible mechanism and strong robustness" with "short computation time" — the same
   trade-off that justifies GLIDE running PID on an Arduino rather than a heavier controller.
2. Optimized PnG (acceleration/deceleration values found via a genetic-algorithm/PSO hybrid) cut
   battery SOC cost by **28.3%** vs. constant-speed cruise control over a 5 km trip.
3. **Regenerative braking made PnG worse, not better**, on this vehicle: forcing regen during the
   glide phase raised the SOC cost above both the optimal (regen-free) PnG and even above plain
   constant-speed cruising, because motor+battery round-trip conversion losses outweigh the
   recovered kinetic energy at this scale. The paper's own conclusion: "vehicle itself could keep
   more energy as the kinetic form than transforming it to the electrical form."

**Relevance to GLIDE:** point 3 is a direct, literature-backed argument for the Herons EV hardware
plan *not* including regenerative braking (handover Section 1 lists none) — for a small,
low-mass, low-speed-delta vehicle, pure glide (no regen) is not just simpler but plausibly more
energy-efficient than glide-with-regen. Point 1 gives GLIDE a citable precedent for "PID is the
right-sized controller for this problem," reusable directly in a methods/justification section of
GLIDE's own write-up.

### Other Shell Eco-marathon / fuel-cell driving-strategy papers (not deep-dived — sciencedirect.com/
researchgate.net blocked; abstract-level only)

- Wiley (2019), ["Design and energy efficiency analysis of a pure fuel cell vehicle for Shell eco
  racer"](https://onlinelibrary.wiley.com/doi/10.1002/er.4487).
- ScienceDirect, ["Multiphysics modeling and optimization of the driving strategy of a light duty
  fuel cell vehicle"](https://www.sciencedirect.com/science/article/abs/pii/S0360319917331336) —
  cited inside Gechev & Punov [17] as achieving 500 Wh/100km at 25 km/h average speed for an Urban
  Concept vehicle; same order of magnitude as the Herons EV's own ~25 Wh/8.53 km budget once
  normalized, worth a direct numeric comparison later.
- ["Establishing an optimal eco-driving strategy for an electric vehicle through testbed
  simulation — A case study from Shell Eco-Marathon
  2018"](https://www.researchgate.net/publication/333520581) — researchgate.net blocked, abstract
  only.

### Gap identified (feeds Section 6)

Across this literature, PnG/ACF strategies are either mathematically optimized offline (Whitehair
et al., Gechev & Punov) or executed via closed-loop automatic control (Tian et al.'s PGACCS) —
and in the one paper from the *same competition* as GLIDE (Gechev & Punov), the authors explicitly
note their optimal strategy is hard for a human pilot to execute precisely. **None of these papers
put a human driver in the loop receiving a live error signal to manually decide pulse/glide
timing.** That is precisely GLIDE's proposal: move optimal-control/ACF logic (already proven and
quantified in this literature) to a discrete instruction consumable by a human, instead of either
leaving the driver unaided or automating propulsion outright.

### Next steps for this section

- [ ] If deeper numeric comparison is wanted later, pull the Wiley Shell Eco Racer paper and the
  ScienceDirect multiphysics paper (both currently blocked — need `sciencedirect.com`/
  `onlinelibrary.wiley.com` allowlisted, or a manually supplied PDF).
- [ ] Check for any SEM/FSAE team technical reports (not peer-reviewed journals) describing
  in-cockpit PnG coaching — closest possible prior art, not yet found in this pass.

---

## 2. Motorsport Energy-Delta Feedback Systems (Formula E, F1, Sim Racing)

**Status:** draft complete

### Formula E

The steering wheel dash displays a live **energy target** for the upcoming lap plus a **+/- delta**
against that target; the driver decides how much to lift-and-coast or regenerate per corner to close
the gap [FIA Formula E — Anatomy of a Formula E steering wheel](https://www.fiaformulae.com/en/news/2020/july/formula-e-steering-wheel);
[FIA Formula E — Energy management: the key to Formula E strategy](https://www.fiaformulae.com/en/news/4099).
Regen is manually triggered via a paddle and can recapture up to ~75% of kinetic energy during
lift-and-coast. This is human-in-the-loop control over a live energy error signal — architecturally
almost identical to GLIDE, except Formula E uses a continuous numeric display where GLIDE uses a
discrete LED.

### Formula 1

The "fuel delta" concept (gap between planned and actual fuel consumption) and "lift and coast"
exist, but are communicated via pit-wall radio ("Target +2", discrete verbal instructions) rather
than a dedicated live driver-facing HMI
[RaceFans — 'Lift and coast': Why F1 drivers are told to save fuel](https://www.racefans.net/2015/06/11/lift-and-coast-why-f1-drivers-are-told-to-save-fuel/);
[F1 Briefing — F1 Fuel Thresholds & Race Strategy](https://f1briefing.com/how-f1-teams-set-fuel-usage-thresholds/).
Precedent for the *logic* (target-vs-actual delta triggering a driving instruction) but not for the
*interface* GLIDE proposes.

### Sim racing (iRacing / ACC endurance overlays)

Tools such as RaceLab, Track Impulse, and iRacingAutoPit compute live consumption and display
continuous overlays of remaining fuel / delta against the stint plan
[RaceLab](https://racelab.app/iracing/); [Track Impulse — Fuel Calculator](https://track-impulse.com/iracing-fuel-calculator).
Same signal shape as GLIDE (continuous delta consumed visually by the driver) but applied to
stint/pit-stop management rather than per-segment brake/coast decisions.

### Positioning (feeds Section 6)

All three precedents use a target-vs-actual energy error signal to drive human coast/regen
decisions — validating GLIDE's conceptual approach — but none uses a discrete 3-state indicator
(LED semaphore) instead of a continuous numeric display. Direct link to the open question in
Section 4 (discrete vs. continuous HMI).

---

## 3. PID / Control-Based Energy Management in Student Competition Vehicles

**Status:** draft complete

### Findings

- **MDPI, *"Design, Construction, and Simulation-Based Validation of a High-Efficiency Electric
  Powertrain for a Shell Eco-Marathon Urban Concept Vehicle"*** (2025/2026;
  [DOI-indexed](https://www.mdpi.com/2411-9660/9/5/113) — `mdpi.com` blocked by this environment's
  network policy, findings below are abstract/search-snippet level only). Same competition class as
  GLIDE. Powertrain: 1500 W / 48 V BLDC motor, custom 12S8P Li-ion pack. A Simulink vehicle-dynamics
  model runs a **PID controller that compares reference vs. actual velocity and outputs the torque
  demand sent directly to the motor** — i.e., the PID's output actuates the drivetrain. Reported
  consumption: <50 Wh/km, smooth tracking through acceleration/braking transitions.
- **Springer, *"A Novel PID Controller-Based Control Strategy for a Formula Student
  Vehicle"*** ([chapter page fetched in full; abstract confirmed, body paywalled](https://link.springer.com/chapter/10.1007/978-981-97-4895-2_15)).
  Parallel-hybrid FSAE vehicle (PM DC motor + ICE via chain drive). PID regulates motor input
  voltage so actual speed tracks target speed with minimal overshoot; gains tuned in
  MATLAB/Simulink via a transfer-function-based tuner. Again, **PID output drives the actuator
  directly** — no human in the loop.
- Cross-reference to Section 1's deep-dive on Tian, Liu & Shi (PGACCS): PI controller explicitly
  chosen over MPC/fuzzy/sliding-mode for being an "intelligible mechanism" with "strong robustness"
  and "short computation time" — the most citable justification found for running PID on
  lightweight embedded hardware (an Arduino, in GLIDE's case) rather than a heavier controller.
- **Methodological note:** a Cal Poly M.S. thesis (Bickel, 2017, *"Optimizing Control of Shell
  Eco-marathon Prototype Vehicle to Minimize Fuel Consumption"*) surfaced repeatedly in searches
  alongside the MDPI paper and looked promising (same competition, "optimizing control"). Downloaded
  and read in full (146 pages, via a scispace.com mirror since the official
  digitalcommons.calpoly.edu PDF is behind a Cloudflare bot challenge) — confirmed **zero mentions
  of PID**; it is actually an offline speed/gear-ratio optimization tool for a gasoline-powered
  prototype, unrelated to closed-loop control. Excluded to avoid mis-citing it (the search engine's
  auto-summary had conflated it with the MDPI paper).

### Design discussion: should GLIDE ever act directly on the powertrain?

Raised during this session by comparison to Formula 1 engine maps (Push/Hybrid/Save), which
throttle power delivery automatically once selected. Every PID precedent found in this section
(MDPI, Springer/FSAE, and Tian et al. in Section 1) uses PID output to **actuate the motor
directly** — none of them puts a human between the controller and the actuator. This raised the
question of whether GLIDE should do the same (an automatic power limiter) instead of, or in
addition to, the advisory LED.

**Decision: no — GLIDE stays advisory-only (LED instruction to the driver), not a direct
powertrain actuator.** Reasoning:

1. **Regulatory scope.** Shell Eco-marathon requires a *purpose-built motor controller* as part of
   the vehicle's already-homologated propulsion system (Art. 66a, see handover Section 2). A PID
   loop that outputs torque/power commands to the motor would make GLIDE part of that propulsion
   control system rather than an auxiliary circuit — materially raising the Technical Inspection
   burden (redundancy, fail-safe requirements) beyond what's currently scoped.
2. **Safety / failure mode.** A firmware bug in an advisory LED produces, worst case, a wrong
   color. A firmware bug in a system that gates motor power on-track (mid-corner, needing a burst
   to avoid a hazard, interacting with other vehicles) is a materially more dangerous single point
   of failure — much higher stakes than the F1 comparison suggests, since F1 engine maps are
   driver-selected and built/certified by the manufacturer as part of the homologated power unit,
   not retrofitted by a student team onto an existing vehicle.
3. **Positioning vs. prior art (feeds Section 6).** Automatic PID-actuates-motor control for
   PnG/ACF-style strategies is already a solved, published problem (MDPI, Springer, Tian et al.).
   GLIDE's actual novelty — identified as the gap across Sections 1–3 — is keeping a human in the
   loop with the same error signal. Moving to direct actuation would abandon that novelty and put
   GLIDE in direct competition with already-published, more mature automated solutions instead.

An engine-map-style automatic limiter remains a plausible **v2/future-work** idea, but as a
substantially different, higher-risk project (effectively replacing or integrating with the
vehicle's homologated motor controller) rather than an evolution of the current LED-advisory
scope.

### Conceptual cross-domain note: reactive vs. anticipatory control

A related, differently-applied control-theory framing worth carrying into Section 6: the
distinction between **reactive** controllers (PID — act only once a tracking error is already
measured) and **anticipatory** controllers (MPC and similar — act on a predicted future error
before it materializes). This split is standard in aerospace/hypersonic re-entry GN&C
literature, where PID is the low-cost reactive baseline and MPC/adaptive layers are added
specifically to act ahead of fast-changing disturbances. It's a useful lens for GLIDE: the whole
point of GLIDE's reference energy profile (handover Section 4) is to give the driver
*anticipatory* information (a precomputed target curve, indexed by distance) rather than
purely reactive feedback — i.e., GLIDE's PID error signal is computed against a plan that
already encodes upcoming corners/segments, which is conceptually closer to a predictive
scheme than to naive reactive control, even though the controller itself is a simple PID. Worth
revisiting when drafting Section 6, as a way to frame why a "simple" PID can still behave
anticipatorily if the setpoint itself is forward-looking.

(Source note: this framing was cross-checked against the reactive-vs-anticipatory distinction
as used generically in control theory and MPC literature; a specific re-entry-vehicle paper
surfaced during this session was not used as a citation here, since its reported results are
explicitly synthetic/unvalidated rather than experimental — only the general, well-established
control-theory concept is carried forward.)

---

## 4. Driver-Coaching HMI Design (Discrete vs. Continuous Feedback)

**Status:** draft complete

### Deep-dive 1 — Sanguinetti, "Onboard Feedback to Promote Eco-Driving: Average Impact and
Important Features", National Center for Sustainable Transportation / UC Davis white paper
(2018), and the underlying design framework Sanguinetti, Dombrovski & Sikand, *"Information,
timing, and display: A design-behavior framework for improving the effectiveness of
eco-feedback"*, Energy Research & Social Science (2018)

This is the most directly relevant source found in this review to date. The design framework
splits eco-feedback along three axes — **information, timing, display** — each tied to a
behavior-change mechanism (salience → attention, precision → learning, meaning →
motivation). One of its explicit sub-dimensions is **data granularity**: "the resolution of data
presented... numeric data typically have high data granularity, whereas **a light that changes
colors between green, yellow, and red has low data granularity**." That is a direct, named
description of GLIDE's LED semaphore design, used by this literature as the canonical example
of a *low-granularity* display.

The framework's own reasoning for when low granularity is the right choice maps closely onto
GLIDE's context: "greater granularity would be expected to support learning... However,
**ambient displays often call for reduced data granularity so that information can be absorbed
while the user is attending to some other task, such as driving**." A high-speed Eco-marathon
attempt, where the driver's primary task is precise track control, is exactly this case — an
argument for GLIDE's discrete LED over a continuous numeric delta, grounded in attentional
load rather than aesthetics.

The accompanying statistical meta-analysis (17 studies, 23 effect sizes, random-effects model)
found onboard eco-driving feedback produces an average **6.6% fuel-economy improvement**
(95% CI: 4.9–8.3%, p < .001) — a solid quantitative anchor for what a well-designed feedback
system can be expected to deliver, independent of GLIDE's own eventual field results. Of 14
pre-registered hypotheses about what makes feedback more effective, most (multi-modal
display, both fine- and coarse-grained information, feedback standards, gamification) trended
in the predicted direction but were **not** statistically significant at this sample size — with one
exception: effectiveness reliably **decays with intervention length** (longer exposure → smaller
effect), a caution relevant to any competition-season plan that assumes GLIDE's benefit stays
constant across the full practice-to-competition timeline. Directly relevant to GLIDE's own open
question (handover Section 4, item 4) about discrete-vs-continuous data granularity
specifically: the meta-analysis flagged this exact hypothesis (H4) but had **insufficient studies**
to test it empirically — this remains a genuine open gap in the literature, not just in GLIDE's
own review.

### Deep-dive 2 — Chada, Görges, Ebert, Teutsch & Puttige Subramanya, *"Evaluation of the
Driving Performance and User Acceptance of a Predictive Eco-Driving Assistance System for
Electric Vehicles"*, Transportation Research Part C (2022/2023), University of Kaiserslautern
([arXiv:2208.11429](https://arxiv.org/pdf/2208.11429))

A rigorous, real experimental study (not synthetic) — N=41 participants, six-DOF motion-platform
driving simulator (IPG CarMaker + SUMO traffic co-simulation), a parameterized Nissan Leaf BEV
model, MPC-based reference/car-following speed optimization, and formal TAM/TPB
statistical analysis of user acceptance.

Directly relevant to the discrete-vs-continuous question: pEDAS's HMI is itself a **hybrid**
design — a *continuous* arrow (length encodes how far off the optimal speed the driver is,
color encodes direction) combined with a *discrete* state change (digits turn from colored to
**black** when the driver enters a ±2 km/h "optimal band," at which point the advice becomes
categorical: stop touching the pedals and coast). This mirrors GLIDE's own planned
architecture almost exactly — a continuous underlying error signal collapsed into a small number
of discrete driver-facing states.

Results: participants achieved average energy savings of **9.82%** overall (11% on highway
segments), reduced speed-limit violations by ~46%, and reduced red-light stops by ~60%. Two
findings are particularly load-bearing for GLIDE's design decisions:

1. **A concrete failure mode of under-specified continuous feedback.** In participant debriefs,
   at least one driver reported difficulty interpreting the *arrow-length* continuous signal and,
   instead of coasting smoothly inside the optimal band as intended, ended up oscillating
   between throttle and brake — which *increased* jerk and energy consumption relative to no
   assistance at all. This is a directly documented instance of a continuous display creating
   worse outcomes than a clearer discrete instruction would have, for at least a subset of
   drivers — a concrete data point (not just theory) in favor of GLIDE's simpler 3-state approach.
2. **Perceived usefulness (not perceived ease of use) was the strongest predictor of intent to
   use the system** (TAM/TPB structural model, hierarchical regression), and **perceived
   behavioral control** was the strongest TPB predictor. Ease-of-interpretation matters mainly
   insofar as it feeds into perceived usefulness — supporting the idea that a system's clarity
   (discrete, unambiguous) is instrumentally important primarily because it makes the system's
   *usefulness* legible to the driver, not as an end in itself.

### Positioning for Section 6

Across both sources, the literature does not cleanly say "discrete beats continuous" or vice
versa — it says each has a distinct behavior-change mechanism (continuous → precision/fine
motor calibration; discrete → salience/low attentional cost) and that **the right choice depends
on how much spare attention the driver has**. For GLIDE's use case — a driver at speed on a
competition track, where the primary task is vehicle control, not fuel-economy
micromanagement — the literature's own reasoning (ambient/ low-granularity displays for
divided-attention contexts) directly supports the discrete LED choice already made on
regulatory grounds (handover Section 2). It also surfaces a concrete, real risk in the *rejected*
alternative (a continuous numeric/arrow display): documented cases of drivers misinterpreting
continuous magnitude cues and behaving worse than with no assistance. This is probably the
strongest, most literature-grounded argument found so far for GLIDE's HMI choice.

### Next steps for this section

- [ ] If useful later, look specifically for eco-driving HMI studies inside motorsport/high-speed
  contexts (karting shift-lights, sim-racing delta bars under time pressure) rather than
  commuter-car studies, to further validate the "divided attention" argument at racing speeds
  rather than urban traffic speeds.

---

## 5. Regulatory Positioning Follow-Up

**Status:** partial — Art. 230d resolved by design; Art. 57i classification question deliberately
left open (not literature-resolvable; requires direct correspondence with Shell Eco-marathon,
tracked here for completeness).

### Art. 230d (no manipulation of onboard telemetry between finish line and Technical
Inspection) — resolved by design

**Decision (this session):** add an SD card module + RTC (real-time clock, e.g. DS3231) to
GLIDE's Arduino, and a physical pushbutton on the steering wheel, with the following behavior:

- Every attempt is logged to its own timestamped file on the SD card (distance, instantaneous
  power, cumulative energy, error vs. reference profile, LED state over time). The RTC provides a
  real wall-clock timestamp that survives power cycles — `millis()` alone would not, since it
  resets on every power-up and can't distinguish sessions.
- The "live" accumulators (current attempt's running distance/energy) are **not** reset
  automatically. They persist until the driver presses the steering-wheel pushbutton, which
  closes out the current log file (writing a final summary line) and zeroes the accumulators for
  the next attempt.
- **Compliance condition (team protocol, not just hardware):** the pushbutton must only be
  pressed *after* the Technical Team has released the vehicle from its post-attempt inspection —
  never between crossing the finish line and that release. Art. 230d's prohibited window is
  specifically *finish line → Technical Inspection*, not *finish line → forever*; since the
  completed attempt's log is already written to SD with an immutable timestamp before anyone
  touches the button, the just-finished attempt's data is available for inspection regardless of
  when the button is later pressed.
- This should be stated explicitly in the electrical documentation submitted under Art. 67c — the
  team pre-declaring its own "don't touch the button until cleared" procedure strengthens the
  compliance position by showing the constraint was designed in, not improvised after the fact.
- Side benefit (not the primary motivation, but worth noting): per-attempt SD logs with real
  timestamps are exactly the telemetry the handover's calibration plan (Section 4, step 4) needs
  to build the non-uniform, segment-based reference profile from practice-session data — this
  hardware addition serves both the regulatory-compliance goal and the reference-profile
  refinement goal at once.

### Art. 57i (driver-display exemption classification) — deliberately left open

Per the handover's existing analysis: the Arduino+LED system most likely does not qualify for
the "unmodified, self-contained" driver-display exemption, and the safer position is to treat
GLIDE as part of the vehicle's electrical circuit (joulemeter-metered, fused, documented). The
question of whether Shell Eco-marathon organizers would rule differently if asked directly
remains **open on purpose** — not being resolved in this session. Next step, whenever the team
is ready to pursue it, is still to email shellecomarathon@shell.com per the handover's original
recommendation (Section 2).

---

## 6. Synthesis — How GLIDE Is Positioned Relative to Prior Art

**Status:** _pending completion of Sections 1–4_

---

## References

_(consolidated bibliography, filled in as each section is completed)_
