# Project IronLog — Architecture Overview

> **Document Classification:** INTERNAL — Architecture Reference  
> **Version:** 1.0.0  
> **Date:** 2026-04-30  
> **Personas:** Lead Security Architect · Senior Blockchain Engineer · Threat Modeller · Distributed Systems Engineer

---

## 1. System Purpose

Project IronLog provides **cryptographically verifiable, tamper-evident file and log integrity monitoring** for enterprise Linux endpoints. Every file-change event is:

1. **Hashed** (BLAKE3 + SHA-256) at the edge
2. **Encrypted** (AES-256-GCM) and stored off-chain in private IPFS
3. **Signed** (ECDSA-P384 via FIPS 140-2 Level 3 HSM) and committed to a permissioned blockchain ledger
4. **Analyzed** by a streaming ML anomaly detection engine
5. **Forwarded** to SIEM for SOC consumption

The system is designed under the assumption of **breach at every layer**. No single component compromise should allow an attacker to silently tamper with the audit trail.

---

## 2. Trust Boundaries

```text
┌──────────────────────────────────────────────────────────┐
│                    UNTRUSTED_ZONE                        │
│  Monitored endpoint: filesystem, user processes,         │
│  Edge Agent (Go binary), eBPF hooks                      │
│                                                          │
│  Threat assumption: Kernel-level compromise possible.    │
│  Agent binary may be replaced. Network is hostile.       │
└──────────────┬───────────────────────────┬───────────────┘
               │ HSM PKCS#11              │ mTLS
               ▼                          ▼
┌──────────────────────┐   ┌──────────────────────────────┐
│   TRUSTED_ENCLAVE    │   │       STORAGE_ZONE           │
│  HSM (FIPS 140-2 L3) │   │  Private IPFS Cluster        │
│  Intel SGX Enclave   │   │  (≥3 nodes, encrypted-at-rest)│
│                      │   │                              │
│  Keys never leave    │   │  CID proves existence only;  │
│  hardware boundary.  │   │  content encrypted before    │
│                      │   │  storage.                    │
└──────────────────────┘   └──────────────────────────────┘
                                      │
               ┌──────────────────────┘
               ▼
┌──────────────────────────────────────────────────────────┐
│                    LEDGER_ZONE                           │
│  Hyperledger Fabric network (Raft consensus)             │
│  3-of-5 org endorsement policy                           │
│  Smart contract (chaincode) validates signatures,        │
│  verifies ZKP proofs, emits indexed events               │
│                                                          │
│  Threat assumption: Up to 2-of-5 orgs compromised.      │
└──────────────────────┬───────────────────────────────────┘
                       │ Kafka (TLS + SASL)
                       ▼
┌──────────────────────────────────────────────────────────┐
│                      AI_ZONE                             │
│  Streaming ML inference (ONNX Runtime)                   │
│  Isolation Forest baseline, LSTM future                  │
│  FP SLA: ≤ 0.1%                                         │
│  Explainability output mandatory on every alert          │
│                                                          │
│  Threat assumption: Model weights may be poisoned.       │
│  Inference results are advisory, not authoritative.      │
└──────────────────────┬───────────────────────────────────┘
                       │ OpenTelemetry → Kafka
                       ▼
┌──────────────────────────────────────────────────────────┐
│                     SIEM_ZONE                            │
│  Splunk/Elastic SIEM                                     │
│  SOAR playbook integration                               │
│  SOC Tier-1/Tier-2 escalation                            │
└──────────────────────────────────────────────────────────┘
```

---

## 3. Component Detail

### 3.1 Edge Agent (UNTRUSTED_ZONE)

| Attribute | Value | Threat Mitigated |
|-----------|-------|------------------|
| **Language** | Go 1.22+ | — |
| **File monitor** | eBPF (cilium/ebpf library) | T1006 (Direct Volume Access) — inotify can be bypassed via raw block device writes; eBPF hooks at VFS layer cannot. |
| **Self-attestation** | BLAKE3 hash of own binary checked against remote attestation service at startup, then every 15 min | T1036 (Masquerading), T1554 (Compromise Client Software Binary) |
| **Hashing** | BLAKE3 (primary, high-throughput) + SHA-256 (FIPS compliance) | T1565.001 (Stored Data Manipulation) — dual hashing prevents algorithm-specific collision attacks |
| **Rate limiter** | Token bucket: 1000 events/sec/agent, configurable | Denial-of-service via event storm flooding the pipeline |
| **Config sealing** | Intel SGX sealed storage; no plaintext credentials on disk | T1552.001 (Credentials in Files) |
| **Payload signing** | ECDSA-P384 via HSM PKCS#11 | T1557 (Adversary-in-the-Middle) — signature binds payload to agent identity |

**Open question / environment dependency:** SGX availability depends on CPU generation (Ice Lake+). Fallback to TPM 2.0 sealed storage if SGX unavailable — flag as deployment-time configuration.

### 3.2 Off-Chain Storage — Private IPFS (STORAGE_ZONE)

