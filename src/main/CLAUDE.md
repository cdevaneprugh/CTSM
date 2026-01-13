# src/main - Driver, Types, and Control

Core CTSM infrastructure: driver loop, type definitions, initialization, and runtime control.

---

## Key Files

| File | LOC | Purpose |
|------|-----|---------|
| `clm_driver.F90` | ~2,500 | Main physics calling sequence |
| `clm_initializeMod.F90` | ~1,800 | Two-phase initialization |
| `clm_instMod.F90` | ~800 | Global type instance registry |
| `controlMod.F90` | ~1,500 | Namelist reading and control |
| `clm_varctl.F90` | ~600 | Runtime control flags |
| `clm_varcon.F90` | ~800 | Physical constants |
| `clm_varpar.F90` | ~400 | Array dimension parameters |
| `GridcellType.F90` | ~400 | Gridcell data structure |
| `LandunitType.F90` | ~500 | Land unit data structure |
| `ColumnType.F90` | ~600 | Column data structure |
| `PatchType.F90` | ~700 | Patch/PFT data structure |
| `filterMod.F90` | ~400 | Activity filters |
| `decompMod.F90` | ~600 | Domain decomposition |
| `atm2lndType.F90` | ~500 | Atmospheric forcing input |
| `lnd2atmType.F90` | ~400 | Land surface output |

---

## Subgrid Type Hierarchy

### GridcellType.F90
```fortran
type, public :: gridcell_type
  ! Indices
  integer, pointer :: luni(:)      ! First land unit index
  integer, pointer :: lunf(:)      ! Last land unit index
  integer, pointer :: coli(:)      ! First column index
  integer, pointer :: colf(:)      ! Last column index
  integer, pointer :: patchi(:)    ! First patch index
  integer, pointer :: patchf(:)    ! Last patch index

  ! Geography
  real(r8), pointer :: lat(:)      ! Latitude (degrees)
  real(r8), pointer :: lon(:)      ! Longitude (degrees)
  real(r8), pointer :: area(:)     ! Grid cell area (km2)

  ! Topography
  real(r8), pointer :: elevation(:)  ! Mean elevation (m)
  real(r8), pointer :: slope(:)      ! Mean slope (degrees)
  real(r8), pointer :: aspect(:)     ! Mean aspect (degrees)
contains
  procedure :: Init
  procedure :: Clean
end type
```

### LandunitType.F90
```fortran
type, public :: landunit_type
  ! Indices
  integer, pointer :: gridcell(:)  ! Parent gridcell
  integer, pointer :: coli(:)      ! First column
  integer, pointer :: colf(:)      ! Last column
  integer, pointer :: patchi(:)    ! First patch
  integer, pointer :: patchf(:)    ! Last patch

  ! Type identifier
  integer, pointer :: itype(:)     ! Land unit type (1-9)
  ! 1=vegetated, 2=crop, 3=wetland, 4=glacier,
  ! 5=glacier_mec, 6=lake, 7=urban_tbd, 8=urban_hd, 9=urban_md

  ! Fractions
  real(r8), pointer :: wtgcell(:)  ! Weight on gridcell
end type
```

### ColumnType.F90
```fortran
type, public :: column_type
  ! Indices
  integer, pointer :: gridcell(:)  ! Parent gridcell
  integer, pointer :: landunit(:)  ! Parent land unit
  integer, pointer :: patchi(:)    ! First patch
  integer, pointer :: patchf(:)    ! Last patch

  ! Type identifier
  integer, pointer :: itype(:)     ! Column type

  ! Topography (hillslope)
  real(r8), pointer :: hslp_col(:)      ! Hillslope
  real(r8), pointer :: hand_col(:)      ! Height above nearest drainage
  real(r8), pointer :: nhillcolumns(:)  ! Number of hillslope columns

  ! Vertical structure
  integer,  pointer :: snl(:)           ! Number of snow layers (<0)
  real(r8), pointer :: z(:,:)           ! Layer center depth (m)
  real(r8), pointer :: dz(:,:)          ! Layer thickness (m)
  real(r8), pointer :: zi(:,:)          ! Layer interface depth (m)
end type
```

