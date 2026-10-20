# Changelog

## [1.0.0] – 2026-XX-XX
First public release. Changes are relative to the CAESB version of WaterPaths in Araújo et al. (2024) ([lbl59/federal_district](https://github.com/lbl59/federal_district), doi:10.5281/zenodo.8221350).

### Added (9 files)
- `src/DataCollector/`: `EmergencyTransferParanoaDataCollector.cpp`, `EmergencyTransferParanoaDataCollector.h`, `WSSDataCollector.cpp`, `WSSDataCollector.h`
- `src/DroughtMitigationInstruments/`: `EmergencyTransferParanoa.cpp`, `EmergencyTransferParanoa.h`
- `src/SystemComponents/Utility/`: `WaterSupplySystems.cpp`, `WaterSupplySystems.h`
- `src/Utils/`: `Constants.cpp`

### Modified (72 files)
- `src/`: `main.cpp`
- `src/ContinuityModels/`: `ContinuityModelROF.cpp`, `ContinuityModelROF.h`, `ContinuityModelRealization.cpp`, `ContinuityModelRealization.h`
- `src/ContinuityModels/Base/`: `ContinuityModel.cpp`, `ContinuityModel.h`
- `src/Controls/`: `WwtpDischargeRule.cpp`, `WwtpDischargeRule.h`
- `src/Controls/Base/`: `MinEnvFlowControl.cpp`, `MinEnvFlowControl.h`, `MinEnvironFlowControl.cpp`, `MinEnvironFlowControl.h`
- `src/DataCollector/`: `AllocatedReservoirDataCollector.cpp`, `AllocatedReservoirDataCollector.h`, `MasterDataCollector.cpp`, `MasterDataCollector.h`, `TransfersBilateralDataCollector.cpp`, `TransfersDataCollector.cpp`, `UtilitiesDataCollector.cpp`, `UtilitiesDataCollector.h`
- `src/DroughtMitigationInstruments/`: `InsuranceStorageToROF.cpp`, `InsuranceStorageToROF.h`, `Restrictions.cpp`, `Restrictions.h`, `Transfers.cpp`, `Transfers.h`, `TransfersBilateral.cpp`, `TransfersBilateral.h`
- `src/DroughtMitigationInstruments/Base/`: `DroughtMitigationPolicy.cpp`, `DroughtMitigationPolicy.h`
- `src/Problem/`: `Caesb.cpp`, `Caesb.h`
- `src/Problem/Base/`: `Problem.cpp`, `Problem.h`
- `src/Simulation/`: `Simulation.cpp`, `Simulation.h`
- `src/SystemComponents/`: `Catchment.cpp`
- `src/SystemComponents/Bonds/`: `BalloonPaymentBond.cpp`, `BalloonPaymentBond.h`, `FloatingInterestBalloonPaymentBond.cpp`, `FloatingInterestBalloonPaymentBond.h`, `LevelDebtServiceBond.cpp`, `LevelDebtServiceBond.h`
- `src/SystemComponents/Bonds/Base/`: `Bond.cpp`, `Bond.h`
- `src/SystemComponents/Utility/`: `InfrastructureManager.cpp`, `InfrastructureManager.h`, `Utility.cpp`, `Utility.h`
- `src/SystemComponents/WaterSources/`: `AllocatedReservoir.cpp`, `AllocatedReservoir.h`, `Intake.cpp`, `Quarry.cpp`, `Relocation.cpp`, `Relocation.h`, `Reservoir.cpp`, `Reservoir.h`, `ReservoirExpansion.cpp`, `ReservoirExpansion.h`, `SequentialJointTreatmentExpansion.cpp`, `SequentialJointTreatmentExpansion.h`
- `src/SystemComponents/WaterSources/Base/`: `WaterSource.cpp`, `WaterSource.h`
- `src/Utils/`: `Constants.h`, `Matrices.h`, `ObjectivesCalculator.cpp`, `ObjectivesCalculator.h`, `Utils.cpp`, `Utils.h`
- `src/Utils/Graph/`: `Graph.cpp`
- `src/Utils/QPSolver/`: `QuadProg++.cpp`

### Removed
- none

### Summary
- Multi-subsystem `Utility` → `WaterSupplySystems` architecture (per-WSS demands, tariffs, ROF, infrastructure, finances).
- Per-WSS ROF tables (`tables_r{R}_wss{W}.csv`).
- WSS-level restrictions with tariff surcharges, bilateral transfers with sender demand protection, and the new `EmergencyTransferParanoa`.
- New infrastructure option: Santa Maria WTP expansion (ID 12).
- New objectives (WSS reliability, affordability, severity) and experiment modes 1–8 (`-E`, `-w`, `-V`).
- `NUM_DEC_VAR` 19 → 22; `NUM_OBJECTIVES` 5 → 7.
- New data collectors and outputs; per-realization deep copies for thread safety; Borg failure handling; portable CLI parsing; CMake/OpenMP build.
