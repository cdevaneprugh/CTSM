# CTSM Tools Directory

This directory contains CLI wrappers and tools for creating/modifying CTSM input data.

## Key Architecture

**Most Python tools in this directory are thin wrappers** (20-40 lines) that import and call implementations from `python/ctsm/`. When investigating tool behavior or debugging, look in `python/ctsm/` for the actual code.

Exception: `mksurfdata_esmf/` contains actual Fortran implementation (not a wrapper).

---

## Tool Inventory

| Tool | Type | Wrapper Location | Implementation |
|------|------|------------------|----------------|
| **mksurfdata_esmf** | Fortran | `mksurfdata_esmf/` | Same directory (Fortran source) |
| **subset_data** | Python wrapper | `site_and_regional/subset_data` | `python/ctsm/subset_data.py` |
| **run_neon** | Python wrapper | `site_and_regional/run_neon` | `python/ctsm/site_and_regional/run_neon.py` |
| **run_tower** | Python wrapper | `site_and_regional/run_tower` | `python/ctsm/site_and_regional/run_tower.py` |
| **mesh_maker** | Python wrapper | `site_and_regional/mesh_maker` | `python/ctsm/mesh_maker.py` |
| **mesh_plotter** | Python wrapper | `site_and_regional/mesh_plotter` | `python/ctsm/mesh_plotter.py` |
| **fsurdat_modifier** | Python wrapper | `modify_input_files/fsurdat_modifier` | `python/ctsm/modify_input_files/fsurdat_modifier.py` |
| **mesh_mask_modifier** | Python wrapper | `modify_input_files/mesh_mask_modifier` | `python/ctsm/modify_input_files/mesh_mask_modifier.py` |
| **generate_gdds** | Python wrapper | `crop_calendars/generate_gdds` | `python/ctsm/crop_calendars/generate_gdds.py` |

---

## Decision Tree: Which Tool for Which Task?

### Creating Surface Data

**Need a global surface dataset?**
→ Use `mksurfdata_esmf/` (Fortran tool, requires HPC resources)

**Need a single-point or regional surface dataset?**
→ Use `site_and_regional/subset_data` (extracts from existing global data)

**Need to modify an existing surface dataset?**
→ Use `modify_input_files/fsurdat_modifier` (changes PFTs, soil, etc.)

### Single-Point Simulations

**Setting up a NEON tower site?**
→ Use `site_and_regional/run_neon` (automated NEON workflow)

**Setting up a custom single-point?**
→ Use `site_and_regional/subset_data point --lat X --lon Y`

**Need custom atmospheric forcing?**
→ Use `site_and_regional/subset_data` with `--datm-*` options

### Mesh Operations

**Creating a new mesh file?**
→ Use `site_and_regional/mesh_maker`

**Modifying mesh mask (land/ocean)?**
→ Use `modify_input_files/mesh_mask_modifier`

**Visualizing mesh structure?**
→ Use `site_and_regional/mesh_plotter`

### Crop Calendars

**Processing GGCMI crop data?**
→ Use `crop_calendars/` tools

---

## Input Data Creation Workflow

The traditional 6-step workflow from `tools/README`:

```
1. Create SCRIP grid files (if needed)
   └─> mkmapgrids/ or site_and_regional/mknoocnmap.pl

2. Create ocean-to-atmosphere mapping
   └─> CIME gen_maps.sh (or skip if no ocean)

3. Add SCRIP files to XML database (optional)
   └─> Manual XML editing

4. Generate domain files
   └─> CIME gen_domain

5. Create surface datasets
   └─> mksurfdata_esmf/

6. Add files to XML or user_nl_clm
   └─> Manual configuration
```

**For single-point runs, skip most of this:** Use `subset_data` which handles steps 1-5 automatically by extracting from existing global data.

---

## HiPerGator-Specific Notes

### What Works

- **subset_data**: Works with `MPILIB=openmpi` setting (our fork modification)
- **mksurfdata_esmf**: Works after our GCC 14 fixes (format specifiers, compiler flags)
- **fsurdat_modifier**: Works without modification
- **mesh_maker/plotter**: Work without modification

### What Doesn't Work (or Has Issues)

- **mpi-serial**: Our fork uses openmpi instead; mpi-serial had linking issues
- **run_neon**: Requires NEON data access; may need network configuration
- **Default input paths**: Changed to `/blue/gerber/earth_models/inputdata` in our fork

