# FedClear: Contract-Backed Clearing Layer for Auditable Federated Learning

FedClear is a reproducible research prototype for blockchain-coordinated federated learning. It implements a contract-backed control plane where clients reserve update tickets, publish update identifiers, and claim entitlements, while model tensors remain off chain in content-addressed storage.

The goal of this repository is to validate the implementation claims of the accompanying paper:

- a Solidity coordinator contract for FL round coordination,
- a local Ethereum-compatible execution backend using Anvil,
- a fast Python mock backend for stress sweeps,
- real FL workloads on handwritten-digit datasets,
- SHA-256 content-addressed off-chain artifact storage,
- transaction/gas measurements and paper-ready CSV outputs.

This is a research artifact, not a production public-chain deployment.

## Overview

FedClear separates FL coordination metadata from model artifacts.

The coordinator stores:

- model listings,
- round funding state,
- ticket reservations,
- update identifiers,
- global model identifiers,
- refund and claim state.

The artifact store holds:

- initial models,
- client update tensors,
- finalized global models.

Artifacts are addressed locally as:

```text
sha256:<digest>
```

where `<digest>` is the SHA-256 hash of the serialized artifact bytes. On retrieval, the store recomputes the digest and rejects mismatches.

## Backends

The artifact supports two execution backends.

### Mock backend

The mock backend executes the same coordinator workflow in Python. It is useful for fast tests, debugging, and failure sweeps.

### Anvil/EVM backend

The Anvil backend deploys the Solidity `BCFLCoordinator` contract on a local Ethereum-compatible Anvil chain. It produces real local EVM transaction receipts and gas measurements.

## Repository layout

```text
contracts/BCFLCoordinator.sol      Solidity coordinator contract
bcfl_contract/store.py             SHA-256 content-addressed artifact store
bcfl_contract/fl.py                Dataset loading, non-IID split, local training, aggregation
bcfl_contract/mock_chain.py        Python coordinator backend
bcfl_contract/evm_chain.py         Anvil/EVM deployment backend
bcfl_contract/experiment.py        End-to-end experiment runner
scripts/run_contract_mvp.py        Main Plain FL vs FedClear experiment
scripts/run_failure_sweep.py       Failure/stress sweep runner
scripts/plot_results.py            Plot and gas-summary generator
requirements.txt                   Core Python dependencies
requirements-evm.txt               EVM/web3/Solidity dependencies
```

## Requirements

Tested with:

```text
Ubuntu 24.04
Python 3.12
Foundry/Anvil
Solidity 0.8.24
```

Install Python dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate

python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
```

For Anvil/EVM experiments, also install:

```bash
python3 -m pip install -r requirements-evm.txt
```

Install Foundry/Anvil:

```bash
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc
foundryup

anvil --version
```

## Quick mock run

Run a small mock-backend smoke test:

```bash
source .venv/bin/activate

python3 scripts/run_contract_mvp.py \
  --quick \
  --out results_mock_quick
```

Expected outputs:

```text
results_mock_quick/plain_fl_rounds.csv
results_mock_quick/bcfl_mock_rounds.csv
results_mock_quick/contract_mock_receipts.csv
results_mock_quick/contract_mvp_summary.csv
results_mock_quick/paper_ready_notes.md
```

## Anvil/EVM smoke test

Start Anvil in one terminal:

```bash
anvil --host 127.0.0.1 --port 8545 --accounts 20 --silent
```

In another terminal:

```bash
source .venv/bin/activate

python3 -m pip install -r requirements.txt -r requirements-evm.txt

python3 scripts/run_contract_mvp.py \
  --backend anvil \
  --quick \
  --out results_anvil_quick
```

The first Anvil run may download Solidity compiler 0.8.24 through `py-solc-x`.

## Main MNIST experiment

This is the main contract-backed experiment.

It runs:

- MNIST from OpenML,
- 20,000 samples,
- 10 clients,
- non-IID Dirichlet split,
- 12 FL rounds,
- local Anvil/EVM backend.

Start Anvil:

```bash
anvil --host 127.0.0.1 --port 8545 --accounts 20 --silent
```

Run the experiment:

```bash
source .venv/bin/activate

python3 scripts/run_contract_mvp.py \
  --backend anvil \
  --dataset mnist \
  --max-samples 20000 \
  --rounds 12 \
  --out results_anvil_mnist_20k_12r
