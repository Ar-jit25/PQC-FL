# Adaptive Post-Quantum Cryptography in Federated Learning (PQC-FL)

**Securing Distributed Systems Against Quantum Threats**

---

## 📌 Overview

Classical cryptographic primitives (Diffie-Hellman, RSA, ECDSA) secure most Federated Learning (FL) deployments today, but they are vulnerable to Shor's algorithm on sufficiently powerful quantum computers — including "harvest now, decrypt later" attacks on data collected today.

This project implements and evaluates an **Adaptive Post-Quantum Cryptography Federated Learning (Adaptive PQC-FL)** framework that replaces classical key exchange and signatures with NIST-standardized post-quantum algorithms, and studies the resulting trade-offs in accuracy, latency, and communication overhead — including under heterogeneous (Non-IID) client data and an adaptive workload-delegation scheme.

---

## 🧩 System Architecture

The framework is a Python-based simulation with a client-server FL topology (1 server, 5 simulated clients), organized into three layers:

| Layer | Responsibility |
|---|---|
| **Application Layer** | Local model training (PyTorch CNN), FedAvg global aggregation |
| **Security Layer** | Per-round PQC key exchange (KEM) and digital signatures over client-server communication, via the `liboqs` (Open Quantum Safe) library |
| **Delegation Layer** | Adaptive offloading of KEM operations from resource-constrained clients to an edge server, based on a configurable load threshold |

**Per-round protocol:** server generates a fresh KEM keypair → global model + public key sent to clients → clients encapsulate a shared secret → clients train locally (1 epoch, Adam) and sign their update → server decapsulates + verifies signatures → verified updates aggregated via FedAvg.

---

## 🔐 PQC Algorithms Evaluated

| Role | Algorithms | Basis |
|---|---|---|
| Key Encapsulation (KEM) | **CRYSTALS-Kyber512**, **BIKE-L1** | Lattice-based / Code-based |
| Digital Signatures | **CRYSTALS-Dilithium2**, **Falcon-512** | Lattice-based |

Three combinations were benchmarked: **Kyber512+Dilithium2**, **Kyber512+Falcon-512**, **BIKE-L1+Dilithium2**.

---

## 🧪 Experimental Setup

| Parameter | Value |
|---|---|
| Dataset | MNIST (60K train / 10K test) |
| Clients (N) | 5 |
| Communication rounds (R) | 10 |
| Local epochs / round | 1 |
| Batch size | 64 |
| Optimizer | Adam (lr = 0.001) |
| Model | 2-conv-block CNN (32, 64 filters) |
| Non-IID split | Dirichlet, α = 0.5 |
| Delegation thresholds tested | t ∈ {0.0, 0.3, 0.5, 0.7, 1.0} |
| PQC library | liboqs via pyoqs |
| Platform | Google Colab (Python 3.12, GPU) |
| Seeds | NumPy 42 / PyTorch 42 |

---

## 📊 Key Results

**1. Baseline comparison (Vanilla FL vs. DH-secured FL vs. PQC-FL):**

| Configuration | Final Accuracy | Avg. Round Time |
|---|---|---|
| Vanilla FL (no security) | ~99.09% | ~101.8 s |
| Secure FL (DH masking) | ~99.00% | ~102.85 s |
| PQC-FL (BIKE-L1 + Dilithium2) | ~98.97% | ~99.94 s |

PQC integration cost < 0.15 accuracy points and no meaningful timing penalty.

**2. PQC algorithm combination comparison:**

| Combination | Accuracy | KEM time | Sign time | Signature size |
|---|---|---|---|---|
| Kyber512 + Dilithium2 | ~98.97% | ~0.2 ms | ~15.3 ms | 2420 B |
| Kyber512 + Falcon-512 | ~98.92% | ~0.1 ms | ~16.0 ms | 657 B (**3.68× smaller**) |
| BIKE-L1 + Dilithium2 | ~99.0% | ~0.2 ms | ~15.8 ms | 2420 B |

Kyber512+Dilithium2 is the most balanced default; Falcon-512 is preferred where bandwidth matters.

**3. IID vs. Non-IID:**

