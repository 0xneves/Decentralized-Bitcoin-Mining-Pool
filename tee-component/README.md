# TEE Component (Trusted Execution Environment)

## Overview

The TEE (Trusted Execution Environment) component is the **trust anchor** of the entire system. It provides cryptographically verifiable proof that share validation and reward calculations are executed fairly and cannot be tampered with, even by the server operator.

## What This Component Does

- **Validates mining shares** in a secure, isolated environment
- **Calculates rewards** with cryptographic proof of correctness
- **Provides remote attestation** so anyone can verify the code being executed
- **Protects sensitive operations** from observation or manipulation
- **Eliminates trust** in the pool operator

## Architecture Role

```
[Mining Pool] --shares--> [TEE Component] --attestation--> [Verifiers]
                              ↓
                      [Reward Calculation]
                              ↓
                    [Bridge Service Notification]
```

## Technology Options

### For MVP - Two Paths:

**Option A: TEE Simulator (Recommended for MVP)**
- Simulates TEE behavior with cryptographic signing
- Faster development (1-2 weeks)
- No special hardware required
- Easy to test and debug
- Can migrate to real TEE later

**Option B: Phala Network Phat Contracts (Production)**
- Real TEE using Intel SGX or ARM TrustZone
- Full security guarantees
- Steeper learning curve (4-8 weeks)
- Requires SGX-enabled hardware or Phala testnet

### This README covers BOTH approaches

---

## Option A: TEE Simulator (MVP Implementation)

### What is the Simulator?

A software component that mimics TEE behavior:
- ✅ Cryptographic signing of all operations
- ✅ Audit logging (immutable log of all actions)
- ✅ Deterministic execution
- ✅ API compatible with real TEE
- ⚠️ Does NOT provide true hardware isolation (for production, use real TEE)

### Prerequisites

- Rust 1.70+ (recommended) or Go/Node.js
- PostgreSQL for audit logging
- OpenSSL for cryptographic operations

### Installation

1. **Install Rust:**
   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   source $HOME/.cargo/env
   ```

2. **Clone and setup:**
   ```bash
   cd tee-component
   cargo build --release
   ```

3. **Generate signing keys:**
   ```bash
   ./scripts/generate-keys.sh
   # Creates: keys/private.pem, keys/public.pem
   ```

4. **Configure:**
   ```bash
   cp config.example.toml config.toml
   nano config.toml
   ```

   ```toml
   [server]
   host = "0.0.0.0"
   port = 8080
   
   [keys]
   private_key = "./keys/private.pem"
   public_key = "./keys/public.pem"
   
   [database]
   url = "postgresql://postgres:password@localhost:5432/tee_audit"
   
   [validation]
   max_share_age_seconds = 300
   require_sequential_nonces = false
   ```

### Running the Simulator

```bash
cargo run --release
# or
./target/release/tee-simulator
```

**Output:**
```
TEE Simulator v0.1.0
Public Key: 04f3a2...
Listening on 0.0.0.0:8080
Audit log: PostgreSQL
Ready to validate shares
```

### API Endpoints

**1. Validate Share:**
```bash
POST /validate
Content-Type: application/json

{
  "share": {
    "worker": "miner1_address",
    "job_id": "abc123",
    "nonce": "0x1234567890",
    "result": "0000000000abcdef...",
    "difficulty": 1,
    "timestamp": 1697568000
  },
  "block_template": {
    "previous_block": "00000000000000...",
    "merkle_root": "abcdef1234...",
    "target": "0000ffff..."
  }
}
```

**Response:**
```json
{
  "valid": true,
  "share_id": "share_12345",
  "signature": "3045022100...",
  "attestation": {
    "timestamp": 1697568000,
    "validator_pubkey": "04f3a2...",
    "operation": "share_validation",
    "result": "valid"
  }
}
```

**2. Calculate Rewards:**
```bash
POST /calculate-rewards
Content-Type: application/json

