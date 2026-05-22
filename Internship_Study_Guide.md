# Silicon Photonics PDK Development Study Guide and Development Procedure

This guide is designed for research interns at the Indian Institute of Technology Bhilai (IIT Bhilai) who are starting from absolute basics. It outlines what to study step-by-step and provides the exact code structure and procedure for creating a custom Process Design Kit (PDK) in this Python workspace.

---

## Part 1: Step-by-Step Study Curriculum

### Step 1: Silicon Photonics Basics
Before writing code, you must understand the physical hardware you are designing.
* Core Concept: Silicon Photonics uses silicon as an optical medium to route light on a chip, similar to how copper wires route electrons on an electrical chip.
* Materials: A standard Silicon-on-Insulator (SOI) wafer consists of:
  * Silicon Substrate (handle wafer)
  * Buried Oxide (BOX) Layer: Typically 2 to 3 micrometers of Silicon Dioxide (SiO2) that acts as the bottom cladding.
  * Silicon Device Layer: Typically 220 nanometers of Silicon (Si) that forms the core of the waveguide.
  * Top Cladding: Typically Silicon Dioxide (SiO2) or air.
* Index Contrast: Silicon has a refractive index of approximately 3.47 at a wavelength of 1550 nanometers. Silicon Dioxide has a refractive index of approximately 1.44. This high index contrast confines light strongly inside the silicon core via Total Internal Reflection (TIR).
* Waveguide Dimensions: A standard strip waveguide at 1550nm wavelength is 500nm (0.5µm) wide and 220nm (0.22µm) thick. This ensures single-mode propagation (only one spatial mode of light travels through it).

### Step 2: GDSII Files and KLayout GUI
* GDSII format: A hierarchical database that represents physical layouts using 2D geometric shapes (polygons and paths) placed on specific layer numbers.
* Layers: Foundries use different masks to etch materials. Each mask is represented by a Layer Number and a Datatype Number (e.g., Layer 1/0 for Silicon core, Layer 2/0 for Deep Silicon Etch, Layer 31/0 for Metal Routing).
* KLayout: The industry-standard open-source layout viewer.
  * Exercise: Open KLayout. Create a new layout. Manually draw a polygon on Layer 1/0. Save it as a `.gds` file. Open the file in KLayout and observe the cell hierarchy panel on the left and the layers panel on the right.

### Step 3: Programmable Layout using gdsfactory
* The Core Object: `gf.Component`. A Component is a cell containing:
  * Polygons: Physical shapes on specific layers.
  * References: Pointers to other cells (prevents file size explosion by reusing identical structures).
  * Ports: Logical connection points on the cell's boundaries. A port has a center coordinate (x, y), an orientation angle (in degrees: 0 = East, 90 = North, 180 = West, 270 = South), a width, and a cross-section/layer type.
* Connections: The command `ref2.connect("o1", ref1.ports["o2"])` translates and rotates the component referenced by `ref2` so that its port `"o1"` matches the position and angle of `ref1`'s port `"o2"`.

### Step 4: Anatomy of a Process Design Kit (PDK)
A PDK is a collection of files that adapts a design tool to a specific manufacturing process. It contains:
1. Layer Map: Mappings of names to (Layer, Datatype) pairs.
2. Waveguide Cross-Sections: Definitions of waveguide geometries (width, layers, cladding offsets).
3. Parametric Cells (PCells): Reusable Python functions that generate custom layouts based on user-defined inputs (e.g., waveguide length, ring radius).
4. Layer Stack: 3D representation of layers (materials, thickness, vertical offsets) used for rendering and simulations.
5. DRC Rules: Constraints like minimum feature size and spacing to prevent manufacturing failures.

---

## Part 2: Procedure for Creating PDK Files in This Workspace

To build your custom PDK, you should create a structured directory in your workspace. Do not write everything in a single, unstructured script.

### Proposed Directory Structure
Create a folder named `iitb_pdk/` inside `c:\Users\User\Downloads\Internship/`. Inside it, create the following files:

```
c:/Users/User/Downloads/Internship/
│
├── iitb_pdk/                  # Your custom PDK package folder
│   ├── __init__.py            # Exposes PDK initialization
│   ├── layers.py              # Layer definitions and LayerMap
│   ├── cross_sections.py      # Waveguide cross-sections
│   ├── components.py          # Custom PCells (waveguides, couplers, etc.)
│   └── pdk.py                 # PDK registration and activation script
│
├── test_my_pdk.py             # Script to verify and generate GDS files using your PDK
└── check_libs.py              # Library diagnostic script
```

### Step-by-Step File Implementation

#### 1. Define Layers (`iitb_pdk/layers.py`)
This file maps names to GDS layer/datatype pairs.

```python
from gdsfactory.technology import LayerMap, LayerSpec

class IITB_Layers(LayerMap):
    # Core silicon layer
    WG_CORE: LayerSpec = (1, 0)
    
    # Deep silicon etch (for ribs or cladding isolation)
    WG_CLAD: LayerSpec = (2, 0)
    
    # Metal layer for heaters or contacts
    METAL: LayerSpec = (31, 0)
    
    # Text label layer for documentation
    TEXT: LayerSpec = (66, 0)

# Instantiate the layer map
LAYER = IITB_Layers()
```

#### 2. Define Waveguide Cross-Sections (`iitb_pdk/cross_sections.py`)
This file defines how physical waveguides are drawn when generating paths.

