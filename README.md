# byz-fed-ids-5g
Byzantine-Resilient Federated IDS for Real-Time 5G IoT via Blockchain Governance and IPFS
# Byzantine-Resilient Federated Learning IDS for 5G Networks
### Hyperledger Fabric + IPFS + Multi-Krum | IEEE TDSC Submission

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.9+-blue.svg)](https://www.python.org/)
[![Fabric](https://img.shields.io/badge/Hyperledger%20Fabric-2.4-orange.svg)](https://hyperledger-fabric.readthedocs.io/)
[![IPFS](https://img.shields.io/badge/IPFS-Cluster-65C2CB.svg)](https://ipfscluster.io/)

> **Reproducibility repository** for the paper:  
> *"Byzantine-Resilient Federated Learning for Intrusion Detection in 5G Networks via Hyperledger Fabric and IPFS"*  
> Submitted to IEEE Transactions on Dependable and Secure Computing (TDSC)


## Table of Contents

1. [Overview](#1-overview)
2. [System Architecture](#2-system-architecture)
3. [Testbed: 10-VM Infrastructure](#3-testbed-10-vm-infrastructure)
4. [Prerequisites](#4-prerequisites)
5. [Repository Structure](#5-repository-structure)
6. [Environment Setup](#6-environment-setup)
   - 6.1 [Clone & Configure](#61-clone--configure)
   - 6.2 [Fabric Network Bootstrap](#62-fabric-network-bootstrap)
   - 6.3 [IPFS Cluster Bootstrap](#63-ipfs-cluster-bootstrap)
   - 6.4 [FL Client Setup](#64-fl-client-setup)
   - 6.5 [Prometheus Monitoring](#65-prometheus-monitoring)
7. [Dataset Preparation](#7-dataset-preparation)
8. [Running the Experiments](#8-running-the-experiments)
   - 8.1 [S1 – IDS Detection Quality](#81-s1--ids-detection-quality)
   - 8.2 [S2–S3 – Fabric + FL Round Latency](#82-s2s3--fabric--fl-round-latency)
   - 8.3 [S4 – Byzantine Resilience](#83-s4--byzantine-resilience)
   - 8.4 [S5 – Fault Tolerance](#84-s5--fault-tolerance)
9. [Caliper Benchmark](#9-caliper-benchmark)
10. [Results Reproduction](#10-results-reproduction)
11. [Generating Figures and Tables](#11-generating-figures-and-tables)
12. [Expected Results Summary](#12-expected-results-summary)
13. [Troubleshooting](#13-troubleshooting)
14. [Citation](#14-citation)
15. [License](#15-license)


## 1. Overview

This repository contains the complete implementation of a **Byzantine-resilient Federated Learning Intrusion Detection System (FL-IDS)** for 5G networks, evaluated on a 10-VM private OpenStack testbed.

The system integrates three layers:

| Layer | Technology | Role |
|---|---|---|
| **FL / Detection** | PyTorch + Multi-Krum | Distributed model training, Byzantine defense |
| **Off-chain Storage** | IPFS Cluster (5 nodes) | Content-addressed persistence of FL updates |
| **Accountability** | Hyperledger Fabric 2.4 (Raft) | Tamper-evident audit trail, on-chain security |

**Key results (reproducible):**

| Metric | Value |
|---|---|
| F1-score (UNSW-NB15, t=0.50) | 0.9535 ± 0.0004 |
| ROC-AUC | 0.9883 ± 0.0001 |
| Inference latency | 0.137 µs/sample |
| Fabric throughput | 627 TPS, 0 failures |
| IPFS add+replicate (5.6 KB) | 108 ms p50 |
| Multi-Krum overhead | 0.69 ms p50 |
| Byzantine detection rate | 20/20 rounds (100%) |

---

## 2. System Architecture


┌─────────────────────────────────────────────────────────────────────┐
│                        FL / DETECTION LAYER                         │
│                                                                     │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐   │
│  │edge-client1│  │edge-client3│  │edge-client4│  │edge-client │   │
│  │10.0.0.11   │  │10.0.0.13   │  │10.0.0.14   │  │2-byz       │   │
│  │Honest FL   │  │Honest FL   │  │Honest FL   │  │10.0.0.12   │   │
│  │client      │  │client      │  │client      │  │Byzantine   │   │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘   │
│        └───────────────┴───────────────┴────────────────┘          │
│                                 │                                   │
│                    ┌────────────▼────────────┐                      │
│                    │   fl-ids-vm1-orch        │                      │
│                    │   10.0.0.10              │                      │
│                    │   Orchestrator + Krum    │                      │
│                    └────────────┬────────────┘                      │
└─────────────────────────────────┼───────────────────────────────────┘
                                  │  store artifact (CID)
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         IPFS STORAGE LAYER                          │
│                                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ipfs-node0│ │ipfs-node1│ │ipfs-node2│ │ipfs-node3│ │ipfs-node4│ │
│  │10.0.0.20 │ │10.0.0.21 │ │10.0.0.22 │ │10.0.0.23 │ │10.0.0.24 │ │
│  │(cluster) │ │(cluster) │ │(cluster) │ │(cluster) │ │(cluster) │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
│              Replication min=3, max=5 | Payload 5.6 KB              │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │  anchor CID + hash
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      FABRIC ACCOUNTABILITY LAYER                    │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │peer0.org1        │  │peer0.org2        │  │orderer.example   │  │
│  │10.0.0.30         │  │10.0.0.31         │  │10.0.0.32         │  │
│  │Endorsing peer    │  │Endorsing peer    │  │Raft orderer      │  │
│  │LevelDB           │  │LevelDB           │  │(3 orderers, f=1) │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
│                                                                     │
│  On-chain defenses: Anti-replay | Anti-rollback | MSP Sybil guard  │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                    ┌─────────────▼────────────┐
                    │    prometheus             │
                    │    10.0.0.40             │
                    │    Monitoring + Caliper   │
                    └───────────────────────────┘


## 3. Testbed: 10-VM Infrastructure

All VMs run **Ubuntu 20.04 LTS** on a private OpenStack cloud (5G-emulated network).

| # | VM Name | IP Address | vCPUs | RAM | Role |
|---|---------|-----------|-------|-----|------|
| 1 | `fl-ids-vm1-orch` | `10.0.0.10` | 4 | 8 GB | FL orchestrator, Multi-Krum aggregator, Caliper client gateway |
| 2 | `edge-client-1` | `10.0.0.11` | 2 | 4 GB | FL honest client #1 (largest workload, round p50 = 9.18 s) |
| 3 | `edge-client-2-byz` | `10.0.0.12` | 2 | 4 GB | FL Byzantine client — label flip / model poison / noise |
| 4 | `edge-client-3` | `10.0.0.13` | 2 | 4 GB | FL honest client #3 |
| 5 | `edge-client-4` | `10.0.0.14` | 2 | 4 GB | FL honest client #4 (smallest workload, round p50 = 3.68 s) |
| 6 | `fabric-peer-org1` | `10.0.0.30` | 4 | 8 GB | Fabric peer0.org1 (LevelDB), endorser |
| 7 | `fabric-peer-org2` | `10.0.0.31` | 4 | 8 GB | Fabric peer0.org2 (LevelDB), endorser |
| 8 | `fabric-orderer` | `10.0.0.32` | 4 | 8 GB | Raft orderers ×3 (orderer1/2/3), BatchTimeout=500 ms |
| 9 | `ipfs-cluster` | `10.0.0.20–24` | 2×5 | 4 GB×5 | IPFS cluster-0 to cluster-4 (replication min=3, max=5) |
| 10 | `prometheus` | `10.0.0.40` | 2 | 4 GB | Prometheus + Node Exporter + Grafana, Caliper HTML reports |

> **Note:** VMs 9 uses 5 separate instances sharing the `10.0.0.20/24` range.  
> SSH key: `~/.ssh/fl-ids-key.pem` (replace with your key path throughout).


## 4. Prerequisites

### All VMs
bash
sudo apt-get update && sudo apt-get install -y \
    docker.io docker-compose curl wget git \
    python3.9 python3-pip python3-venv \
    build-essential libssl-dev
sudo usermod -aG docker $USER && newgrp docker

### Orchestrator VM only (`10.0.0.10`)
bash
# Node.js 18 (for Caliper)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Hyperledger Caliper CLI
npm install --save-dev @hyperledger/caliper-cli@0.5.0
npx caliper --version

# Go 1.19 (for chaincode)
wget https://go.dev/dl/go1.19.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.19.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc && source ~/.bashrc


### Python dependencies (all FL VMs: `10.0.0.10–14`)
```bash
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt


`requirements.txt`:

torch==2.0.1
torchvision==0.15.2
numpy==1.24.3
pandas==2.0.2
scikit-learn==1.3.0
ipfshttpclient==0.8.0a2
requests==2.31.0
prometheus-client==0.17.0
pyyaml==6.0
tqdm==4.65.0


---

## 5. Repository Structure


byz-fed-ids-5g/
├── README.md
├── requirements.txt
│
├── fabric/                          # Hyperledger Fabric network
│   ├── network/
│   │   ├── docker-compose.yaml      # Peers + orderers
│   │   ├── configtx.yaml            # Channel + consortium config
│   │   ├── crypto-config.yaml       # MSP identity generation
│   │   └── scripts/
│   │       ├── bootstrap.sh         # Full network up script
│   │       ├── teardown.sh
│   │       └── create-channel.sh
│   └── chaincode/
│       └── fl-ids-cc/               # Go chaincode
│           ├── go.mod
│           └── fl_ids.go            # Anti-replay, anti-rollback, anchor logic
│
├── ipfs/                            # IPFS cluster config
│   ├── cluster-config.json
│   ├── bootstrap-cluster.sh
│   └── test-ipfs.sh
│
├── fl/                              # Federated Learning core
│   ├── orchestrator.py              # Main FL loop + Multi-Krum aggregation
│   ├── client.py                    # FL client (honest)
│   ├── client_byzantine.py          # Byzantine client (attack modes)
│   ├── model.py                     # Neural network architecture
│   ├── aggregation/
│   │   ├── multikrum.py             # Multi-Krum implementation
│   │   ├── fedavg.py
│   │   ├── trimmed_mean.py
│   │   └── coord_median.py
│   ├── data/
│   │   ├── preprocess.py            # UNSW-NB15 preprocessing + balancing
│   │   └── split.py                 # Federated data partitioning
│   └── utils/
│       ├── ipfs_utils.py            # IPFS add/get with cluster confirmation
│       └── fabric_utils.py          # Fabric transaction submission
│
├── caliper/                         # Hyperledger Caliper benchmark
│   ├── benchmarks/
│   │   └── fl-ids-workload.yaml     # 4 rounds × 50,000 tx, 5 workers
│   ├── networks/
│   │   └── fabric-network.yaml      # Network topology for Caliper
│   └── workload/
│       └── dt-anchor.js             # Workload module (DT decision pattern)
│
├── monitoring/                      # Prometheus + Grafana
│   ├── prometheus.yml
│   ├── docker-compose-monitoring.yml
│   └── dashboards/
│       └── fl-ids-dashboard.json
│
├── scripts/                         # End-to-end experiment runners
│   ├── run_s1_detection.sh          # S1: IDS quality
│   ├── run_s2s3_performance.sh      # S2+S3: latency + Fabric bench
│   ├── run_s4_byzantine.sh          # S4: Byzantine resilience
│   └── run_s5_faults.sh             # S5: fault injection
│
├── analysis/                        # Post-processing + figure generation
│   ├── parse_logs.py
│   ├── compute_metrics.py
│   ├── figures/
│   │   ├── fig1_convergence.py
│   │   ├── fig2_detection_rate.py
│   │   ├── fig3_threshold.py
│   │   ├── fig4_byzantine_f1.py
│   │   ├── fig5_defense.py
│   │   └── fig6_system.py
│   └── tables/
│       └── generate_tables.py
│
├── data/                            # Dataset (not included — see §7)
│   └── UNSW_NB15/
│       ├── UNSW_NB15_training-set.csv
│       └── UNSW_NB15_testing-set.csv
│
├── logs/                            # Experiment output logs (git-ignored)
│   └── .gitkeep
│
└── paper/                           # LaTeX paper source
    ├── main.tex
    ├── sections/
    │   └── empirical_evaluation.tex
    └── figures/
        └── final/

## 6. Environment Setup

### 6.1 Clone & Configure

Run on **all VMs**:
bash
git clone https://github.com/YOUR_USERNAME/byz-fed-ids-5g.git
cd byz-fed-ids-5g

# Copy and edit your IP configuration
cp config/network.yaml.example config/network.yaml
nano config/network.yaml


config/network.yaml`:
yaml
orchestrator:
  host: 10.0.0.10
  port: 5000

clients:
  - id: client1
    host: 10.0.0.11
    port: 5001
    byzantine: false
  - id: client2
    host: 10.0.0.12
    port: 5002
    byzantine: true          # Set to false for honest-only runs
  - id: client3
    host: 10.0.0.13
    port: 5003
    byzantine: false
  - id: client4
    host: 10.0.0.14
    port: 5004
    byzantine: false

fabric:
  peer0_org1: 10.0.0.30:7051
  peer0_org2: 10.0.0.31:7051
  orderer:    10.0.0.32:7050
  channel:    fl-ids-channel
  chaincode:  fl-ids-cc

ipfs:
  cluster_api: http://10.0.0.20:9094
  gateway:     http://10.0.0.20:8080
  nodes:
    - 10.0.0.20:5001
    - 10.0.0.21:5001
    - 10.0.0.22:5001
    - 10.0.0.23:5001
    - 10.0.0.24:5001

fl:
  rounds: 20
  local_epochs: 3
  learning_rate: 0.005
  batch_size: 256
  aggregation: multikrum    # options: multikrum | fedavg | trimmed_mean | coord_median
  krum_f: 1                 # Byzantine budget f

prometheus:
  host: 10.0.0.40
  port: 9090

### 6.2 Fabric Network Bootstrap

Run on **`fabric-orderer` VM (`10.0.0.32`)**:

bash
cd byz-fed-ids-5g/fabric/network

# 1. Generate crypto material
export PATH=$PATH:$HOME/byz-fed-ids-5g/fabric/bin
cryptogen generate --config=./crypto-config.yaml

# 2. Generate genesis block and channel artifacts
configtxgen -profile TwoOrgsOrdererGenesis \
  -channelID system-channel -outputBlock ./channel-artifacts/genesis.block

configtxgen -profile TwoOrgsChannel \
  -outputCreateChannelTx ./channel-artifacts/fl-ids-channel.tx \
  -channelID fl-ids-channel

# 3. Bring up the network
docker-compose -f docker-compose.yaml up -d

# 4. Create and join channel (from peer org1)
./scripts/create-channel.sh

# 5. Deploy chaincode
./scripts/deploy-chaincode.sh

# Verify
docker ps | grep -E "peer|orderer"


Expected output:

peer0.org1.example.com   Up      0.0.0.0:7051->7051/tcp
peer0.org2.example.com   Up      0.0.0.0:9051->9051/tcp
orderer1.example.com     Up      0.0.0.0:7050->7050/tcp
orderer2.example.com     Up      0.0.0.0:8050->8050/tcp
orderer3.example.com     Up      0.0.0.0:9050->9050/tcp

**Chaincode verification:**
bash
# Should return: {"status":"OK","rounds":0,"alerts":0}
peer chaincode query -C fl-ids-channel -n fl-ids-cc \
  -c '{"function":"GetSystemStatus","Args":[]}'

### 6.3 IPFS Cluster Bootstrap

Run on **`ipfs-cluster` VMs (`10.0.0.20`–`10.0.0.24`)**:

bash
# Install IPFS and ipfs-cluster-service
wget https://dist.ipfs.tech/kubo/v0.20.0/kubo_v0.20.0_linux-amd64.tar.gz
tar -xzf kubo_v0.20.0_linux-amd64.tar.gz && sudo mv kubo/ipfs /usr/local/bin/

wget https://dist.ipfs.tech/ipfs-cluster-service/v1.0.6/ipfs-cluster-service_v1.0.6_linux-amd64.tar.gz
tar -xzf ipfs-cluster-service_v1.0.6_linux-amd64.tar.gz
sudo mv ipfs-cluster-service /usr/local/bin/

# Initialize IPFS daemon (run on all 5 nodes)
ipfs init
ipfs daemon &

# On node 0 (10.0.0.20) — initialize cluster
ipfs-cluster-service init --consensus crdt
# Copy the generated cluster secret to all other nodes

# On nodes 1–4 — join the cluster
ipfs-cluster-service init --consensus crdt \
  --bootstrap /ip4/10.0.0.20/tcp/9096/p2p/<PEER_ID_OF_NODE0>

# Start cluster service on all nodes
ipfs-cluster-service daemon &


**Verify cluster (from `10.0.0.20`):**
bash
ipfs-cluster-ctl peers ls
# Expected: 5 peers listed

# Test with FL payload size (5.6 KB)
echo "test" | ipfs-cluster-ctl add --replication-min=3 --replication-max=5 -
ipfs-cluster-ctl status <CID>
# Expected: PINNED on 3+ nodes


### 6.4 FL Client Setup

Run on **each edge VM (`10.0.0.11`–`10.0.0.14`) and orchestrator (`10.0.0.10`)**:

```bash
cd byz-fed-ids-5g
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# Place UNSW-NB15 data (see §7)
ls data/UNSW_NB15/
# UNSW_NB15_training-set.csv  UNSW_NB15_testing-set.csv

# Preprocess and partition data
python fl/data/preprocess.py \
  --train data/UNSW_NB15/UNSW_NB15_training-set.csv \
  --test  data/UNSW_NB15/UNSW_NB15_testing-set.csv \
  --balance 1:1 \
  --output data/processed/

python fl/data/split.py \
  --input  data/processed/train_balanced.csv \
  --n-clients 4 \
  --strategy iid \
  --output data/partitions/


### 6.5 Prometheus Monitoring

Run on **`prometheus` VM (`10.0.0.40`)**:

bash
cd byz-fed-ids-5g/monitoring
docker-compose -f docker-compose-monitoring.yml up -d

# Verify
curl http://10.0.0.40:9090/api/v1/targets | python3 -m json.tool | grep "health"
# All targets should show "up"


Access Grafana: `http://10.0.0.40:3000` (admin/admin)  
Import dashboard: `monitoring/dashboards/fl-ids-dashboard.json`

---

## 7. Dataset Preparation

**Dataset:** UNSW-NB15 (publicly available)

bash
# Download from the official source:
# https://research.unsw.edu.au/projects/unsw-nb15-dataset

mkdir -p data/UNSW_NB15
# Place the CSV files:
# - UNSW_NB15_training-set.csv (82,332 samples, 9 attack categories)
# - UNSW_NB15_testing-set.csv  (25,879 samples)

# Verify dataset integrity
python3 - <<'EOF'
import pandas as pd
train = pd.read_csv("data/UNSW_NB15/UNSW_NB15_training-set.csv")
test  = pd.read_csv("data/UNSW_NB15/UNSW_NB15_testing-set.csv")
print(f"Train: {len(train):,} samples | Test: {len(test):,} samples")
print(f"Attack categories: {train['attack_cat'].nunique()}")
print(train['attack_cat'].value_counts())
EOF


Expected output:

Train: 82,332 samples | Test: 25,879 samples
Attack categories: 10
Normal         56,000
Generic        40,000
Exploits       33,393
...

## 8. Running the Experiments

### 8.1 S1 – IDS Detection Quality

Run on **orchestrator (`10.0.0.10`)**. Clients are started automatically via SSH.

bash
cd byz-fed-ids-5g
source venv/bin/activate

# Full S1 run — 3 independent seeds, 20 rounds each
bash scripts/run_s1_detection.sh

# Or manually with custom parameters:
python fl/orchestrator.py \
  --config config/network.yaml \
  --rounds 20 \
  --aggregation multikrum \
  --krum-f 1 \
  --seeds 42 123 456 \
  --threshold-sweep 0.30 0.40 0.50 0.60 0.70 \
  --output logs/s1/

**Expected runtime:** ~35–45 min for 3 seeds × 20 rounds.

**Output files:**
logs/s1/
├── seed_42/
│   ├── round_metrics.json        # F1, Precision, Recall, FPR, AUC per round
│   ├── threshold_sweep.json      # Metrics at each threshold
│   └── phase7_cids.json          # IPFS CIDs from round 7 (used in M4)
├── seed_123/
├── seed_456/
└── summary.json                  # Mean ± std across 3 seeds

**Verify S1 result:**
bash
python3 -c "
import json
s = json.load(open('logs/s1/summary.json'))
print(f\"F1  = {s['f1_mean']:.4f} ± {s['f1_std']:.4f}  (target: 0.9535 ± 0.0004)\")
print(f\"AUC = {s['auc_mean']:.4f} ± {s['auc_std']:.4f}  (target: 0.9883 ± 0.0001)\")
"

---

### 8.2 S2–S3 – Fabric + FL Round Latency

bash
# S2: FL round latency decomposition (runs in parallel with S3)
python fl/orchestrator.py \
  --config config/network.yaml \
  --rounds 20 \
  --aggregation multikrum \
  --log-component-latency \      # enables per-component timing
  --output logs/s2/

# S3: Caliper benchmark (run simultaneously)
bash scripts/run_s2s3_performance.sh

# Or run Caliper directly:
cd caliper
npx caliper launch manager \
  --caliper-workspace . \
  --caliper-networkconfig networks/fabric-network.yaml \
  --caliper-benchconfig benchmarks/fl-ids-workload.yaml \
  --caliper-flow-only-test


**Caliper workload config** (`caliper/benchmarks/fl-ids-workload.yaml`):
yaml
test:
  name: FL-IDS Fabric Benchmark
  description: 200,000 transactions, 4 rounds x 50,000, 5 workers
  workers:
    number: 5
  rounds:
    - label: "500TPS"
      txNumber: 50000
      rateControl:
        type: fixed-rate
        opts:
          tps: 500
    - label: "1000TPS"
      txNumber: 50000
      rateControl:
        type: fixed-rate
        opts:
          tps: 1000
    - label: "2000TPS"
      txNumber: 50000
      rateControl:
        type: fixed-rate
        opts:
          tps: 2000
    - label: "3000TPS"
      txNumber: 50000
      rateControl:
        type: fixed-rate
        opts:
          tps: 3000


**Expected Caliper output** (in `caliper/reports/`):
```
| Name     | Succ  | Fail | Send Rate | Throughput | Avg Lat | Min Lat | Max Lat |
|----------|-------|------|-----------|------------|---------|---------|---------|
| 500TPS   | 50000 | 0    | 500 TPS   | 498 TPS    | 0.59 s  | 0.28 s  | 1.12 s  |
| 1000TPS  | 50000 | 0    | 1000 TPS  | 800 TPS    | 0.61 s  | 0.26 s  | 1.38 s  |
| 2000TPS  | 50000 | 0    | 2000 TPS  | 801 TPS    | 0.78 s  | 0.28 s  | 2.14 s  |
| 3000TPS  | 50000 | 0    | 3000 TPS  | 809 TPS    | 1.26 s  | 0.22 s  | 2.74 s  |


---

### 8.3 S4 – Byzantine Resilience

bash
# All 3 attack types (label flip, model poison, noise)
bash scripts/run_s4_byzantine.sh

# Or individually:
python fl/orchestrator.py \
  --config config/network.yaml \
  --rounds 20 \
  --aggregation multikrum \
  --byzantine-mode label_flip \    # options: label_flip | model_poison | noise
  --output logs/s4/label_flip/

python fl/orchestrator.py \
  --config config/network.yaml \
  --rounds 20 \
  --aggregation multikrum \
  --byzantine-mode model_poison \
  --output logs/s4/model_poison/

# Aggregator comparison
for agg in fedavg trimmed_mean coord_median multikrum; do
  python fl/orchestrator.py \
    --config config/network.yaml \
    --rounds 20 \
    --aggregation $agg \
    --byzantine-mode model_poison \
    --output logs/s4/comparison/$agg/
done

# Byzantine fraction sweep
for frac in 0 1 2; do    # f=0 (0%), f=1 (20%), f=2 (25% approx.)
  python fl/orchestrator.py \
    --config config/network.yaml \
    --rounds 20 \
    --aggregation multikrum \
    --krum-f $frac \
    --byzantine-mode label_flip \
    --output logs/s4/fraction_sweep/f${frac}/
done


**On-chain attack verification:**
```bash
# Replay attack test
python scripts/test_replay_attack.py \
  --fabric-peer 10.0.0.30:7051 \
  --channel fl-ids-channel \
  --chaincode fl-ids-cc
# Expected: "Error: replay detected: alert already exists"

# Rollback attack test
python scripts/test_rollback_attack.py \
  --fabric-peer 10.0.0.30:7051
# Expected: "Error: rollback detected: round < last round"

# Sybil attack test
python scripts/test_sybil_attack.py \
  --fabric-peer 10.0.0.30:7051
# Expected: "Error: access denied: channel creator org unknown"


---

### 8.4 S5 – Fault Tolerance

bash
# Full fault injection suite
bash scripts/run_s5_faults.sh

# --- Raft orderer fault (manual steps) ---
# 1. Start FL experiment in background
python fl/orchestrator.py \
  --config config/network.yaml --rounds 80 \
  --output logs/s5/raft_fault/ &
FL_PID=$!

# 2. Wait until round 60, then stop orderer3
sleep 600   # adjust to your round timing
ssh ubuntu@10.0.0.32 "docker stop orderer3.example.com"

# 3. After 5 rounds (test fault window), restart
sleep 150
ssh ubuntu@10.0.0.32 "docker start orderer3.example.com"

# 4. Wait for completion
wait $FL_PID

# --- IPFS node fault (manual steps) ---
python fl/orchestrator.py \
  --config config/network.yaml --rounds 40 \
  --output logs/s5/ipfs_fault/ &
FL_PID=$!

# Stop IPFS node on VM3
ssh ubuntu@10.0.0.22 "docker stop ipfs-node2 ipfs-cluster-2"

# Let experiment complete (replication min=3 still satisfied)
wait $FL_PID

# Restart IPFS node
ssh ubuntu@10.0.0.22 "docker start ipfs-node2 ipfs-cluster-2"

---

## 9. Caliper Benchmark

The Caliper benchmark is the sole measurement tool for Fabric throughput and latency (M3). Two latency figures are reported:

| Metric | Source | Measurement point |
|---|---|---|
| Submit-to-commit (E2E) | Caliper HTML report | Client submission → commit event |
| Peer-internal commit | Prometheus `kvledger` | `peer_ledger_commit_duration` |

```bash
# Query Prometheus for peer-internal commit latency
curl -s "http://10.0.0.40:9090/api/v1/query" \
  --data-urlencode 'query=histogram_quantile(0.50, peer_ledger_commit_duration_seconds_bucket)' \
  | python3 -m json.tool

# Expected: ~0.057 (57 ms p50)

curl -s "http://10.0.0.40:9090/api/v1/query" \
  --data-urlencode 'query=histogram_quantile(0.95, peer_ledger_commit_duration_seconds_bucket)' \
  | python3 -m json.tool

# Expected: ~0.107 (107 ms p95)


## 10. Results Reproduction

After running all scenarios, compute all metrics:

bash
source venv/bin/activate
python analysis/compute_metrics.py \
  --logs-dir logs/ \
  --caliper-reports caliper/reports/ \
  --output results/metrics_summary.json


**Full reproduction script (runs all 5 scenarios sequentially):**

bash
# WARNING: Full run takes approximately 3–4 hours
bash scripts/run_all_experiments.sh 2>&1 | tee logs/full_run.log


scripts/run_all_experiments.sh`:
bash
#!/bin/bash
set -e
echo "=== S1: IDS Detection Quality ==="
bash scripts/run_s1_detection.sh

echo "=== S2+S3: Performance Benchmark ==="
bash scripts/run_s2s3_performance.sh

echo "=== S4: Byzantine Resilience ==="
bash scripts/run_s4_byzantine.sh

echo "=== S5: Fault Tolerance ==="
bash scripts/run_s5_faults.sh

echo "=== Computing all metrics ==="
python analysis/compute_metrics.py \
  --logs-dir logs/ \
  --caliper-reports caliper/reports/ \
  --output results/metrics_summary.json

echo "=== Generating figures ==="
python analysis/figures/fig1_convergence.py
python analysis/figures/fig2_detection_rate.py
python analysis/figures/fig3_threshold.py
python analysis/figures/fig4_byzantine_f1.py
python analysis/figures/fig5_defense.py
python analysis/figures/fig6_system.py

echo "=== Done. Results in results/ and paper/figures/final/ ==="


---

## 11. Generating Figures and Tables

bash
source venv/bin/activate

# Individual figures
python analysis/figures/fig1_convergence.py   --input logs/s1/   --output paper/figures/final/
python analysis/figures/fig2_detection_rate.py --input logs/s1/  --output paper/figures/final/
python analysis/figures/fig3_threshold.py      --input logs/s1/  --output paper/figures/final/
python analysis/figures/fig4_byzantine_f1.py   --input logs/s4/  --output paper/figures/final/
python analysis/figures/fig5_defense.py        --input logs/s4/  --output paper/figures/final/
python analysis/figures/fig6_system.py         --input logs/ caliper/reports/ --output paper/figures/final/

# LaTeX tables
python analysis/tables/generate_tables.py \
  --input results/metrics_summary.json \
  --output paper/sections/tables_auto.tex

# Transfer to local machine
scp -i ~/.ssh/fl-ids-key.pem \
  ubuntu@10.0.0.10:/home/ubuntu/byz-fed-ids-5g/paper/figures/final/* \
  ~/Downloads/fl-ids-paper/figures/

## 12. Expected Results Summary

The following values should be reproduced within reported confidence intervals:

### S1 — IDS Detection Quality (Table II)
| Metric | Expected | Tolerance |
|---|---|---|
| F1-score (t=0.50, r20 mean) | 0.9518 | ±0.0010 |
| F1-score (t=0.50, best run r19) | 0.9535 | ±0.0004 |
| ROC-AUC | 0.9883 | ±0.0001 |
| Precision | 0.9699 | ±0.002 |
| Recall | 0.9376 | ±0.002 |
| FPR | 0.0619 | ±0.003 |
| Inference latency (p50) | 3.55 ms | ±0.5 ms |

### S2–S3 — System Performance (Table III)
| Metric | Expected |
|---|---|
| Fabric throughput (plateau) | ~620–630 TPS |
| Failures (total) | 0 |
| Peer-internal commit p50 | 57 ms |
| Client submit-to-commit p50 | 591 ms |
| IPFS add+replicate p50 | 108 ms |
| Multi-Krum aggregation p50 | 0.69 ms |

### S4 — Byzantine Resilience (Table IV)
| Attack | Expected F1 | Expected Δ |
|---|---|---|
| Label Flip (20/20 detected) | 0.9511 | −0.0007 |
| Model Poison (20/20 detected) | 0.9512 | −0.0006 |
| FedAvg + Model Poison | 0.1413 | −0.8122 |

### S5 — Fault Tolerance (Table IV)
| Test | Expected |
|---|---|
| Orderer crash recovery | ≤ 20 s |
| IPFS node down → add latency | unchanged (~108 ms) |
| Replay / Rollback / Sybil | REJECTED (100%) |

---

## 13. Troubleshooting

**Fabric peer not joining channel:**
bash
# Check MSP certificates
docker exec peer0.org1.example.com \
  peer channel list
# If empty: re-run ./scripts/create-channel.sh


**IPFS cluster not pinning:**
bash
# Check replication status
ipfs-cluster-ctl status --filter=error
# If nodes unreachable, check firewall: port 9096 (cluster), 4001 (swarm)
sudo ufw allow 9096/tcp && sudo ufw allow 4001/tcp

**FL client timeout on IPFS add:**
bash
# Verify IPFS API is accessible from FL VM
curl http://10.0.0.20:5001/api/v0/id
# Check config: fl/utils/ipfs_utils.py → timeout parameter (default 30s)


**Caliper transaction failures:**
bash
# Check Fabric channel health
peer channel fetch config -c fl-ids-channel -o 10.0.0.32:7050
# If BatchTimeout is causing timeouts, verify docker-compose.yaml:
# CORE_PEER_GOSSIP_USELEADERELECTION=true


**Prometheus targets down:**
bash
# Restart Node Exporter on affected VM
docker restart node-exporter
# Verify port 9100 is open
curl http://10.0.0.30:9100/metrics | head -5


## 15. License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.

Dataset (UNSW-NB15) is provided by the University of New South Wales under its own terms.  
Hyperledger Fabric is licensed under Apache 2.0.  
IPFS and ipfs-cluster are licensed under MIT/Apache 2.0.

---

<p align="center">
  <em>Developed at the Université du Québec en Outaouais (UQO) — TDSC Submission 2025</em>
</p>

