# System Architecture

## Overview

This document provides a detailed architectural overview of the Decentralized TEE-Based Bitcoin Mining Pool system.

## Component Communication Flow

```
                                   [User/Miner]
                                        │
                                        ▼
                              ┌─────────────────┐
                              │   Miners (3+)   │
                              │   cpuminer      │
                              └────────┬────────┘
                                       │ Stratum Protocol
                                       │ (TCP 3333)
                                       ▼
                    ┌──────────────────────────────────┐
                    │      Mining Pool Server          │
                    │  • Job distribution              │
                    │  • Share validation              │
                    │  • Block submission              │
                    └───┬────────────────────────┬─────┘
                        │                        │
          RPC           │                        │ HTTP/gRPC
    (Bitcoin commands)  │                        │
                        ▼                        ▼
        ┌───────────────────────┐    ┌──────────────────┐
        │   Bitcoin Node        │    │  TEE Component   │
        │   (regtest/testnet)   │    │  • Share verify  │
        │   • Block templates   │    │  • Attestation   │
        │   • Block validation  │    │  • Audit log     │
        └───────────────────────┘    └─────────┬────────┘
                                               │
                                               │ Validated rewards
                                               ▼
                                    ┌──────────────────────┐
                                    │   Bridge Service     │
                                    │  • BTC monitoring    │
                                    │  • WBTC conversion   │
                                    │  • Token minting     │
                                    └──────────┬───────────┘
                                               │
                                               │ Web3 calls
                                               ▼
    ┌──────────────────────────────────────────────────────────┐
    │                  Ethereum Network (Sepolia)               │
    ├──────────────────────────────────────────────────────────┤
    │  ┌──────────────────┐         ┌────────────────────┐    │
    │  │  WBTC Vault      │◄────────┤  Reward Token      │    │
    │  │  Contract        │         │  Contract (ERC-20) │    │
    │  │                  │         │                    │    │
    │  │  • Store WBTC    │         │  • Mint tokens     │    │
    │  │  • Redemption    │         │  • Burn tokens     │    │
    │  │  • NAV calc      │         │  • Transfer        │    │
    │  └──────────────────┘         └────────────────────┘    │
    └──────────────────────────────────────────────────────────┘
                                ▲
                                │ Token balances & trading
                                │
                        [Miners' Wallets]
```

## Data Flow

### 1. Mining Flow

```
1. Pool fetches block template from Bitcoin node
   └─> GET getblocktemplate

2. Pool creates Stratum jobs from template
   └─> Distributes to miners

3. Miners hash and submit shares
   └─> Submit via Stratum protocol

4. Pool validates shares
   └─> Sends to TEE for verification

5. TEE validates and signs result
   └─> Returns attestation

6. Pool records share in database
   └─> Updates miner statistics

7. Valid block found?
   └─> Yes: Submit to Bitcoin node
   └─> No: Continue mining
```

### 2. Reward Flow

```
1. Block confirmed on Bitcoin network
   └─> Bitcoin node validates block

2. BTC sent to pool wallet
   └─> Bridge service monitors wallet

3. Bridge accumulates BTC (threshold: 0.001)
   └─> Waits for conversion trigger

4. Bridge converts BTC → WBTC
   └─> Manual for MVP
   └─> Automated in production

5. Bridge deposits WBTC to vault
   └─> Calls vault.deposit(amount)

6. Bridge calculates rewards
   └─> Pro-rata based on shares
   └─> Fetches BTC/USD price

7. Bridge mints tokens
   └─> Calls token.batchMint(miners, amounts)

8. Tokens distributed to miners
   └─> Event emitted on-chain
```

### 3. Redemption Flow

```
1. Miner initiates redemption
   └─> Calls token.approve(vault, amount)

2. Miner calls vault.redeem(amount)
   └─> Vault calculates pro-rata WBTC

3. Vault burns reward tokens
   └─> Calls token.burn(miner, amount)

4. Vault transfers WBTC to miner
   └─> Transfers from vault balance

5. Miner receives WBTC
   └─> Can convert to BTC via official process
```

