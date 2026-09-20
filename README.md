# charlib_EE671 — Standard Cell Characterization Template

A ready-to-run template for characterizing digital standard cells with
[CharLib](https://github.com/stineje/CharLib) and the open-source
[sky130](https://github.com/google/skywater-pdk) PDK.

You edit a SPICE netlist and a YAML config, push to GitHub, and a
GitHub Actions workflow runs the full characterization in the cloud and
produces a Liberty (`.lib`) timing/power model — **no local tool install
needed**.

---

## What you get

- A SPICE cell netlist (`cells/inverter.spice`) — the transistor-level circuit.
- A CharLib config (`config/config.yaml`) — corner, supply, temperature,
  slews, loads, and logic function.
- A CI workflow (`.github/workflows/generate-lib.yml`) that installs ngspice +
  CharLib, downloads the sky130 PDK, runs characterization, and uploads the
  resulting `.lib` as a downloadable artifact.

---

## Quick start

### 1. Create your own copy

Click **"Use this template" → "Create a new repository"** on GitHub
(or **Fork** it). This gives you your own repo you can freely edit.

### 2. Make a change and push

Every push to the `main` branch runs the workflow automatically. To try it
right away, edit `config/config.yaml` (or just push any commit).

```bash
git clone git@github.com:<your-username>/<your-repo>.git
cd <your-repo>
# edit cells/ and config/ ...
git add .
git commit -m "Characterize my cell"
git push origin main
```

You can also trigger a run without pushing: on GitHub go to the **Actions**
tab → **Generate Liberty (.lib) Files** → **Run workflow**.

### 3. Download your results

1. Open the **Actions** tab on your repo.
2. Click the most recent **Generate Liberty (.lib) Files** run.
3. Wait for the green check (about 2–4 minutes).
4. Scroll to **Artifacts** at the bottom and download:
   - **`sky130-lib-files`** — your generated `.lib` file(s).
   - **`charlib-log`** — the full run log (useful for debugging failures).

---

## Repository layout

```
.
├── cells/
│   └── inverter.spice        # transistor-level SPICE netlist (the DUT)
├── config/
│   ├── config.yaml           # CharLib characterization config
│   └── mc_switches.spice     # sky130 Monte-Carlo switches (see notes)
└── .github/workflows/
    └── generate-lib.yml       # CI: runs CharLib and uploads the .lib
```

---

## How to characterize your own cell

### Step 1 — Write the SPICE netlist

Put your cell in `cells/`. The cell must be a `.subckt`. The subckt name
must match the cell name you use in the config. Example
(`cells/inverter.spice`):

```spice
* Sky130 CMOS Inverter
.subckt inverter IN OUT VDD VSS
XM1 OUT IN VSS VSS sky130_fd_pr__nfet_01v8 w=1.0 l=0.15
XM2 OUT IN VDD VDD sky130_fd_pr__pfet_01v8 w=2.0 l=0.15
.ends inverter
```

- Use sky130 device models: `sky130_fd_pr__nfet_01v8`, `sky130_fd_pr__pfet_01v8`.
- Widths/lengths (`w`/`l`) are in microns.
- Pin order in the `.subckt` line is up to you, but the connections in the
  netlist must be correct.

### Step 2 — Describe it in the config

Edit `config/config.yaml`:

```yaml
settings:
  lib_name: inverter_sky130_tt_1v80_025C   # name of the generated library
  results_dir: output
  temperature: 25                          # °C
  named_nodes:
    primary_power:
      name: VDD
      voltage: 1.8                          # supply voltage (V)
    primary_ground:
      name: VSS
      voltage: 0.0

cells:
  inverter:                                 # must match the .subckt name
    netlist: "cells/inverter.spice"
    area: 3.7536                            # layout area in um^2 (you set this)
    models:
      - "config/mc_switches.spice"          # keep this line first (see notes)
      - "sky130A/libs.tech/ngspice/sky130.lib.spice tt"   # PDK + corner
    data_slews: [0.01, 0.05, 0.1, 0.2]      # input transition times (ns)
    loads: [0.001, 0.005, 0.01, 0.05]       # output load caps (pF)
    functions:
      - "OUT = !IN"                          # boolean function of the cell
```

**To add a second cell**, add its netlist under `cells/` and a second entry
under the `cells:` key (same shape, matching its `.subckt` name and function).

### Step 3 — Commit and push

Push to `main`, then grab the `.lib` from the Actions artifacts (see above).

---

## Config field reference

| Field | Where | Meaning |
|-------|-------|---------|
| `lib_name` | settings | Name of the generated Liberty library / output file. |
| `temperature` | settings | Characterization temperature in °C. |
| `named_nodes.*.voltage` | settings | Supply / ground voltages in volts. |
| `netlist` | per-cell | Path to the cell's SPICE `.subckt` file. |
| `area` | per-cell | Physical layout area in µm². Not derivable from SPICE — set it from your layout (Magic/KLayout). Defaults to `0.0` if omitted. |
| `models` | per-cell | SPICE files/libraries to include. `"file corner"` becomes a `.lib`; a bare `"file"` becomes an `.include`. |
| `data_slews` | per-cell | Input transition times swept (ns). |
| `loads` | per-cell | Output load capacitances swept (pF). |
| `functions` | per-cell | Boolean logic function(s), e.g. `"OUT = !IN"`. |

The `tt` at the end of the sky130 model line selects the **typical-typical**
process corner. Other corners include `ss` (slow), `ff` (fast), etc.

---

## Notes & gotchas

### Why `config/mc_switches.spice` exists

The sky130 transistor models gate their statistical mismatch/process terms on
two parameters — `mc_mm_switch` and `mc_pr_switch` — but the `tt` corner does
not define them. Without them, ngspice aborts with:

```
Undefined parameter [mc_mm_switch]
Subckt Stack underflow.
```

`config/mc_switches.spice` defines both to `0` (nominal corner, no Monte-Carlo
variation) and is included **before** the PDK library so the models find them.
Keep that line first in every cell's `models` list.

### The run says `area : 0.00`

You didn't set the `area` field for that cell. Add `area: <value>` under the
cell (see the reference table). Area comes from your physical layout — SPICE
cannot know it.

### A characterization point failed

Download the **`charlib-log`** artifact and read the tail of the log. Common
causes: a typo in a device model name, a wrong `.subckt` pin connection, or a
`functions` expression that doesn't match the circuit.

---

## Running locally (optional)

You don't need this — CI does everything — but if you want to run on your own
machine:

```bash
# System dependency
sudo apt-get install -y ngspice libngspice0-dev

# Python tools
pip install git+https://github.com/stineje/CharLib.git@2.0.0 ciel
pip install "numpy<2" --force-reinstall

# Download the sky130 PDK and link it next to this repo as ./sky130A
export PDK_ROOT="$HOME/pdk"
ciel enable --pdk-family sky130 78b7bc32ddb4b6f14f76883c2e2dc5b5de9d1cbc
ln -sfn "$(readlink -f $PDK_ROOT/sky130A)" ./sky130A

# Run
charlib run config/
# → output/<lib_name>.lib
```

---

## Credits

Built on [CharLib](https://github.com/stineje/CharLib) by James Stine et al.
and the [SkyWater sky130 PDK](https://github.com/google/skywater-pdk).
Template maintained for **EE671**.
