# Secure command execution for UAV swarms: code and experimental data

This repository contains the Go prototype, the benchmarks, and the experimental data of a framework for secure command execution in UAV swarms. The base station authorizes each command with an Ed25519 signature, a rotating temporary leader collects threshold-BLS approvals from the swarm, and each UAV executes a command only after verifying the resulting certificate. Signing keys come from GJKR distributed key generation without a trusted dealer.

## Contents

| File | Content |
|---|---|
| `Experimental_results.zip` | Results of the original campaigns: application-layer emulation, Linux netem, ns-3, PX4/Gazebo, and the first baseline comparison |
| `source_code.zip` | `reviewer_experiments/protocol_go/`: the Go prototype (module `uav-bft`), with protocol packages in `internal/` and programs in `cmd/`; `reviewer_experiments/scripts/`: parsing of key-generation logs; `experiment_harness/`: harness that builds the tools, runs the repeated campaign, and analyzes it (see its README) |
| `repeated_campaign_results.zip` | `experiment_results/campaign_20261121_133412/`: results of the repeated campaign, i.e., 1,350 baseline trials, 25 key-generation runs, 120 loss-margin runs, and 30 runs of the original baseline computation, with every run's command and log |

## Main programs

| Program | Purpose |
|---|---|
| `cmd/protocolbench` | Application-layer emulator with ideal, WiFi-light, and WiFi-stressed profiles (simulated time, deterministic for a given seed) |
| `cmd/revisionprotocolbench` | `protocolbench` plus `-fault-bound`, `-loss-margin` (threshold t = n − f − u), and `-seed-start`; identical output with default options |
| `cmd/dkgdemo` | GJKR key generation and threshold-BLS signing, recovery, and verification |
| `cmd/publishedbaselinebench` | Original common-platform comparison with FHotStuff/FROST and authenticated PBFT (unchanged) |
| `cmd/revisionbaselinebench` | The same comparison with one method and trial per process, critical-path latency, and `-mode fair` or `-mode legacy` |
| `cmd/bs`, `cmd/uavnode` | Base station and UAV node executables for real UDP execution |
| `cmd/px4gate`, `cmd/px4faultgate` | Gate between certified commands and PX4 |
| `cmd/trustbench` | Sensitivity of the availability score to its cutoff |

Unzip `source_code.zip` and build with Go 1.26 or newer:

```bash
cd reviewer_experiments/protocol_go
go build -o bin/ ./cmd/...
```

To repeat the campaign, see `experiment_harness/README.md`. It takes about 2.5 hours on the machine below.

## Repeated campaign: main results

Machine: Intel Core i9-13900HX (32 threads), 16 GB RAM, WSL2 with Ubuntu 22.04, Go 1.26.5, Kyber v4.0.2. All 1,525 runs succeeded. Full numbers are in `experiment_results/campaign_20261121_133412/analysis/REPORT.md` inside `repeated_campaign_results.zip`.

Per-command computation and communication for 100 UAVs in normal operation (fair mode, mean ± SD over 30 trials):

| Method | Total computation (ms) | Critical-path computation (ms) | Deliveries |
|---|---|---|---|
| Proposed (threshold BLS) | 77.85 ± 3.93 | 63.83 ± 3.22 | 265 |
| FHotStuff with FROST | 50.89 ± 1.97 | 42.88 ± 1.71 | 498 |
| Authenticated PBFT | 540.16 ± 16.73 | 7.72 ± 1.31 | 19,899 |

`publishedbaselinebench` recomputes each signer's public key share by polynomial evaluation in every share verification, verifies every partial signature twice during threshold-BLS recovery, and generates FROST nonces inside the timed interval. Rerun in legacy mode, it reproduces the earlier values: 402.28 ± 8.22 ms (proposed), 1145.52 ± 20.74 ms (FHotStuff with FROST), and 540.92 ± 14.97 ms (PBFT). Fair mode applies the same standard optimizations to both threshold schemes; the table above uses it. Deliveries come from a per-message model, not from network measurements.

Certification success with 100 UAVs, f silent Byzantine followers, and threshold t = n − f − u (ten seeds of 50 rounds each):

| f | u | t | WiFi-light | WiFi-stressed |
|---|---|---|---|---|
| 33 | 0 | 67 | 25.6% | 0.0% |
| 30 | 4 | 66 | 100.0% | 0.2% |
| 25 | 12 | 63 | 100.0% | 94.6% |

GJKR key generation, all parties in one process (mean ± SD over five runs): 0.91 ± 0.01 s for 10 UAVs, 6.32 ± 0.18 s for 20, 87.76 ± 0.96 s for 50, and 673.64 ± 6.41 s for 100.
