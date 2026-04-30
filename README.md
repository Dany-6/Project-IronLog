# Project IronLog

> **Enterprise-Grade File & Log Integrity Monitoring System**

## 📖 Project Overview
Project IronLog is a cryptographically verifiable, tamper-evident file and log integrity monitoring architecture designed for high-security enterprise environments. By combining endpoint eBPF monitoring, off-chain IPFS storage, Hardware Security Module (HSM) signing, and a permissioned blockchain ledger, IronLog ensures that critical audit trails cannot be silently altered or deleted—even by an attacker with root access.

## ✨ Key Features & Functionality
- **Immutable Audit Trail:** All file changes are logged to a Hyperledger Fabric permissioned blockchain, ensuring zero tampering or retroactive deletion.
- **High-Throughput Edge Monitoring:** Utilizes eBPF hooks combined with BLAKE3 hashing to monitor filesystem changes at the kernel level without performance degradation.
- **Cryptographic Non-Repudiation:** Every log event is signed using ECDSA-P384 via a FIPS 140-2 Level 3 Hardware Security Module (HSM).
- **Decentralized Secure Storage:** Audit payloads are encrypted via AES-256-GCM and stored in a private IPFS cluster, guaranteeing high availability and compliance retention.
- **AI-Driven Anomaly Detection:** Streams blockchain events via Kafka to an ML inference engine to detect sophisticated multi-step attack chains with a strict false-positive SLA.
- **Zero-Knowledge Verification:** Utilizes Groth16 zk-SNARK proofs within the smart contract to verify file integrity against a policy allow-list without exposing the full dataset.

---

## 🚀 Beginner's Installation & Execution Guide

Welcome to the hands-on deployment guide. Even if you are a beginner, this step-by-step tutorial will help you install the necessary tools, start a local test blockchain, and deploy the IronLog smart contract (chaincode) to it.

---

## Step 1: Install Prerequisites

Before we begin, your computer needs three essential tools: **Git**, **Go**, and **Docker**. Choose your operating system below and follow the instructions carefully.

### 🪟 Windows Users