### Our Fork Modifications

| File | Change | Reason |
|------|--------|--------|
| `mksurfdata_esmf/src/mksurfdata.F90` | Format specifiers | GCC 10+ compatibility |
| `mksurfdata_esmf/src/CMakeLists.txt` | STATIC → SHARED | PIO linking |
| `mksurfdata_esmf/gen_mksurfdata_build` | Compiler flags | GCC 14 compatibility |
| `python/ctsm/site_and_regional/single_point_case.py` | MPILIB=openmpi | mpi-serial linking issues |
| `python/ctsm/site_and_regional/default_data_*.cfg` | Input paths | HiPerGator paths |

---

## Directory Structure

```
tools/
├── mksurfdata_esmf/          # Surface data generation (Fortran)
│   ├── gen_mksurfdata_build  # Build script (modified for HiPerGator)
│   ├── gen_mksurfdata_namelist    # Namelist generator
│   ├── gen_mksurfdata_jobscript_* # Job script generators
│   ├── src/                  # Fortran source (modified)
│   └── README.md             # Detailed documentation
│
├── site_and_regional/        # Single-point and regional tools
│   ├── subset_data           # Main subsetting wrapper
│   ├── run_neon              # NEON site automation
│   ├── run_tower             # Tower site automation
│   ├── mesh_maker            # Mesh creation
│   ├── mesh_plotter          # Mesh visualization
│   ├── mknoocnmap.pl         # No-ocean mapping (Perl)
│   └── default_data/         # Default configuration files
│
├── modify_input_files/       # Input modification tools
│   ├── fsurdat_modifier      # Surface dataset modifier
│   ├── mesh_mask_modifier    # Mesh mask modifier
│   └── modify_*.cfg          # Example config files
│
├── crop_calendars/           # Crop calendar processing
│   ├── generate_gdds         # GDD baseline generation
│   ├── process_ggcmi_shdates # GGCMI date processing
│   └── regrid_ggcmi_shdates  # GGCMI regridding
│
├── mkmapgrids/               # SCRIP grid creation (legacy)
├── contrib/                  # Community contributions
└── README                    # Original documentation
```

---

## Common Usage Examples

### Subset Data for Single-Point
```bash
cd tools/site_and_regional
./subset_data point --lat 29.7 --lon -82.0 --site OSBS --create-surface \
    --outdir /path/to/output
```

### Subset Data for Region
```bash
./subset_data region --lat1 25.0 --lat2 35.0 --lon1 -90.0 --lon2 -80.0 \
    --reg southeast --create-surface
```

### Modify Surface Dataset
```bash
cd tools/modify_input_files
./fsurdat_modifier modify_fsurdat.cfg
```

Example config (`modify_fsurdat.cfg`):
```ini
[modify_fsurdat_basic_options]
fsurdat_in = /path/to/input.nc
fsurdat_out = /path/to/output.nc
idealized = False

[modify_fsurdat_subgrid_fractions]
pct_urban = 0.0
pct_lake = 0.0
```

### Build mksurfdata_esmf
```bash
cd tools/mksurfdata_esmf
./gen_mksurfdata_build
```

---

## Troubleshooting

### "Module not found" errors
The wrapper scripts add `python/` to the Python path. If you get import errors:
1. Ensure you're running from the CTSM root or tools/ directory
2. Check that `python/ctsm/` exists and has `__init__.py`

### subset_data fails with MPI errors
Our fork uses openmpi instead of mpi-serial. Ensure:
1. OpenMPI module is loaded
2. Using our fork's `single_point_case.py` (has MPILIB=openmpi)

### mksurfdata_esmf build fails
Common issues:
1. **Missing modules**: Load `intel`, `openmpi`, `netcdf-fortran`, `esmf`
2. **PIO errors**: Our fork uses SHARED libraries (check CMakeLists.txt)
3. **Format errors**: GCC 10+ requires our format specifier fixes

### Input data not found
Check paths in:
- `python/ctsm/site_and_regional/default_data_*.cfg`
- Should point to `/blue/gerber/earth_models/inputdata`

---

## See Also

- `mksurfdata_esmf/CLAUDE.md` - Detailed mksurfdata documentation
- `site_and_regional/CLAUDE.md` - Subset data and single-point tools
- `python/CLAUDE.md` - Python implementation details
- `../CLAUDE.md` - Main repository documentation
