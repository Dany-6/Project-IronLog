# Project IronLog — Technology Stack

> **Document Classification:** INTERNAL — Architecture Reference  
> **Date:** 2026-04-30  

The following stack selections prioritize security, formal verifiability, and high-throughput logging capabilities. Every selection is justified against a concrete alternative.

| Category | Tool Selection | Justification vs. Rejected Alternative |
| :--- | :--- | :--- |
| **Agent Runtime** | **Go (1.22+)** | *Rejected: Rust.* While Rust offers superior memory safety without GC pauses, Go was selected for its robust existing eBPF ecosystem (`cilium/ebpf`), extensive cryptographic libraries, and seamless alignment with the Hyperledger Fabric chaincode environment. Go 1.22's PGO (Profile-Guided Optimization) handles the required throughput. |
| **Blockchain Platform** | **Hyperledger Fabric (v2.5+)** | *Rejected: Quorum (EVM).* Fabric's channel architecture provides superior multi-tenant data isolation at the protocol level. Fabric's Go chaincode is more expressive and less prone to re-entrancy vulnerabilities than Quorum's Solidity smart contracts. Fabric's Raft consensus is better suited for an enterprise permissioned network than Quorum's IBFT. |
| **ZKP Library** | **gnark (Consensys)** | *Rejected: circom/snarkjs.* `gnark` allows writing circuits directly in Go, unifying the agent, smart contract, and prover codebase. It offers significantly faster proof generation times for Groth16 compared to circom's JavaScript/C++ stack. |
| **IPFS Implementation** | **IPFS Cluster + Kubo** | *Rejected: Native IPFS without clustering.* Native IPFS does not provide guaranteed data replication SLAs or automated rebalancing across nodes. IPFS Cluster provides the necessary pinning orchestration to ensure the required 3-node replication factor and compliance retention policies. |
| **ML Framework** | **ONNX Runtime** | *Rejected: TensorFlow Serving.* ONNX provides a framework-agnostic execution engine. This allows data scientists to train the Isolation Forest in `scikit-learn` or the LSTM in `PyTorch`, export to ONNX, and run inference in a lightweight, high-performance C++ runtime without the bloated dependencies of TF Serving. |
| **SIEM Connector** | **Kafka + OpenTelemetry** | *Rejected: Splunk HEC (HTTP Event Collector).* Direct HEC integration creates tight coupling and potential backpressure if the SIEM is degraded. Kafka provides a robust, decoupled buffer (72h retention) allowing the AI middleware to emit events asynchronously. OpenTelemetry standardizes the schema. |
| **Secret Management** | **HashiCorp Vault (Enterprise)** | *Rejected: AWS KMS.* IronLog must be deployable in air-gapped network segments. AWS KMS requires continuous outbound internet access to AWS endpoints. Vault can be deployed entirely on-premise and integrates natively with the PKCS#11 HSM backends for key wrapping. |
| **Container Orchestration** | **Kubernetes (Hardened)** | *Rejected: HashiCorp Nomad.* While Nomad is simpler, Kubernetes provides advanced native constructs required for this architecture: NetworkPolicies for strict zone isolation, native integration with eBPF CNIs (Cilium), and robust Operator patterns for managing the Fabric nodes. |
| **Observability** | **Prometheus + Grafana** | *Rejected: Datadog.* Similar to the AWS KMS rejection, Datadog requires telemetry to leave the organizational boundary. Prometheus/Grafana operates entirely within the air-gapped environment, ensuring sensitive operational metadata (which could be used for reconnaissance) is not exposed to a SaaS provider. |

---

## ⚠️ Supply Chain Risk & CVE Flags

- **Go (Agent Runtime):** Low risk, but requires strict dependency pinning. Go's native toolchain vulnerabilities (e.g., historical `net/http` or `crypto/tls` CVEs) require rapid patching. Ensure GOPROXY is pointed to an internal, scanned artifact registry.
- **Hyperledger Fabric:** The Fabric CA component has had historical CVEs related to certificate parsing. Monitor `hyperledger/fabric-ca` repositories closely.
- **Kubo (IPFS):** IPFS implementations are historically complex and have experienced denial-of-service vulnerabilities. Exposing the IPFS swarm port to the untrusted zone is extremely high risk; it must remain strictly within the `STORAGE_ZONE` boundary.
- **ONNX Runtime:** High supply chain risk. ML dependencies are notorious for arbitrary code execution vulnerabilities during model deserialization (e.g., Pickle vulnerabilities, though ONNX mitigates this by using protobuf). Validate model hashes before loading.