| Configuration | Accuracy | Drop vs. IID |
|---|---|---|
| Vanilla FL — IID | 99.14% | — |
| Vanilla FL — Non-IID | 91.27% | −7.87 pts |
| PQC-FL — IID | 99.12% | — |
| PQC-FL — Non-IID | 85.70% | −13.42 pts |

Non-IID heterogeneity roughly **doubles** the accuracy penalty attributable to PQC overhead.

**4. Adaptive delegation (threshold sweep):**

| Threshold t | Delegating clients | IID Accuracy | Non-IID Accuracy |
|---|---|---|---|
| 0.0 (full) | 5/5 | 98.97% | 83.72% |
| 0.3 | 4/5 | 99.10% | 84.88% |
| **0.5 (optimal)** | 3/5 | 99.05% | **88.19%** |
| 0.7 | 1/5 | 99.08% | 86.13% |
| 1.0 (none) | 0/5 | 99.02% | 85.71% |

A **U-curve** emerges under Non-IID conditions — partial delegation (t = 0.5) recovers roughly **30%** of the Non-IID accuracy gap, outperforming both full and no delegation.

---

## ✅ Conclusions

1. PQC integration causes negligible accuracy loss under IID conditions.
2. Different PQC combinations trade off key/signature size vs. speed similarly in accuracy, letting deployment context drive the choice.
3. Non-IID data heterogeneity is a much bigger factor in accuracy loss than PQC overhead itself — and amplifies PQC's cost.
4. Moderate, threshold-based delegation of cryptographic work meaningfully improves Non-IID performance over the "all or nothing" extremes.

Quantum-secure Federated Learning is practically achievable without sacrificing model utility.

---

## 🔭 Future Work

- Dynamic/adaptive threshold selection (e.g., reinforcement learning or bandit methods) instead of a fixed t
- Evaluation on real heterogeneous hardware (Raspberry Pi, Jetson, smartphones)
- Combining PQC with Differential Privacy
- PQC integration with Non-IID-robust aggregation (FedProx, SCAFFOLD, MOON, FedNova)
- Formal security analysis of the delegation architecture under stronger (non-semi-honest) threat models

---

## ⚠️ Scope & Limitations

- Single-machine simulation only — no real network conditions (latency, packet loss, bandwidth limits)
- Horizontal FL only (shared feature space across clients); vertical FL / federated transfer learning out of scope
- Edge server assumed semi-honest (executes delegated ops correctly, not malicious)
- FedAvg aggregation and core ML pipeline were not modified — only the communication/security layer

---

## ⚙️ Setup

```bash
pip install torch torchvision liboqs-python numpy
```

> Requires the [Open Quantum Safe](https://openquantumsafe.org/) `liboqs` library and its Python bindings (`liboqs-python`). Experiments were run on Google Colab with GPU acceleration.

## ▶️ Usage

The project is organized across multiple Jupyter notebooks, each covering one experimental phase:
1. Baseline comparison (Vanilla FL / DH-secured FL / PQC-FL)
2. PQC algorithm combination comparison
3. IID vs. Non-IID evaluation
4. Adaptive delegation threshold sweep

Run each notebook top-to-bottom in Google Colab (or a local Jupyter environment with a GPU) to reproduce the corresponding results tables and figures.

---

## 📚 Key References

- McMahan et al., *Communication-Efficient Learning of Deep Networks from Decentralized Data* (FedAvg), AISTATS 2017
- Moody et al., *Post-Quantum Cryptography: NIST's PQC Standardization Process*, NIST 2024
- NIST FIPS 203 (Kyber / ML-KEM), FIPS 204 (Dilithium / ML-DSA), FIPS 206 (Falcon)
- Karimireddy et al., *SCAFFOLD: Stochastic Controlled Averaging for Federated Learning*, ICML 2020
- Aragon et al., *BIKE: Bit Flipping Key Encapsulation*, NIST PQC Submission 2017

Full reference list is in the project report.

---

## 📄 License

Academic project developed for the Mini-Project (II) course at IIIT Nagpur. Not licensed for commercial use without permission from the authors.
