# Understanding PDK and iPDK: A Complete Guide

This document explains what a PDK is, what an iPDK is, how they work internally, and how you can build your own using Python. It starts from the absolute basics and builds up.

---

## Table of Contents

1. [What Problem Does a PDK Solve?](#1-what-problem-does-a-pdk-solve)
2. [What Is a PDK?](#2-what-is-a-pdk)
3. [Anatomy of a PDK — Every Piece Explained](#3-anatomy-of-a-pdk--every-piece-explained)
4. [What Is an iPDK and Why Does It Exist?](#4-what-is-an-ipdk-and-why-does-it-exist)
5. [iPDK vs Traditional PDK — Concrete Differences](#5-ipdk-vs-traditional-pdk--concrete-differences)
6. [How a PDK Is Built Using Python and gdsfactory](#6-how-a-pdk-is-built-using-python-and-gdsfactory)
7. [Step-by-Step: Building Your Own PDK](#7-step-by-step-building-your-own-pdk)
8. [How the Generic PDK in gdsfactory Is Structured](#8-how-the-generic-pdk-in-gdsfactory-is-structured)
9. [The Complete Workflow: From PDK to Fabricated Chip](#9-the-complete-workflow-from-pdk-to-fabricated-chip)
10. [What You Can Do in This Project](#10-what-you-can-do-in-this-project)
11. [Key Terms Reference](#11-key-terms-reference)

---

## 1. What Problem Does a PDK Solve?

When you design a chip, you are drawing geometric shapes (rectangles, polygons, paths) on different layers. Each layer corresponds to a real physical material or process step in the factory (called a "foundry").

The problem: the designer needs to know exactly what the foundry can and cannot manufacture. Questions like:

- What layers are available? What do they represent physically?
- How thin can I make a waveguide? (minimum width)
- How close can two shapes be? (minimum spacing)
- How thick is the silicon? What material sits above it?
- What components have already been validated and tested?

Without answers to these questions, the designer is guessing. The chip will either fail to manufacture, or it will not work as expected.

A PDK is the answer to all of these questions, packaged into a set of files that your design tools can understand.

---

## 2. What Is a PDK?

PDK stands for **Process Design Kit**. It is a collection of files provided by a foundry that describes everything about their manufacturing process that a designer needs to know.

Think of it like this:

```
Foundry (factory)  ----provides----> PDK ----used by----> Designer
```

A PDK contains:

| Component | What It Tells You |
|-----------|------------------|
| Layer Map | Which GDS layer numbers correspond to which physical materials |
| Layer Stack | The physical thickness, height, and material of each layer |
| Design Rules | Minimum width, spacing, enclosure constraints for each layer |
| Component Library (PCells) | Pre-built, validated components (waveguides, bends, couplers) |
| Cross Sections | How a waveguide looks in cross-section (width, layer, cladding) |
| Simulation Settings | Refractive indices, wavelength ranges, material properties |
| Connectivity | Which layers connect to each other through vias |
| Layer Views | How each layer should be displayed (color, pattern) in a viewer |

A PDK is not a single file. It is a structured collection of data, code, and configuration.

---

## 3. Anatomy of a PDK — Every Piece Explained

### 3.1 Layer Map

The layer map assigns a human-readable name to each GDS layer number. In GDSII files, layers are identified by a pair of integers: `(layer_number, datatype)`.

Example:
```
WG       = (1, 0)    --> This is the silicon waveguide layer
SLAB150  = (2, 0)    --> This is the 150nm slab (shallow etch)
SLAB90   = (3, 0)    --> This is the 90nm slab (deep etch)
M1       = (41, 0)   --> Metal 1 layer
M2       = (45, 0)   --> Metal 2 layer
HEATER   = (47, 0)   --> TiN heater layer
VIAC     = (40, 0)   --> Via from silicon to metal 1
VIA1     = (44, 0)   --> Via from metal 1 to metal 2
```

The layer map is the foundation. Every other part of the PDK references these layer definitions.

Without the layer map, the GDS file is just anonymous numbered layers. The layer map gives them meaning.


### 3.2 Layer Stack

The layer stack describes the **physical, 3D reality** of the chip. While the GDS file is 2D (top-down view), the real chip has height and depth. The layer stack tells you:

- **zmin**: where each layer starts vertically (height from the bottom)
- **thickness**: how thick each layer is
- **material**: what the layer is made of (silicon, silicon dioxide, aluminum, TiN, etc.)
- **sidewall_angle**: the angle of the walls (etching is never perfectly vertical)

Example of what a layer stack describes in physical terms:

```
                    ┌──────────────────┐
       3.2 um       │    Metal 3 (Al)   │  thickness = 2.0 um
                    ├──────────────────┤
       2.3 um       │    Metal 2 (Al)   │  thickness = 0.7 um
                    ├──────────────────┤
       1.1 um       │    Metal 1 (Al)   │  thickness = 0.7 um
                    ├──────────────────┤
       1.1 um       │    Heater (TiN)   │  thickness = 0.75 um
                    ├──────────────────┤
       0.0 um       │  ┌────┐  Silicon  │  thickness = 0.22 um (waveguide)
                    │  │ WG │  (core)   │
                    ├──┴────┴──────────┤
      -3.0 um       │   Buried Oxide    │  thickness = 3.0 um (SiO2)
                    │     (BOX)         │
                    ├──────────────────┤
     -13.0 um       │    Substrate      │  thickness = 10.0 um (Si)
                    └──────────────────┘
```

The waveguide core (silicon) sits at z=0. Everything is measured relative to it.

This is critical for simulation. If you want to simulate how light travels through the waveguide, you need to know exactly what materials surround it and their thicknesses.


### 3.3 Cross Sections

A cross section defines **how a path (like a waveguide) should be drawn** in 2D, and what its physical structure looks like when you slice through it.

When you draw a "waveguide" in your layout, you are actually drawing a path with a certain width on a certain layer. But a real waveguide is not just a single rectangle on one layer. It might include:

- The core (silicon, 0.5 um wide, on layer WG)
- The cladding (silicon dioxide, wider, on layer WGCLAD)
- Doping regions on either side
- A heater above it

A cross section bundles all of this together into a single reusable definition:

```
Cross Section "strip":
   Main layer:  WG, width = 0.5 um
   Cladding:    WGCLAD, width = 3.0 um (centered on the waveguide)

Cross Section "rib":
   Main layer:  WG, width = 0.5 um
   Slab:        SLAB150, width = 6.0 um
   Cladding:    WGCLAD, width = 6.0 um

Cross Section "metal_routing":
   Main layer:  M3, width = 10.0 um
```

When you create a straight waveguide or a bend, you pass it a cross section, and it knows exactly which layers to draw and at what widths.


### 3.4 Component Library (PCells / Parametric Cells)

A PCell (Parameterized Cell) is a **function** that generates a component's layout based on input parameters.

Instead of drawing a waveguide by hand every time, you call a function:

```python
straight_waveguide = gf.components.straight(length=10.0, width=0.5)
euler_bend = gf.components.bend_euler(radius=10.0, angle=90)
mmi_splitter = gf.components.mmi1x2(width=0.5, length_mmi=5.5)
```

Each PCell function:
1. Takes parameters (length, width, radius, etc.)
2. Creates the geometry (polygons on the correct layers)
3. Defines ports (connection points with position, direction, and width)
4. Returns a Component object

The PDK provides a library of these validated PCells. "Validated" means the foundry has tested them — they know these components work when manufactured.

A key benefit: if you change a parameter, the entire geometry updates automatically. Change `length=10` to `length=20`, and the waveguide redraws itself.


### 3.5 Design Rules (DRC)

Design Rule Check (DRC) rules define the **manufacturing constraints**. These are hard limits set by the foundry's equipment.

Examples:
```
WG layer:
  - Minimum width: 0.15 um
  - Minimum spacing between two shapes: 0.15 um
  - Minimum area: 0.025 um^2

Metal 1 layer:
  - Minimum width: 0.5 um
  - Minimum spacing: 0.5 um

Via Contact:
  - Must be enclosed by Metal 1 on all sides by at least 0.1 um
  - Minimum via size: 0.2 x 0.2 um
```

If you violate any of these rules, the foundry will reject your design. DRC tools check your GDS file against these rules and report violations.


### 3.6 Connectivity

Connectivity defines **which layers connect to each other through vias**. This is needed for electrical routing (connecting metal pads, heaters, etc.).

```
NPP  --via--> VIAC --via--> M1
PPP  --via--> VIAC --via--> M1
M1   --via--> VIA1 --via--> M2
M2   --via--> VIA2 --via--> M3
```

This tells the routing tools: "If I need to go from Metal 1 to Metal 2, I need to place a VIA1 between them."


### 3.7 Simulation Settings

For photonic design, you need to simulate how light behaves. The PDK provides:

- **Refractive index** of each material at different wavelengths
- **Wavelength range** for simulation (e.g., 1.2 to 1.6 um for telecom)
- **Mesh settings** for the simulation grid
- **Material dispersion data** (how refractive index changes with wavelength)

Example from the generic PDK:

```
Silicon (si):
  At 1.55 um wavelength: n = 3.47
  (refractive index decreases at longer wavelengths)

Silicon Dioxide (sio2):
  At 1.55 um wavelength: n = 1.45

Silicon Nitride (sin):
  At 1.55 um wavelength: n = 1.99
```

### 3.8 Layer Views

Layer views define **how layers appear visually** in a layout viewer like KLayout. Each layer gets:

- A color (e.g., WG = red, M1 = blue)
- An opacity level
- A fill pattern (solid, hatched, dotted)
- Visibility (on/off by default)

This is cosmetic but essential for practical use. Without it, all layers look the same and you cannot tell what you are looking at.

---

## 4. What Is an iPDK and Why Does It Exist?

### The Problem iPDK Solves

In the traditional world, PDKs were **proprietary and vendor-locked**:

- A Cadence PDK used the SKILL programming language for PCells. It only worked in Cadence Virtuoso.
- A Synopsys PDK used its own format. It only worked in Synopsys tools.
- A Mentor Graphics PDK had yet another format.

This meant:
- The foundry had to create **3 separate PDKs** for the same process (one for each EDA vendor).
- If a designer wanted to switch tools, they had to start over.
- Maintaining 3 versions of the same PDK was expensive and error-prone.

### The Solution: iPDK

**iPDK = Interoperable Process Design Kit**

An iPDK is a PDK built on **open standards** so it works across multiple EDA tools. The key standards are:

1. **OpenAccess (OA)**: An open-source database format for storing IC design data. All major EDA vendors support reading/writing OpenAccess databases.

2. **Python for PCells**: Instead of writing PCells in a proprietary language (like SKILL), iPDKs use Python. Any tool that can run Python can use these PCells.

3. **Standard file structure**: The directory layout, technology files, and configuration files follow a documented standard.

```
Traditional:
  Foundry --> Cadence PDK (SKILL)    --> works only in Cadence
  Foundry --> Synopsys PDK           --> works only in Synopsys
  Foundry --> Mentor PDK             --> works only in Mentor

iPDK:
  Foundry --> ONE iPDK (Python/OA)   --> works in Cadence, Synopsys, Mentor, KLayout, etc.
```

### Who Pushed for iPDK?

The **IPL Alliance (Interoperable PDK Libraries Alliance)** — which includes foundries like TSMC, Samsung, and GlobalFoundries — drives the iPDK standard. The goal is to reduce PDK development cost and give designers freedom to choose their tools.

---

## 5. iPDK vs Traditional PDK — Concrete Differences

| Aspect | Traditional PDK | iPDK |
|--------|----------------|------|
| PCell language | Proprietary (SKILL, proprietary scripts) | Python (PyCells) |
| Database format | Vendor-specific or OpenAccess | OpenAccess (mandatory) |
| Tool compatibility | One vendor only | Any OpenAccess-compatible tool |
| Callbacks (parameter validation) | Vendor-specific | Tcl or Python |
| Technology file | Vendor-specific format | Standard .tf ASCII format for OA |
| DRC/LVS rules | Vendor-specific rule decks | Still tool-specific, but the underlying device definitions are shared |
| Development effort for foundry | Multiplied per vendor | Single development effort |
| Designer flexibility | Locked to one toolset | Free to choose best tools |

### What an iPDK Directory Looks Like

```
my_ipdk/
├── lib.defs                    # OpenAccess library definitions
├── tech/
│   ├── techfile.tf             # Technology file (layers, rules, units)
│   └── display.drf             # Display rules (colors, patterns)
├── pcells/
│   ├── waveguide.py            # Python PCell for waveguide
│   ├── bend.py                 # Python PCell for bend
│   ├── mmi.py                  # Python PCell for MMI splitter
│   └── callbacks/
│       └── validate_params.py  # Parameter validation callbacks
├── models/
│   ├── waveguide.spi           # SPICE model for waveguide
│   └── bend.spi                # SPICE model for bend
├── drc/
│   └── rules.svrf              # Design Rule Check deck
├── lvs/
│   └── rules.svrf              # Layout vs Schematic deck
└── docs/
    └── user_guide.pdf          # Documentation
```

---

## 6. How a PDK Is Built Using Python and gdsfactory

In the open-source photonic world, **gdsfactory** is the primary tool for creating PDKs in Python. It provides a framework where you define all the PDK components as Python code.

Here is how gdsfactory maps to the traditional PDK concepts:

| PDK Concept | gdsfactory Implementation |
|-------------|--------------------------|
| Layer Map | A Python class inheriting from `LayerEnum` |
| Layer Stack | A `LayerStack` object containing `LayerLevel` entries |
| Cross Sections | Python functions that return `CrossSection` objects |
| PCells (Components) | Python functions decorated with `@gf.cell` |
| Simulation Settings | Pydantic models with material data |
| Connectivity | A list of `(layer1, via, layer2)` tuples |
| Layer Views | A YAML file or `LayerViews` object |
| The PDK itself | A `Pdk` object that bundles everything together |

The entire PDK is Python code. No proprietary languages, no binary blobs, no vendor lock-in.

---

## 7. Step-by-Step: Building Your Own PDK

This is the practical section. Here is exactly what you need to create.

### Step 1: Define Your Layer Map

This is always the first thing you do. You need to decide what layers your process uses and assign them GDS numbers.

```python
# my_pdk/layer_map.py

import gdsfactory as gf

Layer = tuple[int, int]

class LAYER(gf.LayerEnum):
    """Layer map for my custom foundry process."""

    # Waveguide layers
    WG: Layer = (1, 0)           # Silicon waveguide core
    WGCLAD: Layer = (111, 0)     # Waveguide cladding

    # Slab layers (partial etches)
    SLAB150: Layer = (2, 0)      # 150nm slab (shallow etch)
    SLAB90: Layer = (3, 0)       # 90nm slab (deep etch)

    # Doping layers
    N: Layer = (20, 0)           # N-type doping
    P: Layer = (21, 0)           # P-type doping
    NPP: Layer = (24, 0)        # N++ doping (heavy)
    PPP: Layer = (25, 0)        # P++ doping (heavy)

    # Metal layers
    HEATER: Layer = (47, 0)      # TiN heater
    M1: Layer = (41, 0)          # Metal 1
    M2: Layer = (45, 0)          # Metal 2

    # Via layers
    VIAC: Layer = (40, 0)        # Via: silicon to metal 1
    VIA1: Layer = (44, 0)        # Via: metal 1 to metal 2

    # Utility layers
    PORT: Layer = (1, 10)        # Port markers
    TEXT: Layer = (66, 0)        # Text labels
    FLOORPLAN: Layer = (64, 0)   # Chip floorplan boundary
```

Each line says: "Layer named X lives at GDS layer number Y, datatype Z."


### Step 2: Define Your Layer Stack

Now describe the physical structure. What is each layer made of, how thick is it, and where does it sit vertically?

```python
# my_pdk/layer_stack.py

from gdsfactory.technology import LayerLevel, LayerStack, LogicalLayer
from my_pdk.layer_map import LAYER

nm = 1e-3  # convert nanometers to micrometers

def get_layer_stack() -> LayerStack:
    layers = dict(
        # The silicon substrate at the very bottom
        substrate=LayerLevel(
            layer=LogicalLayer(layer=LAYER.WAFER),
            thickness=10.0,
            zmin=-13.0,
            material="si",
        ),
        # Buried oxide (BOX) - the insulator under the waveguide
        box=LayerLevel(
            layer=LogicalLayer(layer=LAYER.WAFER),
            thickness=3.0,
            zmin=-3.0,
            material="sio2",
        ),
        # The waveguide core - this is the most important layer
        core=LayerLevel(
            layer=LogicalLayer(layer=LAYER.WG),
            thickness=220 * nm,       # 220 nm thick
            zmin=0.0,                 # reference point
            material="si",            # silicon
            sidewall_angle=10,        # degrees from vertical
        ),
        # Cladding oxide above the waveguide
        clad=LayerLevel(
            layer=LogicalLayer(layer=LAYER.WAFER),
            thickness=3.0,
            zmin=0.0,
            material="sio2",
        ),
        # Metal 1 for electrical routing
        metal1=LayerLevel(
            layer=LogicalLayer(layer=LAYER.M1),
            thickness=700 * nm,
            zmin=1.1,
            material="Aluminum",
        ),
        # Heater for thermal tuning
        heater=LayerLevel(
            layer=LogicalLayer(layer=LAYER.HEATER),
            thickness=750 * nm,
            zmin=1.1,
            material="TiN",
        ),
    )
    return LayerStack(layers=layers)

LAYER_STACK = get_layer_stack()
```


### Step 3: Define Cross Sections

Cross sections tell the router how to draw waveguides and connections.

```python
# my_pdk/cross_sections.py

import gdsfactory as gf
from gdsfactory.cross_section import cross_section
from my_pdk.layer_map import LAYER

# A standard strip waveguide: 500nm wide silicon core
strip = cross_section(
    width=0.5,
    layer=LAYER.WG,
)

# A wider waveguide for multimode applications
strip_wide = cross_section(
    width=2.0,
    layer=LAYER.WG,
)

# A rib waveguide: partially etched (has a slab underneath)
rib = cross_section(
    width=0.5,
    layer=LAYER.WG,
    sections=[
        gf.Section(width=6.0, layer=LAYER.SLAB150, name="slab"),
    ],
)

# Metal routing for electrical connections
metal_routing = cross_section(
    width=10.0,
    layer=LAYER.M1,
)

# Bundle them for the PDK
cross_sections = {
    "strip": strip,
    "strip_wide": strip_wide,
    "rib": rib,
    "metal_routing": metal_routing,
}
```


### Step 4: Create Your Component Library (PCells)

These are the reusable building blocks of your PDK.

```python
# my_pdk/cells.py

import gdsfactory as gf
from my_pdk.layer_map import LAYER

@gf.cell
def my_straight(length: float = 10.0, width: float = 0.5) -> gf.Component:
    """A straight waveguide.

    Args:
        length: waveguide length in um.
        width: waveguide width in um.
    """
    c = gf.Component()
    # Draw a rectangle on the waveguide layer
    c.add_polygon(
        [(0, -width/2), (length, -width/2), (length, width/2), (0, width/2)],
        layer=LAYER.WG,
    )
    # Add input port (left side)
    c.add_port(
        name="o1",
        center=(0, 0),
        width=width,
        orientation=180,  # pointing left
        layer=LAYER.WG,
    )
    # Add output port (right side)
    c.add_port(
        name="o2",
        center=(length, 0),
        width=width,
        orientation=0,    # pointing right
        layer=LAYER.WG,
    )
    return c

@gf.cell
def my_taper(length: float = 10.0, width1: float = 0.5, width2: float = 3.0) -> gf.Component:
    """A linear taper that transitions between two widths.

    Args:
        length: taper length in um.
        width1: input width in um.
        width2: output width in um.
    """
    c = gf.Component()
    c.add_polygon(
        [
            (0, -width1/2),
            (length, -width2/2),
            (length, width2/2),
            (0, width1/2),
        ],
        layer=LAYER.WG,
    )
    c.add_port(name="o1", center=(0, 0), width=width1, orientation=180, layer=LAYER.WG)
    c.add_port(name="o2", center=(length, 0), width=width2, orientation=0, layer=LAYER.WG)
    return c

# Collect all cells for the PDK
cells = {
    "my_straight": my_straight,
    "my_taper": my_taper,
}
```


### Step 5: Define Simulation Settings

```python
# my_pdk/simulation_settings.py

from functools import partial
import numpy as np
from scipy import interpolate

# Refractive index data (wavelength in um)
wavelengths = [1.2, 1.3, 1.4, 1.5, 1.55, 1.6]

# Silicon refractive index at each wavelength
n_silicon =   [3.518, 3.502, 3.489, 3.479, 3.475, 3.471]

# Silicon dioxide refractive index
n_oxide =     [1.459, 1.458, 1.458, 1.458, 1.457, 1.457]

def get_material_index(wav, wavelengths, n_values):
    f = interpolate.interp1d(wavelengths, n_values)
    return f(wav)

materials_index = {
    "si": partial(get_material_index, wavelengths=wavelengths, n_values=n_silicon),
    "sio2": partial(get_material_index, wavelengths=wavelengths, n_values=n_oxide),
}
```


### Step 6: Assemble the PDK

Now bring everything together into a single `Pdk` object.

```python
# my_pdk/__init__.py

from gdsfactory.pdk import Pdk
from my_pdk.layer_map import LAYER
from my_pdk.layer_stack import LAYER_STACK
from my_pdk.cross_sections import cross_sections
from my_pdk.cells import cells
from my_pdk.simulation_settings import materials_index

LAYER_CONNECTIVITY = [
    ("NPP", "VIAC", "M1"),
    ("PPP", "VIAC", "M1"),
    ("M1", "VIA1", "M2"),
]

PDK = Pdk(
    name="my_custom_pdk",
    version="0.1.0",
    cells=cells,
    cross_sections=cross_sections,
    layers=LAYER,
    layer_stack=LAYER_STACK,
    materials_index=materials_index,
    connectivity=LAYER_CONNECTIVITY,
)
```


### Step 7: Use Your PDK

```python
# design.py
from my_pdk import PDK

# Activate your PDK (makes it the current active PDK)
PDK.activate()

# Now design with it
import gdsfactory as gf

c = gf.Component("my_design")
wg = c << gf.get_component("my_straight", length=20.0)
taper = c << gf.get_component("my_taper", length=5.0)
taper.connect("o1", wg.ports["o2"])

c.write_gds("my_design.gds")
```

---

## 8. How the Generic PDK in gdsfactory Is Structured

Your project already uses the generic PDK that comes built-in with gdsfactory. Here is exactly how it is organized. These files are in your virtual environment at `.venv312/Lib/site-packages/gdsfactory/gpdk/`:

### File: `layer_map.py`
Defines 40+ layers including:
- Waveguide layers: WG (1,0), WGN (34,0) for nitride
- Slab layers: SLAB150 (2,0), SLAB90 (3,0)
- Doping: N (20,0), P (21,0), NP, PP, NPP, PPP
- Metals: M1 (41,0), M2 (45,0), M3 (49,0)
- Vias: VIAC (40,0), VIA1 (44,0), VIA2 (43,0)
- Utility: PORT (1,10), TEXT (66,0), FLOORPLAN (64,0)

### File: `layer_stack.py`
Defines the physical structure:
- Silicon substrate: 10 um thick, below everything
- Buried oxide (BOX): 3 um thick SiO2
- Waveguide core: 220 nm thick silicon at z=0
- Slab layers: 90nm and 150nm partial etches
- Silicon nitride: 350nm thick, sitting above the silicon with a 100nm gap
- Three metal layers: M1 at 1.1um, M2 at 2.3um, M3 at 3.2um
- Heater (TiN): at 1.1um height

Also defines the fabrication **process steps**: etches, implants, annealing, metallization, planarization.

### File: `simulation_settings.py`
Contains:
- Refractive index data for silicon, silicon nitride, and silicon dioxide
- Wavelength range: 0.6 to 2.0 um
- Lumerical FDTD simulation parameters
- Interpolation functions for getting refractive index at any wavelength

### File: `__init__.py`
Assembles everything into a `Pdk` object:
- Registers all built-in gdsfactory components (hundreds of them)
- Defines port types (optical, electrical, vertical TE/TM)
- Sets up routing strategies for automatic waveguide/metal routing
- Defines layer transitions (how to connect different cross sections)
- Creates and exports the `PDK` object

When you call `gf.gpdk.PDK.activate()` in your `demo.py`, you are loading all of this.

---

## 9. The Complete Workflow: From PDK to Fabricated Chip

Here is the full picture of how a PDK fits into the chip design flow:

```
Step 1: Foundry creates the PDK
    - Defines layers, rules, components, models
    - Validates components through test fabrication
    - Packages everything as a PDK (or iPDK)

Step 2: Designer receives the PDK
    - Installs it in their design environment
    - Activates it: PDK.activate()

Step 3: Designer creates the layout
    - Uses PDK components: waveguides, bends, couplers, modulators
    - Connects them together using ports
    - Routes electrical and optical connections

Step 4: Design Rule Check (DRC)
    - Automated tool checks the layout against foundry rules
    - Reports violations (shape too narrow, spacing too small, etc.)
    - Designer fixes violations

Step 5: Layout vs Schematic (LVS)
    - Compares the physical layout to the intended circuit schematic
    - Ensures every connection in the schematic exists in the layout
    - Reports mismatches

Step 6: Simulation
    - Optical simulation: light propagation, coupling, losses
    - Electrical simulation: heater response, modulator bandwidth
    - Uses refractive index data from the PDK

Step 7: Export and Tape-out
    - Final GDS file is generated
    - Sent to the foundry for fabrication
    - Foundry manufactures the chip using their process
```

---

## 10. What You Can Do in This Project

Given that your internship goal is to build your own iPDK using Python, here is what that concretely means:

### Core Tasks

1. **Define a layer map** for your target process. Decide what layers exist and assign them GDS numbers.

2. **Define a layer stack** describing the physical structure: thickness, material, and vertical position of each layer.

3. **Create cross sections** for the waveguide types your process supports (strip, rib, slot, etc.).

4. **Build a component library** of PCells: straight waveguides, bends (Euler, circular), tapers, MMI splitters, directional couplers, grating couplers, ring resonators, crossings.

5. **Add simulation settings**: refractive index data for your materials, wavelength ranges.

6. **Package it as a Pdk object** using gdsfactory's `Pdk` class.

7. **Create KLayout integration**: layer view files (.lyp or .yaml) so the design looks correct when opened in KLayout.

### Validation and Testing

8. **Write DRC rules** (can be done in Python with gdsfactory or as KLayout DRC scripts).

9. **Test every component**: generate test structures, run DRC, verify port connectivity.

10. **Create documentation**: a user guide explaining how to use your PDK.

### Advanced (if time permits)

11. **Add process simulation**: define the fabrication steps (etch, implant, deposit) and simulate the resulting 3D structure.

12. **Add compact models**: behavioral models for each component that can be used in circuit-level simulation.

13. **OpenAccess export**: if needed for interoperability with commercial tools.

### Project Structure You Should Aim For

```
my_ipdk/
├── __init__.py              # PDK assembly and activation
├── layer_map.py             # Layer definitions
├── layer_stack.py           # Physical layer stack
├── cross_sections.py        # Waveguide cross sections
├── cells/                   # Component library
│   ├── __init__.py
│   ├── waveguides.py        # Straight, taper
│   ├── bends.py             # Euler bend, circular bend
│   ├── couplers.py          # Directional coupler, MMI
│   ├── gratings.py          # Grating couplers
│   └── rings.py             # Ring resonators
├── simulation_settings.py   # Material data, wavelength ranges
├── layer_views.yaml         # KLayout display settings
├── drc/
│   └── rules.py             # Design rule checks
├── tests/
│   ├── test_cells.py        # Component tests
│   └── test_drc.py          # DRC tests
└── docs/
    └── user_guide.md        # How to use this PDK
```

---

## 11. Key Terms Reference

| Term | Meaning |
|------|---------|
| PDK | Process Design Kit — the bridge between foundry process and designer tools |
| iPDK | Interoperable PDK — a PDK that works across multiple EDA tools using open standards |
| OpenAccess | An open-source database standard for IC design data |
| PCell | Parameterized Cell — a function that generates layout geometry from parameters |
| Layer Map | Mapping of human-readable names to GDS layer numbers |
| Layer Stack | 3D physical description of all layers (thickness, material, position) |
| Cross Section | Definition of how a path is drawn (which layers, what widths) |
| DRC | Design Rule Check — automated verification against manufacturing constraints |
| LVS | Layout vs Schematic — checks that layout matches the intended circuit |
| GDS/GDSII | The binary file format for IC layout data |
| Foundry | The factory that manufactures chips |
| Tape-out | The act of sending your final design to the foundry for fabrication |
| Port | A connection point on a component (position + direction + width + layer) |
| Component | A reusable layout block (equivalent to a cell in GDS) |
| @gf.cell | A Python decorator that marks a function as a component generator |
| KLayout | Open-source layout viewer and editor |
| gdsfactory | Python framework for photonic IC design and PDK creation |
| EDA | Electronic Design Automation — software tools for chip design |
| BOX | Buried Oxide — the SiO2 layer beneath the silicon waveguide |
| Euler Bend | A waveguide bend that gradually changes curvature to minimize optical loss |
| MMI | Multimode Interference — a type of optical splitter/combiner |
| Routing | Automatically drawing waveguide/metal paths between components |
