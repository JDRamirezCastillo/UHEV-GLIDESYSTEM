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

**Status:** partial — survey-level pass done from abstracts/search snippets. `WebFetch` to
arxiv.org, sciencedirect.com, researchgate.net and iopscience.iop.org is blocked by this
environment's outbound network policy (403 at the proxy CONNECT level), so no full-text extraction
of equations or numeric results yet. Revisit once the environment's network policy is widened, or
paste in full text/PDFs of the priority papers below.

### Definition and measured savings

Pulse-and-glide (PnG) is a cyclic driving technique: a *pulse* phase (accelerate at a fixed power
level) followed by a *glide* phase (motor off/decoupled, vehicle coasts to a lower speed threshold,
then repeats). Reported savings vs. constant-speed driving are up to ~5%, conditional on the motor
actually being deactivated during the glide phase — Springer, *Int. J. Precision Eng.
Manuf.-Green Tech.* (2023), ["Energy-Saving Strategy for Speed Cruise Control Using Pulse and Glide
Driving"](https://link.springer.com/article/10.1007/s40684-023-00516-5); ScienceDirect, ["Pulse and
glide strategy analysis based on engine operating point during pulse
mode"](https://www.sciencedirect.com/science/article/abs/pii/S094735802200022X).

### Shell Eco-marathon-specific papers (highest relevance — same competition)

- ["Establishing an optimal eco-driving strategy for an electric vehicle through testbed
  simulation — A case study from Shell Eco-Marathon
  2018"](https://www.researchgate.net/publication/333520581) — validates PnG via testbed
  simulation for a SEM electric vehicle; motor fully shut off during glide, energy flow driven to
  zero.
- ["Driving strategy for minimal energy consumption of an ultra-energy-efficient vehicle in Shell
  Eco-marathon competition"](https://iopscience.iop.org/article/10.1088/1757-899X/1002/1/012018)
  (IOPscience, 2020) — hydrogen fuel-cell vehicle (1 kW), 1420 m track (SEM Europe 2019); compares
  5 propulsion schemes/driving strategies via vehicle-dynamics + real motor-efficiency simulation.
- Related Shell Eco Racer fuel-cell vehicle papers — Wiley (2019) ["Design and energy efficiency
  analysis of a pure fuel cell vehicle for Shell eco
  racer"](https://onlinelibrary.wiley.com/doi/10.1002/er.4487); ScienceDirect ["Multiphysics
  modeling and optimization of the driving strategy of a light duty fuel cell
  vehicle"](https://www.sciencedirect.com/science/article/abs/pii/S0360319917331336) — same domain,
  energy modeling focus rather than driver feedback.

### Formal optimal-control treatments (relevant to justifying GLIDE's PID choice)

- arXiv 2012.11435, ["Pulse-and-Glide Driving with Drivability Constraints: A Pontryagin
  Approach"](https://arxiv.org/pdf/2012.11435) — treats PnG as a Pontryagin optimal-control problem,
  not just a heuristic.
- arXiv 2205.08682, ["A Pulse-and-Glide-driven Adaptive Cruise Control System for Electric
  Vehicle"](https://arxiv.org/pdf/2205.08682) — implements PnG inside a closed-loop adaptive cruise
  control system; direct precedent for "automatic control executing PnG," though here the system
  acts on the vehicle rather than instructing a human.

### Gap identified (feeds Section 6)

Across this literature, PnG is either mathematically optimized offline or executed via closed-loop
automatic control (ACC). **None of these papers put a human driver in the loop receiving a live
error signal to manually decide pulse/glide timing.** That is precisely GLIDE's proposal: move
optimal-control logic to a discrete instruction consumable by a human, instead of automating
propulsion.

### Next steps for this section

- [ ] Extract exact numeric savings and the control formulation (equations) from the Pontryagin
  paper (arXiv 2012.11435) and the Shell Eco-marathon IOPscience paper — pending network access or
  user-provided PDF/text.
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