### PatchType.F90
```fortran
type, public :: patch_type
  ! Indices
  integer, pointer :: gridcell(:)  ! Parent gridcell
  integer, pointer :: landunit(:)  ! Parent land unit
  integer, pointer :: column(:)    ! Parent column

  ! Type identifier
  integer, pointer :: itype(:)     ! PFT type (1-17)

  ! Weights
  real(r8), pointer :: wtcol(:)    ! Weight on column
  real(r8), pointer :: wtlunit(:)  ! Weight on land unit
  real(r8), pointer :: wtgcell(:)  ! Weight on gridcell
end type
```

---

## Control Variables (`clm_varctl.F90`)

### Physics Options
```fortran
logical :: use_cn           ! Carbon-nitrogen model
logical :: use_crop         ! Crop model
logical :: use_fates        ! FATES vegetation
logical :: use_hillslope    ! Hillslope hydrology
logical :: use_luna         ! LUNA photosynthesis
logical :: use_hydrstress   ! Hydraulic stress
logical :: use_bedrock      ! Bedrock in soil column
```

### Spinup Control
```fortran
logical :: spinup_state                    ! In spinup mode
character(len=*) :: clm_accelerated_spinup ! 'on', 'off', 'sasu'
```

### History Output Control
```fortran
logical :: hist_empty_htapes        ! Clear default history
integer :: hist_nhtfrq(maxhist)     ! Output frequency
integer :: hist_mfilt(maxhist)      ! Samples per file
character :: hist_fincl1(maxflds)   ! Fields to include
character :: hist_fexcl1(maxflds)   ! Fields to exclude
```

---

## Initialization Sequence Detail

### `clm_initializeMod.F90::initialize1()`

```fortran
subroutine initialize1(...)
  ! 1. Parse namelist
  call control_init()

  ! 2. Set array dimensions
  call clm_varpar_init()

  ! 3. Set physical constants
  call clm_varcon_init()

  ! 4. Initialize decomposition
  call decompInit_lnd()

  ! 5. Allocate subgrid types
  call grc%Init(bounds)
  call lun%Init(bounds)
  call col%Init(bounds)
  call patch%Init(bounds)

  ! 6. Read surface data
  call surfrd(...)
end subroutine
```

### `clm_initializeMod.F90::initialize2()`

```fortran
subroutine initialize2(...)
  ! 1. Allocate all state/flux types
  call atm2lnd_inst%Init(bounds)
  call lnd2atm_inst%Init(bounds)
  call canopystate_inst%Init(bounds)
  call soilstate_inst%Init(bounds)
  call water_inst%Init(bounds)
  call temperature_inst%Init(bounds)
  ! ... (~30+ more type allocations)

  ! 2. Initialize history infrastructure
  call hist_init()

  ! 3. Read restart or initialize cold start
  if (nsrest == nsrContinue) then
    call restFile_read(...)
  else
    call initCold(...)
  end if

  ! 4. Initialize biogeochemistry
  call CNInit(...)

  ! 5. Set up activity filters
  call filter_init(...)

  ! 6. FATES initialization (if enabled)
  if (use_fates) call CLMFatesInitialize(...)
end subroutine
```

---

## Driver Loop (`clm_driver.F90`)

```fortran
subroutine clm_drv(doalb, nextsw_cday)
  ! Update land surface properties
  call dynSubgrid_driver()

  ! Pre-flux calculations
  call BiogeophysPreFluxCalcs()

  ! Energy balance calculations
  do_energy: do
    call SoilTemperature()
    if (has_lake) call LakeTemperature()

    call BareGroundFluxes()
    call CanopyFluxes()
    call SoilFluxes()
    if (has_urban) call UrbanFluxes()
    if (has_lake) call LakeFluxes()
  end do do_energy

  ! Hydrology
  call HydrologyNoDrainage()  ! or HydrologyDrainage()
  call CanopyInterception()
  if (has_lake) call LakeHydrology()
  if (use_hillslope) call HillslopeHydrology()

  ! Radiation (if albedo calculation needed)
  if (doalb) then
    call SurfaceAlbedo()
    call SurfaceRadiation()
  end if

  ! Biogeochemistry
  if (use_cn) call CNDriver()
  if (use_lch4) call ch4()

  ! Update history buffer
  call hist_update_hbuf()
end subroutine
```

