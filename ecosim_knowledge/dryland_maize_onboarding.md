# Learning EcoSIM with the dryland-maize case

This is a hands-on route into EcoSIM for someone who is comfortable with
Python and ecosystem science. It begins with one checked-in case, follows its
inputs through the Fortran model, and ends with an analysis of the generated
NetCDF history file.

The companion notebook is
[`dryland_maize_analysis.ipynb`](dryland_maize_analysis.ipynb). It uses the
310-day scratch run described below and can be pointed at another compatible
EcoSIM run with the `ECOSIM_DRYLAND_RUN_DIR` environment variable.

## 1. The repository in one picture

```text
site and experiment data
  |-- grid/soil initialization NetCDF
  |-- hourly climate NetCDF
  |-- plant-management NetCDF
  |-- soil-management NetCDF
  |-- PFT and microbial parameters
  `-- atmospheric greenhouse gases
                    |
                    v
             case namelist
                    |
                    v
       drivers/ecosim/ecosim.F90
                    |
                    v
       hourly coupled process loop
                    |
                    v
       <case>.ecosim.h0.<date>.nc
                    |
                    v
        Python/xarray diagnostics
```

There are two operational halves:

- The Fortran model lives under [`../f90src/`](../f90src/) and is launched by
  [`../drivers/ecosim/ecosim.F90`](../drivers/ecosim/ecosim.F90). It owns the
  state, process equations, timestepping, balances, restart files, and history
  output.
- [`../ecosim-agent/`](../ecosim-agent/) is the Python preprocessing and
  analysis bridge. Its canonical workflows are under `.agents/skills/`; they
  prepare site metadata, climate, soils, management, vegetation, namelists,
  and post-run summaries.

The checked-in examples connect these halves:

- [`../examples/inputs/`](../examples/inputs/) contains case-specific NetCDF
  inputs.
- [`../examples/run_dir/`](../examples/run_dir/) contains the namelists and is
  the intended working-directory depth.
- [`../input_data/`](../input_data/) contains shared PFT, microbial, and
  atmospheric inputs.

## 2. The dryland-maize experiment

The teaching namelist is
[`../examples/run_dir/dryland/dryland.namelist`](../examples/run_dir/dryland/dryland.namelist).
It is a one-grid-cell, hourly case with transient crop and soil management.

| Namelist field | File | Scientific role |
| --- | --- | --- |
| `grid_file_in` | `dryland_grid_20230221.nc` | One grid cell, one topographic unit, 20 soil levels; site, soil physics, initial pools, and boundary conditions |
| `clm_hour_file_in` | `dryland_clm_20230203.nc` | Six years (2001–2006) of hourly temperature, wind, precipitation, vapor pressure, radiation, and annual precipitation chemistry |
| `pft_mgmt_in` | `dryland_pft_20230509.nc` | One active PFT per year, alternating maize and soybean, with planting and harvest strings |
| `soil_mgmt_in` | `dryland_soilmgmt_20230210.nc` | Tillage and fertilizer event tables; irrigation is inactive |
| `pft_file_in` | `ecosim_pftpar_20260915.nc` | Trait records for 83 PFT/climate combinations |
| `micpar_file_in` | `MicrobePars.20260211.nc` | Microbial guild and kinetic parameters |
| `atm_ghg_in` | `fatm_hist_GHGs_1750-2025.nc` | Monthly atmospheric CO2, CH4, and N2O |

The namelist also sets `atm_co2_fix=370.`. In
[`../f90src/IOutils/ClimReadMod.F90`](../f90src/IOutils/ClimReadMod.F90), a
positive fixed value overrides CO2 from the GHG file, while CH4 and N2O still
come from the file.

### 2001 management timeline

Input dates use `DDMMYYYY`.

| Date | Event | Input meaning |
| --- | --- | --- |
| April 9, 10, 16 | Tillage | Type 3 to 0.10 m |
| April 20 | Tillage | Type 7 to 0.05 m |
| May 14 | Plant maize | 6.2 plants m-2 at 0.05 m depth |
| May 16 | Fertilization | Event `me01f`; cumulative output reaches 12.81 g N m-2 |
| October 29 | Harvest/cut | Plant-management event for the active maize PFT |
| November 8 | Tillage | Type 1 to 0.20 m |

Population is per square metre and planting depth is in metres. The grid and
management files all use `NH1=NH2=NV1=NV2=1`, and `NZ=1` is the active PFT
count for the topographic unit.

## 3. What happens during one model hour

The main program reads the namelist, constructs the mesh, initializes modules
and inputs, prepares history tapes, and loops over the forcing periods. The
central sequence is in
[`../drivers/ecosim/EcoSIMAPI.F90`](../drivers/ecosim/EcoSIMAPI.F90):

1. `PrepHourlyWeather` prepares the current climate forcing.
2. `HOUR1` updates surface energy, radiation, and water processes.
3. `WATSUB` solves soil water and heat transfer.
4. `MicrobeModel` updates microbial transformations when enabled.
5. `PlantModel` updates photosynthesis, phenology, allocation, roots, and
   nutrient uptake.
6. `soluteModel` solves geochemical equilibria.
7. `TranspNoSalt` and optional `TranspSalt` move gases and solutes.
8. `EROSION` and `REDIST` update transported state and layer accounting.
9. Diagnostics and balance checks run.
10. History buffers, the model clock, history files, and any restart are
    updated.

Daily work such as management is dispatched before the 24-hour loop. Annual
setup reads the year-specific plant, climate, soil, and management inputs.

Useful source areas when tracing a result are:

- `f90src/HydroTherm/`: water, heat, canopy, snow, litter, and soil physics.
- `f90src/Plant_bgc/`: photosynthesis, phenology, allocation, roots, uptake,
  and plant disturbance.
- `f90src/Microbial_bgc/`: decomposers, nitrifiers, denitrifiers,
  methanogens, methanotrophs, fungi, and cyanobacteria.
- `f90src/Geochem/` and `f90src/Transport/`: equilibrium chemistry and
  aqueous/gas transport.
- `f90src/Balances/` and `f90src/ModelDiags/`: transfers, conservation, and
  diagnostics.
- `f90src/IOutils/HistData/`: the best place to confirm output definitions,
  units, aggregation flags, and the source state/flux for a history variable.

## 4. Build used for this walkthrough

No VSCode terminal profile was present, so the default shell environment was
used. The initial Apple Clang 17 configuration could not compile the bundled
zlib 1.2.12: Clang now predefines `TARGET_OS_MAC`, which activates an obsolete
zlib compatibility branch. The validated build therefore uses GCC/gfortran
16.2 in a separate directory:

```bash
cmake -S . -B build/Darwin-arm64-gcc16-double-Release \
  -DECOSIM_PRECISION=double \
  -DCMAKE_BUILD_TYPE=Release \
  -DAPPLE=1 \
  -DCMAKE_C_COMPILER=/Users/li106/local/bin/gcc-16 \
  -DCMAKE_CXX_COMPILER=/Users/li106/local/bin/g++-16 \
  -DCMAKE_Fortran_COMPILER=/Users/li106/local/bin/gfortran

