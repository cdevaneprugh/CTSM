# CTSM Testing Guide

Comprehensive guide to CTSM's multi-layered testing infrastructure.

---

## Testing Systems Overview

| System | Location | Purpose | When to Use |
|--------|----------|---------|-------------|
| **run_sys_tests** | `./run_sys_tests` | CTSM system test orchestration | Comprehensive validation |
| **CIME create_test** | `cime/scripts/create_test` | Full workflow tests (200+) | Integration testing |
| **Fortran unit tests** | `src/*/test/` | pFUnit module tests | After Fortran changes |
| **Python unit tests** | `python/ctsm/test/` | pytest (~12,300 LOC) | After Python changes |
| **FATES tests** | `src/fates/testing/` | Vegetation model tests | FATES development |

---

## 1. System Tests (run_sys_tests)

High-level test orchestration that wraps CIME testing.

### Location
```
./run_sys_tests
```

### Basic Usage
```bash
# Show help
./run_sys_tests --help

# Run a test suite
./run_sys_tests -s aux_clm

# Generate baseline
./run_sys_tests -s aux_clm --baseline-name my_baseline -g

# Compare to baseline
./run_sys_tests -s aux_clm --baseline-name my_baseline -c
```

### Test Suites
| Suite | Description |
|-------|-------------|
| `aux_clm` | Auxiliary CLM tests |
| `clm_short` | Short integration tests |
| `clm_long` | Long integration tests |
| `prebeta` | Pre-release tests |

### HiPerGator Notes
- Requires proper SLURM allocation
- Use appropriate queue/account
- Consider memory requirements for large tests

---

## 2. CIME create_test

CIME's test framework for full case workflow testing.

### Location
```
cime/scripts/create_test
```

### Basic Usage
```bash
cd cime/scripts

# Single test
./create_test SMS_D.f09_g17.I2000Clm51Bgc.hipergator

# Multiple tests from testlist
./create_test --xml-category clm_developer

# With baseline comparison
./create_test SMS_D.f09_g17.I2000Clm51Bgc.hipergator \
    --baseline-name baseline_v1 --compare
```

### Test Name Format
```
TESTTYPE[_MODIFIERS].GRID.COMPSET.MACHINE[_COMPILER]

Examples:
SMS_D.f09_g17.I2000Clm51Bgc.hipergator
ERS_D_Mmpi-serial.1x1_brazil.I2000Clm51Bgc.hipergator
```

### Test Types
| Type | Description |
|------|-------------|
| `SMS` | Smoke test (basic run) |
| `ERS` | Exact restart test |
| `ERI` | Exact restart with interpolation |
| `ERB` | Exact restart with rebuild |
| `PET` | PE layout test |
| `PEM` | PE layout memory test |

### Test Modifiers
| Modifier | Description |
|----------|-------------|
| `_D` | Debug mode |
| `_P*` | PE count (e.g., `_P16`) |
| `_M*` | MPILIB (e.g., `_Mmpi-serial`) |
| `_N*` | NTHRDS (e.g., `_N2`) |

### Test Definitions
Test definitions are in `cime_config/testdefs/testlist_clm.xml`:

```xml
<test name="SMS_D" grid="f09_g17" compset="I2000Clm51Bgc">
  <machines>
    <machine name="hipergator" compiler="gnu"/>
  </machines>
</test>
```

### HiPerGator Notes
```bash
# Set project/account for job submission
export PROJECT=gerber

# May need to specify queue
./create_test ... --queue gerber-b
```

---

## 3. Fortran Unit Tests (pFUnit)

Unit tests for Fortran modules using the pFUnit framework.

### Location
Test directories within source:
```
src/main/test/
src/biogeophys/test/
src/biogeochem/test/
src/utils/test/
```

### Running Tests
```bash
# From CTSM root
cd /blue/gerber/cdevaneprugh/ctsm5.3

# Run all Fortran unit tests
../cime/scripts/fortran_unit_testing/run_tests.py \
    --build-dir unit_tests.temp

# Run specific test
../cime/scripts/fortran_unit_testing/run_tests.py \
    --build-dir unit_tests.temp \
    --test-spec-dir src/main/test \
    --test-name clm_varcon

# With verbose output
../cime/scripts/fortran_unit_testing/run_tests.py \
    --build-dir unit_tests.temp --verbose
```

### pFUnit Test File Format
```fortran
! test_my_module.pf
module test_my_module
  use funit
  use my_module
contains

  @Test
  subroutine test_my_function()
    real(r8) :: result
    result = my_function(1.0_r8)
    @assertEqual(2.0_r8, result, tolerance=1.e-10_r8)
  end subroutine

end module
```

### HiPerGator Notes
```bash
# Load required modules
module load intel/2021.4.0 openmpi/4.1.5 netcdf-fortran/4.5.3

# Build may require specific compiler settings
export FC=ifort
export CC=icc
```

---

## 4. Python Unit Tests

Comprehensive Python tests using pytest.

### Location
```
python/ctsm/test/
├── test_unit_*.py     # Unit tests (41 files)
├── test_sys_*.py      # System tests (12 files)
└── testinputs/        # Test fixtures
```