1. **Install Git:**
   - Download the installer from [git-scm.com/download/win](https://git-scm.com/download/win) and run it. You can leave all settings at their defaults.
2. **Install Go (Programming Language):**
   - Download the Windows MSI installer from [go.dev/dl](https://go.dev/dl/).
   - Run the installer and follow the prompts.
3. **Install Docker Desktop (Crucial for the Blockchain):**
   - Download Docker Desktop from [docker.com](https://docs.docker.com/desktop/install/windows-install/).
   - Run the installer. **Important:** Ensure the "Use WSL 2 instead of Hyper-V" option is checked during installation.
   - Restart your computer when prompted.
   - After restarting, open the "Docker Desktop" application and accept the terms to ensure the Docker engine is running in the background.

### 🍎 macOS Users

*Tip: We recommend using the Terminal app and Homebrew for the easiest installation.*

1. **Install Homebrew (if you don't have it):**
   - Open your `Terminal` app and paste this command:
     ```bash
     /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
     ```
2. **Install Git and Go:**
   - Run this command in your terminal:
     ```bash
     brew install git go
     ```
3. **Install Docker Desktop:**
   - Run this command:
     ```bash
     brew install --cask docker
     ```
   - Open your `Applications` folder, launch **Docker**, and grant it the necessary permissions. Keep it running in the background.

### 🐧 Linux Users (Ubuntu/Debian)

Open your terminal and run the following commands one by one:

1. **Install Git and Docker:**
   ```bash
   sudo apt-get update
   sudo apt-get install git docker.io docker-compose curl -y
   ```
2. **Configure Docker Permissions:**
   - This prevents you from needing `sudo` every time you use Docker.
   ```bash
   sudo usermod -aG docker $USER
   newgrp docker
   ```
3. **Install Go:**
   ```bash
   wget https://go.dev/dl/go1.22.2.linux-amd64.tar.gz
   sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.22.2.linux-amd64.tar.gz
   export PATH=$PATH:/usr/local/go/bin
   ```
   *(To make Go permanent, add `export PATH=$PATH:/usr/local/go/bin` to your `~/.bashrc` file).*

---

## Step 2: Compile the IronLog Smart Contract

Now that your tools are installed, let's verify that the IronLog smart contract (written in Go) compiles correctly. This proves your Go installation works.

1. Open a terminal (Command Prompt or PowerShell on Windows, Terminal on Mac/Linux).
2. Navigate to the IronLog `chaincode` directory:
   ```bash
   # Adjust this path based on where you saved the project!
   cd "c:\Users\bahar\Downloads\Project IronLog\chaincode"
   ```
3. Run the Go build command:
   ```bash
   go build ./...
   ```
   > **Note:** If the command finishes and returns you to the prompt without printing any errors, **congratulations!** The code compiled perfectly.

---

## Step 3: Start the Local Blockchain Network

To actually run the contract, we need a blockchain! We will use the official Hyperledger Fabric Test Network.

1. Create a new folder anywhere on your computer (e.g., your Desktop or Documents) and open a terminal in that folder.
2. Download the Fabric Samples repository:
   ```bash
   git clone https://github.com/hyperledger/fabric-samples.git
   ```
3. Navigate into the test network folder:
   ```bash
   cd fabric-samples/test-network
   ```
4. Start the blockchain network and create a channel (a private ledger) called `mychannel`:
   ```bash
   ./network.sh up createChannel -c mychannel
   ```
   > **What is happening?** Docker is downloading images and spinning up isolated containers that act as "nodes" in your local blockchain. This might take a few minutes the very first time you run it.

---

## Step 4: Deploy the IronLog Contract to the Blockchain

Now we tell the blockchain network to load our IronLog code.

1. Make sure your terminal is still inside the `fabric-samples/test-network` directory.
2. Run the deployment command. **You must change the `../path/to/Project IronLog/chaincode` part** to the actual path where your IronLog chaincode folder is located!
   
   ```bash
   # Example:
   ./network.sh deployCC -ccn file_integrity -ccp "c:/Users/bahar/Downloads/Project IronLog/chaincode" -ccl go
   ```
   
   - `-ccn`: Chaincode Name (`file_integrity`)
   - `-ccp`: Chaincode Path (Where your code lives)
   - `-ccl`: Chaincode Language (`go`)

   > The network will package your code, install it on the nodes, and approve it. You will see a lot of output. Wait for the success message at the end!

---

## Step 5: Execute Transactions (Testing the App)

Once deployed, you can interact with the blockchain just like a real application would! We will use the command line to simulate the IronLog Edge Agent submitting a file change.

1. First, we have to set up some environment variables so our terminal acts like a blockchain user (Admin of Org1):
   
   **For Linux/Mac (Bash):**
   ```bash
   export PATH=${PWD}/../bin:$PATH
   export FABRIC_CFG_PATH=$PWD/../config/
   export CORE_PEER_TLS_ENABLED=true
   export CORE_PEER_LOCALMSPID="Org1MSP"
   export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
   export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
   export CORE_PEER_ADDRESS=localhost:7051
   ```

2. **Log a File Change Event:**
   Now, we "invoke" the `LogFileChange` function in our smart contract. Paste this huge command (it simulates a single log event being saved to the blockchain):

   ```bash
   peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" -C mychannel -n file_integrity -c '{"function":"LogFileChange","Args":["/var/log/auth.log", "blake3_hash_123", "sha256_hash_123", "ipfs_cid_xyz", "dek_ref_999", "agent_001", "168285999000", "hsm_sig_abc", "attest_hash_def", "zkp_proof_123"]}'
   ```
   *If successful, you will see a message saying `Chaincode invoke successful. result: status:200`.*

3. **Verify the Record was Saved:**
   Let's query the blockchain to prove the record is there. Wait a few seconds for the transaction to be added to a block, then run:

   ```bash
   peer chaincode query -C mychannel -n file_integrity -c '{"function":"GetRecord","Args":["<PASTE_THE_TX_ID_HERE>"]}'
   ```
   *(Note: Because our blueprint code currently focuses on the `LogFileChange` logic, you'll need to look at the Docker logs of the peer to see the event emission, or expand the Go code to include a `GetRecord` read function!)*

---

## Step 6: Clean Up

When you are done testing, you can shut down the blockchain network to free up your computer's memory.

Inside the `fabric-samples/test-network` directory, run:
```bash
./network.sh down
```
This removes the Docker containers and clears the local ledger data.
