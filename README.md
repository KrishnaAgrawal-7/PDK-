# 🔬 Photonic IC Layout Design — Internship Project

## 📌 What This Project Is About

This project is about **designing photonic integrated circuit (PIC) layouts** using Python. Instead of drawing chip layouts by hand in a GUI tool, we use code to **programmatically create, manipulate, and export** photonic component layouts (waveguides, bends, splitters, etc.) as **GDSII files** — the industry-standard format used by semiconductor foundries for chip fabrication.

### In Simple Terms

> Imagine you're designing a tiny optical chip that guides light through microscopic "pipes" (waveguides). This project uses Python to draw those pipes, connect them together, and export the final design as a `.gds` file that a chip factory can use to manufacture the real chip.

---

## 🧠 Core Concepts

### What is a Photonic Integrated Circuit (PIC)?

A PIC is a microchip that uses **light (photons)** instead of electricity to process and transmit information. Key components include:

| Component | What It Does |
|-----------|-------------|
| **Waveguide** | A tiny "channel" that guides light on the chip (like a fiber optic cable, but on-chip) |
| **Bend** | Curves the waveguide to change direction (uses Euler bends to minimize light loss) |
| **Splitter** | Splits one light beam into two |
| **Coupler** | Combines or transfers light between waveguides |
| **Ring Resonator** | A circular waveguide that filters specific wavelengths |

### What is GDSII (.gds)?

**GDSII (Graphic Data System II)** is a binary file format that stores 2D geometric shapes representing IC layouts. It's the standard format that:
- Chip designers create
- Foundries (factories) use to manufacture real chips
- Contains **layers** (different materials/process steps), **cells** (reusable blocks), and **polygons** (shapes)

### Design Hierarchy

```
Library (top-level .gds file)
 └── Cell (a reusable design block, like "simple_waveguide_and_bend")
      ├── Polygons (actual geometric shapes)
      ├── Paths (line-based shapes with width)
      ├── Labels (text annotations)
      └── References (instances of other cells — enables reuse)
```

---

## 📁 Project Structure

```
Internship/
│
├── demo.py               # Main script — creates a waveguide + bend layout
├── view_gds.py           # Utility script — inspects and visualizes .gds files
├── waveguide_bend.gds    # Output — the generated GDSII layout file
├── README.md             # This file — project documentation
│
└── .venv312/             # Python 3.12 virtual environment with all dependencies
    ├── Scripts/          # Python executables (pip, python, etc.)
    └── Lib/
        └── site-packages/  # All installed libraries
```

---

## 📝 File-by-File Explanation

### 1. `demo.py` — The Main Layout Script

This is the **heart of the project**. It creates a simple photonic layout with two components and exports it as a `.gds` file.

**What it does step by step:**

```python
import gdsfactory as gf        # Import the photonic design library
gf.gpdk.PDK.activate()         # Activate the Generic PDK (Process Design Kit)
```
1. **Imports gdsfactory** — the main library for photonic IC design
2. **Activates the Generic PDK** — a set of predefined rules, layer mappings, and component definitions that match a generic fabrication process

```python
c = gf.Component("simple_waveguide_and_bend")   # Create a blank component (cell)
```
3. **Creates a new Component** — think of it as a blank canvas/cell to place shapes on

```python
straight = c << gf.components.straight(length=10.0, width=0.5)   # 10μm long, 0.5μm wide
bend = c << gf.components.bend_euler(radius=10.0, angle=90.0)    # 90° Euler bend, 10μm radius
```
4. **Adds a straight waveguide** — 10 micrometers long, 0.5 micrometers wide
5. **Adds an Euler bend** — a 90-degree curve with 10μm radius (Euler bends minimize optical loss)
6. The `<<` operator adds ("places") components into the parent component

```python
bend.connect("o1", straight.ports["o2"])   # Connect bend input to straight output
```
7. **Connects the components** — aligns the bend's input port (`o1`) to the straight's output port (`o2`). Ports are predefined connection points on each component.

