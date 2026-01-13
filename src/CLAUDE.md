# CTSM Fortran Source Code

This directory contains the core Fortran implementation of CTSM (~197,000 lines across 261+ modules).

---

## Directory Overview

| Directory | LOC | Purpose |
|-----------|-----|---------|
| `main/` | 33,569 | Driver, types, initialization, control |
| `biogeophys/` | 69,069 | Hydrology, energy, radiation, snow, lakes |
| `biogeochem/` | 64,538 | Carbon-nitrogen cycling, phenology, fire |
| `soilbiogeochem/` | 13,607 | Soil organic matter, decomposition |
| `dyn_subgrid/` | 6,758 | Dynamic land units, harvests |
| `utils/` | 10,037 | Utilities, MPI, time manager |
| `cpl/` | ~5,000 | Coupler interfaces (LILAC, NUOPC) |
| `fates/` | (submodule) | FATES vegetation model |
| `init_interp/` | ~2,000 | Initialization interpolation |
| `unit_test_stubs/` | ~1,000 | Stubs for unit testing |

---

## Subgrid Hierarchy

CTSM uses a 4-level spatial hierarchy:

```
Gridcell (grc)
└── Land Unit (lun)        # vegetated, urban, lake, glacier, crop
    └── Column (col)       # soil/snow column
        └── Patch (patch)  # Plant Functional Type (PFT)
```

### Type Definitions (in `main/`)

| File | Instance | Level | Purpose |
|------|----------|-------|---------|
| `GridcellType.F90` | `grc` | 1 | Geographic gridcell |
| `LandunitType.F90` | `lun` | 2 | Land use type |
| `ColumnType.F90` | `col` | 3 | Soil/snow column |
| `PatchType.F90` | `patch` | 4 | Plant functional type |

Each type has:
- `.Init()` method for allocation
- `.Clean()` method for deallocation
- Pointer arrays for state variables

---

## Central Type Registry: clm_instMod.F90

`main/clm_instMod.F90` declares ~50+ global type instances. Key groups:

### Grid/Subgrid Types
```fortran
type(gridcell_type)    :: grc
type(landunit_type)    :: lun
type(column_type)      :: col
type(patch_type)       :: patch
```

### Physics Types (biogeophys)
```fortran
type(aerosol_type)           :: aerosol_inst
type(canopystate_type)       :: canopystate_inst
type(energyflux_type)        :: energyflux_inst
type(temperature_type)       :: temperature_inst
type(waterstatebulk_type)    :: water_inst
type(soilstate_type)         :: soilstate_inst
type(soilhydrology_type)     :: soilhydrology_inst
type(lakestate_type)         :: lakestate_inst
type(solarabs_type)          :: solarabs_inst
type(surfalb_type)           :: surfalb_inst
type(surfrad_type)           :: surfrad_inst
```

### Biogeochemistry Types
```fortran
type(bgc_vegetation_type)    :: bgc_vegetation_inst
type(ch4_type)               :: ch4_inst
type(crop_type)              :: crop_inst
type(photosyns_type)         :: photosyns_inst
```

### Soil Biogeochemistry Types
```fortran
type(soilbiogeochem_state_type)          :: soilbiogeochem_state_inst
type(soilbiogeochem_carbonstate_type)    :: soilbiogeochem_carbonstate_inst
type(soilbiogeochem_carbonflux_type)     :: soilbiogeochem_carbonflux_inst
type(soilbiogeochem_nitrogenstate_type)  :: soilbiogeochem_nitrogenstate_inst
```

### Coupler Types
```fortran
type(atm2lnd_type) :: atm2lnd_inst
type(lnd2atm_type) :: lnd2atm_inst
type(lnd2glc_type) :: lnd2glc_inst
```

---

## Initialization Sequence

`main/clm_initializeMod.F90` implements two-phase initialization:

### Phase 1: `initialize1()`
```
├── control_init()           # Parse namelist
├── clm_varpar_init()        # Set array dimensions
├── clm_varcon_init()        # Set physical constants
├── surfrd_compat_check()    # Verify surface data
├── initGridCells()          # Create grid structure
└── [allocate decomposition]
```

### Phase 2: `initialize2()`
```
├── Allocate all state/flux types
├── Read surface data
├── Initialize biogeochemistry
├── Set up activity filters
└── CLMFatesInitialize()     # If use_fates=.true.
```

---

## Main Driver Sequence

`main/clm_driver.F90` orchestrates the physics calling sequence (~65 module dependencies):

