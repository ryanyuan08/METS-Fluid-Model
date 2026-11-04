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
| Analysis | Steady state only |
| Not yet modeled | GN2 pressurant flow losses, LOX leg, transients |

`fluid_model_OLD.ipynb` is the original GOX / propane version from the METS CoDR (Pc 50 psia, MR 2.2), kept for reference.

## Network

```
GOX:       O-REG → O-TUBE-1 → OMV → O-TUBE-2 → O-INJ (choked) ─┐
                                                               ├→ Chamber → throat
Kerosene:  Tank (PR-REG) → F-TUBE-1 → FMV → F-TUBE-2 → F-ORF → F-INJ ─┘
```

Both legs share one chamber pressure node, closed by

```
Pc = (mdot_ox + mdot_fuel) · c* / A_throat
```

so a change on one leg shifts Pc and therefore the flow on the other leg.

## How it works

Built on [FullFlow](https://pypi.org/project/fullflow/) (network solver) and ThermoProp (CoolProp properties).

- **Tubes:** each run of 1/4" × 0.035" SS tube is a Darcy–Weisbach pipe with the friction factor from Colebrook. Bends, tees, and unions are folded in as an equivalent length, `L_eq = L + ΣK · D / f`, with height change included.
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

## Inputs and placeholders

Each input in the notebook has a comment citing its source (Hopper Torch PDR / TRR, the HopperTorchSizing spreadsheet, the METS BOM and P&ID). Values marked `# PLACEHOLDER` are estimates — valve Cv, fitting K values, tube lengths, fuel injector CdA, and kerosene properties — and should be replaced with datasheet, as-built, or cold-flow data.

## Running

```bash
pip install fullflow thermoprop numpy matplotlib
jupyter notebook fluid_model_NEW.ipynb
```

Run all cells; plots are saved as `pressure_ladder.png` and `regulator_sweep.png` in the working directory.

## Planned

- LOX leg with a cavitating venturi for LOX-fed test articles (Elytra torch, Elytra gas generator)
- Calibrating placeholders against hot-fire and cold-flow data
