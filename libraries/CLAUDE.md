# CTSM Libraries

External libraries for MPI and parallel I/O.

---

## Contents

| Directory | Purpose |
|-----------|---------|
| `mpi-serial/` | Serial MPI stub library |
| `parallelio/` | PIO (Parallel I/O) library |

---

## 1. mpi-serial

### What It Is
A one-processor MPI stub library that allows MPI-based code to run without real MPI. Supports most common MPI calls needed by CTSM.

### Upstream Intent
The upstream CTSM uses mpi-serial for single-point simulations to avoid the overhead of real MPI for trivially parallel cases.

### HiPerGator Status: NOT USED

**Why?** mpi-serial had linking issues on HiPerGator. Rather than debug the library build, we switched to using openmpi for all runs, including single-point.

**Our modification:** In `python/ctsm/site_and_regional/single_point_case.py`:
```python
# Upstream uses: MPILIB=mpi-serial
# Our fork uses: MPILIB=openmpi
```

### Could We Use It?

**Potential benefits:**
- Faster case builds (no MPI library linking)
- Simpler job submission (no MPI overhead)
- Potentially faster single-point runs

**Required investigation:**
1. Identify specific linking error (library path? symbol resolution?)
2. Build mpi-serial with HiPerGator's GCC 14
3. Test with CTSM case.build
4. Verify runtime behavior

**Current assessment:** Low priority. OpenMPI works fine for single-point runs. The overhead is minimal and we get consistent behavior across all case types.

### If You Want to Try

```bash
cd libraries/mpi-serial

# Configure for HiPerGator
module load gcc/14.2.0
./configure FC=gfortran CC=gcc

# Build
make

# Test
make tests
```

Then modify `ccs_config/machines/hipergator/config_machines.xml`:
```xml
<MPILIBS>openmpi,mpi-serial</MPILIBS>
```

And set `MPILIB=mpi-serial` in case configuration.

---

## 2. parallelio (PIO)

### What It Is
Parallel I/O library for reading/writing NetCDF files in parallel. Critical for CTSM performance with large grids.

### HiPerGator Status: SHARED BUILD EXISTS

A pre-built PIO exists at:
```
/blue/gerber/earth_models/shared/parallelio/bld/
```

This is configured in `ccs_config/machines/hipergator/config_machines.xml`:
```xml
<env name="PIO">/blue/gerber/earth_models/shared/parallelio/bld</env>
<env name="PIO_LIBDIR">/blue/gerber/earth_models/shared/parallelio/bld/lib</env>
<env name="PIO_INCDIR">/blue/gerber/earth_models/shared/parallelio/bld/include</env>
```

### Does case.build Use the Shared PIO?

**For model executable:** CIME should use the externally-configured PIO when the `PIO` environment variable is set. The machine configuration above sets this.

**For mksurfdata_esmf:** Our fork modified `CMakeLists.txt` to use SHARED (not STATIC) libraries:
```cmake
add_library(piof SHARED IMPORTED)
add_library(pioc SHARED IMPORTED)
```

This was necessary because the shared PIO build only provides `.so` files, not `.a` static libraries.

### Verification

To check if a case is using the shared PIO:

```bash
cd $CASES/mycase
./xmlquery PIO

# Should show:
# PIO: /blue/gerber/earth_models/shared/parallelio/bld
```

During `case.build`, watch for PIO rebuild messages. If you see PIO compiling, something is misconfigured.

### Rebuilding the Shared PIO

If you need to rebuild (e.g., after module updates):

```bash
cd /blue/gerber/earth_models/shared/parallelio
module load gcc/14.2.0 openmpi/5.0.7 netcdf-c/4.9.3 netcdf-f/4.6.2 hdf5/1.14.6

# Configure
mkdir bld && cd bld
cmake .. \
    -DPIO_ENABLE_FORTRAN=ON \
    -DPIO_ENABLE_TIMING=OFF \
    -DWITH_PNETCDF=OFF \
    -DCMAKE_INSTALL_PREFIX=$(pwd)

# Build and install
make -j8
make install
```

---

## Research Summary

### mpi-serial
- **Status:** Not used on HiPerGator
- **Workaround:** Use openmpi for all cases
- **Recommendation:** Low priority to fix. OpenMPI works fine.
- **Effort to fix:** Medium (debug linking, test thoroughly)

### PIO
- **Status:** Shared build configured and working
- **Location:** `/blue/gerber/earth_models/shared/parallelio/bld`
- **mksurfdata fix:** SHARED library linking (our fork)
- **Recommendation:** Current setup is good. Rebuild if modules change.

---

## Build Notes

### mpi-serial Build Dependencies
- C compiler (gcc)
- Fortran compiler (gfortran)
- autoconf/automake (for configure)

### PIO Build Dependencies
- MPI (openmpi)
- NetCDF-C
- NetCDF-Fortran
- HDF5
- CMake

### Module Compatibility
Both libraries should be built with the same compiler/MPI versions used for CTSM builds. Current HiPerGator setup:
- GCC 14.2.0
- OpenMPI 5.0.7
- NetCDF-C 4.9.3
- NetCDF-Fortran 4.6.2
- HDF5 1.14.6

---

## See Also

- `../ccs_config/machines/hipergator/config_machines.xml` - Machine configuration
- `../tools/mksurfdata_esmf/CLAUDE.md` - mksurfdata PIO usage
- `mpi-serial/README` - mpi-serial documentation
