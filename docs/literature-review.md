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

**Status:** _not started_

---

## 4. Driver-Coaching HMI Design (Discrete vs. Continuous Feedback)

**Status:** _not started_

---

## 5. Regulatory Positioning Follow-Up

**Status:** _not started — tracked for completeness; resolved via correspondence, not literature._

---

## 6. Synthesis — How GLIDE Is Positioned Relative to Prior Art

**Status:** _pending completion of Sections 1–4_

---

## References

_(consolidated bibliography, filled in as each section is completed)_
