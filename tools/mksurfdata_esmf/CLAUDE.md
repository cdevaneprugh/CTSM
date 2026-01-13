# mksurfdata_esmf - Surface Dataset Generation Tool

Fortran tool for generating global surface datasets (fsurdat files) for CTSM.

**Note:** For single-point and regional datasets, consider using `subset_data` instead - it's faster and doesn't require this build process.

---

## Quick Reference

```bash
# Build (one-time)
./gen_mksurfdata_build

# Generate namelist
./gen_mksurfdata_namelist --res 1.9x2.5 --start-year 1850 --end-year 1850

# Generate job script and submit
./gen_mksurfdata_jobscript_single --number-of-nodes 2 --tasks-per-node 128 \
    --namelist-file target.namelist
sbatch mksurfdata_jobscript_single.sh
```

---

## HiPerGator Build Process

### Prerequisites
```bash
# Load required modules
module load intel/2021.4.0 openmpi/4.1.5 netcdf-fortran/4.5.3 esmf/8.6.0

# Ensure submodules are present
cd $CTSMROOT
./bin/git-fleximod update
```

### Build
```bash
cd tools/mksurfdata_esmf
./gen_mksurfdata_build
```

The build creates `mksurfdata` executable in `./tool_bld/`.

---

## Our Fork Modifications

This tool required several fixes to work on HiPerGator with GCC 14:

### 1. Format Specifier Fixes (`src/mksurfdata.F90`)

GCC 10+ is strict about Fortran format specifiers. We fixed format width issues:

```fortran
! Before (upstream)
write(6,'(a,a6,f5.2)') ...

! After (our fork)
write(6,'(a,a6,f6.2)') ...
```

### 2. PIO Library Linking (`src/CMakeLists.txt`)

Changed from STATIC to SHARED imported libraries for PIO:

```cmake
# Before (upstream)
add_library(piof STATIC IMPORTED)
add_library(pioc STATIC IMPORTED)

# After (our fork)
add_library(piof SHARED IMPORTED)
add_library(pioc SHARED IMPORTED)
```

### 3. GCC 14 Compiler Flags (`gen_mksurfdata_build`)

Added flags for GCC 14 compatibility:

```bash
-fallow-argument-mismatch
-fallow-invalid-boz
```

---

## Directory Structure

```
mksurfdata_esmf/
├── gen_mksurfdata_build           # Build script (modified)
├── gen_mksurfdata_namelist        # Namelist generator (Python)
├── gen_mksurfdata_jobscript_single  # Single job script generator
├── gen_mksurfdata_jobscript_multi   # Multi job script generator
├── download_input_data            # Input data downloader
├── src/                           # Fortran source
│   ├── mksurfdata.F90             # Main program (modified)
│   ├── CMakeLists.txt             # CMake config (modified)
│   └── *.F90                      # Supporting modules
├── tool_bld/                      # Build output (after build)
│   └── mksurfdata                 # Executable
├── cmake/                         # CMake modules
├── namelist_defaults/             # Default namelist values
└── README.md                      # Upstream documentation
```

---

## When to Use This vs subset_data

| Use Case | Recommended Tool |
|----------|------------------|
| Global surface dataset | **mksurfdata_esmf** |
| Regional dataset (new mesh) | **mksurfdata_esmf** (but subset_data preferred) |
| Single-point dataset | **subset_data** |
| Subset of existing global data | **subset_data** |
| Modifying existing surface data | **fsurdat_modifier** |

**General rule:** If you can use `subset_data` to extract from existing global data, prefer that. Only use `mksurfdata_esmf` when you need to create from raw input data.

---

## Troubleshooting

### Build Fails: Format Errors

If you see errors like:
```
Error: Expected comma in format specification at (1)
```

Check that our format specifier fixes are present in `src/mksurfdata.F90`.

### Build Fails: PIO Linking

If you see:
```
error while loading shared libraries: libpiof.so
```

Check that `src/CMakeLists.txt` uses SHARED (not STATIC) for PIO libraries.

### Build Fails: GCC Incompatibility

If you see argument mismatch errors:
```
Error: Type mismatch between actual argument at (1) and actual argument at (2)
```

Check that `gen_mksurfdata_build` includes `-fallow-argument-mismatch` flag.

### Runtime Fails: Missing Input Data

Run `./download_input_data` to fetch missing files, or check that input data paths point to `/blue/gerber/earth_models/inputdata`.

### Runtime Fails: MPI Issues

Ensure OpenMPI module is loaded and matches the build environment:
```bash
module load openmpi/4.1.5
```

---

## Input Data Requirements

- All raw input `*.nc` files must be NetCDF3 or CDF5 (not NetCDF4)
- LAI dataset must have unlimited time dimension

Convert files if needed:
```bash
# Convert to CDF5
nccopy -k cdf5 oldfile.nc newfile.nc

# Make time dimension unlimited
ncks --mk_rec_dmn time file.nc -o file_unlimited.nc
```

---

## See Also

- `README.md` - Detailed upstream documentation
- `../CLAUDE.md` - Tools directory overview
- `../site_and_regional/CLAUDE.md` - Alternative subset_data approach
- Upstream issue: https://github.com/ESCOMP/CTSM/issues/2341 (multi-machine support)
