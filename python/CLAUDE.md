# CTSM Python Package

This directory contains the Python implementations for CTSM tools. The `tools/` directory provides thin CLI wrappers that import and call code from here.

**Key insight:** If you're debugging a tool from `tools/`, the actual code is here in `python/ctsm/`.

---

## Package Structure

```
python/
├── ctsm/                           # Main package (~5,600 LOC)
│   ├── __init__.py
│   ├── subset_data.py              # Data subsetting (897 LOC)
│   ├── mesh_maker.py               # Mesh file creation (313 LOC)
│   ├── mesh_plotter.py             # Mesh visualization (178 LOC)
│   ├── run_sys_tests.py            # System test orchestration (849 LOC)
│   ├── lilac_build_ctsm.py         # LILAC build interface (881 LOC)
│   ├── utils.py                    # Common utilities (276 LOC)
│   ├── longitude.py                # Longitude handling (211 LOC)
│   ├── config_utils.py             # Config file utilities (204 LOC)
│   ├── machine.py                  # Machine configuration (203 LOC)
│   │
│   ├── site_and_regional/          # Single-point/regional tools (4,834 LOC)
│   │   ├── single_point_case.py    # SinglePointCase class (738 LOC)
│   │   ├── regional_case.py        # RegionalCase class (575 LOC)
│   │   ├── base_case.py            # BaseCase, DatmFiles (244 LOC)
│   │   ├── tower_site.py           # TowerSite class (500 LOC)
│   │   ├── run_tower.py            # Tower automation (337 LOC)
│   │   ├── run_neon.py             # NEON automation
│   │   ├── mesh_type.py            # Mesh data structure (591 LOC)
│   │   └── default_data_*.cfg      # Default paths (modified for HiPerGator)
│   │
│   ├── modify_input_files/         # Input modification (1,464 LOC)
│   │   ├── fsurdat_modifier.py     # Surface dataset modifier (669 LOC)
│   │   ├── modify_fsurdat.py       # CLI for fsurdat (505 LOC)
│   │   ├── mesh_mask_modifier.py   # Mesh mask modifier (97 LOC)
│   │   └── modify_mesh_mask.py     # CLI for mesh mask (193 LOC)
│   │
│   ├── crop_calendars/             # Crop calendar tools (6,634 LOC)
│   │   ├── generate_gdds.py        # GDD generation
│   │   ├── process_ggcmi_shdates.py  # GGCMI processing
│   │   └── regrid_ggcmi_shdates.py   # GGCMI regridding
│   │
│   ├── toolchain/                  # mksurfdata generation (1,628 LOC)
│   │   ├── gen_mksurfdata_namelist.py
│   │   ├── gen_mksurfdata_jobscript_multi.py
│   │   └── gen_mksurfdata_jobscript_single.py
│   │
│   ├── joblauncher/                # Job submission abstraction (258 LOC)
│   │   ├── job_launcher_base.py    # Base class
│   │   ├── job_launcher_factory.py # Factory pattern
│   │   ├── job_launcher_qsub.py    # PBS/TORQUE
│   │   └── job_launcher_no_batch.py  # Direct execution
│   │
│   ├── param_utils/                # Parameter file utilities
│   │   ├── set_paramfile.py
│   │   └── query_paramfile.py
│   │
│   └── test/                       # Test suite (~12,300 LOC)
│       ├── test_unit_*.py          # Unit tests (41 files)
│       ├── test_sys_*.py           # System tests (12 files)
│       └── testinputs/             # Test fixtures
│
├── run_ctsm_py_tests               # Test runner script
├── Makefile                        # Test automation
├── pyproject.toml                  # Python project config
├── README.md                       # Package documentation
└── conda_env_ctsm_py.yml          # Conda environment spec
```

---

## Wrapper → Implementation Map

| Wrapper (`tools/`) | Implementation (`python/ctsm/`) |
|--------------------|--------------------------------|
| `site_and_regional/subset_data` | `subset_data.py` |
| `site_and_regional/run_neon` | `site_and_regional/run_neon.py` |
| `site_and_regional/run_tower` | `site_and_regional/run_tower.py` |
| `site_and_regional/mesh_maker` | `mesh_maker.py` |
| `site_and_regional/mesh_plotter` | `mesh_plotter.py` |
| `modify_input_files/fsurdat_modifier` | `modify_input_files/fsurdat_modifier.py` |
| `modify_input_files/mesh_mask_modifier` | `modify_input_files/mesh_mask_modifier.py` |
| `crop_calendars/generate_gdds` | `crop_calendars/generate_gdds.py` |
| `param_utils/set_paramfile` | `param_utils/set_paramfile.py` |
| `param_utils/query_paramfile` | `param_utils/query_paramfile.py` |

---

## Key Classes

### SinglePointCase (`site_and_regional/single_point_case.py`)
Creates single-point domain, surface, and DATM files.

```python
from ctsm.site_and_regional.single_point_case import SinglePointCase

case = SinglePointCase(
    lat=29.7, lon=-82.0,
    site_name="OSBS",
    create_domain=True,
    create_surfdata=True,
    create_datm=True
)
case.create_case_files(output_dir="/path/to/output")
```

### RegionalCase (`site_and_regional/regional_case.py`)
Creates regional domain, surface, and DATM files.

### FsurdatModifier (`modify_input_files/fsurdat_modifier.py`)
Modifies surface datasets (PFTs, soil properties, etc.).

---

## Running Tests

```bash
cd python

# All tests
make test

# Unit tests only
make utest

# System tests only
make stest

# Linting
make lint

# Code formatting check
make black
```

### Direct test execution
```bash
# Run specific test file
python -m pytest ctsm/test/test_unit_subset_data.py -v

# Run with coverage
python -m pytest --cov=ctsm ctsm/test/
```

---

## HiPerGator Modifications

### 1. MPILIB Setting (`site_and_regional/single_point_case.py`)

Changed from `mpi-serial` to `openmpi`:
```python
# Line ~200 in single_point_case.py
self._mpilib = "openmpi"  # Was: "mpi-serial"
```

### 2. Default Data Paths (`site_and_regional/default_data_*.cfg`)

Changed input data path:
```ini
[main]
inputdata_path = /blue/gerber/earth_models/inputdata
```

---

## Adding New Functionality

### Pattern for New Tool

1. Create implementation in `python/ctsm/your_module.py`
2. Add a `main()` function as entry point
3. Create wrapper in `tools/your_category/your_tool`:

```python
#!/usr/bin/env python3
"""Wrapper for your_module."""
import os
import sys

_CTSM_PYTHON = os.path.join(
    os.path.dirname(os.path.realpath(__file__)),
    os.pardir, os.pardir, "python"
)
sys.path.insert(1, _CTSM_PYTHON)

from ctsm.your_module import main

if __name__ == "__main__":
    main()
```

4. Add tests in `python/ctsm/test/test_unit_your_module.py`

---

## Code Quality Tools

| Tool | Config File | Purpose |
|------|-------------|---------|
| **pylint** | `.pylintrc` | Static analysis |
| **black** | `pyproject.toml` | Code formatting (line-length=100) |
| **pytest** | (default) | Testing framework |

---

## Dependencies

Core dependencies (see `conda_env_ctsm_py.yml`):
- Python 3.8+
- numpy
- xarray
- netCDF4
- scipy
- configparser

---

## See Also

- `README.md` - Upstream Python package documentation
- `ctsm/site_and_regional/CLAUDE.md` - Detailed site_and_regional docs
- `../tools/CLAUDE.md` - Tools directory overview
- `ctsm/test/README` - Test infrastructure details