| Attribute | Value | Threat Mitigated |
|-----------|-------|------------------|
| **Implementation** | IPFS Cluster (ipfs-cluster) over Kubo nodes | — |
| **Replication factor** | ≥3 nodes (configurable, default 3) | Single-node failure / data loss |
| **Encryption** | AES-256-GCM with HSM-derived DEK; AAD = file_path ‖ timestamp | T1530 (Data from Cloud Storage Object) — CID is public, but content is ciphertext |
| **Key reference** | DEK key reference stored in blockchain payload (NOT the key itself) | T1552.004 (Private Keys) — key material stays in HSM |
| **Retention policy** | CID pinned for compliance window: SOC2 = 7 years, configurable per policy | Compliance violation via premature garbage collection |
| **Pinning SLA** | Content pinned within 500ms of PUT; exempt from GC until retention expiry | Data availability during forensic investigation |

**Design decision:** Content is encrypted *before* IPFS push. This is mandatory because IPFS CIDs prove existence (content-addressing) but provide zero confidentiality. Any node in the cluster can read unpinned content that transits the network.

### 3.3 Permissioned Ledger + Smart Contracts (LEDGER_ZONE)

| Attribute | Value | Threat Mitigated |
|-----------|-------|------------------|
| **Platform** | Hyperledger Fabric 2.5+ | — |
| **Consensus** | Raft (crash fault tolerant, 3-of-5 orderers) | T1195.002 (Supply Chain Compromise) — requires majority compromise |
| **Endorsement policy** | 3-of-5 organizations must endorse | Rogue single-org node cannot commit fraudulent records |
| **Channel architecture** | One channel per tenant; system channel for cross-tenant policy | T1199 (Trusted Relationship) — tenant data isolation |
| **Chaincode** | Go, see `chaincode/file_integrity.go` | — |
| **ZKP** | Groth16 zk-SNARK for Merkle inclusion proof | Compliance verification without exposing the full allow-list |
| **Events** | Indexed events emitted on every state change for SIEM/AI consumption | T1070 (Indicator Removal) — events are immutable once committed |

**Groth16 vs PLONK tradeoff:**
- **Groth16**: Constant-time verification (~10ms), smallest proof size (~192 bytes). Requires per-circuit trusted setup (mitigated by MPC ceremony). Chosen for production due to verification speed at scale.
- **PLONK**: Universal trusted setup (one ceremony for all circuits), but larger proofs (~512 bytes) and slower verification (~30ms). Preferred if circuit changes frequently. Recommend PLONK for development/staging, Groth16 for production.

**"Compliance state" formal definition:** The Merkle root of a balanced binary tree where each leaf is SHA-256(canonical_file_path ‖ authorized_blake3_hash). The security policy smart contract maintains this tree. A file change is compliant if and only if a valid Merkle inclusion proof (verified via zk-SNARK) demonstrates that the file's new hash is a leaf in the tree.

### 3.4 AI Anomaly Detection Middleware (AI_ZONE)

| Attribute | Value | Threat Mitigated |
|-----------|-------|------------------|
| **Transport** | Kafka topic (TLS + SASL/SCRAM), subscribed to Fabric block events | T1557 (AitM on event stream) |
| **Feature vector** | `cadence_delta` (time since last change), `hour_entropy` (Shannon entropy of change-hour distribution), `process_lineage` (pid→parent chain), `user_context` (uid, sudo history, session type), `file_sensitivity_tier` (1–5 admin-assigned scale) | — |
| **Baseline model** | Isolation Forest (scikit-learn trained, ONNX-exported) | T1078 (Valid Accounts) — detects anomalous legitimate-credential usage |
| **Advanced model** | LSTM (PyTorch trained, ONNX-exported) for sequential pattern detection | T1059 (Command/Script Interpreter) — detects multi-step attack chains |
| **Modes** | Online (streaming, <100ms p99 latency) + Offline (batch reanalysis of historical blocks) | — |
| **FP SLA** | ≤ 0.1% on known-good change patterns (measured weekly) | Alert fatigue leading to missed true positives |
| **Explainability** | Every alert includes: anomaly_score, triggered_features[], confidence_interval, nearest-neighbor reference from training set | SOC analyst requires justification to act |

**Adversarial robustness:** Model retraining requires 2-of-3 ML engineer approval + signed model hash committed to blockchain before deployment. This prevents single-actor model poisoning (T1565.001).

---

## 4. Data Flow Summary