cmake --build build/Darwin-arm64-gcc16-double-Release --parallel 4
cmake --build build/Darwin-arm64-gcc16-double-Release \
  --target install --parallel 4
```

The executable used here is:

```text
build/Darwin-arm64-gcc16-double-Release/local/bin/ecosim.f90.x
```

Autoconf, Automake, and Libtool are required because the repository generates
configure scripts for its bundled NetCDF-C and NetCDF-Fortran sources.

## 5. Safe scratch runs

Run simulations in a scratch mirror. The namelist paths are relative to their
working-directory depth, and the smoke workflow preserves that depth with
input symlinks.

### Startup test

```bash
env PATH="$PWD/ecosim-agent/.venv-cmip6/bin:$PATH" \
  .agents/skills/ecosim-smoke-run/scripts/run_smoke.sh \
  -d 20 \
  -b "$PWD/build/Darwin-arm64-gcc16-double-Release" \
  dryland
```

The standalone `dryland/dryland` case completed 480 hourly steps and its
history file passed the finite-value and temperature scan. The script also
lists `dryland/dryland2`; that case is intentionally not standalone because it
requires an untracked 2004 restart generated by a previous dryland run. Its
zero-step failure is not a failure of the executable.

### Crop-season lab

```bash
workdir=$(mktemp -d /private/tmp/ecosim-dryland-310.XXXXXX)

env PATH="$PWD/ecosim-agent/.venv-cmip6/bin:$PATH" \
  .agents/skills/ecosim-smoke-run/scripts/run_smoke.sh \
  -d 310 -t 1800 \
  -b "$PWD/build/Darwin-arm64-gcc16-double-Release" \
  -o "$workdir" \
  dryland
```

The validated run completed 7,440 hourly steps and wrote 310 daily samples.
Its file was:

```text
<workdir>/eco/examples/run_dir/dryland/
  dryland_maize.ecosim.h0.2001-01-01-00000.nc
