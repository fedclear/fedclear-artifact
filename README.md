# Contract-backed BC-FL MVP

This repository contains a reproducible research artifact for a blockchain-coordinated federated learning (BC-FL) control plane. It accompanies an anonymous double-blind submission and is intended to validate the paper's implementation claims, not to serve as a production public-chain deployment.

The artifact has two execution backends:

1. **Mock backend**: a fast Python implementation of the same coordinator interface, useful for quick tests and failure sweeps.
2. **Anvil/EVM backend**: a Solidity `BCFLCoordinator` contract deployed on a local Ethereum-compatible Anvil chain. This backend produces real local EVM transaction receipts and gas measurements.

The FL workload runs on real datasets. The default is the built-in scikit-learn handwritten digits dataset. The stronger paper experiment uses an OpenML MNIST subset.

## What this artifact demonstrates

The implementation validates the control-plane claims of the paper:

- The coordinator stores only metadata, ticket state, reward/funding state, entitlements, and content identifiers.
- Model tensors and client updates are stored off chain in a SHA-256 content-addressed artifact store.
- Clients reserve tickets, train locally, publish update identifiers, and claim entitlements.
- The lister fetches retrievable artifacts, filters invalid updates, aggregates valid updates with the same FedAvg-style rule as Plain FL, writes the next global artifact, and finalizes the round on chain.
- Plain FL and contract-backed BC-FL are evaluated on the same data split and training parameters.
- The Anvil backend reports transaction count and gas used by operation.

## Important scope statement

This repository is a **contract-backed research MVP**. It is not an audited production system, public mainnet deployment, complete staking/slashing protocol, privacy-preserving FL stack, or wide-area storage system.

Production features left for future work include secure aggregation, differential privacy, real validator staking/slashing, wide-area networking, persistent replicated storage, monitoring, and contract/security audits.

## IPFS and storage note

This implementation does **not** run IPFS. It uses a local SHA-256 content-addressed artifact store. This is intentional: the protocol requires an off-chain content-addressed storage layer, not a specific IPFS deployment. IPFS, S3/MinIO with hashes, Filecoin-backed storage, or another replicated backend can be used in a production deployment if it provides:

- fetch-by-identifier,
- integrity verification,
- object-size limits,
- availability during the aggregation window,
- and operational monitoring/pinning/retention policies.

Paper-safe wording:

> The prototype uses a local SHA-256 content-addressed artifact store. IPFS is a compatible deployment option, but the reported implementation does not depend on or run IPFS.

## Repository layout

```text
contracts/BCFLCoordinator.sol      Solidity coordinator contract
bcfl_contract/store.py             SHA-256 content-addressed artifact store
bcfl_contract/fl.py                Dataset loading, non-IID split, local training, aggregation
bcfl_contract/mock_chain.py        Fast Python coordinator backend
bcfl_contract/evm_chain.py         Anvil/EVM Solidity deployment backend
bcfl_contract/experiment.py        End-to-end experiment runner
scripts/run_contract_mvp.py        Main Plain FL vs BC-FL experiment
scripts/run_failure_sweep.py       Mock-backend stress sweeps
scripts/plot_results.py            Paper-ready plots and gas summaries
requirements.txt                   Core Python dependencies
requirements-evm.txt               EVM/web3/solidity dependencies
```

## Environment

Tested on Ubuntu 24.04 with Python 3.12 and Foundry/Anvil. Other recent Linux distributions should work.

