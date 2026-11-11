# METS Fluid Model

Steady-state fluid network model for **METS** (Mini Engine Test Stand), a small wall-mounted GOX / kerosene test stand used to hot-fire small combustion devices — currently the **Hopper torch igniter** — and to train members in feed system design, analysis, and testing.

The model answers two questions about the stand's feed system:

1. **Design:** what do the regulators need to be set to in order to hit a test article's target mass flows, mixture ratio, and chamber pressure?
2. **Prediction:** given a set of regulator setpoints, what mass flows, MR, Pc, and line pressures will the stand actually deliver?

## Scope

| | Current (`fluid_model_NEW.ipynb`) |
|---|---|
| Oxidizer | GOX from a 2000 psia bottle through O-REG |
| Fuel | Kerosene / RP-1, tank pressurized with GN2 through PR-REG |
| Test article | Hopper torch (MR 1.8, 19.29 / 10.71 g/s ox / fuel, Pc 200 psia, 0.1952" throat) |
| LOX leg (added) | LOX tank pressurized with GN2 through PR-REG-2, built from the *METS LOX Feed System Design* pressure ladder |
| Test article (added) | Elytra torch igniter, LOX / kerosene (MR 1.81, 9.4 / 5.2 g/s, Pc 265 psia, one O-F-O triplet) |
| Analysis | Steady state only |
| Not yet modeled | GN2 pressurant flow losses, LOX cavitating venturi (gas gen), transients |

`fluid_model_OLD.ipynb` is the original GOX / propane version from the METS CoDR (Pc 50 psia, MR 2.2), kept for reference.

## Network

```
GOX:       O-REG → O-TUBE-1 → OMV → O-TUBE-2 → O-INJ (choked) ─┐
                                                               ├→ Chamber → throat
Kerosene:  Tank (PR-REG) → F-TUBE-1 → FMV → F-TUBE-2 → F-INJ ──┘
```

Both legs share one chamber pressure node, closed by

```
Pc = (mdot_ox + mdot_fuel) · c* / A_throat
```

so a change on one leg shifts Pc and therefore the flow on the other leg.

For the Elytra torch the GOX leg is replaced by the LOX leg (venturi left out; it returns for the gas generator):

```
LOX:       Tank (PR-REG-2) → C-TUBE-1 → O-PV → C-TUBE-2 → T-OV → C-TUBE-3 → O-INJ (liquid) ─┐
                                                                                         ├→ Chamber → throat
Kerosene:  Tank (PR-REG-1) → F-TUBE-1 → FMV  → F-TUBE-2 → F-INJ ───────────────────────────┘
```

O-PV is stand hardware; T-OV is the torch oxygen valve (part of the test article). FMV doubles as the torch fuel solenoid.
The Elytra chamber closure uses c*/A_t anchored at the design point (Pc 265 psia at 14.6 g/s), so no throat diameter is needed.

**Code layout:** each Elytra / LOX function sits directly after its Hopper / GOX twin and follows the same steps
(`make_hopper_torch` → `make_elytra_torch`, `build_gox_leg` / `build_fuel_leg` → `build_lox_leg`,
`build_mets_system` → `build_elytra_system`, `find_regulator_setpoints` → `elytra_find_setpoints`, `predict` → `elytra_predict`, ...).
All stand hardware, GOX, kerosene and LOX, lives in one `STAND` dict. The two main cells run the same steps
(targets → design → self-check → prediction → sweeps), and every plot is drawn in the final VISUALIZATION cell.

**LOX leg inputs** come from the *METS LOX Feed System Design* pressure ladder (lengths, K, Cv 0.04 for O-PV and T-OV),
modeled with the real 0.18" ID tube, tank head adding pressure at the bottom outlet, and the ladder's 0.3 m elevation
change applied as a drop to the torch. Before this was committed, the FullFlow LOX leg was checked against that
spreadsheet with its own inputs and matched every row to within 0.002 psi.

## How it works

Built on [FullFlow](https://pypi.org/project/fullflow/) (network solver) and ThermoProp (CoolProp properties).

- **Tubes:** each run of 1/4" × 0.035" SS tube is a Darcy–Weisbach pipe with the friction factor from Churchill. Bends, tees, and unions are folded in as an equivalent length, `L_eq = L + ΣK · D / f`, with height change included.
- **Valves and orifices:** valve Cv and fitting K values are converted to an effective CdA. Gas restrictions use FullFlow's `CompressibleOrifice` (which handles choking); liquid restrictions use `DischargeCoefficient`.
- **Fluids:** GOX is an ideal gas whose density updates with local pressure; kerosene has constant density and viscosity.
- **Continuity:** each internal node has a mass balance (flow in = flow out), and the chamber balance couples the legs.
- **Modes:**
  - *Design mode* makes the two regulator setpoints unknowns and adds two balances forcing the injector flows to equal their targets — one solve returns both setpoints.
  - *Predict mode* fixes the setpoints and solves for everything else.

## Outputs

- Required O-REG and PR-REG setpoints, with checks against the 1 ksi component rating and bottle pressure
- Self-check: predicting at the design setpoints must return the targets
- Prediction at rounded, real-world setpoints
- Regulator sweeps (mdot and MR vs. setpoint for each leg)
- Pressure ladder plot of every node on both legs
- Elytra: required PR-REG-2 / PR-REG-1 setpoints, prediction, sweeps and ladder (`elytra_regulator_sweep.png`, `elytra_pressure_ladder.png`)

## Inputs and placeholders

Each input in the notebook has a comment citing its source (Hopper Torch PDR / TRR, the HopperTorchSizing spreadsheet, the METS BOM and P&ID). Values marked `# PLACEHOLDER` are estimates — valve Cv, fitting K values, tube lengths, fuel injector CdA, and kerosene properties — and should be replaced with datasheet, as-built, or cold-flow data.

## Running

```bash
pip install fullflow thermoprop numpy matplotlib
jupyter notebook fluid_model_NEW.ipynb
```

Run all cells; plots are saved as `pressure_ladder.png` and `regulator_sweep.png` in the working directory.

## Planned

- Cavitating venturi on the LOX leg for the Elytra gas generator (LOX leg and Elytra torch are done)
- Calibrating placeholders against hot-fire and cold-flow data