{
  "block_hash": "00000000000abc...",
  "block_reward": 6.25,
  "shares": [
    {"worker": "miner1", "count": 100, "difficulty": 1},
    {"worker": "miner2", "count": 150, "difficulty": 1},
    {"worker": "miner3", "count": 50, "difficulty": 1}
  ],
  "pool_fee": 0.01
}
```

**Response:**
```json
{
  "rewards": [
    {"worker": "miner1", "btc": 2.06, "usd_value": 134000},
    {"worker": "miner2", "btc": 3.09, "usd_value": 201000},
    {"worker": "miner3", "btc": 1.03, "usd_value": 67000}
  ],
  "pool_fee_btc": 0.0625,
  "signature": "3045022100...",
  "attestation": {
    "timestamp": 1697568100,
    "validator_pubkey": "04f3a2...",
    "operation": "reward_calculation",
    "total_distributed": 6.1875
  }
}
```

**3. Get Attestation:**
```bash
GET /attestation

Response:
{
  "version": "0.1.0",
  "public_key": "04f3a2...",
  "code_hash": "sha256:abcdef...",
  "uptime_seconds": 3600,
  "total_shares_validated": 1523,
  "total_rewards_calculated": 15
}
```

**4. Verify Signature:**
```bash
POST /verify
Content-Type: application/json

{
  "data": {...},
  "signature": "3045022100..."
}

Response:
{
  "valid": true,
  "signer": "04f3a2..."
}
```

### Audit Logging

All operations are logged to PostgreSQL:

```sql
CREATE TABLE audit_log (
  id SERIAL PRIMARY KEY,
  operation_type VARCHAR(50) NOT NULL,
  timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  input_hash VARCHAR(64) NOT NULL,
  output_hash VARCHAR(64) NOT NULL,
  signature VARCHAR(512) NOT NULL,
  pubkey VARCHAR(128) NOT NULL,
  metadata JSONB
);

CREATE INDEX idx_audit_timestamp ON audit_log(timestamp);
CREATE INDEX idx_audit_operation ON audit_log(operation_type);
```

**Query audit log:**
```sql
-- Recent validations
SELECT * FROM audit_log 
WHERE operation_type = 'share_validation' 
ORDER BY timestamp DESC 
LIMIT 10;

-- Verify integrity
SELECT 
  COUNT(*) as total_operations,
  COUNT(DISTINCT signature) as unique_signatures,
  MIN(timestamp) as first_operation,
  MAX(timestamp) as last_operation
FROM audit_log;
```

### Testing the Simulator

**Test 1: Share validation**
```bash
curl -X POST http://localhost:8080/validate \
  -H "Content-Type: application/json" \
  -d '{
    "share": {
      "worker": "test_miner",
      "job_id": "test_job",
      "nonce": "0x12345678",
      "result": "0000000000abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
      "difficulty": 1,
      "timestamp": 1697568000
    },
    "block_template": {
      "previous_block": "0000000000000000000000000000000000000000000000000000000000000000",
      "merkle_root": "abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890",
      "target": "0000ffff00000000000000000000000000000000000000000000000000000000"
    }
  }'
```

**Test 2: Reward calculation**
```bash
curl -X POST http://localhost:8080/calculate-rewards \
  -H "Content-Type: application/json" \
  -d '{
    "block_hash": "00000000000abc123",
    "block_reward": 6.25,
    "shares": [
      {"worker": "miner1", "count": 100, "difficulty": 1},
      {"worker": "miner2", "count": 200, "difficulty": 1}
    ],
    "pool_fee": 0.01
  }'
```

**Test 3: Signature verification**
```bash
# Get public key
curl http://localhost:8080/attestation

# Verify a previous operation's signature
curl -X POST http://localhost:8080/verify \
  -H "Content-Type: application/json" \
  -d '{
    "data": {...},
    "signature": "3045022100..."
  }'
```

---

## Option B: Phala Network Phat Contracts (Production)

### What is Phala Network?

Phala is a decentralized cloud computing platform that uses TEEs (Intel SGX) to provide confidential and verifiable computation.

**Key features:**
- ✅ True hardware isolation
- ✅ Remote attestation via blockchain
- ✅ Decentralized (no single point of failure)
- ✅ Polkadot parachain integration

### Prerequisites

- Rust 1.70+ with `wasm32-unknown-unknown` target
- Phala development environment
- Intel SGX-enabled hardware OR access to Phala testnet
- Polkadot.js wallet

### Installation

1. **Install Phala SDK:**
   ```bash
   cargo install cargo-contract --force
   rustup target add wasm32-unknown-unknown
   ```

2. **Install Phala CLI:**
   ```bash
   npm install -g @phala/fn
   ```

3. **Clone Phala examples:**
   ```bash
   git clone https://github.com/Phala-Network/phat-contract-examples.git
   cd phat-contract-examples
   ```

4. **Setup TEE development environment:**
   ```bash
   # Option A: Use Phala testnet (no SGX needed)
   phala-fn login
   
   # Option B: Local SGX (requires SGX CPU)
   ./scripts/setup-sgx.sh
   ```

### Creating a Phat Contract

**Contract structure (`src/lib.rs`):**

```rust
#![cfg_attr(not(feature = "std"), no_std)]

