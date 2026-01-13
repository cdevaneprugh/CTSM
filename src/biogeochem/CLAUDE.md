# src/biogeochem - Biogeochemistry

Carbon-nitrogen cycling, phenology, photosynthesis, and vegetation dynamics.

**~64,500 lines** across 71 modules.

---

## Module Categories

### Carbon-Nitrogen Drivers
| File | Purpose |
|------|---------|
| `CNDriverMod.F90` | Main C-N driver (orchestrates all BGC) |
| `CNAllocationMod.F90` | Carbon/nitrogen allocation |
| `CNBalanceCheckMod.F90` | C-N mass balance verification |
| `CNStateUpdate1Mod.F90` | State update phase 1 (gap mortality) |
| `CNStateUpdate2Mod.F90` | State update phase 2 (phenology) |
| `CNStateUpdate3Mod.F90` | State update phase 3 (fire) |
| `CNGapMortalityMod.F90` | Gap-phase mortality |

### Photosynthesis
| File | Purpose |
|------|---------|
| `PhotosynthesisMod.F90` | Main photosynthesis model |
| `PhotosynthesisHydraulicsMod.F90` | Hydraulic-limited photosynthesis |
| `LunaMod.F90` | LUNA optimal photosynthesis |

### Phenology
| File | Purpose |
|------|---------|
| `CNPhenologyMod.F90` | Phenology driver |
| `CNDVMod.F90` | Dynamic vegetation phenology |
| `CNDVType.F90` | DV state variables |

### Vegetation Carbon/Nitrogen State
| File | Purpose |
|------|---------|
| `CNVegStateType.F90` | Vegetation structure |
| `CNVegCarbonStateType.F90` | Vegetation C pools |
| `CNVegCarbonFluxType.F90` | Vegetation C fluxes |
| `CNVegNitrogenStateType.F90` | Vegetation N pools |
| `CNVegNitrogenFluxType.F90` | Vegetation N fluxes |

### Fire
| File | Purpose |
|------|---------|
| `CNFireBaseMod.F90` | Base fire model |
| `CNFireAreaMod.F90` | Fire area calculation |
| `CNFireEmissionsMod.F90` | Fire emissions |
| `CNFireMethodType.F90` | Fire method interface |
| `CNFireFactoryMod.F90` | Fire model factory |

### Methane
| File | Purpose |
|------|---------|
| `ch4Mod.F90` | Methane biogeochemistry |
| `ch4varcon.F90` | Methane constants |

### Crop
| File | Purpose |
|------|---------|
| `CropMod.F90` | Crop model driver |
| `CropType.F90` | Crop state variables |
| `CNNDynamicsMod.F90` | Crop nitrogen dynamics |

### Isotopes
| File | Purpose |
|------|---------|
| `CNCIsoAtmMod.F90` | C13/C14 atmospheric source |
| `CNVegCarbonFluxType.F90` | Includes C13/C14 variants |

---

## Key Carbon Pools

### Vegetation Carbon (`CNVegCarbonStateType`)

```fortran
! Patch-level carbon pools (kg C/m2)
real(r8), pointer :: leafc_patch(:)          ! Leaf carbon
real(r8), pointer :: frootc_patch(:)         ! Fine root carbon
real(r8), pointer :: livestemc_patch(:)      ! Live stem carbon
real(r8), pointer :: deadstemc_patch(:)      ! Dead stem carbon
real(r8), pointer :: livecrootc_patch(:)     ! Live coarse root carbon
real(r8), pointer :: deadcrootc_patch(:)     ! Dead coarse root carbon
real(r8), pointer :: gresp_storage_patch(:)  ! Growth respiration storage
real(r8), pointer :: cpool_patch(:)          ! Temporary photosynthate pool

! Storage pools (for allocation)
real(r8), pointer :: leafc_storage_patch(:)
real(r8), pointer :: frootc_storage_patch(:)
real(r8), pointer :: livestemc_storage_patch(:)
real(r8), pointer :: deadstemc_storage_patch(:)

! Transfer pools (for growth)
real(r8), pointer :: leafc_xfer_patch(:)
real(r8), pointer :: frootc_xfer_patch(:)
real(r8), pointer :: livestemc_xfer_patch(:)
real(r8), pointer :: deadstemc_xfer_patch(:)

! Total vegetation carbon
real(r8), pointer :: totvegc_patch(:)        ! Total veg C (kg C/m2)
```

### Key Carbon Fluxes (`CNVegCarbonFluxType`)

```fortran
! Photosynthesis and respiration (kg C/m2/s)
real(r8), pointer :: gpp_patch(:)            ! Gross primary production
real(r8), pointer :: npp_patch(:)            ! Net primary production
real(r8), pointer :: ar_patch(:)             ! Autotrophic respiration
real(r8), pointer :: mr_patch(:)             ! Maintenance respiration
real(r8), pointer :: gr_patch(:)             ! Growth respiration

! Component respiration
real(r8), pointer :: leaf_mr_patch(:)        ! Leaf maintenance resp
real(r8), pointer :: froot_mr_patch(:)       ! Fine root maint resp
real(r8), pointer :: livestem_mr_patch(:)    ! Live stem maint resp
real(r8), pointer :: livecroot_mr_patch(:)   ! Live coarse root maint resp

! Turnover fluxes
real(r8), pointer :: leafc_to_litter_patch(:)    ! Leaf litter flux
real(r8), pointer :: frootc_to_litter_patch(:)   ! Fine root litter flux
```

---

## Nitrogen Pools and Fluxes

### Vegetation Nitrogen (`CNVegNitrogenStateType`)

```fortran
! Patch-level nitrogen pools (kg N/m2)
real(r8), pointer :: leafn_patch(:)          ! Leaf nitrogen
real(r8), pointer :: frootn_patch(:)         ! Fine root nitrogen
real(r8), pointer :: livestemn_patch(:)      ! Live stem nitrogen
real(r8), pointer :: deadstemn_patch(:)      ! Dead stem nitrogen
real(r8), pointer :: livecrootn_patch(:)     ! Live coarse root nitrogen
real(r8), pointer :: deadcrootn_patch(:)     ! Dead coarse root nitrogen
real(r8), pointer :: retransn_patch(:)       ! Retranslocated nitrogen

! Total vegetation nitrogen
real(r8), pointer :: totvegn_patch(:)        ! Total veg N (kg N/m2)
```

### Nitrogen Fluxes (`CNVegNitrogenFluxType`)

```fortran
! Uptake and retranslocation (kg N/m2/s)
real(r8), pointer :: sminn_to_plant_patch(:)     ! Soil mineral N uptake
real(r8), pointer :: retransn_to_npool_patch(:)  ! Retranslocation flux

! Turnover
real(r8), pointer :: leafn_to_litter_patch(:)    ! Leaf litter N flux
real(r8), pointer :: frootn_to_litter_patch(:)   ! Fine root litter N flux
```

---

## CNDriver Calling Sequence

`CNDriverMod.F90::CNDriver()` orchestrates all biogeochemistry:

```fortran
subroutine CNDriver(...)
  ! 1. Nitrogen deposition and fixation
  call CNNDeposition()
  call CNNFixation()

  ! 2. Phenology - leaf onset/offset
  call CNPhenology()

  ! 3. Photosynthesis
  call Photosynthesis()       ! or LUNA if enabled

  ! 4. Respiration
  call CNMResp()              ! Maintenance respiration
  call CNGResp()              ! Growth respiration

  ! 5. Allocation
  call CNAllocation()

  ! 6. Mortality
  call CNGapMortality()

  ! 7. Soil biogeochemistry (in soilbiogeochem/)
  call SoilBiogeochemDriver()

  ! 8. State updates
  call CNStateUpdate1()       ! Gap mortality
  call CNStateUpdate2()       ! Phenology
  call CNStateUpdate3()       ! Fire

  ! 9. Fire (if enabled)
  if (use_cn_fires) then
    call CNFireArea()
    call CNFireFluxes()
  end if

  ! 10. Methane (if enabled)
  if (use_lch4) call ch4()

  ! 11. Mass balance check
  call CNBalanceCheck()
end subroutine
```

---

## Photosynthesis Models

### Standard Model (`PhotosynthesisMod.F90`)

Farquhar-Ball-Berry model with:
- Rubisco-limited photosynthesis (Ac)
- Light-limited photosynthesis (Aj)
- Export-limited photosynthesis (Ap)

```fortran
! Key variables
real(r8) :: vcmax    ! Maximum carboxylation rate (umol/m2/s)
real(r8) :: jmax     ! Maximum electron transport (umol/m2/s)
real(r8) :: ci       ! Internal CO2 concentration (Pa)
real(r8) :: an       ! Net leaf photosynthesis (umol/m2/s)
real(r8) :: gs       ! Stomatal conductance (mol/m2/s)
```

### LUNA Model (`LunaMod.F90`)

Optimal photosynthesis model that predicts Vcmax from N allocation.

Enable with: `use_luna = .true.` in namelist.

---

## Phenology

### Phenology Types
```fortran
! Phenology states (in CNPhenologyMod.F90)
integer, parameter :: iphen_dormant = 0       ! Dormant
integer, parameter :: iphen_dormant_heat = 1  ! Waiting for heat
integer, parameter :: iphen_evergreen = 2     ! Evergreen
integer, parameter :: iphen_stress = 3        ! Under stress
```

### Key Phenology Variables
```fortran
real(r8) :: onset_flag     ! Leaf onset trigger
real(r8) :: offset_flag    ! Leaf offset trigger
real(r8) :: dayl           ! Daylength (seconds)
real(r8) :: gdd            ! Growing degree days
```

---

## Fire Model

### Fire Area Calculation
```fortran
! CNFireAreaMod.F90
real(r8) :: fire_m         ! Fire moisture factor
real(r8) :: fire_spread    ! Fire spread rate
real(r8) :: burned_area    ! Burned fraction
```

### Fire Emissions
```fortran
! CNFireEmissionsMod.F90
real(r8) :: fire_closs     ! Fire carbon loss (kg C/m2/s)
real(r8) :: fire_nloss     ! Fire nitrogen loss (kg N/m2/s)
```

---

## Methane Model (`ch4Mod.F90`)

Lake and wetland methane production and oxidation.

Enable with: `use_lch4 = .true.` in namelist.

### Key Variables
```fortran
real(r8) :: ch4_prod       ! CH4 production (mol/m2/s)
real(r8) :: ch4_oxid       ! CH4 oxidation (mol/m2/s)
real(r8) :: ch4_surf_flux  ! Surface CH4 flux (mol/m2/s)
```

---

## Crop Model

Enable with: `use_crop = .true.` and crop compset.

### Crop Types
```fortran
! Crop PFT indices (17-78 for crops)
integer, parameter :: nc3crop = 17      ! Generic C3 crop
integer, parameter :: nc3irrig = 18     ! Irrigated C3 crop
integer, parameter :: ncorn = 19        ! Corn
integer, parameter :: ncornirrig = 20   ! Irrigated corn
integer, parameter :: nwheat = 21       ! Wheat
! ... etc.
```

### Crop-Specific Variables
```fortran
real(r8) :: hui            ! Heat unit index
real(r8) :: gddmaturity    ! GDD to maturity
real(r8) :: grainfrac      ! Grain fraction
real(r8) :: yield          ! Crop yield
```

---

## Key History Output Variables

```fortran
! Carbon fluxes
GPP           ! Gross primary production (gC/m2/s)
NPP           ! Net primary production (gC/m2/s)
AR            ! Autotrophic respiration (gC/m2/s)
HR            ! Heterotrophic respiration (gC/m2/s)
NEE           ! Net ecosystem exchange (gC/m2/s)

! Carbon pools
TOTECOSYSC    ! Total ecosystem carbon (gC/m2)
TOTVEGC       ! Total vegetation carbon (gC/m2)
TOTSOMC       ! Total soil organic matter C (gC/m2)
TOTLITC       ! Total litter carbon (gC/m2)

! Nitrogen
NPOOL         ! Plant nitrogen pool (gN/m2)
SMINN         ! Soil mineral nitrogen (gN/m2)

! Leaf
TLAI          ! Total LAI (m2/m2)
LEAFC         ! Leaf carbon (gC/m2)
```

---

## Adding New BGC Processes

### 1. Create Module
```fortran
module MyBGCMod
  use clm_varctl, only : use_mybgc
contains
  subroutine MyBGCProcess(...)
    if (.not. use_mybgc) return
    ! ... implementation
  end subroutine
end module
```

### 2. Add Control Variable
In `main/clm_varctl.F90`:
```fortran
logical :: use_mybgc = .false.
```

### 3. Add Namelist Entry
In `bld/namelist_files/namelist_definition_ctsm.xml`:
```xml
<entry id="use_mybgc" type="logical" ...>
```

### 4. Call from CNDriver
In `CNDriverMod.F90`:
```fortran
call MyBGCProcess(...)
```

---

## Spinup Considerations

BGC spinup requires long simulations for C-N equilibrium:

### Accelerated Spinup
```fortran
clm_accelerated_spinup = 'on'   ! AD spinup (~200 years)
clm_accelerated_spinup = 'off'  ! Post-AD (~hundreds years)
```

### Monitoring Variables
- `TOTECOSYSC` - Should stabilize
- `TOTSOMC` - Slowest pool
- `GPP`, `NPP` - Should be reasonable for biome

---

## See Also

- `../CLAUDE.md` - Source overview
- `../soilbiogeochem/` - Soil C-N cycling
- `../main/CLAUDE.md` - Driver integration
- `../../bld/namelist_files/` - BGC namelist options