```python
import gdsfactory as gf
from iitb_pdk.layers import LAYER

# Define a standard strip waveguide cross-section
def xs_strip(width: float = 0.5) -> gf.CrossSection:
    return gf.CrossSection(
        sections=(
            gf.Section(
                width=width,
                layer=LAYER.WG_CORE,
                name="core",
                port_names=("o1", "o2"),
                port_types=("optical", "optical")
            ),
        )
    )

# Define a rib waveguide cross-section (with a slab layer underneath)
def xs_rib(width: float = 0.5, slab_width: float = 2.0) -> gf.CrossSection:
    return gf.CrossSection(
        sections=(
            # Main core ridge
            gf.Section(
                width=width,
                layer=LAYER.WG_CORE,
                name="core",
                port_names=("o1", "o2"),
                port_types=("optical", "optical")
            ),
            # Thinner slab layer
            gf.Section(
                width=slab_width,
                layer=LAYER.WG_CLAD,
                name="slab"
            )
        )
    )

# Register cross sections in gdsfactory
cross_sections = {
    "xs_strip": xs_strip,
    "xs_rib": xs_rib
}
```

#### 3. Define Custom Components/PCells (`iitb_pdk/components.py`)
Create parametric cells that generate layout geometries based on variables.

```python
import gdsfactory as gf
from iitb_pdk.layers import LAYER

@gf.cell
def custom_straight(length: float = 10.0, width: float = 0.5) -> gf.Component:
    """A basic straight waveguide using the custom PDK layers."""
    c = gf.Component()
    
    # Create the rectangular core geometry
    c.add_polygon(
        [
            (0, -width/2),
            (length, -width/2),
            (length, width/2),
            (0, width/2)
        ],
        layer=LAYER.WG_CORE
    )
    
    # Add input and output ports
    c.add_port(
        name="o1",
        center=(0, 0),
        width=width,
        orientation=180,
        port_type="optical",
        layer=LAYER.WG_CORE
    )
    c.add_port(
        name="o2",
        center=(length, 0),
        width=width,
        orientation=0,
        port_type="optical",
        layer=LAYER.WG_CORE
    )
    return c

@gf.cell
def y_splitter(width: float = 0.5, length: float = 10.0, separation: float = 3.0) -> gf.Component:
    """A Y-junction splitter layout."""
    c = gf.Component()
    
    # Input waveguide segment
    input_wg = c << custom_straight(length=length/3, width=width)
    
    # Output ports definition
    c.add_port(name="o1", port=input_wg.ports["o1"])
    
    # Output waveguide branches
    out_wg1 = c << custom_straight(length=2*length/3, width=width)
    out_wg2 = c << custom_straight(length=2*length/3, width=width)
    
    # Position the outputs with the specified separation
    out_wg1.movey(separation / 2)
    out_wg1.movex(length/3)
    out_wg2.movey(-separation / 2)
    out_wg2.movex(length/3)
    
    # Connect waveguides using path routing
    # Add port definitions for the split outputs
    c.add_port(name="o2", port=out_wg1.ports["o2"])
    c.add_port(name="o3", port=out_wg2.ports["o2"])
    
    return c

# Map custom PCells to a components list
components = {
    "custom_straight": custom_straight,
    "y_splitter": y_splitter
}
```

#### 4. Define and Register the PDK (`iitb_pdk/pdk.py`)
This file registers your custom components, cross-sections, and layers inside the gdsfactory environment.

```python
import gdsfactory as gf
from iitb_pdk.layers import LAYER
from iitb_pdk.cross_sections import cross_sections
from iitb_pdk.components import components

# Define the custom PDK object
PDK = gf.Pdk(
    name="IITB_PDK",
    cells=components,
    cross_sections=cross_sections,
    layers=dict(LAYER),
)

def register_pdk():
    """Register and activate the IIT Bhilai PDK."""
    PDK.register()
    PDK.activate()
    print("IITB_PDK successfully registered and activated.")
```

#### 5. Expose PDK Initialization (`iitb_pdk/__init__.py`)
Make the directory importable as a Python package.

```python
from iitb_pdk.pdk import register_pdk, PDK, LAYER

__all__ = ["register_pdk", "PDK", "LAYER"]
```

---

## Part 3: Testing Your Custom PDK

Create a file named `test_my_pdk.py` in the main directory `c:\Users\User\Downloads\Internship\` to test if the custom PDK works, creates routing, and exports a GDS layout.

```python
import gdsfactory as gf
from iitb_pdk import register_pdk

# 1. Register and activate the custom PDK
register_pdk()

# 2. Create a top-level component
top_cell = gf.Component("iitb_pdk_test_layout")

# 3. Instantiate components from the custom PDK registry
splitter = top_cell << gf.get_component("y_splitter", width=0.5, length=12.0, separation=4.0)
straight1 = top_cell << gf.get_component("custom_straight", length=15.0, width=0.5)
straight2 = top_cell << gf.get_component("custom_straight", length=15.0, width=0.5)

# 4. Physically connect components using their ports
straight1.connect("o1", splitter.ports["o2"])
straight2.connect("o1", splitter.ports["o3"])

# 5. Export layout to GDSII format
gds_filename = "my_custom_pdk_layout.gds"
top_cell.write_gds(gds_filename)

print(f"Successfully generated layout using custom PDK and saved to: {gds_filename}")
```

To run the test script:
```powershell
python test_my_pdk.py
```
This generates `my_custom_pdk_layout.gds` containing the layout drawn entirely using your custom PDK layer configuration and cells. You can open and inspect it using KLayout or `view_gds.py`.