use pink_extension as pink;

#[pink::contract(env=PinkEnvironment)]
mod tee_validator {
    use super::pink;
    use ink_storage::traits::SpreadAllocate;
    
    #[ink(storage)]
    #[derive(SpreadAllocate)]
    pub struct TeeValidator {
        owner: AccountId,
        shares_validated: u64,
        rewards_calculated: u64,
    }
    
    impl TeeValidator {
        #[ink(constructor)]
        pub fn new() -> Self {
            ink_lang::utils::initialize_contract(|contract: &mut Self| {
                contract.owner = Self::env().caller();
                contract.shares_validated = 0;
                contract.rewards_calculated = 0;
            })
        }
        
        #[ink(message)]
        pub fn validate_share(
            &mut self,
            worker: String,
            nonce: u64,
            result: Vec<u8>,
            target: Vec<u8>
        ) -> Result<bool, String> {
            // Validate proof-of-work
            if self.check_pow(&result, &target) {
                self.shares_validated += 1;
                
                // Log to blockchain
                pink::ext().log(&format!(
                    "Valid share from {} at nonce {}",
                    worker, nonce
                ));
                
                Ok(true)
            } else {
                Ok(false)
            }
        }
        
        #[ink(message)]
        pub fn calculate_rewards(
            &mut self,
            block_reward: u128,
            shares: Vec<(String, u64)>,
            pool_fee: u8
        ) -> Vec<(String, u128)> {
            let total_shares: u64 = shares.iter().map(|(_, s)| s).sum();
            let fee_amount = (block_reward * pool_fee as u128) / 100;
            let distributable = block_reward - fee_amount;
            
            let rewards: Vec<(String, u128)> = shares
                .iter()
                .map(|(worker, share_count)| {
                    let reward = (distributable * *share_count as u128) / total_shares as u128;
                    (worker.clone(), reward)
                })
                .collect();
            
            self.rewards_calculated += 1;
            rewards
        }
        
        fn check_pow(&self, result: &[u8], target: &[u8]) -> bool {
            // Compare result against target
            for (r, t) in result.iter().zip(target.iter()) {
                if r > t {
                    return false;
                } else if r < t {
                    return true;
                }
            }
            true
        }
    }
}
```

### Building and Deploying

**Build contract:**
```bash
cargo contract build --release
```

**Deploy to Phala testnet:**
```bash
# Deploy contract
phala-fn upload ./target/ink/tee_validator.contract

# Instantiate contract
phala-fn instantiate --constructor new

# Get contract ID
phala-fn info
```

### Calling Phat Contract from Pool

**JavaScript/TypeScript:**
```typescript
import { ApiPromise, WsProvider } from '@polkadot/api';
import { ContractPromise } from '@polkadot/api-contract';

// Connect to Phala
const wsProvider = new WsProvider('wss://poc5.phala.network/ws');
const api = await ApiPromise.create({ provider: wsProvider });

// Load contract
const contract = new ContractPromise(
  api,
  contractABI,
  contractAddress
);

// Validate share
const { result, output } = await contract.query.validateShare(
  callerAddress,
  { gasLimit: -1 },
  worker,
  nonce,
  result,
  target
);

// Calculate rewards
const { result, output } = await contract.query.calculateRewards(
  callerAddress,
  { gasLimit: -1 },
  blockReward,
  shares,
  poolFee
);
```

### Remote Attestation

Phala provides built-in remote attestation:

```rust
use pink_extension::chain_extension::{signing, SigType};

