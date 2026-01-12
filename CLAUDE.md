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

## Repository Structure

```
CTSM/
├── bld/                    # Build-time configuration
│   └── namelist_files/     # Namelist definitions and defaults
├── cime/                   # CIME framework (submodule)
│   └── scripts/            # Case management scripts
├── cime_config/            # CTSM-specific CIME configuration
├── python/                 # Python tools
│   └── ctsm/
│       ├── subset_data.py  # Data subsetting tool
│       └── site_and_regional/  # Single-point case tools
├── src/                    # CTSM Fortran source code
│   ├── biogeochem/         # Biogeochemistry
│   ├── biogeophys/         # Biophysics
│   └── main/               # Main CLM code
└── tools/                  # Build and data tools
    ├── mksurfdata_esmf/    # Surface data generation tool
    └── site_and_regional/  # Regional/single-point configs
```

## Key Paths

| Purpose | Path |
|---------|------|
| CIME scripts | `cime/scripts/` |
| Create case | `cime/scripts/create_newcase` |
| Namelist defaults | `bld/namelist_files/namelist_defaults_ctsm.xml` |
| Namelist definitions | `bld/namelist_files/namelist_definition_ctsm.xml` |
| Subset data tool | `tools/site_and_regional/subset_data` |
| Single point tool | `tools/site_and_regional/run_neon` |

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

## Submodule Management

This repo uses `git-fleximod` for submodule management:
```bash
./bin/git-fleximod update    # Initialize/update all submodules
./bin/git-fleximod status    # Check submodule status
```

## Related Resources

- **hpg-esm-tools**: User scripts and analysis tools
- **~/.cime/**: Machine configuration files (config_machines.xml, etc.)
- **Upstream**: https://github.com/ESCOMP/CTSM
