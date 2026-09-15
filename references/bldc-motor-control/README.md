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

6 more papers to be added in a second batch.
