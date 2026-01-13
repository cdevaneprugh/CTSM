# CTSM - Community Terrestrial Systems Model

This is a fork of ESCOMP/CTSM customized for UF HiPerGator.

## Branch: uf-ctsm5.3.085

Custom branch based on upstream tag `ctsm5.3.085` with the following modifications:

### Build Tool Fixes (mksurfdata_esmf)
- **mksurfdata.F90**: Fixed Fortran format specifiers for GCC 10+ compatibility
- **CMakeLists.txt**: Changed STATIC to SHARED for PIO imported libraries
- **gen_mksurfdata_build**: Added GCC 14 compiler flags (-fallow-argument-mismatch, etc.)

### HiPerGator Configuration
- **single_point_case.py**: MPILIB=openmpi (not mpi-serial)
- **default_data_*.cfg**: Input path set to `/blue/gerber/earth_models/inputdata`

### Research Features
- **spillheight parameter**: Namelist parameter for hillslope hydrology research

---

## Repository Structure (Detailed)

```
CTSM/
├── bld/                        # Build-time configuration
│   ├── namelist_files/         # Namelist definitions and defaults
│   │   ├── namelist_defaults_ctsm.xml    # Default values
│   │   └── namelist_definition_ctsm.xml  # Variable definitions
│   └── unit_testers/           # Build system unit tests
│
├── cime/                       # CIME framework (submodule)
│   ├── scripts/                # Case management scripts
│   │   ├── create_newcase      # Create new simulation case
│   │   ├── create_test         # Create test cases (200+ tests)
│   │   └── fortran_unit_testing/  # pFUnit test framework
│   └── CIME/                   # Python CIME library
│
├── cime_config/                # CTSM-specific CIME configuration
│   ├── buildnml                # Namelist generation
│   ├── config_component.xml    # Component settings
│   └── testdefs/               # Test definitions
│       └── testlist_clm.xml    # 200+ test case definitions
│
├── ccs_config/                 # Machine configurations (submodule - our fork)
│   └── machines/               # Machine-specific files
│       ├── config_machines.xml # Machine definitions (includes hipergator)
│       └── config_batch.xml    # Batch system config
│
├── doc/                        # Documentation (Sphinx RST)
│   ├── source/                 # RST source files (mirrors official docs)
│   └── ChangeLog               # Version history
│
├── libraries/                  # External libraries
│   ├── mpi-serial/             # Serial MPI stub library
│   └── parallelio/             # PIO parallel I/O library
│
├── python/                     # Python package (ACTUAL IMPLEMENTATIONS)
│   ├── ctsm/                   # Main package (~5,600 LOC)
│   │   ├── site_and_regional/  # SinglePointCase, RegionalCase classes
│   │   ├── modify_input_files/ # fsurdat_modifier, mesh_mask_modifier
│   │   ├── crop_calendars/     # Crop calendar processing
│   │   ├── toolchain/          # mksurfdata job generation
│   │   ├── joblauncher/        # Job submission abstraction
│   │   ├── subset_data.py      # Data subsetting implementation
│   │   └── test/               # Python tests (~12,300 LOC)
│   └── README.md               # Python package documentation
│
├── src/                        # Fortran source code (~197,000 LOC)
│   ├── main/                   # Driver, types, initialization
│   ├── biogeophys/             # Hydrology, energy, radiation, hillslope
│   ├── biogeochem/             # Carbon-nitrogen cycling, phenology
│   ├── soilbiogeochem/         # Soil organic matter, decomposition
│   ├── dyn_subgrid/            # Dynamic land units/patches
│   ├── utils/                  # Utilities, time management
│   ├── cpl/                    # Coupler interfaces (LILAC, NUOPC)
│   ├── fates/                  # FATES vegetation model (submodule)
│   └── unit_test_stubs/        # Unit test stub modules
│
└── tools/                      # CLI wrappers (thin shells, 20-40 lines each)
    ├── mksurfdata_esmf/        # Surface dataset generation (Fortran)
    ├── site_and_regional/      # Wrappers for python/ctsm/
    ├── modify_input_files/     # Wrappers for surface modification
    ├── crop_calendars/         # Wrappers for crop processing
    └── contrib/                # Community-contributed tools
```

