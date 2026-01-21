# site_and_regional - Single-Point and Regional Tools

Tools for creating and running single-point and regional CTSM simulations.

**Important:** Most scripts in this directory are **thin wrappers** (20-40 lines) that import implementations from `python/ctsm/`. For actual code logic, see `python/ctsm/site_and_regional/` and `python/ctsm/subset_data.py`.

---

## Tool Overview

| Wrapper Script | Implementation | Purpose |
|----------------|----------------|---------|
| `subset_data` | `python/ctsm/subset_data.py` | Extract single-point/regional data |
| `run_neon` | `python/ctsm/site_and_regional/run_neon.py` | NEON tower site automation |
| `run_tower` | `python/ctsm/site_and_regional/run_tower.py` | General tower site automation |
| `mesh_maker` | `python/ctsm/mesh_maker.py` | Create ESMF mesh files |
| `mesh_plotter` | `python/ctsm/mesh_plotter.py` | Visualize mesh files |
| `modify_singlept_site_neon` | `python/ctsm/site_and_regional/modify_singlept_site_neon.py` | NEON-specific modifications |
| `neon_surf_wrapper` | `python/ctsm/site_and_regional/neon_surf_wrapper.py` | NEON surface data |
| `plumber2_surf_wrapper` | `python/ctsm/site_and_regional/plumber2_surf_wrapper.py` | PLUMBER2 surface data |
| `mknoocnmap.pl` | (Perl script, self-contained) | Create no-ocean mapping files |

---

## subset_data - Primary Tool

The most commonly used tool for single-point simulations.

### Single-Point Example
```bash
./subset_data point --lat 29.7 --lon -82.0 --site OSBS --create-surface \
    --outdir /path/to/output
```

### Regional Example
```bash
./subset_data region --lat1 25.0 --lat2 35.0 --lon1 -90.0 --lon2 -80.0 \
    --reg southeast --create-surface
```

### Key Options
```
--lat, --lon          Coordinates for single-point
--lat1, --lat2, etc.  Bounding box for regional
--site                Site name (used in output filenames)
--create-surface      Generate surface dataset
--create-datm         Generate atmospheric forcing
--create-user-mods    Generate user_mods directory
--outdir              Output directory
--crop                Include crop data
--dompft              Dominant PFT to use
```

### What subset_data Creates

```
output_directory/
├── surfdata_*.nc           # Surface dataset
├── domain_*.nc             # Domain file
├── user_mods/              # User mods directory
│   ├── user_nl_clm         # CLM namelist additions
│   ├── user_nl_datm_streams  # DATM stream modifications
│   └── shell_commands      # Post-setup commands
└── datm_*/                 # Atmospheric forcing (if requested)
```

---

## HiPerGator-Specific Notes

### Our Fork Modification: MPILIB=openmpi

The upstream code uses `MPILIB=mpi-serial` by default. Our fork changes this to `MPILIB=openmpi` in `python/ctsm/site_and_regional/single_point_case.py` because mpi-serial has linking issues on HiPerGator.

### Input Data Paths

Our fork changes default input paths in `python/ctsm/site_and_regional/default_data_*.cfg`:
```
/blue/gerber/earth_models/inputdata
```

### Running subset_data

```bash
# Ensure conda environment
module load conda
conda activate ctsm  # or ctsm_pylib

# Run from this directory
cd tools/site_and_regional
./subset_data point --lat 29.7 --lon -82.0 --site mysite --create-surface
```

---

## Directory Structure

```
site_and_regional/
├── subset_data               # Main subsetting wrapper
├── run_neon                  # NEON automation wrapper
├── run_tower                 # Tower site wrapper
├── mesh_maker                # Mesh creation wrapper
├── mesh_plotter              # Mesh visualization wrapper
├── modify_singlept_site_neon # NEON modification wrapper
├── neon_surf_wrapper         # NEON surface data wrapper
├── plumber2_surf_wrapper     # PLUMBER2 surface data wrapper
├── mknoocnmap.pl             # No-ocean mapping (Perl)
├── default_data/             # Configuration files
│   ├── default_data_1x1.cfg  # Default paths (modified in our fork)
│   └── ...
└── README                    # Original documentation
```

---

## Workflow: Setting Up a Single-Point Simulation

### 1. Generate Input Data
```bash
cd tools/site_and_regional
./subset_data point --lat 29.7 --lon -82.0 --site OSBS --create-surface \
    --create-user-mods --outdir ~/osbs_data
```

### 2. Create Case
```bash
cd $CIME_SCRIPTS
./create_newcase --case $CASES/osbs_test \
    --compset I2000Clm51Bgc --res CLM_USRDAT --machine hipergator \
    --user-mods-dir ~/osbs_data/user_mods
```

### 3. Setup and Build
```bash
cd $CASES/osbs_test
./case.setup
./case.build
```

### 4. Run
```bash
./case.submit
```

---

## Key Implementation Classes

In `python/ctsm/site_and_regional/`:

| Class | File | Purpose |
|-------|------|---------|
| `SinglePointCase` | `single_point_case.py` | Single-point domain/surface/DATM creation |
| `RegionalCase` | `regional_case.py` | Regional case setup |
| `BaseCase` | `base_case.py` | Shared functionality |
| `TowerSite` | `tower_site.py` | Tower site metadata |
| `DatmFiles` | `base_case.py` | DATM forcing file handling |

---

## Troubleshooting

### "No module named ctsm"
The wrapper adds `python/` to the path. Ensure:
1. You're running from CTSM root or tools/site_and_regional/
2. `python/ctsm/__init__.py` exists

### MPI errors during case.build
Our fork uses `MPILIB=openmpi`. Ensure OpenMPI module is loaded:
```bash
module load openmpi/4.1.5
```

### Input data not found
Check paths in `default_data/default_data_*.cfg` point to `/blue/gerber/earth_models/inputdata`.

### "Invalid longitude" errors
subset_data expects longitude in -180 to 180 range. Convert if needed:
```python
# 270° -> -90°
lon = lon - 360 if lon > 180 else lon
```

---

## See Also

- `python/ctsm/subset_data.py` - Implementation code
- `python/ctsm/site_and_regional/CLAUDE.md` - Implementation details
- `../CLAUDE.md` - Tools directory overview
- `../mksurfdata_esmf/CLAUDE.md` - Alternative for global datasets