### Running Tests
```bash
cd python

# All tests
make test

# Unit tests only
make utest

# System tests only
make stest

# With coverage
python -m pytest --cov=ctsm ctsm/test/

# Specific test file
python -m pytest ctsm/test/test_unit_subset_data.py -v

# Specific test function
python -m pytest ctsm/test/test_unit_subset_data.py::test_point_extraction -v
```

### Make Targets
```bash
make test      # All tests
make utest     # Unit tests only
make stest     # System tests only
make lint      # pylint
make black     # Code formatting check
```

### Test Coverage Summary
| Test File | LOC | Coverage Area |
|-----------|-----|---------------|
| `test_unit_subset_data.py` | 27,395 | Data subsetting |
| `test_unit_fsurdat_modifier.py` | 15,140 | Surface modification |
| `test_unit_singlept_data.py` | 13,030 | Single-point setup |
| `test_unit_mesh_type.py` | ~500 | Mesh operations |

### HiPerGator Notes
```bash
# Activate conda environment
module load conda
conda activate ctsm  # or ctsm_pylib

# Run tests
cd python
make test
```

---

## 5. FATES Tests

Testing for the FATES vegetation model submodule.

### Location
```
src/fates/testing/
```

### Running FATES Tests
```bash
cd src/fates/testing

# See README for specific instructions
cat README.testing.md
```

### Test Types
- Functional tests
- Unit tests
- Integration tests

---

## Test Categories by Purpose

### Quick Validation (5-10 min)
```bash
# Python unit tests
cd python && make utest

# Smoke test a case
cd cime/scripts
./create_test SMS.f09_g17.I2000Clm51Bgc.hipergator
```

### Pre-Commit Testing (30-60 min)
```bash
# Full Python tests
cd python && make test

# Fortran unit tests
../cime/scripts/fortran_unit_testing/run_tests.py --build-dir build

# Quick system test
./run_sys_tests -s clm_short
```

### Comprehensive Testing (hours)
```bash
# Full system test suite
./run_sys_tests -s aux_clm -g -b baseline_name

# All CIME tests
cd cime/scripts
./create_test --xml-category clm_developer
```

---

## Baseline Management

### Creating Baselines
```bash
# System tests
./run_sys_tests -s aux_clm --baseline-name v1.0 -g

# CIME tests
./create_test SMS.f09_g17.I2000Clm51Bgc.hipergator \
    --baseline-name v1.0 --generate
```

### Comparing to Baselines
```bash
# System tests
./run_sys_tests -s aux_clm --baseline-name v1.0 -c

# CIME tests
./create_test SMS.f09_g17.I2000Clm51Bgc.hipergator \
    --baseline-name v1.0 --compare
```

### Baseline Location
Baselines are typically stored in:
```
$CESMDATAROOT/inputdata/ccsm4_baselines/
```

---

## Debugging Failed Tests

### 1. Check Test Output
```bash
# CIME tests create case directories
ls /path/to/scratch/SMS.f09_g17.I2000Clm51Bgc.hipergator.*/

# Check CaseStatus
cat CaseStatus

# Check run logs
cat /path/to/run/*.log.*
```

### 2. Run in Debug Mode
```bash
# CIME test with debug
./create_test SMS_D.f09_g17.I2000Clm51Bgc.hipergator

# Python with verbose
python -m pytest ctsm/test/test_unit_xyz.py -v --tb=long
```

### 3. Reproduce Locally
```bash
# Copy test case
./create_test SMS.f09_g17.I2000Clm51Bgc.hipergator --test-root /my/test/dir

# Modify and rerun
cd /my/test/dir/SMS...
./case.submit
```

---

## Writing New Tests

### Python Unit Test
```python
# python/ctsm/test/test_unit_my_module.py
import unittest
from ctsm.my_module import MyClass

class TestMyModule(unittest.TestCase):
    def setUp(self):
        self.obj = MyClass()

    def test_basic_function(self):
        result = self.obj.do_something(1, 2)
        self.assertEqual(result, 3)

    def test_edge_case(self):
        with self.assertRaises(ValueError):
            self.obj.do_something(-1, 0)

if __name__ == '__main__':
    unittest.main()
```

### Fortran Unit Test
```fortran
! src/main/test/test_my_module.pf
module test_my_module
  use funit
  use my_module
contains

  @Test
  subroutine test_calculation()
    real(r8) :: expected, actual
    expected = 2.0_r8
    actual = my_calculation(1.0_r8)
    @assertEqual(expected, actual, tolerance=1.e-10_r8)
  end subroutine

end module
```

### System Test
Add to `cime_config/testdefs/testlist_clm.xml`:
```xml
<test name="SMS" grid="f09_g17" compset="I2000Clm51Bgc">
  <machines>
    <machine name="hipergator" compiler="gnu"/>
  </machines>
  <options>
    <option name="wallclock">00:30:00</option>
  </options>
</test>
```

---

## See Also

- `CLAUDE.md` - Repository overview
- `python/README.md` - Python package testing
- `cime/scripts/fortran_unit_testing/README` - pFUnit documentation
- `cime_config/testdefs/testlist_clm.xml` - Test definitions
- `src/fates/testing/README.testing.md` - FATES tests