```python
c.write_gds("waveguide_bend.gds")   # Export to GDSII format
```
8. **Exports everything** to a `.gds` file that can be opened in KLayout or sent to a foundry

### 2. `view_gds.py` — GDS File Inspector & Visualizer

A utility tool to **inspect and visualize** any `.gds` file without needing KLayout.

**Features:**
- Prints file metadata (name, size, units)
- Lists all cells and their contents (polygons, paths, labels, references)
- Shows all unique layers used
- Visualizes the layout with matplotlib (color-coded by layer)

**Usage:**
```bash
python view_gds.py waveguide_bend.gds    # Direct mode
python view_gds.py                        # Interactive mode (prompts for file path)
```

### 3. `waveguide_bend.gds` — The Output Layout

The generated GDSII file containing:

| Cell Name | Description |
|-----------|-------------|
| `simple_waveguide_and_bend` | Top-level cell — the complete layout (references the two sub-cells below) |
| `straight_...` | A 10μm straight waveguide (1 polygon) |
| `bend_euler_...` | A 90° Euler bend with R=10μm (1 polygon) |
| `$$$CONTEXT_INFO$$$` | Metadata cell auto-generated by gdsfactory |

**Open this file with:**
- **KLayout** (GUI) → `File → Open` → select the file
- **Python** → `python view_gds.py waveguide_bend.gds`

---

## 📚 Libraries Used

### Core Libraries

| Library | Version | What It Does | Why We Use It |
|---------|---------|-------------|---------------|
| **gdsfactory** | 9.43.0 | Photonic IC design framework | The main library — provides components (waveguides, bends, etc.), PDKs, routing, and GDS export |
| **kfactory** | 2.5.2 | Backend for gdsfactory | Handles low-level cell/component management using KLayout's engine |
| **klayout** | 0.30.8 | Layout engine (Python bindings) | Core engine that kfactory uses for geometry operations and GDS I/O |

### Geometry & Math

| Library | Version | What It Does |
|---------|---------|-------------|
| **numpy** | 2.4.6 | Numerical computing (arrays, math) — used for coordinate calculations |
| **scipy** | 1.17.1 | Scientific computing — used for curve generation (Euler spirals, etc.) |
| **shapely** | 2.1.2 | 2D geometric operations (boolean ops, offsets, intersections) |

### Visualization

| Library | Version | What It Does |
|---------|---------|-------------|
| **matplotlib** | 3.10.9 | Plotting and visualization of layouts |
| **pillow** | 12.2.0 | Image processing |
| **scikit-image** | 0.26.0 | Image analysis (used for some rasterization tasks) |

### Data & Configuration

| Library | Version | What It Does |
|---------|---------|-------------|
| **pydantic** | 2.12.5 | Data validation — gdsfactory uses it for component parameter validation |
| **pydantic-settings** | 2.14.1 | Settings management |
| **PyYAML** / **ruamel.yaml** | 6.0.3 / 0.19.1 | YAML parsing for configuration files |
| **orjson** | 3.11.9 | Fast JSON parsing |
| **pandas** | 3.0.3 | Data analysis (tabular data) |

### Development & Interactive

| Library | Version | What It Does |
|---------|---------|-------------|
| **ipython** | 9.13.0 | Enhanced interactive Python shell |
| **ipykernel** | 7.2.0 | Jupyter kernel for running notebooks |
| **ipywidgets** | 8.1.8 | Interactive widgets for Jupyter |
| **graphviz** | 0.21 | Graph visualization (component hierarchy diagrams) |

### Utilities