```

Generate plots and gas summaries:

```bash
python3 scripts/plot_results.py results_anvil_mnist_20k_12r
```

Useful outputs:

```text
results_anvil_mnist_20k_12r/contract_mvp_summary.csv
results_anvil_mnist_20k_12r/contract_anvil_receipts.csv
results_anvil_mnist_20k_12r/plain_fl_rounds.csv
results_anvil_mnist_20k_12r/bcfl_anvil_rounds.csv
results_anvil_mnist_20k_12r/paper_ready_notes.md
results_anvil_mnist_20k_12r/plots/
```

## Representative MNIST result

One representative run with the command above produced:

| Metric | Value |
|---|---:|
| Backend | Anvil local EVM |
| Dataset | MNIST |
| Samples | 20,000 |
| Clients | 10 |
| Rounds | 12 |
| Plain FL test accuracy | 0.8930 |
| FedClear test accuracy | 0.8930 |
| Accuracy gap vs Plain FL | 0.0000 |
| Transactions, including deployment | 276 |
| Total gas used | 41,818,892 |
| Artifact puts / gets | 133 / 120 |
| Storage get failures | 0 |

Gas breakdown:

| Operation | Count | Mean gas | Total gas |
|---|---:|---:|---:|
| Deploy | 1 | 1,391,031 | 1,391,031 |
| createListing | 1 | 210,815 | 210,815 |
| fundRound | 12 | 48,356 | 580,272 |
| reserveTicket | 120 | 130,943 | 15,713,120 |
| publishUpdate | 120 | 170,289 | 20,434,692 |
| finalizeRound | 12 | 264,961 | 3,179,532 |
| claim | 10 | 30,943 | 309,430 |

Transaction hashes and block numbers may differ across runs. Accuracy should be reproducible under the same software versions, seed, dataset subset, and command-line parameters.

## Full MNIST mock run

The full OpenML MNIST dataset can be run with the mock backend:

```bash
python3 scripts/run_contract_mvp.py \
  --backend mock \
  --dataset mnist \
  --max-samples 0 \
  --rounds 12 \
  --out results_mock_mnist_full_12r
```

Use `--max-samples 0` for the full dataset. When `--max-samples` is nonzero, the run uses a subset.

## Failure sweeps

Run the MNIST 20k failure sweep:

```bash
python3 scripts/run_failure_sweep.py \
  --dataset mnist \
  --max-samples 20000 \
  --out results_mock_mnist_20k_failure_sweep
```

This evaluates controlled stress scenarios such as:

- artifact retrieval failures,
- client dropout,
- invalid updates.

Inspect the output:

```bash
cat results_mock_mnist_20k_failure_sweep/*.csv
```

## Fashion-MNIST

Fashion-MNIST is supported when OpenML is reachable:

```bash
python3 scripts/run_contract_mvp.py \
  --backend mock \
  --dataset fashion_mnist \
  --max-samples 12000 \
  --rounds 12 \
  --out results_mock_fashion_12k_12r
```

## Packaging lightweight results

To zip the paper-relevant outputs without large model artifacts:

```bash
zip -r mnist_20k_paper_results.zip \
  results_anvil_mnist_20k_12r/*.csv \
  results_anvil_mnist_20k_12r/*.md \
  results_anvil_mnist_20k_12r/plots
```

The generated `artifact_store/` directories can be reproduced by rerunning the experiments and are usually not needed for lightweight artifact review.

## Storage note

This implementation does not run IPFS. It uses a local SHA-256 content-addressed artifact store.

The protocol only requires an off-chain artifact layer that supports:

- fetch-by-identifier,
- integrity verification,
- object-size limits,
- availability during the aggregation window.

IPFS, S3/MinIO with hashes, Filecoin-backed storage, or another replicated backend could be used in a production deployment, but the reported experiments use the local SHA-256 backend.

## Common issues

### `anvil: command not found`

Install Foundry:

```bash
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc
foundryup
```

### `Need at least 11 unlocked Anvil accounts`

Start Anvil with more accounts:

```bash
anvil --host 127.0.0.1 --port 8545 --accounts 20 --silent
```

### Solidity `Stack too deep`

The EVM backend compiles with optimizer and `viaIR: true`. The expected compiler settings are in `bcfl_contract/evm_chain.py`:

```python
"optimizer": {"enabled": True, "runs": 200},
"viaIR": True,
```

### `ModuleNotFoundError`

Create and activate the virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt -r requirements-evm.txt
```

### Gas summary column name

The receipts CSV uses `event`, not `op`:

```python
import pandas as pd

df = pd.read_csv("results_anvil_mnist_20k_12r/contract_anvil_receipts.csv")
print(df.groupby("event")["gas_used"].agg(["count", "mean", "sum"]))
```