## Component Details

### Bitcoin Node

**Purpose:** Provide blockchain data and validate blocks

**Key Functions:**
- `getblocktemplate` - Fetch mining work
- `submitblock` - Submit found blocks
- `getbalance` - Check wallet balance
- `generatetoaddress` - Mine blocks (regtest only)

**Data Storage:** 
- Blockchain data (~50GB for testnet)
- Wallet data (~1MB)

### Mining Pool

**Purpose:** Coordinate miners and distribute work

**Key Functions:**
- Accept miner connections (Stratum)
- Distribute jobs to miners
- Validate submitted shares
- Track miner contributions
- Submit found blocks

**Data Storage:**
- PostgreSQL database:
  - `shares` table (~1GB/month)
  - `blocks` table (~1MB/month)
  - `miners` table (~1MB)

**API Endpoints:**
- Stratum (TCP 3333) - Miner communication
- HTTP (3334) - Admin/monitoring

### TEE Component

**Purpose:** Provide verifiable, tamper-proof share validation

**Key Functions:**
- Validate share proof-of-work
- Sign validation results
- Provide remote attestation
- Audit log all operations

**Data Storage:**
- PostgreSQL audit log (~500MB/month)
- Signing keys (secured)

**API Endpoints:**
- POST `/validate` - Validate share
- POST `/calculate-rewards` - Calculate rewards
- GET `/attestation` - Get TEE attestation
- POST `/verify` - Verify signature

### Bridge Service

**Purpose:** Convert BTC to WBTC and trigger token minting

**Key Functions:**
- Monitor Bitcoin wallet
- Convert BTC to WBTC
- Deposit WBTC to vault
- Calculate miner rewards
- Mint reward tokens

**Data Storage:**
- PostgreSQL database:
  - `conversions` table
  - `minting_operations` table

**API Endpoints:**
- GET `/api/status` - Service status
- GET `/api/btc-wallet` - Bitcoin wallet info
- POST `/api/convert` - Start conversion
- POST `/api/mint-tokens` - Mint tokens

### Smart Contracts

#### WBTC Vault

**Purpose:** Secure WBTC storage and redemption

**Key Functions:**
```solidity
function deposit(uint256 amount) external
function redeem(uint256 tokenAmount) external
function getBalance() external view returns (uint256)
function getNavPerToken() external view returns (uint256)
```

**State Variables:**
- `wbtc` - WBTC token address
- `rewardToken` - Reward token address
- `totalDeposited` - Total WBTC deposited
- `totalRedeemed` - Total WBTC redeemed

#### Reward Token

**Purpose:** ERC-20 token representing mining rewards

**Key Functions:**
```solidity
function mint(address to, uint256 amount) external
function batchMint(address[] to, uint256[] amounts) external
function burn(address from, uint256 amount) external
```

**State Variables:**
- `vault` - Vault contract address
- `treasury` - Treasury address
- `totalMinted` - Total tokens minted
- `totalBurned` - Total tokens burned

## Security Model

### Trust Assumptions

**What we trust:**
- TEE hardware (Intel SGX / ARM TrustZone)
- Ethereum consensus
- Bitcoin consensus
- Open-source code review

**What we don't trust:**
- Pool operator
- Bridge service operator (for MVP)
- Individual miners
- External services

### Security Measures

1. **TEE Isolation**
   - Share validation in secure enclave
   - Cryptographic attestation
   - Audit logging

2. **Smart Contract Security**
   - OpenZeppelin contracts
   - Access control (roles)
   - Reentrancy guards
   - Pausable (emergency)

3. **Bridge Security**
   - Multi-signature (production)
   - Time locks (production)
   - Monitoring and alerts

4. **Network Security**
   - TLS for all communications
   - API authentication
   - Rate limiting
   - DDoS protection

