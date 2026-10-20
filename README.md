# WaterPaths-WSS: multi-subsystem WaterPaths for the Federal District of Brazil (CAESB)

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.XXXXXXX.svg)](https://doi.org/10.5281/zenodo.XXXXXXX)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

**WaterPaths-WSS** is a modified version of [WaterPaths](https://github.com/bernardoct/WaterPaths) (Trindade et al., 2020), an open-source C++ stochastic simulation and many-objective optimization framework for regional water supply planning under deep uncertainty. This version adds an explicit **Water Supply System (WSS)** layer, so that **one utility can operate several hydraulically distinct subsystems**. Each subsystem has its own demands, sources, treatment plants, risk-of-failure (ROF) triggers, drought responses, infrastructure pathway, finances and performance objectives, while the utility still pools debt, contingency funds and shared infrastructure.

The model is configured for the **CAESB** (Companhia de Saneamento Ambiental do Distrito Federal) system in Brasília, split into two subsystems:

| WSS ID | Subsystem | Sources (WaterPaths IDs) |
|---|---|---|
| 0 | **Descoberto** | Descoberto Reservoir (0), Corumbá IV (2, allocated), *candidates:* Corumbá WTP stage 1 (5), stage 2 (6), Descoberto storage expansion (10) |
| 1 | **Torto / Santa Maria (TortoSM)** | Santa Maria / Torto Reservoir (1), Bananal & Torto intakes (4), Lake Paranoá (3, allocated, emergency use), *candidates:* Paranoá WTP stages 1–3 (7, 8, 9), Santa Maria WTP expansion (12) |

> **Original work.** WaterPaths was developed by Bernardo C. Trindade and co-authors (Cornell University / UNC Chapel Hill). The CAESB application was built by A. P. Giacomazzo (2020) and B. M. Araújo et al. (2024) ([lbl59/federal_district](https://github.com/lbl59/federal_district), [10.5281/zenodo.8221350](https://doi.org/10.5281/zenodo.8221350)). In that version the two service areas were modelled as two separate `Utility` objects. **This repository does not claim authorship of WaterPaths or of the original CAESB model**; it documents and releases the modifications listed below.

---

## What is new compared with the original model

**1. Multi-subsystem utility architecture**
- New class `WaterSupplySystems` (`src/SystemComponents/Utility/`). A `Utility` now owns any number of WSS, added through `Utility::addWaterSupplySystem(...)` and accessed with `systemById(id)`.
- Each WSS has its own demand series, demand-class fractions, tariff table, wastewater return rule, source-to-WTP mapping, owned treatment capacities, demand buffer and per-source failure thresholds.
- Each WSS also holds its own `InfrastructureManager`, with an ROF-triggered construction order and triggers. When one WSS triggers shared infrastructure, construction is forced in the other WSS (`forceInfrastructureConstruction`).
- Financial accounting is calculated by the utility and stored per WSS: gross revenue, drought-mitigation cost, contingency-fund share, debt-service share, infrastructure NPC and residential price. Average monthly income and number of households are also stored per WSS, for affordability.
- Each realization owns deep copies of its WSS objects, including the price series, which removes data races between OpenMP threads.

**2. Risk-of-failure and continuity models per WSS**
- `ContinuityModelROF` and `ContinuityModelRealization` compute short- and long-term ROF for each WSS through a reservoir-to-WSS connectivity matrix.
- Pre-computed ROF tables are stored per realization and subsystem (`rof_tables/tables_r{R}_wss{W}.csv`).

**3. Drought-mitigation instruments at WSS level**
- `Restrictions`: three-stage demand restrictions for each WSS (multipliers 0.98 / 0.96 / 0.90), with stage-dependent **tariff surcharges** (`caesbPriceRestrictionMultipliers.csv`). The stage-3 residential multiplier is a decision variable.
- `TransfersBilateral`: treated-water transfers between WSS (TortoSM → Descoberto, 1.0 m³/s), with a **sender demand-protection factor** (decision variable) that reserves part of the sender's demand before any water is transferred.
- **New** `EmergencyTransferParanoa`: ROF-triggered emergency supply from Lake Paranoá to TortoSM. The pipe capacity is 0.1 m³/s, plus 0.1 m³/s for each Paranoá WTP expansion that is online, and is limited by TortoSM's allocated share of the lake (0.8 %). Paranoá infrastructure can only be built after the emergency transfer has fired once. Transfer cost = receiver tariff × 1.1.

**4. New objectives and experiment modes**
- WSS-level reliability: the share of weeks/years in which no source falls below 20 % of its storage capacity (the threshold can be set per source).
- **Affordability index**: weekly residential water bill divided by average household income, computed separately for each WSS (95th percentile across realizations of the worst week in each realization).
- Optional **failure-severity** objective, plus WSS-level restriction frequency and infrastructure NPC.
- Experiment modes, selected with `-E` (see [Objectives](#objectives-and-experiment-modes)): worst-case (minimax) or average aggregation, per-WSS objectives, single-WSS optimization, penalty-based formulation and disparity (equity-gap) formulation.

**5. Output and robustness**
- New data collectors: `WSSDataCollector` (weekly time series and pathways for each WSS) and `EmergencyTransferParanoaDataCollector`.
- New CSV outputs: annual reliability per WSS and per source.
- When a Borg evaluation fails, the decision variables are written to `eval_failure_rank_N.csv` and the objectives are set to 1e5, so the run continues instead of aborting.
- Portable argument parsing (no `getopt`) and a CMake build that finds OpenMP.
- The number of decision variables rises from 19 to **22**; up to **7** objectives are supported.

---

## Decision variables (22)

| # | Variable | # | Variable |
|---|---|---|---|
| 0 | Restriction trigger – Descoberto (ROF) | 11 | Ranking – Paranoá Sul WTP stage 2 (ID 8) |
| 1 | Restriction trigger – TortoSM (ROF) | 12 | Ranking – Paranoá Sul/Norte WTP stage 3 (ID 9) |
| 2 | Δ restriction trigger between stages – Descoberto | 13 | Ranking – Corumbá WTP stage 1 (ID 5) |
| 3 | Δ restriction trigger between stages – TortoSM | 14 | Ranking – Corumbá WTP stage 2 (ID 6) |
| 4 | Transfer trigger – Descoberto (ROF) | 15 | Ranking – Descoberto storage expansion (ID 10) |
| 5 | Transfer trigger – TortoSM (ROF) | 16 | Infrastructure buffer – Descoberto |
| 6 | Annual contingency-fund payment – Descoberto (*fixed at 0*) | 17 | Infrastructure buffer – TortoSM |
| 7 | Annual contingency-fund payment – TortoSM (*fixed at 0*) | 18 | Paranoá emergency-transfer / Paranoá-infrastructure trigger (ROF) |
| 8 | Infrastructure trigger – Descoberto (long-term ROF) | 19 | Sender demand-protection factor (0–2) |
| 9 | Infrastructure trigger – TortoSM (long-term ROF) | 20 | Ranking – Santa Maria WTP expansion (ID 12) |
| 10 | Ranking – Paranoá Sul WTP stage 1 (ID 7) | 21 | Stage-3 residential tariff multiplier |

## Objectives and experiment modes

| `-E` | No. of objectives | Formulation |
|---|---|---|
| 1 | 4 | Reliability (min over WSS), restriction frequency, infrastructure NPC, worst-case cost |
| 2 | 4 | As 1, reliability averaged over WSS |
| **3** (default) | 5 | As 1 + affordability (worst WSS) |
| 4 | 5 | As 3, reliability and affordability averaged |
| 5 | 7 | Reliability and affordability reported separately for each WSS |
| 6 | 5 | Single-WSS optimization (`-w <WSS id>`) |
| 7 | 5 | Penalty: Σ max(0.98 − rel, 0)² and Σ max(afford − 0.03, 0)² over WSS |
| 8 | 5 | Disparity: normalized gaps in reliability and affordability between WSS 0 and WSS 1 |

Add `-V 1` to modes 1–4 to include the failure-severity objective.

---

## Requirements

- C++14 compiler with OpenMP (GCC ≥ 7 tested; Intel `icc` targets are also provided)
- MPI (e.g. OpenMPI) for parallel optimization
- **Borg MOEA** (multi-master version, `libborgms.a`), only for optimization. Borg is **not distributed with this repository**: request it from the authors at <http://borgmoea.org>.


## Build

```bash
make gcc          # serial / OpenMP simulation
make gcc-debug    # debug build
make borg         # MPI + Borg multi-master optimization (needs lib/libborgms.a)
# or: mkdir build && cd build && cmake .. && make
```

The executable is `FDBsimulation`.

## Run

**1. Generate ROF tables (run once):**
```bash
./FDBsimulation -T 16 -t 2086 -r 999 -d ./ -C 1 -O rof_tables/ \
  -U InputFiles/utilities_rdm.csv -W InputFiles/water_sources_rdm.csv \
  -P InputFiles/policies_rdm.csv -m 0 -s FBD_DVs.csv
```

**2. Simulate one policy from a decision-variable file:**
```bash
./FDBsimulation -T 16 -t 2086 -r 999 -d ./ -C -1 -O rof_tables/ \
  -U InputFiles/utilities_rdm.csv -W InputFiles/water_sources_rdm.csv \
  -P InputFiles/policies_rdm.csv -s FDB_DVs.csv -m 0 -E 3 -p 1
```

**3. Optimize with Borg (SLURM example: `borg_caesb.sh`):**
```bash
mpirun -np 252 ./FDBsimulation -T 1 -t 2086 -r 999 -d ./ -C -1 -O rof_tables/ -b true \
  -U InputFiles/utilities_rdm.csv -W InputFiles/water_sources_rdm.csv \
  -P InputFiles/policies_rdm.csv -e 0 -o 1000 -n 50000 -E 6 -w 1
```


Run `./FDBsimulation -?` for the full list of options. The most important ones are: `-T` threads, `-r` realizations, `-t` weeks, `-d` data directory, `-C` ROF tables (1 export / 0 none / −1 import), `-O` ROF-table directory, `-s` solutions file, `-m` / `-f` / `-l` solution index or range, `-b` run Borg, `-n` NFE, `-e` seed, `-E` experiment mode, `-w` target WSS, `-V` severity objective, `-U/-W/-P` RDM files.

---

## Repository structure

```
src/
  Problem/Caesb.cpp            CAESB two-subsystem problem definition (sources, WSS, policies, DVs)
  SystemComponents/Utility/    Utility, WaterSupplySystems (new), InfrastructureManager
  SystemComponents/WaterSources/  reservoirs, allocated reservoirs, intakes, expansions
  DroughtMitigationInstruments/   Restrictions, TransfersBilateral, EmergencyTransferParanoa (new)
  ContinuityModels/            ROF and realization continuity models (per WSS)
  DataCollector/               output collectors incl. WSSDataCollector (new)
  Utils/                       objectives, constants / experiment modes, QP solver, graph
InputFiles/
  inflows/ evaporation/ demands/   999 synthetic weekly realizations (see Data)
  *_rdm.csv, *_rdm_reeval.csv      deep-uncertainty (DU) factor samples for optimization / re-evaluation
  caesb*DemandClassesFractions.csv, caesb*UserClassesWaterPrices.csv, caesbPriceRestrictionMultipliers.csv
  demand_to_wastewater_fraction_caesb_*.csv
FDB_DVs.csv                    example decision-variable vector (22 values)
Makefile, CMakeLists.txt       build files
```

## Data

| Input | Size | Notes |
|---|---|---|
| Inflows (Descoberto, Santa Maria/Torto, Bananal/Torto, Paranoá, Corumbá IV) | 999 realizations × 4,685 weeks | weekly, hm³ |
| Evaporation (4 reservoirs) | 999 × 4,685 | weekly |
| Demands (Descoberto, TortoSM) | 999 × 2,086 weeks (~40 yr) | weekly, hm³ |
| DU factors | 999 samples × (5 utility + 29 source + 2 policy) factors | multipliers |
| Tariffs / demand classes | 12 months × 5 user classes, per WSS | R$/m³, fractions |

The inputs and (example) pre-computed ROF tables are archived separately on Zenodo: **https://doi.org/10.5281/zenodo.23492993**. Download them and unpack into `InputFiles/` and `rof_tables/`.

## Outputs

The output files are written to `output/`: `Utilities_s*`, `WSS_s*`, `WaterSources_s*`, `Policies_s*` (compact weekly time series), `Pathways_s*` (infrastructure construction per realization), `annualWSS_s*` / `annualSource_s*` (reliability), `Objectives_s*` (objective values), and Borg `.set` / `.runtime` files (`NC_output_MS_E{mode}_S{seed}_N{nfe}.set`).

---

## How to cite

If you use this software, please cite **both** this release and the original works:

- Mohamed, A. (2026). *WaterPaths-WSS: multi-subsystem WaterPaths for the Federal District of Brazil* (v1.0.0) [Software]. Zenodo. https://doi.org/10.5281/zenodo.XXXXXXX
- Trindade, B. C., Gold, D. F., Reed, P. M., Zeff, H. B., & Characklis, G. W. (2020). Water pathways: An open source stochastic simulation system for integrated water supply portfolio management and infrastructure investment planning. *Environmental Modelling & Software*, 132, 104772. https://doi.org/10.1016/j.envsoft.2020.104772
- Araújo, B. M., Gold, D. F., Lau, L. B., Reed, P. M., & Alves, C. M. A. (2024). Exploring equity challenges within deeply uncertain water supply investment pathways in the Federal District of Brazil. *Journal of Water Resources Planning and Management*, 150(8), 04024030. https://doi.org/10.1061/JWRMD5.WRENG-6353

See also `CITATION.cff`.

## License

Released under the **Apache License 2.0**, the same license as the original WaterPaths (see `LICENSE` and `NOTICE`). 

## Acknowledgements

Original WaterPaths: B. C. Trindade, D. F. Gold, P. M. Reed, H. B. Zeff, G. W. Characklis. Original CAESB model: A. P. Giacomazzo, B. M. Araújo, D. F. Gold, L. B. Lau, P. M. Reed, C. M. A. Alves. Borg MOEA: D. Hadka & P. Reed. [Add funding / HPC acknowledgement, e.g. SURF Snellius grant no.]

## Contact

Aseel Mohamed – Utrecht University – https://orcid.org/0009-0008-7735-6343 – open an issue on GitHub.