```text
File Modified on Endpoint
        │
        ▼
  [1] eBPF hook captures event (UNTRUSTED_ZONE)
        │
        ▼
  [2] Agent rate-limits, computes BLAKE3+SHA-256 (UNTRUSTED_ZONE)
        │
        ▼
  [3] Agent requests DEK from HSM (→ TRUSTED_ENCLAVE)
        │
        ▼
  [4] Agent encrypts content, pushes to IPFS (→ STORAGE_ZONE)
        │ Returns: CID
        ▼
  [5] Agent constructs payload, signs via HSM (→ TRUSTED_ENCLAVE)
        │ Returns: ECDSA-P384 signature
        ▼
  [6] Agent submits signed TX to Fabric peer (→ LEDGER_ZONE)
        │
        ▼
  [7] Chaincode validates: agent identity, signature, ZKP
        │
        ▼
  [8] Endorsement (3-of-5 orgs) → Orderer → Block committed
        │ Emits: FileChangeRecorded event
        ▼
  [9] Kafka streams event to AI engine (→ AI_ZONE)
        │
        ▼
  [10] AI scores anomaly, enriches event
        │
        ├─ NORMAL → forward to SIEM (→ SIEM_ZONE)
        │
        └─ ALERT → flag on blockchain + trigger SOAR playbook
```

**Latency budget (end-to-end, p99):**

| Stage | Budget | Notes |
|-------|--------|-------|
| eBPF capture → Agent | <1ms | Kernel → userspace via perf ring buffer |
| Hashing (BLAKE3) | <5ms | For files ≤100MB |
| HSM DEK derivation | <10ms | PKCS#11 over USB/PCIe |
| IPFS PUT + pin | <200ms | Local cluster, 3-node replication |
| HSM signing | <10ms | ECDSA-P384 |
| Fabric TX submit → commit | <500ms | Raft consensus, 3-of-5 endorsement |
| AI inference | <100ms | ONNX Runtime, GPU optional |
| **Total** | **<826ms** | Excluding network jitter |

---

## 5. Cryptographic Operations Inventory

| Operation | Algorithm | Key Size | Location | Purpose |
|-----------|-----------|----------|----------|---------|
| File hashing (throughput) | BLAKE3 | 256-bit output | Agent (CPU) | Integrity fingerprint |
| File hashing (FIPS) | SHA-256 | 256-bit output | Agent (CPU) | FIPS compliance record |
| Content encryption | AES-256-GCM | 256-bit key, 96-bit nonce | Agent (CPU, key from HSM) | IPFS content confidentiality |
| DEK derivation | HKDF-SHA-256 | 256-bit | HSM | Per-file encryption key |
| Payload signing | ECDSA-P384 | 384-bit | HSM | Non-repudiation, agent identity |
| Self-attestation hash | BLAKE3 | 256-bit output | Agent (CPU) + SGX verify | Binary integrity |
| ZKP proof generation | Groth16 (BN254) | — | Agent or dedicated prover | Privacy-preserving compliance |
| ZKP verification | Groth16 (BN254) | — | Chaincode (on-chain) | Compliance check without full allow-list |
| TLS (inter-component) | TLS 1.3 | X25519 + AES-256-GCM | All network boundaries | Transport confidentiality |

---

## 6. Failure Modes & Recovery

| Failure | Detection | Impact | Recovery |
|---------|-----------|--------|----------|
| HSM unavailable | Agent health check, PKCS#11 timeout | Cannot sign or derive keys; events queued locally (encrypted, max 10,000) | Failover to secondary HSM; if no secondary, agent enters read-only mode and alerts SOC |
| IPFS node down | Cluster health API, replication factor drops below 3 | Content still available if ≥1 replica survives | Auto-rebalance by IPFS Cluster; if all replicas lost, recover from blockchain CID + re-push from agent local cache |
| Fabric peer down | Peer health gRPC, Prometheus metrics | TX submission fails; agent retries with exponential backoff (max 5 min) | Raft leader election; if <3-of-5 orderers available, ledger is read-only until quorum restored |
| AI engine crash | Kafka consumer lag alert, health endpoint | Events accumulate in Kafka; no anomaly scoring | Kafka retention (72h default) buffers events; AI restart replays from last committed offset |
| Agent binary tampered | Self-attestation fails at startup or 15-min interval | Agent refuses to start / shuts down | SOC notified; agent binary re-deployed from signed artifact registry; endpoint quarantined |

---

## 7. Compliance Mapping

| Requirement | IronLog Component | Evidence |
|-------------|-------------------|----------|
| SOC2 CC6.1 (Logical access) | Fabric MSP + chaincode ACL | Only registered agent identities can write |
| SOC2 CC7.2 (System monitoring) | eBPF agent + AI anomaly detection | Continuous file monitoring with ML scoring |
| SOC2 CC8.1 (Change management) | Blockchain immutable ledger | Every file change recorded with cryptographic proof |
| NIST 800-53 AU-9 (Audit record protection) | HSM signing + blockchain immutability | Signed records on append-only ledger |
| NIST 800-53 SI-7 (Software/info integrity) | BLAKE3+SHA-256 dual hashing | Integrity verification at capture and on-chain |
| PCI-DSS 10.5 (Secure audit trails) | Private IPFS (encrypted) + blockchain | Off-chain encrypted storage + on-chain proof |

---

*See `architecture_diagram.puml` for the full PlantUML sequence diagram.*  
*See `chaincode/file_integrity.go` for the smart contract implementation.*  
*See `threat_model.md` for STRIDE + MITRE ATT&CK analysis.*
