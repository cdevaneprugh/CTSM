# src/biogeophys - Biophysics and Hydrology

Physical processes: energy balance, hydrology, radiation, snow physics, and hillslope hydrology.

**~69,000 lines** across 87 modules.

---

## Module Categories

### Hydrology
| File | Purpose |
|------|---------|
| `HydrologyDrainageMod.F90` | Main hydrology with lateral drainage |
| `HydrologyNoDrainageMod.F90` | Hydrology without lateral flow |
| `CanopyHydrologyMod.F90` | Canopy interception and throughfall |
| `SoilHydrologyMod.F90` | Vertical soil water movement |
| `HillslopeHydrologyMod.F90` | Hillslope lateral flow |
| `HillslopeHydrologyUtilsMod.F90` | Hillslope utilities |
| `InfiltrationExcessRunoffMod.F90` | Infiltration excess |
| `SaturatedExcessRunoffMod.F90` | Saturation excess |
| `IrrigationMod.F90` | Irrigation application |
| `LakeHydrologyMod.F90` | Lake water balance |
| `GlacierSMBMod.F90` | Glacier surface mass balance |

### Energy/Fluxes
| File | Purpose |
|------|---------|
| `SoilTemperatureMod.F90` | Soil heat equation |
| `BareGroundFluxesMod.F90` | Bare ground sensible/latent heat |
| `CanopyFluxesMod.F90` | Canopy turbulent exchange |
| `SoilFluxesMod.F90` | Soil surface energy (was Biogeophysics2) |
| `UrbanFluxesMod.F90` | Urban energy balance |
| `LakeFluxesMod.F90` | Lake surface fluxes |
| `LakeTemperatureMod.F90` | Lake thermal structure |
| `FrictionVelocityMod.F90` | Friction velocity calculation |
| `QSatMod.F90` | Saturation vapor pressure |

### Radiation
| File | Purpose |
|------|---------|
| `SurfaceRadiationMod.F90` | Net radiation calculation |
| `SurfaceAlbedoMod.F90` | Snow/vegetation albedo |
| `AerosolMod.F90` | Aerosol deposition on snow |
| `SnowSnicarMod.F90` | SNICAR snow albedo model |
| `UrbanAlbedoMod.F90` | Urban canyon albedo |
| `LakeCon.F90` | Lake optical properties |

### State Types
| File | Purpose |
|------|---------|
| `SoilStateType.F90` | Soil properties |
| `CanopyStateType.F90` | Canopy structure |
| `WaterStateType.F90` | Water state variables |
| `WaterFluxType.F90` | Water flux variables |
| `EnergyFluxType.F90` | Energy flux variables |
| `TemperatureType.F90` | Temperature state |
| `LakeStateType.F90` | Lake state variables |
| `SolarAbsType.F90` | Absorbed solar radiation |
| `SurfaceAlbedoType.F90` | Albedo state |

---

## Hillslope Hydrology

The hillslope hydrology module enables lateral subsurface flow between columns arranged by topographic position.

### Key Files
- `HillslopeHydrologyMod.F90` - Main driver
- `HillslopeHydrologyUtilsMod.F90` - Helper functions

### Conceptual Model
```
            Ridge (upslope)
              |
              v  (lateral flow direction)
            Upper
              |
              v
            Lower
              |
              v
            Outlet --> Stream
```

### Key Variables
| Variable | Type | Purpose |
|----------|------|---------|
| `hslp_col` | column | Hillslope identifier |
| `hand_col` | column | Height above nearest drainage (m) |
| `nhillcolumns` | column | Number of columns in hillslope |
| `col_dndx` | column | Downslope column index |
| `col_updx` | column | Upslope column indices |
| `qflx_hillslope` | column | Lateral flow between columns (mm/s) |

### Spillheight Parameter (Research Feature)

Our fork adds `spillheight` namelist parameter for controlling saturation-limited lateral flow:

```fortran
! In HillslopeHydrologyMod.F90
real(r8) :: spillheight  ! Height threshold for lateral overflow (m)
```

Set in `user_nl_clm`:
```fortran
spillheight = 0.1
```

---

## Water State Variables

### WaterStateType (`WaterStateType.F90`)

```fortran
! Column-level water storage
real(r8), pointer :: h2osoi_liq(:,:)    ! Liquid soil water (kg/m2)
real(r8), pointer :: h2osoi_ice(:,:)    ! Ice soil water (kg/m2)
real(r8), pointer :: h2osoi_vol(:,:)    ! Volumetric water content (m3/m3)
real(r8), pointer :: h2osno_col(:)      ! Snow water equivalent (kg/m2)
real(r8), pointer :: h2osfc_col(:)      ! Surface water (kg/m2)

! Water table
real(r8), pointer :: zwt_col(:)         ! Water table depth (m)
real(r8), pointer :: zwt_perched_col(:) ! Perched water table (m)

! Total water storage
real(r8), pointer :: total_plant_stored_h2o(:)  ! Plant water (kg/m2)
```

### WaterFluxType (`WaterFluxType.F90`)

