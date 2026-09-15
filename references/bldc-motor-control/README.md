# References — BLDC Motor Control Review

Source papers cited in `docs/bldc-motor-control-review/bldc-motor-control-review.md`. Kept
separate from `references/papers/` (which backs the first literature review, GLIDE feedback
system) since this is an independent review on BLDC motor control.

## `papers/`

PDFs of cited papers, for offline reference. Naming convention: `Author(s) - Short Title.pdf`.

Batch 1 (5 papers, read in full):

- `Su & McKeever - Low-Cost Sensorless Control of BLDC Motors with Improved Speed Range.pdf` —
  IEEE 2002, Oak Ridge National Laboratory. Single-terminal-voltage back-EMF sensing with
  look-up-table phase-delay correction.
- `Karthika & Nisha - Review on Torque Ripple Reduction Techniques of BLDC Motor.pdf` — IEEE
  ICICT 2020. Survey of commutation-torque-ripple-reduction topologies (Cuk converter, charged
  capacitor, DVM scheme); derives the "DC bus must be ≥4×back-EMF" condition.
- `Rad et al. - Embedded Speed Control of BLDC Motors Using LPC1549 Microcontroller.pdf` — IEEE
  2016, Technical University of Cluj-Napoca. Root-locus-tuned PI speed loop on a hardware
  state-configurable timer, with software Hall-sensor calibration.
- `Li et al. - Torque Ripple Suppression of BLDCM With Optimal Duty Cycle and Switch State by
  FCS-MPC.pdf` — IEEE Open Journal of Power Electronics, 2024. Finite-control-set model
  predictive control (duty cycle + switch-state insertion) to hold non-commutation phase current
  constant during commutation.
- `Nurtriartono et al. - Performance Comparison of BLDC Motor Controllers, Trapezoidal vs
  FOC.pdf` — AIP Conf. Proc. 2019, ITS Surabaya. Experimental comparison of six-step trapezoidal
  commutation vs. field-oriented control on the same 30 kW BLDC motor.

Batch 2 (4 papers, read in full):

- `Hazizi et al. - Design, Construction and Simulation-Based Validation of a High-Efficiency
  Electric Powertrain for a Shell Eco-marathon Urban Concept Vehicle.pdf` — Designs (MDPI) 2025,
  Coventry University. Full powertrain case study, same SEM Battery Electric rules as the Herons
  EV but the **Urban Concept** category (the Herons EV races **Prototype**) — see the category
  caveat in Section 6 of the review before treating its numbers as directly comparable. BLDC motor
  selection, two-stage chain drive, custom 12S8P Li-ion pack, and a MATLAB/Simulink PID-driven
  vehicle model validated at ~45.8 Wh/km (a UC-category figure, not a Prototype-class target).
- `Prabhu, Thirumalaivasan & Ashok - Critical Review on Torque Ripple Sources and Mitigation
  Control Strategies of BLDC Motors in EV Applications.pdf` — IEEE Access, 2023, Vellore
  Institute of Technology. Large-scale survey (~240 references) taxonomizing BLDC torque-ripple
  control into 7 families (FOC, DTC, intelligent/fuzzy/ANN, controlling input voltage, current
  shaping, MPC, SMC) with comparison tables of dozens of specific published techniques each.
- `Hazari et al. - Design of a Brushless DC (BLDC) Motor Controller.pdf` — ICEEICT 2014, American
  International University-Bangladesh. From-scratch derivation of two-phase BLDC commutation
  switching logic, implemented as both a digital logic circuit and an ATmega32 controller;
  reports an $8.62 total component cost (~50% below commercial BLDC controllers).
- `Kim, Toliyat, Panahi & Kim - BLDC Motor Control Algorithm for Low-Cost Industrial
  Applications.pdf` — IEEE 2007, Texas A&M / UT Dallas / Yeungnam College. Generates three-phase
  six-step PWM using general I/O pins and a single on-chip timer (MSP430F123) instead of a
  dedicated multi-channel PWM peripheral, exploiting BLDC's complementary switching to avoid
  needing dead-time; reports ~37% MCU cost reduction vs. parts with on-chip PWM generation.

Batch 3 (1 paper, read in full — closes out the currently planned set):

- `Surakasi et al. - A Novel Methodology to Enhance the Smooth Running of the PM BLDC Motor Drive
  Using PWM-PWM Logic and Advance Angle Method.pdf` — Machines (MDPI) 2023, Anil Neerukonda
  Institute of Technology / Aditya Institute of Technology & Management / Alexandria University /
  Cardiff University. Advance-angle (lead-angle) compensation for inductance-caused current lag,
  plus a synchronous-rectification "PWM-PWM" commutation scheme with K-map-derived suppression
  logic for the extra reverse-current noise it introduces; validated on a 470 V / ~100 Nm
  industrial BLDC motor in simulation and on real hardware.
