# Attesta — Contracts
**Soroban Smart Contracts for On-Chain Verifiable Credential Registry**

This repository contains the Soroban smart contracts powering **Attesta** — a privacy-first on-chain verifiable credential registry built on Stellar. The contracts store cryptographic commitment hashes of credentials signed by authoritative institution public keys, enforce a strict access control matrix across roles, and expose efficient view functions for proof verification — all without storing any sensitive user data on-chain.

> **Monorepo siblings:**
> - Frontend → [`attesta-web`](https://github.com/your-org/attesta-web)
> - Backend API → [`attesta-api`](https://github.com/your-org/attesta-api)

---

## 🚀 Contract Overview

### `attesta_registry`
The core credential registry. Stores cryptographic commitment hashes of issued credentials indexed by credential ID and issuer public key. Manages credential status (active, revoked, expired) and maintains an efficient on-chain index for fast lookups. Exposes view functions for third-party proof verification.

### `attesta_issuer`
Handles institution registration and access control. Maintains the access control matrix mapping institution public keys to their permitted credential types. Verifies institution signatures on credential issuance requests and enforces role-based write permissions across all registry operations.

---

## 🛠 Prerequisites

- **Rust** (stable toolchain) — [Install](https://rustup.rs/)
- **Soroban CLI** — Instructions below
- **Stellar testnet account** — We'll create this during setup

---

## 📦 Installation

### Install Soroban CLI

```bash
cargo install --locked stellar-cli --features opt
```

Or use the install script:

```bash
curl -fsSL https://github.com/stellar/stellar-cli/raw/main/install.sh | sh
```

Verify:

```bash
stellar --version
```

### Add WASM Target

```bash
rustup target add wasm32-unknown-unknown
```

### Clone the Repository

```bash
git clone https://github.com/your-org/attesta-contracts.git
cd attesta-contracts
```

---

## ⚙️ Testnet Setup

### Configure Network

```bash
stellar network add --global testnet \
  --rpc-url https://soroban-testnet.stellar.org:443 \
  --network-passphrase "Test SDF Network ; September 2015"
```

### Generate Identity & Fund Account

```bash
stellar keys generate --global deployer --network testnet
stellar keys address deployer
curl "https://friendbot.stellar.org?addr=$(stellar keys address deployer)"
stellar account balance --id deployer --network testnet
```

---

## 🔨 Build

```bash
cargo build --target wasm32-unknown-unknown --release
```

Compiled WASM artifacts will be located at:

```text
target/wasm32-unknown-unknown/release/attesta_registry.wasm
target/wasm32-unknown-unknown/release/attesta_issuer.wasm
```

---

## 🚀 Deploy

### Deploy Issuer Contract

```bash
stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/attesta_issuer.wasm \
  --source deployer \
  --network testnet
```

### Deploy Registry Contract

```bash
stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/attesta_registry.wasm \
  --source deployer \
  --network testnet
```

**Save both contract IDs** — you will need them when initializing the contracts and configuring `attesta-api` and `attesta-web`.

---

## 🔧 Initialize

### Initialize Issuer Contract

```bash
stellar contract invoke \
  --id YOUR_ISSUER_CONTRACT_ID \
  --source deployer \
  --network testnet \
  -- initialize \
  --admin $(stellar keys address deployer) \
  --registry YOUR_REGISTRY_CONTRACT_ID
```

### Initialize Registry Contract

```bash
stellar contract invoke \
  --id YOUR_REGISTRY_CONTRACT_ID \
  --source deployer \
  --network testnet \
  -- initialize \
  --admin $(stellar keys address deployer) \
  --issuer YOUR_ISSUER_CONTRACT_ID \
  --max_credentials_per_issuer 10000 \
  --credential_expiry_window 31536000
```

### Register a First Institution

```bash
stellar contract invoke \
  --id YOUR_ISSUER_CONTRACT_ID \
  --source deployer \
  --network testnet \
  -- register_institution \
  --institution_key INSTITUTION_PUBLIC_KEY \
  --credential_types "medical_license,certification" \
  --name "Example Medical Board"
```

---

## ⚙️ Contract Parameters

| Parameter | Default | Description |
|---|---|---|
| `max_credentials_per_issuer` | 10,000 | Max credentials an institution can issue |
| `credential_expiry_window` | 31,536,000s (1 yr) | Default validity period for issued credentials |
| `max_credential_types_per_issuer` | 10 | Max distinct credential types per institution |
| `proof_verification_window` | 300s | Grace window for time-bound proof submissions |

---

## 🔐 Access Control Matrix

| Action | Admin | Institution | User | Verifier |
|---|---|---|---|---|
| Register institution | ✅ | ❌ | ❌ | ❌ |
| Issue credential | ❌ | ✅ (own types only) | ❌ | ❌ |
| Revoke credential | ❌ | ✅ (own credentials) | ❌ | ❌ |
| Read commitment hash | ✅ | ✅ | ✅ | ✅ |
| Verify proof | ✅ | ✅ | ✅ | ✅ |
| Update institution keys | ✅ | ✅ (own keys) | ❌ | ❌ |
| Update contract params | ✅ | ❌ | ❌ | ❌ |

---

## 🔌 Key Contract Functions

### Issuer Contract

```rust
// Register a new institution with permitted credential types
attesta_issuer::register_institution(env, institution_key, credential_types, name)

// Rotate an institution's signing key
attesta_issuer::rotate_institution_key(env, old_key, new_key, admin_sig)

// Check if a public key is a registered institution
attesta_issuer::is_registered_institution(env, public_key) -> bool

// Get credential types permitted for an institution
attesta_issuer::get_permitted_types(env, institution_key) -> Vec<String>
```

### Registry Contract

```rust
// Issue a credential commitment (institution-signed)
attesta_registry::issue_credential(env, credential_id, commitment_hash, issuer_key, credential_type, expiry)

// Revoke a credential by ID
attesta_registry::revoke_credential(env, credential_id, issuer_key, issuer_sig)

// Verify a proof against an on-chain commitment
attesta_registry::verify_proof(env, credential_id, proof_hash) -> VerificationResult

// Get credential status (active, revoked, expired)
attesta_registry::get_credential_status(env, credential_id) -> CredentialStatus

// Get commitment hash for a credential ID
attesta_registry::get_commitment(env, credential_id) -> Hash
```

### Example Verification Flow

```rust
// Third-party verifier checks a user's credential proof
let result = attesta_registry::verify_proof(
    &env,
    credential_id,
    user_submitted_proof_hash
);

match result {
    VerificationResult::Valid => { /* proceed */ },
    VerificationResult::Revoked => return Err(ContractError::CredentialRevoked),
    VerificationResult::Expired => return Err(ContractError::CredentialExpired),
    VerificationResult::Invalid => return Err(ContractError::ProofMismatch),
}
```

---

## 📁 Project Structure

```text
/
├── registry/                 # Credential registry contract
│   ├── src/
│   │   ├── lib.rs            # Contract entry points
│   │   ├── storage.rs        # Commitment hash indexing and state
│   │   ├── verification.rs   # Proof verification logic
│   │   └── types.rs          # CredentialStatus, VerificationResult types
│   └── Cargo.toml
├── issuer/                   # Institution access control contract
│   ├── src/
│   │   ├── lib.rs            # Contract entry points
│   │   ├── access.rs         # Access control matrix enforcement
│   │   ├── keys.rs           # Public key registration and rotation
│   │   └── types.rs          # Institution, CredentialType types
│   └── Cargo.toml
└── Cargo.toml                # Workspace root
```

---

## 🧪 Testing

### Run All Tests

```bash
cargo test
```

### Test Individual Contracts

```bash
cargo test -p attesta_registry
cargo test -p attesta_issuer
```

### Test With Output Logging

```bash
cargo test -- --nocapture
```

---

## 🐛 Troubleshooting

**`insufficient balance` on deploy**

```bash
curl "https://friendbot.stellar.org?addr=$(stellar keys address deployer)"
```

**`wasm32-unknown-unknown target not found`**

```bash
rustup target add wasm32-unknown-unknown
```

**`UnauthorizedIssuer` on credential issuance**
1. Confirm the institution public key is registered via `register_institution`
2. Confirm the credential type being issued is in the institution's permitted types list
3. Verify the institution signature on the issuance request is valid

**`InvalidCommitmentHash` on proof verification**
1. Confirm the proof hash was generated from the correct credential data
2. Check that the credential ID matches the one stored in the registry
3. Verify the credential has not been revoked or expired before testing

**`KeyRotationFailed`**
1. Ensure the old key signature is valid and the account is still active
2. Confirm the admin signature is provided if rotating from an inactive key

---

## 📚 Resources

- [Stellar Documentation](https://developers.stellar.org/docs/build/smart-contracts)
- [Soroban Docs](https://soroban.stellar.org/docs)
- [Soroban Examples](https://github.com/stellar/soroban-examples)
- [W3C Verifiable Credentials Spec](https://www.w3.org/TR/vc-data-model/)
- [Attesta Frontend Repo](https://github.com/your-org/attesta-web)
- [Attesta API Repo](https://github.com/your-org/attesta-api)

---

## 🤝 Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for Rust/Soroban coding standards, Git workflow, and the PR process.

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

---

**Built with ❤️ on Stellar**