---

## Activity Filters (`filterMod.F90`)

Filters optimize loops by skipping inactive points:

```fortran
! Filter definition
type filter_col_type
  integer :: num            ! Number of active columns
  integer, pointer :: idx(:) ! Indices of active columns
end type

! Global filter instances
type(filter_col_type) :: filter_nolakec     ! Non-lake columns
type(filter_col_type) :: filter_hydrologyc  ! Hydrology-active columns
type(filter_col_type) :: filter_soilc       ! Soil columns
type(filter_col_type) :: filter_snowc       ! Snow-covered columns
```

### Usage Pattern
```fortran
do fc = 1, filter_hydrologyc%num
  c = filter_hydrologyc%idx(fc)
  ! Process only active hydrology column c
  h2osoi_liq(c,:) = ...
end do
```

---

## Physical Constants (`clm_varcon.F90`)

### Thermodynamic Constants
```fortran
real(r8), parameter :: sb = 5.67e-8_r8       ! Stefan-Boltzmann (W/m2/K4)
real(r8), parameter :: vkc = 0.4_r8          ! von Karman constant
real(r8), parameter :: grav = 9.80616_r8     ! Gravity (m/s2)
real(r8), parameter :: tfrz = 273.15_r8      ! Freezing point (K)
```

### Heat Capacities
```fortran
real(r8), parameter :: cpair = 1.00464e3_r8  ! Dry air (J/kg/K)
real(r8), parameter :: cpliq = 4.188e3_r8    ! Liquid water (J/kg/K)
real(r8), parameter :: cpice = 2.11727e3_r8  ! Ice (J/kg/K)
```

### Latent Heats
```fortran
real(r8), parameter :: hvap = 2.501e6_r8     ! Vaporization (J/kg)
real(r8), parameter :: hsub = 2.501e6_r8 + 3.337e5_r8  ! Sublimation
real(r8), parameter :: hfus = 3.337e5_r8     ! Fusion (J/kg)
```

### Missing Values
```fortran
real(r8), parameter :: spval = 1.e36_r8      ! Real missing value
integer,  parameter :: ispval = -9999        ! Integer missing value
```

---

## Namelist Reading (`controlMod.F90`)

```fortran
subroutine control_init()
  ! Read lnd_in namelist
  namelist /clm_inparm/ &
    finidat, nrevsn, nsrest, &
    hist_nhtfrq, hist_mfilt, &
    hist_empty_htapes, hist_fincl1, &
    use_cn, use_crop, use_fates, use_hillslope, &
    ! ... many more parameters

  ! Open and read namelist
  open(newunit=unitn, file='lnd_in', status='old')
  read(unitn, nml=clm_inparm)
  close(unitn)

  ! Validate and set derived quantities
  call control_consistency_check()
end subroutine
```

---

## Unit Tests

Test directory: `test/`

Key test files:
- `test_clm_varcon.pf` - Physical constants
- `test_filterMod.pf` - Filter functionality
- `test_decompMod.pf` - Decomposition

Run:
```bash
../../../cime/scripts/fortran_unit_testing/run_tests.py \
  --build-dir build_test --test-spec-dir . --test-name clm_varcon
```

---

## See Also

- `../CLAUDE.md` - Source overview
- `../biogeophys/CLAUDE.md` - Physics modules
- `../biogeochem/CLAUDE.md` - BGC modules
- `../../bld/namelist_files/` - Namelist definitions