#[ink(message)]
pub fn get_attestation(&self) -> Vec<u8> {
    // Get TEE attestation quote
    let quote = pink::ext().get_attestation_quote();
    
    // Sign with enclave key
    let signature = signing::sign(&quote, SigType::Sr25519);
    
    // Return attestation report
    [quote, signature].concat()
}
```

**Verify attestation:**
```bash
phala-fn verify-attestation --quote [attestation_data]
```

---

## Comparison: Simulator vs Phala

| Feature | TEE Simulator | Phala Network |
|---------|--------------|---------------|
| Security | ⚠️ Software only | ✅ Hardware TEE |
| Setup Time | 1-2 days | 1-2 weeks |
| Hardware | Any computer | SGX CPU or testnet |
| Attestation | Software signing | Intel SGX attestation |
| Decentralization | Single server | Distributed network |
| Cost | Free | Gas fees on Phala |
| **Use Case** | **MVP Development** | **Production** |

**Recommendation:** Start with simulator, migrate to Phala for production.

---

## Integration with Other Components

### Mining Pool Integration

```javascript
// Pool sends share to TEE
const response = await fetch('http://localhost:8080/validate', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    share: shareData,
    block_template: template
  })
});

const { valid, signature, attestation } = await response.json();

// Store attestation proof
await db.query(
  'UPDATE shares SET tee_signature = $1, tee_valid = $2 WHERE id = $3',
  [signature, valid, shareId]
);
```

### Bridge Service Integration

```javascript
// TEE notifies bridge after reward calculation
const rewards = await teeClient.calculateRewards(blockData);

await bridgeService.processRewards({
  rewards: rewards.rewards,
  signature: rewards.signature,
  attestation: rewards.attestation
});
```

---

## Testing & Validation

### Unit Tests

```bash
cd tee-component
cargo test
```

### Integration Tests

```bash
# Start simulator
cargo run &

# Run integration tests
npm run test:tee-integration
```

### Load Testing

```bash
# Benchmark share validation
ab -n 10000 -c 100 -p share.json http://localhost:8080/validate
```

---

## Security Considerations

### Simulator Limitations:
- ⚠️ No hardware isolation (can be debugged)
- ⚠️ Relies on software signing
- ⚠️ Single point of failure
- ✅ Sufficient for MVP testing

### Phala Production Security:
- ✅ Hardware-enforced isolation
- ✅ Remote attestation via Intel SGX
- ✅ Decentralized execution
- ✅ Cryptographic proofs

---

## Monitoring & Maintenance

**Health check:**
```bash
curl http://localhost:8080/health

Response:
{
  "status": "healthy",
  "uptime_seconds": 3600,
  "shares_validated": 1523,
  "last_operation": "2025-10-17T10:30:00Z"
}
```

**Metrics:**
```bash
curl http://localhost:8080/metrics

Response:
{
  "shares_validated_total": 1523,
  "shares_valid": 1500,
  "shares_invalid": 23,
  "rewards_calculated": 15,
  "average_validation_time_ms": 5.2
}
```

---

## Next Steps

Once TEE component is running:

1. ✅ TEE API is accessible
2. ✅ Share validation works
3. ➡️ Integrate with mining pool
4. ➡️ Test reward calculations
5. ➡️ Set up audit log monitoring

---

## Resources

### Simulator Resources:
- [Rust Cryptography](https://docs.rs/ring/latest/ring/)
- [ECDSA Signing](https://docs.rs/secp256k1/latest/secp256k1/)

### Phala Resources:
- [Phala Documentation](https://docs.phala.network/)
- [Phat Contracts Guide](https://docs.phala.network/developers/phat-contract)
- [Intel SGX](https://www.intel.com/content/www/us/en/developer/tools/software-guard-extensions/overview.html)
- [Phala GitHub](https://github.com/Phala-Network/phala-blockchain)

---

## Troubleshooting

### Simulator Issues:

**Issue 1: "Failed to load private key"**
```bash
# Regenerate keys
./scripts/generate-keys.sh
chmod 600 keys/private.pem
```

**Issue 2: "Database connection failed"**
```bash
# Check PostgreSQL
sudo systemctl status postgresql
psql -U postgres -c "SELECT 1;"
```

### Phala Issues:

**Issue 1: "SGX not available"**
- Use Phala testnet instead of local SGX
- Or use simulator for development

**Issue 2: "Contract deployment failed"**
```bash
# Check balance
phala-fn balance

# Top up testnet tokens
# https://faucet.phala.network
```

---

*Choose Simulator for MVP, migrate to Phala for production deployment.*