Create a Python environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
```

For Anvil/EVM runs, also install:

```bash
python3 -m pip install -r requirements-evm.txt
```

Install Foundry/Anvil if it is not already installed:

```bash
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc
foundryup
anvil --version
```

## Quick mock smoke test

```bash
source .venv/bin/activate
python3 scripts/run_contract_mvp.py --quick --out results_mock_quick
```

Expected outputs:

```text
results_mock_quick/plain_fl_rounds.csv
results_mock_quick/bcfl_mock_rounds.csv
results_mock_quick/contract_mock_receipts.csv
results_mock_quick/contract_mvp_summary.csv
results_mock_quick/paper_ready_notes.md
```

## Mock failure sweeps

The mock backend is useful for stress tests because it is fast and deterministic.

```bash
source .venv/bin/activate
python3 scripts/run_failure_sweep.py --out results_mock_sweep
python3 scripts/plot_results.py results_mock_sweep
```

This produces sweep summaries and plots under `results_mock_sweep/plots/`.

## Anvil/EVM smoke test

Start Anvil in one terminal. Use at least 20 accounts so there are enough unlocked accounts for one lister plus 10 clients.

```bash
anvil --host 127.0.0.1 --port 8545 --accounts 20 --silent
```

In a second terminal:

```bash
cd <repo-root>
source .venv/bin/activate
python3 -m pip install -r requirements.txt -r requirements-evm.txt
python3 scripts/run_contract_mvp.py --backend anvil --quick --out results_anvil_quick
```

The first Anvil run may download Solidity compiler 0.8.24 through `py-solc-x`.

## Main MNIST experiment used in the paper

The main paper result uses an OpenML MNIST subset with 20,000 examples, 10 clients, a non-IID Dirichlet split, 12 FL rounds, and a local Anvil/EVM backend.

Terminal 1:

```bash
anvil --host 127.0.0.1 --port 8545 --accounts 20 --silent
```

Terminal 2:

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

Zip only the lightweight paper-relevant files, not the model artifacts:

```bash
zip -r mnist_20k_paper_results.zip \
  results_anvil_mnist_20k_12r/*.csv \
  results_anvil_mnist_20k_12r/*.md \
  results_anvil_mnist_20k_12r/plots
```

## Expected main-result numbers

With the configuration above, one representative run produced:

| Metric | Value |
|---|---:|
| Backend | Anvil local EVM |
| Dataset | MNIST, 20,000 samples |
| Clients | 10 |
| Rounds | 12 |
| Plain FL test accuracy | 0.8930 |
| Contract-backed BC-FL test accuracy | 0.8930 |
| Accuracy gap vs Plain FL | 0.0000 |
| On-chain transactions, including deployment | 276 |
| Total gas used | 41,818,892 |
| Artifact puts / gets | 133 / 120 |
| Storage get failures | 0 |

Gas breakdown from the representative run:

| Operation | Count | Mean gas | Total gas |
|---|---:|---:|---:|
| Deploy | 1 | 1,391,031 | 1,391,031 |
| createListing | 1 | 210,815 | 210,815 |
| fundRound | 12 | 48,356 | 580,272 |
| reserveTicket | 120 | 130,943 | 15,713,120 |
| publishUpdate | 120 | 170,289 | 20,434,692 |
| finalizeRound | 12 | 264,961 | 3,179,532 |
| claim | 10 | 30,943 | 309,430 |

The exact transaction hashes and block numbers may differ between runs. Accuracy should be reproducible for the same software versions, seed, dataset subset, and command-line parameters.

## Optional full MNIST run

Use `--max-samples 0` for the full OpenML MNIST dataset. This is slower and produces larger artifacts.

```bash
python3 scripts/run_contract_mvp.py \
  --backend anvil \
  --dataset mnist \
  --max-samples 0 \
  --rounds 12 \
  --out results_anvil_mnist_full_12r
```

When `--max-samples` is not zero, describe the experiment as an MNIST subset.

## Fashion-MNIST

Fashion-MNIST is supported if OpenML is reachable:

```bash
python3 scripts/run_contract_mvp.py \
  --backend mock \
  --dataset fashion_mnist \
  --max-samples 12000 \
  --quick \
  --out results_fashion_quick
```

## Reproducibility checklist for anonymous review

Before uploading an anonymous repository:

- Do not use a GitHub account, commit metadata, path, README text, or issue links that identify the authors.
- Do not include local absolute paths containing author names.
- Do not include private keys other than public Anvil development keys shown by Anvil itself.
- Include this README, the contract, scripts, requirements, and a small set of paper-relevant CSVs.
- Avoid uploading the full `artifact_store/` unless the reviewer specifically needs model artifacts. The CSVs and scripts are enough to reproduce them.
- State clearly that the implementation uses local content-addressed storage, not IPFS.
- During double-blind review, refer to the artifact as an anonymous artifact repository. Reveal author-identifying metadata only after review.

## Paper-safe implementation paragraph

> We implemented a contract-backed BC-FL prototype with two backends. The mock backend executes the coordinator interface in Python and is used for fast failure sweeps. The contract backend deploys a Solidity `BCFLCoordinator` contract on a local Anvil Ethereum-compatible chain and records real transaction receipts and gas usage. In both backends, model tensors remain off chain in a SHA-256 content-addressed artifact store, while the coordinator stores listings, tickets, update identifiers, global identifiers, reward pools, refunds, and claims. On a non-IID 20,000-sample MNIST split with 10 clients and 12 rounds, the contract-backed run matched the Plain FL baseline at 89.30% test accuracy, issued 276 transactions, and consumed 41.8M gas.

## Troubleshooting

### `anvil: command not found`

Install Foundry:

```bash
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc
foundryup
```

### `Need at least 11 unlocked Anvil accounts`

Restart Anvil with more accounts:

```bash
anvil --host 127.0.0.1 --port 8545 --accounts 20 --silent
```

### `Stack too deep` during Solidity compilation

The EVM backend compiles with optimizer and `viaIR: true`. If you still see this error, check `bcfl_contract/evm_chain.py` and make sure the compiler settings include:

```python
"optimizer": {"enabled": True, "runs": 200},
"viaIR": True,
```

### `ModuleNotFoundError: No module named 'numpy'`

You are probably in a new folder without an activated virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt -r requirements-evm.txt
```

### `KeyError: 'op'` when summarizing gas

The receipts CSV uses column name `event`, not `op`:

```python
import pandas as pd
df = pd.read_csv("results_anvil_mnist_20k_12r/contract_anvil_receipts.csv")
print(df.groupby("event")["gas_used"].agg(["count", "mean", "sum"]))
```

## Citation and artifact policy

If this repository is linked during double-blind review, use an anonymous repository or artifact service. Do not link a personal GitHub account if the venue requires double-blind review.