```fortran
subroutine clm_drv()
  ! Dynamic subgrid
  call dynSubgrid_driver()

  ! Pre-flux calculations
  call BiogeophysPreFluxCalcs()

  ! Temperature/energy
  call SoilTemperature()
  call LakeTemperature()        ! if lake

  ! Surface fluxes
  call BareGroundFluxes()
  call CanopyFluxes()
  call SoilFluxes()
  call UrbanFluxes()            ! if urban
  call LakeFluxes()             ! if lake

  ! Hydrology
  call HydrologyNoDrainage()    ! or HydrologyDrainage()
  call CanopyInterception()
  call LakeHydrology()          ! if lake
  call HillslopeHydrology()     ! if hillslope

  ! Radiation
  call SurfaceAlbedo()
  call SurfaceRadiation()

  ! Biogeochemistry
  call CNDriver()               ! Carbon-nitrogen
  call ch4()                    ! Methane

  ! Output
  call hist_update_hbuf()       # Update history buffer
end subroutine
```

---

## Variable Naming Conventions

### Physical Constants (`clm_varcon.F90`)
```fortran
sb        ! Stefan-Boltzmann constant
vkc       ! von Karman constant
rwat      ! Gas constant for water vapor
cpliq     ! Specific heat of liquid water
cpice     ! Specific heat of ice
hvap      ! Latent heat of vaporization
hsub      ! Latent heat of sublimation
hfus      ! Latent heat of fusion
denh2o    ! Density of liquid water
denice    ! Density of ice
tkair     ! Thermal conductivity of air
```

### Special Values
```fortran
spval = 1.e36_r8      ! Missing value for reals
ispval = -9999        ! Missing value for integers
```

### Array Naming Pattern
```fortran
! Pattern: variable_level
tg_col(:)           ! Ground temperature [column]
h2osoi_liq(:,:)     ! Liquid soil water [column, layer]
lai_patch(:)        ! Leaf area index [patch]
fv_col(:)           ! Friction velocity [column]
```

### Suffixes by Level
| Suffix | Level |
|--------|-------|
| `_grc` | Gridcell |
| `_lun` | Land unit |
| `_col` | Column |
| `_patch` | Patch/PFT |

---

## Key Entry Points for Development

### Adding a New Physics Process

1. **Create type module** in appropriate directory (e.g., `biogeophys/MyProcessType.F90`)
2. **Declare instance** in `main/clm_instMod.F90`
3. **Initialize** in `main/clm_initializeMod.F90`
4. **Call process** from `main/clm_driver.F90`
5. **Add history output** in your type's `InitHistory()` method

### Adding a New Namelist Parameter

1. **Define** in `bld/namelist_files/namelist_definition_ctsm.xml`
2. **Set default** in `bld/namelist_files/namelist_defaults_ctsm.xml`
3. **Read** in `main/controlMod.F90`
4. **Use** via `clm_varctl` module

### Adding History Output

In your type module:
```fortran
subroutine InitHistory(this, bounds)
  call hist_addfld1d( &
       fname='MY_VAR', &
       units='kg/m2', &
       avgflag='A', &
       long_name='My new variable', &
       ptr_col=this%my_var_col)
end subroutine
```

---

## Control Variables (`main/clm_varctl.F90`)

Key runtime flags:
```fortran
use_cn              ! Use carbon-nitrogen model
use_crop            ! Use crop model
use_fates           ! Use FATES vegetation
use_hillslope       ! Use hillslope hydrology
use_luna            ! Use LUNA photosynthesis
use_hydrstress      ! Use hydraulic stress
create_glacier_mec_landunit  ! Create glacier MEC
```

---

## Filtering (`main/filterMod.F90`)

Filters mask inactive columns/patches for efficient looping:

```fortran
! Example: loop only over active vegetated columns
do fc = 1, num_hydrologyc
  c = filter_hydrologyc(fc)
  ! ... process column c
end do
```

Key filters:
- `filter_nolakec` - Non-lake columns
- `filter_hydrologyc` - Columns with active hydrology
- `filter_soilc` - Soil columns
- `filter_vegp` - Vegetated patches

---

## Unit Tests

Fortran unit tests use pFUnit framework. Test directories:
- `main/test/`
- `biogeophys/test/`
- `biogeochem/test/`
- `utils/test/`

Run tests:
```bash
cd $CTSMROOT
../cime/scripts/fortran_unit_testing/run_tests.py --build-dir unit_tests.temp
```

---

## See Also

- `main/CLAUDE.md` - Driver, types, control details
- `biogeophys/CLAUDE.md` - Hydrology, energy, hillslope
- `biogeochem/CLAUDE.md` - Carbon-nitrogen cycling
- `../CLAUDE.md` - Repository overview
