# site_and_regional Implementation

This directory contains the actual Python implementations for single-point and regional CTSM tools. The wrapper scripts in `tools/site_and_regional/` call code from here.

---

## Module Overview

| File | LOC | Purpose |
|------|-----|---------|
| `single_point_case.py` | 738 | SinglePointCase class - main single-point logic |
| `regional_case.py` | 575 | RegionalCase class - regional setup |
| `base_case.py` | 244 | BaseCase parent class, DatmFiles helper |
| `tower_site.py` | 500 | TowerSite class for tower metadata |
| `run_tower.py` | 337 | Tower simulation orchestration |
| `run_neon.py` | ~300 | NEON site automation |
| `modify_singlept_site_neon.py` | 712 | NEON-specific surface modifications |
| `mesh_type.py` | 591 | Mesh data structure and operations |
| `mesh_plot_type.py` | 140 | Mesh plotting configuration |
| `neon_surf_wrapper.py` | 227 | NEON surface data wrapper |
| `plumber2_surf_wrapper.py` | 196 | PLUMBER2 site wrapper |
| `default_data_*.cfg` | - | Default configuration (HiPerGator paths) |

---

## Class Hierarchy

```
BaseCase
├── SinglePointCase    # Single gridcell simulations
└── RegionalCase       # Multi-gridcell regional simulations

TowerSite              # Tower metadata container (standalone)
DatmFiles              # DATM file management (helper in base_case.py)
MeshType               # Mesh data structure (standalone)
```

---

## SinglePointCase (`single_point_case.py`)

Main class for setting up single-point simulations.

### Key Methods

```python
class SinglePointCase(BaseCase):
    def __init__(self, lat, lon, site_name, ...):
        """Initialize with coordinates and options."""

    def create_domain(self, output_dir):
        """Create domain.lnd.nc file."""

    def create_surfdata(self, output_dir):
        """Create surfdata_*.nc file."""

    def create_datm(self, output_dir):
        """Create DATM forcing files."""

    def create_user_mods(self, output_dir):
        """Create user_mods directory with namelists."""
```

### HiPerGator Modification

Line ~200 (approximately):
```python
# Original (upstream)
self._mpilib = "mpi-serial"

# Our fork
self._mpilib = "openmpi"
```

This change was necessary because mpi-serial had linking issues on HiPerGator.

### Key Attributes

| Attribute | Type | Purpose |
|-----------|------|---------|
| `lat`, `lon` | float | Site coordinates |
| `site_name` | str | Site identifier |
| `pft_16` | bool | Use 16-PFT vs 17-PFT |
| `create_domain` | bool | Generate domain file |
| `create_surfdata` | bool | Generate surface data |
| `create_datm` | bool | Generate DATM forcing |
| `create_user_mods` | bool | Generate user_mods dir |
| `dom_pft` | int | Dominant PFT (if overriding) |

---

## RegionalCase (`regional_case.py`)

Similar to SinglePointCase but for multi-gridcell regions.

### Key Differences from SinglePointCase

- Takes bounding box (lat1, lat2, lon1, lon2) instead of single point
- Handles multi-cell domain files
- More complex surface data subsetting

---

## BaseCase (`base_case.py`)

Parent class with shared functionality.

### DatmFiles Helper Class

Manages DATM atmospheric forcing files:

```python
class DatmFiles:
    """Handle DATM forcing file operations."""

    def __init__(self, datm_type):
        """Initialize with DATM type (e.g., 'GSWP3v1')."""

    def get_stream_files(self):
        """Return list of stream files for this DATM type."""

    def subset_streams(self, lat, lon, output_dir):
        """Subset stream files to point/region."""
```

---

## TowerSite (`tower_site.py`)

Container for tower site metadata.

```python
class TowerSite:
    def __init__(self, name, lat, lon, **kwargs):
        """Initialize tower site."""

    @classmethod
    def from_neon(cls, neon_site_code):
        """Create TowerSite from NEON site code."""

    @classmethod
    def from_config(cls, config_file):
        """Create TowerSite from config file."""
```

---

## Configuration Files

### default_data_1x1.cfg (HiPerGator Modified)

```ini
[main]
inputdata_path = /blue/gerber/earth_models/inputdata

[surfdata]
# Surface data defaults

[datm]
# DATM defaults
```

### Key Configuration Sections

| Section | Purpose |
|---------|---------|
| `[main]` | Input data paths |
| `[surfdata]` | Surface data generation options |
| `[datm]` | DATM forcing options |
| `[domain]` | Domain file options |

---

## Common Usage Patterns

### Creating a Single-Point Case Programmatically

```python
from ctsm.site_and_regional.single_point_case import SinglePointCase

# Create case object
case = SinglePointCase(
    lat=29.6896,
    lon=-82.0212,
    site_name="OSBS",
    create_domain=True,
    create_surfdata=True,
    create_datm=False,
    create_user_mods=True
)

# Generate files
case.create_case_files("/path/to/output")
```

### Creating from Command Line (via subset_data)

```bash
# The subset_data wrapper calls SinglePointCase internally
./subset_data point --lat 29.7 --lon -82.0 --site OSBS --create-surface
```

---

## Key Implementation Details

### Coordinate Handling

- Longitude expected in -180 to 180 range
- Automatic conversion in `longitude.py`
- Grid cell matching uses nearest neighbor

### Surface Data Subsetting

1. Load global surface data file
2. Find nearest grid cell to coordinates
3. Extract that cell's data
4. Create new NetCDF with single cell

### Domain File Creation

1. Create single-cell ESMF mesh
2. Set land fraction to 1.0
3. Write domain.lnd.nc

### User Mods Generation

Creates directory with:
- `user_nl_clm` - Points to new surface data
- `user_nl_datm_streams` - Points to subset DATM data
- `shell_commands` - Post-setup commands

---

## Testing

Tests are in `../test/`:

| Test File | Coverage |
|-----------|----------|
| `test_unit_singlept_data.py` | SinglePointCase |
| `test_unit_subset_data.py` | subset_data main |
| `test_unit_mesh_type.py` | MeshType |
| `test_sys_subset_data.py` | End-to-end tests |

Run tests:
```bash
cd python
python -m pytest ctsm/test/test_unit_singlept_data.py -v
```

---

## Troubleshooting

### "No surface data found for coordinates"
- Check coordinates are within global data coverage
- Check longitude is in -180 to 180 range

### "MPILIB not found"
- Our fork uses `openmpi`, ensure module is loaded
- Check `single_point_case.py` has our modification

### "Input data path not found"
- Check `default_data_*.cfg` points to correct path
- Default: `/blue/gerber/earth_models/inputdata`

---

## See Also

- `../../CLAUDE.md` - Python package overview
- `../subset_data.py` - Top-level subset_data implementation
- `../../../tools/site_and_regional/CLAUDE.md` - Wrapper documentation