| Library | Version | What It Does |
|---------|---------|-------------|
| **click** | 8.4.0 | CLI framework (used by gdsfactory's command-line tools) |
| **rich** | 15.0.0 | Beautiful terminal output (progress bars, tables) |
| **loguru** | 0.7.3 | Logging |
| **networkx** | 3.6.1 | Graph/network analysis (used for routing algorithms) |
| **trimesh** | 4.12.2 | 3D mesh operations (for 3D visualization of layouts) |
| **watchdog** | 6.0.0 | File system monitoring (auto-reload on file changes) |

---

## 🚀 How to Run the Project

### Prerequisites
- Python 3.12
- KLayout installed (for GUI viewing)

### Step 1: Activate the Virtual Environment

```powershell
# In PowerShell, navigate to the project folder
cd C:\Users\User\Downloads\Internship

# Activate the virtual environment
.\.venv312\Scripts\Activate.ps1
```

### Step 2: Generate the Layout

```powershell
python demo.py
```

**Expected output:**
```
Successfully created layout 'simple_waveguide_and_bend' and saved to 'waveguide_bend.gds'.
Ports details:
  Straight o1: Port(name='o1', ...)
  Straight o2: Port(name='o2', ...)
  Bend o1: Port(name='o1', ...)
  Bend o2: Port(name='o2', ...)
```

### Step 3: View the Layout

**Option A — KLayout (GUI):**
```powershell
# Open in KLayout directly
& "$env:APPDATA\KLayout\klayout_app.exe" waveguide_bend.gds
```

**Option B — Python (terminal + matplotlib):**
```powershell
python view_gds.py waveguide_bend.gds
```

---

## 🔄 Workflow Diagram

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Write Python   │────▶│   gdsfactory     │────▶│  .gds file      │
│   code (demo.py) │     │   generates      │     │  (layout data)  │
│                  │     │   geometry        │     │                 │
└─────────────────┘     └──────────────────┘     └────────┬────────┘
                                                          │
                                          ┌───────────────┼───────────────┐
                                          ▼               ▼               ▼
                                   ┌────────────┐  ┌────────────┐  ┌────────────┐
                                   │  KLayout   │  │  Python    │  │  Foundry   │
                                   │  (view/    │  │  viewer    │  │  (chip     │
                                   │   edit)    │  │  (inspect) │  │   fab)     │
                                   └────────────┘  └────────────┘  └────────────┘
```

---

## 🔑 Key Concepts Glossary

| Term | Definition |
|------|-----------|
| **Component** | A reusable design block (cell) — like a function in programming |
| **Port** | A connection point on a component (has position, direction, width, layer) |
| **PDK** | Process Design Kit — rules and layer definitions for a specific foundry process |
| **Layer** | A specific material or process step (e.g., silicon waveguide layer, metal layer) |
| **Euler Bend** | A type of waveguide curve that gradually changes curvature to minimize optical loss |
| **`<<` operator** | Adds (places) a component reference into a parent component |
| **`.connect()`** | Aligns two ports so components are physically connected |
| **Top Cell** | The highest-level cell that contains the entire design |
| **Flatten** | Resolves all cell references into a single level of polygons |

---

## 📖 Useful Resources

- [gdsfactory Documentation](https://gdsfactory.github.io/gdsfactory/) — Main library docs
- [KLayout Documentation](https://www.klayout.de/doc.html) — Layout viewer/editor docs
- [GDSII Format Specification](https://boolean.klaasholwerda.nl/interface/bnf/gdsformat.html) — File format details
- [Photonic IC Design Basics](https://gdsfactory.github.io/gdsfactory/notebooks/00_geometry.html) — Getting started with photonic design

---

## 🛠 Troubleshooting

| Problem | Solution |
|---------|---------|
| `ModuleNotFoundError: No module named 'gdsfactory'` | Activate the venv first: `.\.venv312\Scripts\Activate.ps1` |
| KLayout doesn't open `.gds` files by double-click | Right-click → Open With → Choose KLayout |
| `python` command not found | Use `python3` or ensure Python is on your PATH |
| Layout looks empty in KLayout | Check layer visibility — click the layer panel on the left |
| Virtual environment won't activate in PowerShell | Run: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |
