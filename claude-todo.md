# CTSM Documentation Enhancement - Phase 4.4

Tracking progress for comprehensive CTSM documentation effort.

**Plan file:** `~/.claude/plans/stateful-brewing-mist.md`

## Status

| Phase | Description | Status |
|-------|-------------|--------|
| 1 | Root CLAUDE.md enhancement | Complete |
| 2 | tools/ + python/ documentation | Complete |
| 3 | src/ documentation | Complete |
| 4 | Testing documentation | Complete |
| 5 | Library research (mpi-serial/PIO) | Complete |
| 6 | Verification & commit | In Progress |

---

## Phase 1: Root Enhancement

- [x] Expand CLAUDE.md with detailed directory tree
- [x] Add testing overview section
- [x] Add documentation index
- [x] Create this claude-todo.md

## Phase 2: tools/ + python/

- [x] `tools/CLAUDE.md` - Tool inventory, decision tree, wrapper→implementation map
- [x] `tools/mksurfdata_esmf/CLAUDE.md` - Build process, HiPerGator modifications
- [x] `tools/site_and_regional/CLAUDE.md` - subset_data, run_tower, mesh tools
- [x] `python/CLAUDE.md` - Package structure, module relationships
- [x] `python/ctsm/site_and_regional/CLAUDE.md` - SinglePointCase, RegionalCase

## Phase 3: src/

- [x] `src/CLAUDE.md` - Directory overview, module load order, type hierarchy
- [x] `src/main/CLAUDE.md` - Driver, types, control variables
- [x] `src/biogeophys/CLAUDE.md` - Hydrology, energy, radiation, hillslope
- [x] `src/biogeochem/CLAUDE.md` - Carbon-nitrogen cycling, phenology

## Phase 4: Testing

- [x] `TESTING.md` - All 5 testing systems, invocations, HiPerGator notes

## Phase 5: Libraries

- [x] Research mpi-serial feasibility
- [x] Research shared PIO builds
- [x] `libraries/CLAUDE.md` - Findings and recommendations

## Phase 6: Verification

- [x] Cross-reference all CLAUDE.md files
- [x] Verify file paths and examples
- [x] Update hpg-esm-tools documentation index
- [ ] Commit and push to fork

---

## Key Findings (from planning phase)

**Architecture:**
- `python/ctsm/` contains actual implementations (~5,600 LOC)
- `tools/` contains lightweight wrappers (20-40 lines each)
- Pattern: wrapper imports from python/ctsm and calls main()

**Testing Systems:**
| System | Location | Purpose |
|--------|----------|---------|
| run_sys_tests | `./run_sys_tests` | CTSM system test orchestration |
| CIME create_test | `cime/scripts/create_test` | 200+ integration tests |
| Fortran unit tests | `src/*/test/` | pFUnit framework |
| Python unit tests | `python/ctsm/test/` | ~12,300 LOC |
| FATES tests | `src/fates/testing/` | Functional + unit |

**Source Code:**
- ~197,000 lines of Fortran in src/
- Subgrid hierarchy: GridcellType → LandunitType → ColumnType → PatchType
- ~50+ type instances in `clm_instMod.F90`