```fortran
! Precipitation/interception
real(r8), pointer :: qflx_prec_grnd(:)      ! Ground precipitation (mm/s)
real(r8), pointer :: qflx_rain_grnd(:)      ! Ground rain (mm/s)
real(r8), pointer :: qflx_snow_grnd(:)      ! Ground snow (mm/s)

! Runoff
real(r8), pointer :: qflx_surf(:)           ! Surface runoff (mm/s)
real(r8), pointer :: qflx_drain(:)          ! Subsurface drainage (mm/s)
real(r8), pointer :: qflx_qrgwl(:)          ! Groundwater runoff (mm/s)

! Evapotranspiration
real(r8), pointer :: qflx_evap_tot(:)       ! Total ET (mm/s)
real(r8), pointer :: qflx_evap_soi(:)       ! Soil evaporation (mm/s)
real(r8), pointer :: qflx_tran_veg(:)       ! Transpiration (mm/s)

! Infiltration
real(r8), pointer :: qflx_infl(:)           ! Infiltration (mm/s)
```

---

## Energy Flux Variables

### EnergyFluxType (`EnergyFluxType.F90`)

```fortran
! Surface fluxes
real(r8), pointer :: eflx_sh_tot(:)     ! Sensible heat flux (W/m2)
real(r8), pointer :: eflx_lh_tot(:)     ! Latent heat flux (W/m2)
real(r8), pointer :: eflx_lwrad_out(:)  ! Outgoing longwave (W/m2)
real(r8), pointer :: eflx_lwrad_net(:)  ! Net longwave (W/m2)

! Ground heat flux
real(r8), pointer :: eflx_soil_grnd(:)  ! Ground heat flux (W/m2)

! Energy balance
real(r8), pointer :: errsoi(:)          ! Soil energy balance error (W/m2)
real(r8), pointer :: errseb(:)          ! Surface energy balance error (W/m2)
```

---

## Key Calling Sequence

From `clm_driver.F90`:

```fortran
! 1. Pre-flux calculations
call Hydrology_PreFlux()        ! Update saturation

! 2. Energy balance
call SoilTemperature()          ! Solve heat equation
call BareGroundFluxes()         ! Sensible/latent over bare ground
call CanopyFluxes()             ! Sensible/latent through canopy
call SoilFluxes()               ! Ground heat flux

! 3. Hydrology
call HydrologyNoDrainage()      ! Infiltration, soil water
! or
call HydrologyDrainage()        ! With lateral drainage

call CanopyInterception()       ! Interception of precip

! 4. Hillslope (if enabled)
if (use_hillslope) then
  call HillslopeHydrology()     ! Lateral subsurface flow
end if

! 5. Radiation (when needed)
call SurfaceAlbedo()            ! Compute albedo
call SurfaceRadiation()         ! Net radiation
```

---

## Snow Physics

### Key Modules
- `SnowSnicarMod.F90` - SNICAR snow albedo model
- `AerosolMod.F90` - Aerosol deposition on snow

### Snow Layer Variables
```fortran
integer :: snl           ! Number of snow layers (negative, -5 to 0)
real(r8) :: h2osno       ! Snow water equivalent (kg/m2)
real(r8) :: snow_depth   ! Snow depth (m)
real(r8) :: frac_sno     ! Fraction of ground covered by snow
real(r8) :: snw_rds(:)   ! Snow grain radius (microns)
```

### Snow Layer Structure
```
  snl = -3 means 3 snow layers:
  Layer -3: top (newest snow)
  Layer -2: middle
  Layer -1: bottom (oldest snow)
  Layer  0: (not used for snow)
  Layer  1: top soil layer
```

---

## Lake Model

### Key Modules
- `LakeTemperatureMod.F90` - Lake thermal diffusion
- `LakeHydrologyMod.F90` - Lake water balance
- `LakeFluxesMod.F90` - Lake surface fluxes
- `LakeCon.F90` - Lake optical/thermal properties

### Lake Variables
```fortran
! LakeStateType
real(r8), pointer :: t_lake(:,:)     ! Lake temperature (K)
real(r8), pointer :: lake_icefrac(:,:)  ! Lake ice fraction
real(r8), pointer :: savedtke1(:)    ! Top layer diffusivity
```

---

## Urban Model

### Key Modules
- `UrbanFluxesMod.F90` - Urban canyon energy balance
- `UrbanAlbedoMod.F90` - Urban surface albedo
- `UrbanParamsType.F90` - Urban parameters
- `UrbanTimeVarType.F90` - Time-varying urban properties

### Urban Canyon Geometry
```
     Sky
      |
  Wall | | Wall
      | |
  ----   ----
    Floor (pervious/impervious)
```

---

## Adding New Hydrology Processes

### 1. Create Type Module
```fortran
! MyHydroType.F90
module MyHydroType
  type, public :: myhydro_type
    real(r8), pointer :: my_flux(:)  ! column level
  contains
    procedure :: Init
    procedure :: InitHistory
  end type
end module
```

### 2. Register in clm_instMod
```fortran
use MyHydroType, only : myhydro_type
type(myhydro_type) :: myhydro_inst
```

### 3. Initialize
```fortran
! In clm_initializeMod.F90::initialize2()
call myhydro_inst%Init(bounds)
```

### 4. Call from Driver
```fortran
! In clm_driver.F90
call MyHydroProcess(myhydro_inst, ...)
```

---

## Unit Tests

Test directory: `test/`

Key tests:
- Water balance verification
- Energy balance closure
- Hillslope flow direction

Run:
```bash
../../../cime/scripts/fortran_unit_testing/run_tests.py \
  --build-dir build_test --test-spec-dir .
```

---

## See Also

- `../CLAUDE.md` - Source overview
- `../main/CLAUDE.md` - Driver and types
- `../biogeochem/CLAUDE.md` - Carbon-nitrogen coupling
- `../../CLAUDE.md` - Repository overview