### Key Architecture Note

**tools/ vs python/ctsm/**: The `tools/` directory contains lightweight CLI wrappers (20-40 lines each) that import and call implementations from `python/ctsm/`. When investigating tool behavior, look in `python/ctsm/` for the actual code.

Example: `tools/site_and_regional/subset_data` imports from `python/ctsm/subset_data.py`

---

## Quick Reference: Where Does X Live?

| Looking for... | Location |
|----------------|----------|
| **Case creation** | `cime/scripts/create_newcase` |
| **Namelist defaults** | `bld/namelist_files/namelist_defaults_ctsm.xml` |
| **Namelist definitions** | `bld/namelist_files/namelist_definition_ctsm.xml` |
| **Machine config** | `ccs_config/machines/config_machines.xml` |
| **Compset definitions** | `cime_config/config_compsets.xml` |
| **Test definitions** | `cime_config/testdefs/testlist_clm.xml` |
| **Surface data tool** | Wrapper: `tools/mksurfdata_esmf/`, Code: Fortran in same dir |
| **Subset data tool** | Wrapper: `tools/site_and_regional/subset_data`, Code: `python/ctsm/subset_data.py` |
| **Single-point setup** | Wrapper: `tools/site_and_regional/`, Code: `python/ctsm/site_and_regional/` |
| **Hydrology code** | `src/biogeophys/` (HydrologyDrainageMod.F90, HillslopeHydrologyMod.F90) |
| **Carbon cycling** | `src/biogeochem/` (CNDriverMod.F90, etc.) |
| **Main driver** | `src/main/clm_driver.F90` |
| **Type definitions** | `src/main/` (GridcellType.F90, ColumnType.F90, etc.) |
| **All type instances** | `src/main/clm_instMod.F90` |
| **Python tests** | `python/ctsm/test/` |
| **Fortran unit tests** | `src/*/test/` directories |

---

## Testing Overview

CTSM has 5 testing systems:

| System | Location | Purpose | When to Use |
|--------|----------|---------|-------------|
| **run_sys_tests** | `./run_sys_tests` | CTSM system test orchestration | Comprehensive testing |
| **CIME create_test** | `cime/scripts/create_test` | Full workflow tests (200+) | Integration testing |
| **Fortran unit tests** | `src/*/test/` | pFUnit framework | After Fortran changes |
| **Python unit tests** | `python/ctsm/test/` | pytest (~12,300 LOC) | After Python changes |
| **FATES tests** | `src/fates/testing/` | Vegetation model tests | FATES development |

### Quick Test Commands

```bash
# Python tests (from python/ directory)
make test              # All tests
make utest             # Unit tests only
make stest             # System tests only

# System tests (from CTSM root)
./run_sys_tests --help

# CIME tests (from cime/scripts/)
./create_test --help
```

See `TESTING.md` for detailed testing documentation.

---

## Key Paths

| Purpose | Path |
|---------|------|
| CIME scripts | `cime/scripts/` |
| Create case | `cime/scripts/create_newcase` |
| Namelist defaults | `bld/namelist_files/namelist_defaults_ctsm.xml` |
| Namelist definitions | `bld/namelist_files/namelist_definition_ctsm.xml` |
| Subset data tool | `tools/site_and_regional/subset_data` |
| Single point tool | `tools/site_and_regional/run_neon` |
| Machine config | `ccs_config/machines/config_machines.xml` |

---

## Common Workflows

### Create a Case
```bash
cd cime/scripts
./create_newcase --case /path/to/case --compset I2000Clm51Bgc --res f09_g17 --machine hipergator
```

### Subset Data for Single-Point Run
```bash
cd tools/site_and_regional
./subset_data point --lat 29.7 --lon -82.0 --site OSBS --create-surface
```

### Build mksurfdata_esmf
```bash
cd tools/mksurfdata_esmf
./gen_mksurfdata_build
```

---

## Submodule Management

This repo uses `git-fleximod` for submodule management:
```bash
./bin/git-fleximod update    # Initialize/update all submodules
./bin/git-fleximod status    # Check submodule status
```

Key submodules:
- `cime/` - CIME framework
- `ccs_config/` - Machine configurations (our fork: `cdevaneprugh/ccs_config_cesm`)
- `src/fates/` - FATES vegetation model

---

## Documentation Index

### In This Repository
| Document | Location | Content |
|----------|----------|---------|
| This file | `CLAUDE.md` | Navigation and quick reference |
| Progress tracking | `claude-todo.md` | Phase 4.4 documentation progress |
| Testing guide | `TESTING.md` | Detailed testing documentation |
| Official docs | `doc/source/` | Sphinx RST (mirrors online docs) |
| Change log | `doc/ChangeLog` | Version history |
| Python README | `python/README.md` | Python package documentation |

### Subdirectory Documentation
| Directory | CLAUDE.md | Purpose |
|-----------|-----------|---------|
| `tools/` | `tools/CLAUDE.md` | Tool inventory, decision tree |
| `tools/mksurfdata_esmf/` | `tools/mksurfdata_esmf/CLAUDE.md` | Build process, HiPerGator fixes |
| `tools/site_and_regional/` | `tools/site_and_regional/CLAUDE.md` | Subset data, mesh tools |
| `python/` | `python/CLAUDE.md` | Package structure, module map |
| `python/ctsm/site_and_regional/` | `python/ctsm/site_and_regional/CLAUDE.md` | Implementation details |
| `src/` | `src/CLAUDE.md` | Fortran organization, types |
| `src/main/` | `src/main/CLAUDE.md` | Driver, types, control |
| `src/biogeophys/` | `src/biogeophys/CLAUDE.md` | Hydrology, energy, hillslope |
| `src/biogeochem/` | `src/biogeochem/CLAUDE.md` | Carbon-nitrogen cycling |
| `libraries/` | `libraries/CLAUDE.md` | PIO, mpi-serial notes |

### External Documentation
| Resource | Location | Content |
|----------|----------|---------|
| hpg-esm-tools | `/blue/gerber/cdevaneprugh/hpg-esm-tools/` | Analysis scripts, utilities |
| CTSM Development Guide | `hpg-esm-tools/docs/CTSM_DEVELOPMENT_GUIDE.md` | Development quick reference |
| Research Notes | `hpg-esm-tools/docs/CTSM_RESEARCH_NOTES.md` | Detailed findings |
| Fork Audit | `hpg-esm-tools/docs/FORK_AUDIT_2025-01-11.md` | Local modifications |
| Official CTSM Wiki | https://github.com/ESCOMP/ctsm/wiki | Upstream documentation |
| CTSM Tech Note | https://escomp.github.io/ctsm-docs/ | Technical reference |

---

## Source Code Overview

### Subgrid Hierarchy
```
Gridcell (grc) → Land Unit (lun) → Column (col) → Patch (patch/PFT)
```

- **Gridcell**: Geographic cell
- **Land Unit**: vegetated, urban, lake, glacier, crop
- **Column**: Soil/snow column within land unit
- **Patch**: Plant Functional Type within column

### Key Type Files (src/main/)
| File | Instance | Purpose |
|------|----------|---------|
| `GridcellType.F90` | `grc` | Geographic gridcell |
| `LandunitType.F90` | `lun` | Land unit (veg, urban, etc.) |
| `ColumnType.F90` | `col` | Soil/snow column |
| `PatchType.F90` | `patch` | Plant functional type |
| `clm_instMod.F90` | - | Registry of all ~50+ type instances |

### Main Entry Points
| File | Purpose |
|------|---------|
| `src/main/clm_driver.F90` | Main physics calling sequence |
| `src/main/clm_initializeMod.F90` | Two-phase initialization |
| `src/main/controlMod.F90` | Runtime control and namelist |

---

## Related Resources

- **hpg-esm-tools**: `/blue/gerber/cdevaneprugh/hpg-esm-tools/` - User scripts and analysis tools
- **~/.cime/**: Machine configuration files (config_machines.xml, etc.)
- **Upstream**: https://github.com/ESCOMP/CTSM
- **ccs_config fork**: https://github.com/cdevaneprugh/ccs_config_cesm (branch: uf-hipergator)