## Scalability

### Current MVP Capacity

- **Miners:** 3-100
- **Hash rate:** <1 GH/s (CPU mining)
- **Shares/second:** 10-50
- **Database:** <10GB
- **Network:** <10 Mbps

### Production Targets

- **Miners:** 1,000-10,000
- **Hash rate:** 10-100 PH/s
- **Shares/second:** 1,000-10,000
- **Database:** 100GB-1TB
- **Network:** 100-1000 Mbps

### Scaling Strategies

1. **Horizontal Scaling**
   - Multiple pool servers
   - Load balancing
   - Database replication

2. **Vertical Scaling**
   - Larger servers
   - SSD storage
   - More memory

3. **Optimization**
   - Redis caching
   - Database indexing
   - Connection pooling
   - Batch processing

## Monitoring & Observability

### Key Metrics

**Mining Pool:**
- Active miners
- Total hash rate
- Shares per second
- Valid/invalid share ratio
- Blocks found
- Pool efficiency

**TEE Component:**
- Validations per second
- Average validation time
- Signature verification rate
- Audit log size

**Bridge Service:**
- BTC balance
- Pending conversions
- WBTC deposited
- Tokens minted
- Conversion success rate

**Smart Contracts:**
- Vault WBTC balance
- Token total supply
- Redemption rate
- NAV per token

### Logging

All components use structured logging:

```json
{
  "timestamp": "2025-10-17T10:30:00Z",
  "level": "INFO",
  "component": "mining-pool",
  "event": "share_submitted",
  "miner": "miner1",
  "difficulty": 1.0,
  "valid": true
}
```

### Alerts

- Block found
- Invalid shares spike
- TEE validation failure
- Bridge conversion failure
- Low WBTC balance
- High gas prices
- Contract paused

## Deployment Architecture

### MVP (Local/Testnet)

```
┌─────────────────────────────────────┐
│     Developer Machine               │
│  ┌──────────────────────────────┐  │
│  │ Docker Compose               │  │
│  │  • Bitcoin node              │  │
│  │  • PostgreSQL                │  │
│  │  • Mining pool               │  │
│  │  • TEE component             │  │
│  │  • Bridge service            │  │
│  └──────────────────────────────┘  │
│                                     │
│  Miners running locally             │
└─────────────────────────────────────┘
              │
              ▼
    [Ethereum Sepolia Testnet]
```

### Production (Mainnet)

```
┌─────────────────────────────────────────────┐
│              AWS/GCP Cloud                  │
│  ┌───────────────────────────────────────┐ │
│  │  Kubernetes Cluster                   │ │
│  │   ┌──────────────────────────────┐   │ │
│  │   │ Mining Pool (3 replicas)     │   │ │
│  │   └──────────────────────────────┘   │ │
│  │   ┌──────────────────────────────┐   │ │
│  │   │ TEE Workers (Phala Network)  │   │ │
│  │   └──────────────────────────────┘   │ │
│  │   ┌──────────────────────────────┐   │ │
│  │   │ Bridge Service (2 replicas)  │   │ │
│  │   └──────────────────────────────┘   │ │
│  └───────────────────────────────────────┘ │
│  ┌───────────────────────────────────────┐ │
│  │  Database (PostgreSQL + replicas)    │ │
│  └───────────────────────────────────────┘ │
│  ┌───────────────────────────────────────┐ │
│  │  Load Balancer                        │ │
│  └───────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
              │
              ▼
  [Miners worldwide] → [Pool]
              ↓
  [Bitcoin Mainnet] + [Ethereum Mainnet]
```

## Future Enhancements

1. **Stratum V2** - More efficient protocol
2. **Layer 2** - Cheaper token transactions
3. **Multi-pool** - Support multiple pools
4. **Advanced governance** - DAO voting
5. **Mobile apps** - iOS/Android miners
6. **Advanced analytics** - ML-based optimization

---

For implementation details, see individual component READMEs.
