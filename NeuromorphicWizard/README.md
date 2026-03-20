# NeuromorphicWizard

A Python web application for designing **Intracellular Artificial Neural Networks (IANNs)** using genetic circuits, predicting their behavior with the Biocompiler, and generating OT-2 liquid handling robot protocols for automated transfection.

Built for [HTGAA 2026](https://2026a.htgaa.org/) (How to Grow Almost Anything) - Week 7: Genetic Circuits II.

## What It Does

NeuromorphicWizard takes a simple CSV describing which plasmids to combine and how much DNA to use, then:

1. **Validates** your circuit design (part names, DNA limits, slot conflicts)
2. **Predicts** circuit behavior using the Biocompiler API (heatmap visualization)
3. **Generates** OT-2 robot protocols with plate layouts and color-coded Excel files
4. **Simulates** the Opentrons protocol before running on hardware

## Background

### How Neuromorphic Circuits Work

These circuits perform **analog computation** inside living HEK293 cells using a library of plasmids from the Ron Weiss Lab at MIT. Unlike traditional digital genetic circuits (ON/OFF), IANNs produce a continuous spectrum of expression levels — like real neurons.

The core mechanism uses **endoribonucleases (ERNs)** — enzymes that cut RNA at specific recognition sequences:

```
DNA (plasmid)  →  mRNA  →  Protein
                    ↑
              ERN cuts here
          (if recognition sequence present)
```

Three ERNs are available, each recognizing a different RNA sequence:

| ERN | Function |
|-----|----------|
| **CasE** | Cuts mRNA with CasE recognition sequences |
| **Csy4** | Cuts mRNA with Csy4 recognition sequences |
| **PgU** | Cuts mRNA with PgU recognition sequences |

### Part Naming Convention

The naming convention encodes the biology:

| Pattern | Example | Meaning |
|---------|---------|---------|
| Plain name | `CasE` | ERN enzyme, freely expressed |
| `X_rec_Y` | `PgU_rec_Csy4` | Encodes protein Y, but X can cut its mRNA |
| `X_rec_Color` | `CasE_rec_mNeonGreen` | Encodes fluorescent protein, but X can cut its mRNA |
| `X_rec_Y_rec_Z` | `CasE_rec_Csy4_rec_mKO2` | Dual recognition — either X or Y can cut it |
| Color name | `eBFP2`, `mKO2` | Constitutive fluorescent reporter (always ON) |

### Chains of Inhibition

Circuits are built by chaining inhibitions:

```
Single:  Csy4 ──inhibits──▶ mNeonGreen              → green OFF

Double:  Csy4 ──inhibits──▶ CasE ──inhibits──▶ mNG  → green ON  (double negative)

Triple:  PgU ──▶ Csy4 ──▶ CasE ──▶ mNG              → green OFF (triple negative)
```

Even number of inhibitions = output ON. Odd = OFF. The **amounts matter** — the ratio between plasmids determines expression levels, making the computation analog rather than digital.

### Available Parts

**ERNs:** `CasE`, `Csy4`, `PgU`

**ERN-regulated ERNs:** `PgU_rec_Csy4`, `PgU_rec_CasE`, `Csy4_rec_CasE`, `CasE_rec_Csy4`

**ERN-regulated Reporters:** `Csy4_rec_mNeonGreen`, `CasE_rec_mNeonGreen`, `PgU_rec_mNeonGreen`, `CasE_rec_Csy4_rec_mKO2`

**Fluorescent Reporters (constitutive):**

| Protein | Color |
|---------|-------|
| `mNeonGreen` | Green |
| `mKO2` | Orange |
| `eBFP2` | Blue |
| `mMaroon1` | Maroon/Red |
| `mCherry` | Red |
| `PacBlue` | Blue (Pacific Blue) |

## Installation

### Prerequisites

- [Anaconda Distribution](https://www.anaconda.com/download/success)
- Python 3.10

### Setup

```bash
# 1. Create conda environment
conda create -n neuro_wiz python==3.10
conda activate neuro_wiz

# 2. Install dependencies
pip install -r requirements.txt

# 3. Run tests (optional)
pytest tests/ -v

# 4. Start the application
python3 main.py
```

Opens at **http://localhost:8080**

## Input Format

A CSV with 5 columns:

| Column | Description | Notes |
|--------|-------------|-------|
| **Circuit name** | Name for your circuit | All rows in one circuit share the same name |
| **Transfection group** | Group label (X1, X2, Bias, etc.) | Plasmids in the same group are co-transfected |
| **Contents** | Plasmid name | Must match a name from the parts list |
| **Concentration (ng/uL)** | Stock concentration | Typically 50 |
| **DNA wanted (ng)** | Amount of this plasmid | All rows in a circuit should sum to <= 650 ng |

### Example CSV

```csv
Circuit name,Transfection group,Contents,Concentration (ng/uL),DNA wanted (ng)
MyCircuit,X1,Csy4,50,150
MyCircuit,X1,mKO2,50,100
MyCircuit,X2,Csy4_rec_CasE,50,100
MyCircuit,X2,eBFP2,50,100
MyCircuit,Bias,CasE_rec_mNeonGreen,50,200
```

This defines a double-inhibition cascade: Csy4 inhibits CasE, CasE inhibits mNeonGreen. Result: green ON (double negative = positive).

## Application Tabs

### Build

The main experiment design interface:

- **Upload** circuit design CSV or Biocompiler JSON
- **Edit** data in an interactive table (AG Grid)
- **Generate** plate layouts with automatic slot assignment
- **Visualize** color-coded plate layouts
- **Simulate** Opentrons protocols
- **Download** generated outputs (Excel + Python script)

### Predict

Simulates circuit behavior using the [Biocompiler API](https://biocomp-api.rachael.jdisset.com):

- Select a circuit from your uploaded data
- Converts to Biocompiler recipe format
- Displays a prediction **heatmap** (32x32 grid) showing output fluorescence vs. input concentrations
- Shows the recipe JSON sent to the API

**Predict API constraints:** For prediction, circuits must have exactly X1 and X2 transfection groups, each containing a marker (fluorescent protein), an ERN, and an output (ERN_rec_fluorescent). The API currently recognizes `eBFP2` and `mKO2` as markers, and `mNeonGreen` as the output fluorescent protein.

### Generate

*Coming soon* — AI Agent Interface to Biocompiler-Designer for automated circuit design.

### Analyze

*Coming soon* — Flow Cytometry Results Visualizer for analyzing wet lab results.

## Example Circuits

The `examples/` directory contains ready-to-use CSV files:

| File | Description | Works in Predict? |
|------|-------------|:-:|
| `biocompiler_predict_example.csv` | Dual ERN circuit (CasE/PgU) — reference design | Yes |
| `project1.csv` | PgU/Csy4 cascade — cross-inhibition | Yes |
| `project2.csv` | CasE/Csy4 cascade — cross-inhibition | Yes |
| `project3.csv` | StrongInhibitor — asymmetric PgU-heavy dosing | Yes |
| `project4.csv` | BalancedDuel — perfectly symmetric Csy4/CasE | Yes |
| `project5.csv` | SelfInhibit — each ERN targets its own output, no cross-talk | Yes |
| `project6.csv` | DoubleStrike — 4 parts per group, dual ERN/output | Experimental |
| `project7.csv` | OneWay — only X2's ERN controls both outputs | Yes |
| `project8.csv` | SharedPool — same ERN (PgU) in both groups | Yes |
| `project9.csv` | TiltedPool — shared PgU with asymmetric dosing | Yes |
| `project10.csv` | SplitChannel — cross-inhibition with tilted ratios | Yes |
| `project11.csv` | Escapee — one output immune to inhibition | Yes |
| `simple_two_circuits.csv` | Two simple circuits | Partial |
| `large_five_circuits.csv` | Five circuits for batch testing | Build only |
| `with_dilution.csv` | Demonstrates automatic dilution handling | Build only |
| `with_plate_destinations.csv` | Pre-assigned plate destinations | Build only |

### Circuit Topology Guide

Different circuit topologies produce different heatmap patterns:

**Cross-inhibition** (project1, project2): Each group's ERN attacks the other group's output. Standard pattern — both inputs suppress each other's output.

**Self-inhibition** (project5): Each ERN targets its own group's output. The two groups are independent — no cross-talk. Produces a separable heatmap.

**One-way** (project7): Only one ERN has a valid target. One input axis controls the output; the other is inert. Produces directional bands.

**Shared pool** (project8, project9): Same ERN in both groups. Both inputs contribute to the same inhibitor pool. Produces diagonal gradients.

**Escapee** (project11): One output has no matching ERN — it's always ON, creating a constant baseline floor in the heatmap.

## Output Files

When you export from the Build tab, the Wizard generates:

| File | Description |
|------|-------------|
| `experiment_config.csv` | Full config with source/destination slot assignments |
| `opentrons_protocol.py` | OT-2 Python script ready to run on the robot |
| `plate_layouts.xlsx` | Color-coded plate layout diagrams |
| `biocompiler_format.json5` | Circuit in Biocompiler format with plasmid ratios |

## Architecture

```
NeuromorphicWizard/
  main.py                         # Entry point (NiceGUI app, port 8080)
  requirements.txt                # Dependencies
  core/
    config.py                     # Constants, column names, part lists, layouts
    state.py                      # AppState and TemplateState classes
    validation.py                 # Input validation rules
    layout.py                     # Slot assignment and dilution detection
    json_converter.py             # CSV ↔ Biocompiler JSON conversion
    exporters.py                  # Excel and Opentrons script generation
    script_utils.py               # OT-2 script helpers
    utils.py                      # Data normalization utilities
  ui/
    tabs/
      build.py                    # Main experiment design interface
      predict.py                  # Biocompiler prediction + heatmaps
      generate.py                 # AI circuit designer (placeholder)
      analyze.py                  # Flow cytometry analysis (placeholder)
    components/
      upload.py                   # CSV/JSON file upload
      table.py                    # AG Grid editable table
      layout_gen.py               # Layout generation handler
      visualization.py            # Plate layout visualization
      download.py                 # File download buttons
      simulation.py               # Opentrons simulation runner
      grid_manager.py             # AG Grid state management
      plate_renderer.py           # Plate visual rendering
  data/
    OT2_automated_transfection_v3.9.py    # OT-2 protocol template (24-tube)
    OT2_automated_transfection_v3.8.py    # Legacy template
    OT2_automated_transfection_test96well_format.py  # 96-well template
  static/
    css/styles.css                # Application styles
    js/file_operations.js         # Frontend file handling
  examples/                       # Sample CSV files
  tests/
    unit/                         # Dilution, inference, slot, validation tests
    integration/                  # Full workflow tests (24-tube, 96-well)
    regression/                   # Example file consistency tests
    example_input/                # Test fixtures (valid/invalid)
```

## Physical Protocol Flow

The columns in the generated config trace each plasmid's journey through the OT-2 robot:

```
[DNA source]           Human places stock DNA tubes on the rack
       │
       ▼ (robot pipettes DNA, or dilutes first if < 2 µL)
[Diluted source]       Robot dilutes low-volume DNA with water
       │
       ▼ (robot pipettes DNA into mixing tubes by transfection group)
[DNA destination]      DNA mixing tube — co-transfected plasmids combined
       │                 + Opti-MEM + P3000 added
       │
       ▼ (robot transfers DNA+P3K mix into Lipofectamine tube)
[L3K/OM destination]   Lipofectamine tube — lipid-DNA complexes form (10 min)
       │
       ▼ (robot pipettes final complex onto cells)
[Plate destination]    Output well — HEK293 cells receive the transfection mix
```

## Hardware Layouts

Two OT-2 deck layouts are supported:

**24-Tube Layout** (default): Three 24-tube racks (slots 4, 5, 6) with two 24-well output plates (slots 2, 3). Best for small experiments.

**96-Well Layout**: Mixed tube racks (slots 4, 5) with a 96-well plate (slot 6) for high-throughput experiments.

## Key Constraints

- Total DNA per circuit: **<= 650 ng** (recommended), 800 ng hard limit
- Minimum pipette volume: **2 µL** (below this, automatic dilution is triggered)
- Stock concentration: typically **50 ng/µL**
- The **ratio** between plasmids matters more than absolute amounts for circuit behavior

## Dependencies

```
nicegui==3.3.1        # Web UI framework
fastapi==0.121.3      # Backend framework
pandas>=2.0.0         # Data processing
openpyxl>=3.1.0       # Excel file generation
json5>=0.9.0          # JSON5 parsing for Biocompiler format
opentrons             # OT-2 robot API
httpx>=0.27.0         # Async HTTP client (Biocompiler API)
matplotlib>=3.8.0     # Heatmap plotting
pytest>=7.4.0         # Testing
```

## Testing

```bash
# Run all tests
pytest tests/ -v

# Run specific test categories
pytest tests/unit/ -v
pytest tests/integration/ -v
pytest tests/regression/ -v
```

## License

Developed for HTGAA 2026 at the MIT Media Lab.