```

This pass demonstrates successful input reading, initialization, management,
timestepping, output, and finite values. It is not evidence that the simulated
crop or its budgets are scientifically correct.

## 6. Reading the history file correctly

History files are written to the process working directory, not generally to
`<case>_outputs/`. Their name is:

```text
<case_name>.ecosim.h<tape>.<start-date>.nc
```

The `<case>_outputs/timing/` directory is used when runtime profiling is
enabled.

For the dryland h0 file, the important dimensions are one column, five PFT
slots, 20 soil levels, and 310 daily time records. There is no usable `time`
coordinate variable, so the notebook reconstructs daily dates from the
filename and record count.

Never aggregate from the variable name alone. Read `units`, `long_name`, and
`cell_method`, and trace the registration/update code if needed.

| Variable | Units | Interpretation in this file |
| --- | --- | --- |
| `ECO_GPP_col` | g C m-2 | Cumulative year-to-date GPP; use the last value, not the sum of daily rows |
| `ECO_NPP_col` | g C m-2 | Cumulative year-to-date NPP |
| `ECO_ET_col` | mm H2O m-2 | Cumulative year-to-date evapotranspiration |
| `tPRECIP_col` | mm m-2 | Cumulative precipitation including irrigation |
| `ECO_LAI_col` | m2 m-2 | Daily mean ecosystem LAI |
| `SHOOT_C_pft`, `Root_C_pft`, `GRAIN_C_pft` | g C m-2 | Daily mean live pools per active PFT slot |
| `AMENDED_N_col` | g N m-2 | Cumulative fertilizer N applied during the year, though its history cell method is a daily mean |
| `UPTK_NH4_FLX_pft`, `UPTK_NO3_FLX_pft` | g N m-2 h-1 | Daily-mean hourly uptake rate; multiplying by 24 estimates that day's uptake |
| `QVegTransp_pft` | mm H2O m-2 h-1 | Daily-mean hourly transpiration, negative toward the atmosphere |

This distinction between stocks, rates, interval means, and cumulative
year-to-date variables is essential for C/N/water accounting.

## 7. Run the analysis notebook

The global Python installation on this machine had incompatible NumPy and
NetCDF binary packages. An isolated environment was created at
`ecosim-agent/.venv-cmip6` with compatible packages. Recreate it with:

```bash
env UV_CACHE_DIR=/private/tmp/ecosim-uv-cache \
  uv venv ecosim-agent/.venv-cmip6 --python 3.12

env UV_CACHE_DIR=/private/tmp/ecosim-uv-cache \
  uv pip install --python ecosim-agent/.venv-cmip6/bin/python \
  numpy netCDF4 xarray pandas matplotlib jupyterlab nbclient ipykernel
```

Point the notebook at a run and execute it:

```bash
export ECOSIM_DRYLAND_RUN_DIR="<workdir>/eco/examples/run_dir/dryland"
export MPLCONFIGDIR=/private/tmp/ecosim-mpl

ecosim-agent/.venv-cmip6/bin/jupyter nbconvert \
  --to notebook --execute \
  ecosim_knowledge/dryland_maize_analysis.ipynb \
  --output /private/tmp/dryland_maize_analysis.executed.ipynb
```

The notebook also auto-detects the newest
`/private/tmp/ecosim-dryland-310.*` run when the environment variable is not
set. Its plots cover climate and water, crop C pools and production, N
amendment and uptake, phenology, transpiration, and event timing.

## 8. From this example to a new site

For a real site, do not start by copying arbitrary numbers from this example.
Start in `ecosim-agent` and use its site-run workflow:

1. Initialize the site-run tracker and perform a scoped data-readiness check.
2. Establish site ID, coordinates, elevation, vegetation, Koppen code, target
   simulation period, and benchmark observations.
3. Prepare and validate hourly meteorology and precipitation chemistry.
4. Prepare the soil/grid file, checking layer order, depths, units, missing
   values, and fallback assumptions.
5. Validate the PFT code against `f90src/Modelconfig/ElmIDMod.F90` and the
   selected PFT parameter file.
6. Create editable plant-management and, when active, soil-management
   workbooks/JSON/NetCDF. Read the generated NetCDF back.
7. Assemble `ecosim-agent/result/<SITE_ID>/input/` and `run_dir/`; all namelist
   paths should resolve to `../input/<file>`.
8. Run a short scratch test, inspect finite output, then run the intended
   spinup and evaluation period.
9. Compare output with observations using matched variables, time support,
   spatial basis, and units. Check C/N/P, water, and energy accounting before
   calibration.

The reusable site workflow is documented in
[`../ecosim-agent/.agents/skills/ameriflux-namelist-generator/references/runfolder_workflow.md`](../ecosim-agent/.agents/skills/ameriflux-namelist-generator/references/runfolder_workflow.md).

## 9. A practical learning sequence

Use the guide as five checkpoints:

1. Explain what each dryland input controls and verify its dimensions with
   `ncdump -h`.
2. Trace one simulated hour through `Run_EcoSIM_one_step` and identify which
   subsystem owns each process.
3. Run the 20-day scratch test and locate the actual history output.
4. Run and execute the 310-day crop-season notebook; explain each plot using
   the management schedule and variable metadata.
5. Choose a real site and repeat the workflow with explicit data provenance,
   validation, and benchmark observations.

The first four checkpoints establish mechanical fluency. Scientific use begins
at checkpoint five, where site data, spinup design, mass balance, and
observation comparability become part of the model experiment.
