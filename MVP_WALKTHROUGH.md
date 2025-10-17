# Decentralized TEE-Based Bitcoin Mining Pool - MVP Walkthrough

**Version:** 1.0  
**Date:** October 17, 2025  
**Purpose:** Step-by-step guide to build and test the MVP system

---

## Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Phase 1: Environment Setup](#phase-1-environment-setup)
4. [Phase 2: Bitcoin Network](#phase-2-bitcoin-network)
5. [Phase 3: Smart Contracts](#phase-3-smart-contracts)
6. [Phase 4: Mining Pool](#phase-4-mining-pool)
7. [Phase 5: TEE Component](#phase-5-tee-component)
8. [Phase 6: Bridge Service](#phase-6-bridge-service)
9. [Phase 7: Miners](#phase-7-miners)
10. [Phase 8: End-to-End Testing](#phase-8-end-to-end-testing)
11. [Troubleshooting](#troubleshooting)
12. [Next Steps](#next-steps)

---

## Introduction

This walkthrough guides you through building a **Minimum Viable Product (MVP)** of the decentralized mining pool system. By the end, you'll have:

✅ A local Bitcoin regtest network  
✅ 3 working CPU miners  
✅ A mining pool coordinating work  
✅ TEE component validating shares (simulated or real)  
✅ Smart contracts on Ethereum testnet  
✅ Bridge converting BTC → WBTC → Tokens  
✅ Full end-to-end mining reward flow  

**Estimated Time:** 2-4 weeks (depending on experience)  
**Difficulty:** Intermediate to Advanced  

---

## Prerequisites

### Required Knowledge
- Basic understanding of Bitcoin and Ethereum
- Familiarity with command line
- Basic programming (JavaScript/TypeScript or Python)
- Understanding of smart contracts (helpful but not required)

### Required Software

Install these before starting:

```bash
# 1. Node.js 18+
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 18
nvm use 18

# 2. Python 3.10+ (optional, if using Python)
sudo apt-get install python3.10 python3-pip

# 3. Docker & Docker Compose
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo apt-get install docker-compose-plugin

# 4. Bitcoin Core
# See bitcoin-node/README.md for installation

# 5. Git
sudo apt-get install git

# 6. PostgreSQL
sudo apt-get install postgresql postgresql-contrib

# 7. Rust (for TEE component)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### Required Accounts

- Infura or Alchemy account (free tier) for Ethereum
- Ethereum wallet with Sepolia testnet ETH (get from faucet)
- GitHub account (for cloning repositories)

### Hardware Requirements

**Minimum:**
- 4 CPU cores
- 8 GB RAM
- 50 GB disk space
- Stable internet connection

**Recommended:**
- 8+ CPU cores
- 16 GB RAM
- 100 GB SSD
- High-speed internet

---

## Phase 1: Environment Setup

### Step 1.1: Clone or Create Project

```bash
# If starting fresh
mkdir decentralized-mining-pool
cd decentralized-mining-pool

# Initialize git
git init

# Create .gitignore
cat > .gitignore << EOF
node_modules/
.env
*.log
target/
build/
deployments/
cache/
artifacts/
.DS_Store
*.swp
EOF
```

### Step 1.2: Set Up Project Structure

```bash
# Create all folders
mkdir -p bitcoin-node/{scripts}
mkdir -p miners/{miner1,miner2,miner3,scripts}
mkdir -p mining-pool/{src,config,logs}
mkdir -p tee-component/{src,tests,keys}
mkdir -p bridge-service/{src,config}
mkdir -p smart-contracts/{contracts,scripts,test,deployments}
mkdir -p coordinator/{src,config}
mkdir -p dashboard/{src,public}
mkdir -p docker
mkdir -p docs
mkdir -p scripts

echo "✅ Folder structure created"
```

### Step 1.3: Setup PostgreSQL

```bash
# Start PostgreSQL
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create databases
sudo -u postgres psql << EOF
CREATE DATABASE mining_pool;
CREATE DATABASE bridge_service;
CREATE DATABASE tee_audit;
\q
EOF

echo "✅ Databases created"
```

### Step 1.4: Install Global Dependencies

```bash
# Hardhat for smart contracts
npm install -g hardhat

# PM2 for process management
npm install -g pm2

# Testing tools
npm install -g newman artillery

echo "✅ Global dependencies installed"
```

---

## Phase 2: Bitcoin Network

### Step 2.1: Install Bitcoin Core

```bash
cd bitcoin-node

# Download Bitcoin Core (adjust version)
wget https://bitcoincore.org/bin/bitcoin-core-25.0/bitcoin-25.0-x86_64-linux-gnu.tar.gz

# Extract
tar -xvf bitcoin-25.0-x86_64-linux-gnu.tar.gz
sudo mv bitcoin-25.0/bin/* /usr/local/bin/

# Verify
bitcoin-cli --version
```

### Step 2.2: Configure Bitcoin Node

```bash
# Create config
mkdir -p ~/.bitcoin
cat > ~/.bitcoin/bitcoin.conf << EOF
# Network
regtest=1
server=1

# RPC Settings
rpcuser=bitcoinrpc
rpcpassword=$(openssl rand -hex 32)
rpcallowip=127.0.0.1
rpcport=18443

# Mining
blockfilterindex=1
txindex=1

# Performance
maxmempool=50
EOF

echo "✅ Bitcoin config created"
echo "⚠️  IMPORTANT: Save your RPC password from ~/.bitcoin/bitcoin.conf"
```

### Step 2.3: Start Bitcoin Node

```bash
# Start node
bitcoind -regtest -daemon

# Wait for startup
sleep 5

# Check status
bitcoin-cli -regtest getblockchaininfo

# Create wallet
bitcoin-cli -regtest createwallet "mining_pool_wallet"

# Generate address for mining
MINING_ADDRESS=$(bitcoin-cli -regtest getnewaddress)
echo "Mining address: $MINING_ADDRESS"

# Mine initial blocks (need 101 for coins to be spendable)
bitcoin-cli -regtest generatetoaddress 101 $MINING_ADDRESS

# Check balance
bitcoin-cli -regtest getbalance

echo "✅ Bitcoin node running with spendable coins"
```

**Checkpoint:** You should see balance of 50 BTC

---

## Phase 3: Smart Contracts

### Step 3.1: Setup Hardhat Project

```bash
cd ../smart-contracts

# Initialize project
npm init -y
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox
npm install @openzeppelin/contracts

# Initialize Hardhat
npx hardhat init
# Choose: "Create a TypeScript project"
```

### Step 3.2: Configure Hardhat

Create `.env`:
```bash
cat > .env << EOF
SEPOLIA_RPC_URL=https://sepolia.infura.io/v3/YOUR_INFURA_KEY
ETHEREUM_PRIVATE_KEY=0xYOUR_PRIVATE_KEY
ETHERSCAN_API_KEY=YOUR_ETHERSCAN_KEY
EOF

echo "⚠️  IMPORTANT: Update .env with your actual keys!"
```

### Step 3.3: Copy Contract Files

Copy the contract code from `smart-contracts/README.md` into:
- `contracts/TestWBTC.sol`
- `contracts/WBTCVault.sol`
- `contracts/RewardToken.sol`

### Step 3.4: Deploy Contracts

```bash
# Compile contracts
npx hardhat compile

# Run tests (optional but recommended)
npx hardhat test

# Deploy to local network for testing
npx hardhat node &
sleep 5
npx hardhat run scripts/deploy.js --network localhost

# Deploy to Sepolia testnet
npx hardhat run scripts/deploy.js --network sepolia

# Save addresses!
# Output will show:
# TestWBTC: 0x...
# WBTCVault: 0x...
# RewardToken: 0x...

echo "✅ Smart contracts deployed"
echo "📝 Save these addresses - you'll need them later!"
```

**Checkpoint:** You should have 3 contract addresses on Sepolia

### Step 3.5: Verify Contracts (Optional)

```bash
# Verify on Etherscan
npx hardhat verify --network sepolia 0xYOUR_CONTRACT_ADDRESS

# Check on Sepolia Etherscan
# https://sepolia.etherscan.io/address/0xYOUR_CONTRACT_ADDRESS
```

---

## Phase 4: Mining Pool

### Step 4.1: Setup Mining Pool

```bash
cd ../mining-pool

# Initialize project
npm init -y

# Install dependencies
npm install express bitcoind-rpc pg redis ws dotenv
npm install --save-dev nodemon typescript @types/node

# Create TypeScript config
npx tsc --init
```

### Step 4.2: Create Configuration

```bash
# Create .env
cat > .env << EOF
# Bitcoin Node
BITCOIN_RPC_HOST=localhost
BITCOIN_RPC_PORT=18443
BITCOIN_RPC_USER=bitcoinrpc
BITCOIN_RPC_PASSWORD=YOUR_PASSWORD_FROM_BITCOIN_CONF

# Pool Settings
POOL_PORT=3333
POOL_DIFFICULTY=1

# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/mining_pool

# TEE Component
TEE_VALIDATOR_URL=http://localhost:8080/validate

# Bridge Service
BRIDGE_SERVICE_URL=http://localhost:3500/new-block
EOF

echo "⚠️  Update BITCOIN_RPC_PASSWORD with actual password"
```

### Step 4.3: Setup Database

```bash
# Connect to PostgreSQL
psql -U postgres -d mining_pool << EOF
CREATE TABLE shares (
  id SERIAL PRIMARY KEY,
  worker VARCHAR(255) NOT NULL,
  difficulty NUMERIC NOT NULL,
  timestamp TIMESTAMPTZ DEFAULT NOW(),
  is_valid BOOLEAN DEFAULT true,
  block_hash VARCHAR(64),
  reward_calculated BOOLEAN DEFAULT false
);

CREATE TABLE blocks (
  id SERIAL PRIMARY KEY,
  block_hash VARCHAR(64) UNIQUE NOT NULL,
  height INTEGER NOT NULL,
  finder VARCHAR(255),
  timestamp TIMESTAMPTZ DEFAULT NOW(),
  reward NUMERIC,
  confirmed BOOLEAN DEFAULT false
);

CREATE TABLE miners (
  id SERIAL PRIMARY KEY,
  address VARCHAR(255) UNIQUE NOT NULL,
  total_shares INTEGER DEFAULT 0,
  last_active TIMESTAMPTZ,
  total_earned NUMERIC DEFAULT 0
);

CREATE INDEX idx_shares_worker ON shares(worker);
CREATE INDEX idx_shares_timestamp ON shares(timestamp);
CREATE INDEX idx_blocks_height ON blocks(height);
EOF

echo "✅ Mining pool database ready"
```

### Step 4.4: Implement Pool Server

For MVP, you can use an existing pool implementation or build a simple one. 

**Quick Start Option - Use existing pool:**

```bash
# Clone a basic Stratum pool
git clone https://github.com/zone117x/node-stratum-pool.git temp-pool
cp -r temp-pool/lib ./src/
rm -rf temp-pool

# Modify for your needs (see mining-pool/README.md for details)
```

**Or implement from scratch** (see `mining-pool/README.md` for full code examples)

### Step 4.5: Start Mining Pool

```bash
# Start pool
npm run dev

# In another terminal, check it's running
netstat -an | grep 3333
# Should show: LISTEN on 0.0.0.0:3333

echo "✅ Mining pool running on port 3333"
```

**Checkpoint:** Pool should be accepting connections on port 3333

---

## Phase 5: TEE Component

### Step 5.1: Choose TEE Implementation

For MVP, we recommend **TEE Simulator** (Option A):
- Faster development
- No special hardware needed
- Can migrate to real TEE later

### Step 5.2: Setup TEE Simulator

```bash
cd ../tee-component

# Initialize Rust project
cargo init --name tee-simulator

# Add dependencies to Cargo.toml
cat >> Cargo.toml << EOF
[dependencies]
tokio = { version = "1.32", features = ["full"] }
axum = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio-native-tls"] }
ring = "0.17"
hex = "0.4"
chrono = "0.4"
dotenv = "0.15"
EOF
```

### Step 5.3: Generate Signing Keys

```bash
# Generate keys
mkdir -p keys
openssl ecparam -name secp256k1 -genkey -noout -out keys/private.pem
openssl ec -in keys/private.pem -pubout -out keys/public.pem

chmod 600 keys/private.pem

echo "✅ TEE signing keys generated"
```

### Step 5.4: Create Configuration

```bash
cat > config.toml << EOF
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
EOF
```

### Step 5.5: Setup Database

```bash
psql -U postgres -d tee_audit << EOF
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
EOF

echo "✅ TEE audit database ready"
```

### Step 5.6: Implement TEE Simulator

Create `src/main.rs` with the implementation from `tee-component/README.md`

### Step 5.7: Build and Run

```bash
# Build
cargo build --release

# Run
./target/release/tee-simulator

# Test
curl http://localhost:8080/attestation

echo "✅ TEE component running on port 8080"
```

**Checkpoint:** TEE should respond to health checks

---

## Phase 6: Bridge Service

### Step 6.1: Setup Bridge Service

```bash
cd ../bridge-service

# Initialize project
npm init -y

# Install dependencies
npm install express bitcoinjs-lib ethers pg axios dotenv
npm install --save-dev nodemon typescript
```

### Step 6.2: Configure Bridge

```bash
cat > .env << EOF
# Bitcoin
BITCOIN_RPC_HOST=localhost
BITCOIN_RPC_PORT=18443
BITCOIN_RPC_USER=bitcoinrpc
BITCOIN_RPC_PASSWORD=YOUR_PASSWORD
BITCOIN_WALLET_NAME=mining_pool_wallet

# Ethereum
ETHEREUM_RPC_URL=https://sepolia.infura.io/v3/YOUR_KEY
VAULT_CONTRACT_ADDRESS=0xYOUR_VAULT_ADDRESS
TOKEN_CONTRACT_ADDRESS=0xYOUR_TOKEN_ADDRESS
TEST_WBTC_ADDRESS=0xYOUR_WBTC_ADDRESS
ETHEREUM_PRIVATE_KEY=0xYOUR_PRIVATE_KEY

# Bridge
BRIDGE_SERVICE_PORT=3500
BRIDGE_MODE=manual
CHECK_INTERVAL_SECONDS=60
MIN_BTC_THRESHOLD=0.001

# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/bridge_service
EOF

echo "⚠️  Update with your actual values!"
```

### Step 6.3: Setup Database

```bash
psql -U postgres -d bridge_service << EOF
CREATE TABLE conversions (
  id SERIAL PRIMARY KEY,
  btc_txid VARCHAR(64) UNIQUE NOT NULL,
  btc_amount NUMERIC NOT NULL,
  btc_confirmations INTEGER DEFAULT 0,
  wbtc_txid VARCHAR(66),
  wbtc_amount NUMERIC,
  eth_block_number INTEGER,
  status VARCHAR(50) DEFAULT 'pending',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE minting_operations (
  id SERIAL PRIMARY KEY,
  conversion_id INTEGER REFERENCES conversions(id),
  block_hash VARCHAR(64),
  rewards JSONB NOT NULL,
  mint_txid VARCHAR(66),
  status VARCHAR(50) DEFAULT 'pending',
  created_at TIMESTAMPTZ DEFAULT NOW()
);
EOF

echo "✅ Bridge database ready"
```

### Step 6.4: Implement Bridge

Implement the code from `bridge-service/README.md`

### Step 6.5: Start Bridge Service

```bash
# Start bridge
npm run dev

# Test
curl http://localhost:3500/api/status

echo "✅ Bridge service running on port 3500"
```

**Checkpoint:** Bridge should monitor Bitcoin wallet

---

## Phase 7: Miners

### Step 7.1: Install cpuminer

```bash
cd ../miners

# Install dependencies
sudo apt-get install build-essential libcurl4-openssl-dev libssl-dev git autoconf automake

# Clone cpuminer-multi
git clone https://github.com/tpruvot/cpuminer-multi.git
cd cpuminer-multi

# Compile
./autogen.sh
./configure CFLAGS="-O3"
make

# Install
sudo make install

# Verify
cpuminer --version

cd ..
```

### Step 7.2: Configure Miners

```bash
# Miner 1
cat > miner1/config.json << EOF
{
  "url": "stratum+tcp://localhost:3333",
  "user": "miner1_address",
  "pass": "miner1",
  "algo": "sha256d",
  "threads": 2
}
EOF

# Miner 2
cat > miner2/config.json << EOF
{
  "url": "stratum+tcp://localhost:3333",
  "user": "miner2_address",
  "pass": "miner2",
  "algo": "sha256d",
  "threads": 2
}
EOF

# Miner 3
cat > miner3/config.json << EOF
{
  "url": "stratum+tcp://localhost:3333",
  "user": "miner3_address",
  "pass": "miner3",
  "algo": "sha256d",
  "threads": 1
}
EOF
```

### Step 7.3: Create Start Script

```bash
cat > scripts/start-miners.sh << EOF
#!/bin/bash
echo "Starting miners..."

cd miner1
cpuminer --config=config.json > ../logs/miner1.log 2>&1 &
echo "Miner 1 started (PID: \$!)"

cd ../miner2
cpuminer --config=config.json > ../logs/miner2.log 2>&1 &
echo "Miner 2 started (PID: \$!)"

cd ../miner3
cpuminer --config=config.json > ../logs/miner3.log 2>&1 &
echo "Miner 3 started (PID: \$!)"

cd ..
echo "✅ All miners started"
echo "Check logs: tail -f logs/miner*.log"
EOF

chmod +x scripts/start-miners.sh
mkdir -p logs
```

### Step 7.4: Start Miners

```bash
./scripts/start-miners.sh

# Check miners are running
ps aux | grep cpuminer

# Watch logs
tail -f logs/miner1.log
# Should see: "Stratum session connected"
# Should see: "accepted: 1/1"

echo "✅ Miners are hashing!"
```

**Checkpoint:** Miners should connect to pool and submit shares

---

## Phase 8: End-to-End Testing

### Step 8.1: Verify All Components

```bash
# Create verification script
cat > scripts/verify-all.sh << EOF
#!/bin/bash
echo "Verifying all components..."

echo -n "Bitcoin node: "
bitcoin-cli -regtest getblockchaininfo > /dev/null 2>&1 && echo "✅" || echo "❌"

echo -n "Mining pool: "
nc -zv localhost 3333 > /dev/null 2>&1 && echo "✅" || echo "❌"

echo -n "TEE component: "
curl -s http://localhost:8080/health > /dev/null && echo "✅" || echo "❌"

echo -n "Bridge service: "
curl -s http://localhost:3500/api/status > /dev/null && echo "✅" || echo "❌"

echo -n "Miners: "
pgrep cpuminer > /dev/null && echo "✅ (\$(pgrep cpuminer | wc -l) running)" || echo "❌"

echo -n "Smart contracts: "
# Check if contracts are deployed
# (Add your check here)
echo "⚠️  Manual check required"
EOF

chmod +x scripts/verify-all.sh
./scripts/verify-all.sh
```

### Step 8.2: Test Mining Flow

```bash
# 1. Monitor pool for shares
tail -f mining-pool/logs/pool.log &

# 2. Monitor TEE for validations
tail -f tee-component/logs/tee.log &

# 3. Wait for miners to submit shares (should be within 30 seconds)
sleep 60

# 4. Check database for shares
psql -U postgres -d mining_pool -c "SELECT COUNT(*) as total_shares, COUNT(DISTINCT worker) as active_miners FROM shares;"

# Expected: shares > 0, active_miners = 3
```

### Step 8.3: Test Block Finding

```bash
# In regtest, we can mine blocks instantly
# This simulates a block being found

# 1. Mine a block to the pool address
POOL_ADDRESS=$(bitcoin-cli -regtest getnewaddress)
bitcoin-cli -regtest generatetoaddress 1 $POOL_ADDRESS

# 2. Check pool detected the block
curl http://localhost:3333/api/blocks

# 3. Check bridge service detected new BTC
curl http://localhost:3500/api/btc-wallet
```

### Step 8.4: Test BTC → WBTC → Token Flow

This is the most important test!

```bash
# 1. Check Bitcoin balance
BALANCE=$(bitcoin-cli -regtest getbalance)
echo "Pool BTC balance: $BALANCE"

# 2. Manual conversion (MVP mode)
curl -X POST http://localhost:3500/api/convert \
  -H "Content-Type: application/json" \
  -d "{
    \"btc_amount\": 0.001,
    \"btc_txid\": \"manual_test\"
  }"

# 3. For testing, mint test WBTC and deposit to vault
# (Use the testWBTC contract's mint function)

# 4. Trigger token minting
curl -X POST http://localhost:3500/api/mint-tokens \
  -H "Content-Type: application/json" \
  -d "{
    \"conversion_id\": 1,
    \"block_hash\": \"00000000test\",
    \"rewards\": [
      {\"miner\": \"0xMiner1Address\", \"amount\": 0.0003},
      {\"miner\": \"0xMiner2Address\", \"amount\": 0.0007}
    ]
  }"

# 5. Verify tokens were minted
# Check Sepolia Etherscan for the transaction
# Or use:
npx hardhat console --network sepolia
> const token = await ethers.getContractAt("RewardToken", "0xYOUR_TOKEN_ADDRESS");
> await token.balanceOf("0xMiner1Address");
```

### Step 8.5: Test Token Redemption

```bash
# 1. Approve vault to burn tokens
npx hardhat console --network sepolia
> const token = await ethers.getContractAt("RewardToken", "0xYOUR_TOKEN_ADDRESS");
> const vault = await ethers.getContractAt("WBTCVault", "0xYOUR_VAULT_ADDRESS");
> const [signer] = await ethers.getSigners();
> await token.approve(await vault.getAddress(), ethers.parseUnits("50", 18));

# 2. Redeem tokens
> await vault.redeem(ethers.parseUnits("50", 18));

# 3. Check WBTC balance
> const wbtc = await ethers.getContractAt("TestWBTC", "0xYOUR_WBTC_ADDRESS");
> await wbtc.balanceOf(signer.address);
```

### Step 8.6: Performance Testing

```bash
# Monitor system resources
htop &

# Monitor pool performance
watch -n 5 'psql -U postgres -d mining_pool -c "SELECT COUNT(*) as shares_last_minute FROM shares WHERE timestamp > NOW() - INTERVAL '1 minute';"'

# Check hash rates
curl http://localhost:3333/api/stats

# Expected results:
# - 3 active miners
# - 5-15 shares per minute (CPU mining)
# - All shares validated by TEE
# - No errors in logs
```

---

## Troubleshooting

### Issue: Miners not connecting

**Symptoms:** Miners show "Connection refused"

**Solutions:**
```bash
# Check pool is running
netstat -an | grep 3333

# Check pool logs
tail -f mining-pool/logs/pool.log

# Restart pool
cd mining-pool && npm run dev

# Test connection
telnet localhost 3333
```

### Issue: No shares being submitted

**Symptoms:** Miners connected but no shares

**Solutions:**
```bash
# Check difficulty is not too high
# For regtest, difficulty should be 1

# Check miner logs
tail -f miners/logs/miner1.log

# Restart miners
killall cpuminer
./miners/scripts/start-miners.sh
```

### Issue: TEE validation failing

**Symptoms:** All shares rejected

**Solutions:**
```bash
# Check TEE is running
curl http://localhost:8080/health

# Check TEE logs
tail -f tee-component/logs/tee.log

# Restart TEE
cd tee-component
cargo run --release

# Test validation manually
curl -X POST http://localhost:8080/validate \
  -H "Content-Type: application/json" \
  -d '{"share": {...}, "block_template": {...}}'
```

### Issue: Smart contract calls failing

**Symptoms:** "Transaction reverted" errors

**Solutions:**
```bash
# Check you have Sepolia ETH
npx hardhat console --network sepolia
> await ethers.provider.getBalance("0xYOUR_ADDRESS")

# Check contract addresses are correct
> await token.getAddress()

# Check you have the right role
> await token.hasRole(await token.MINTER_ROLE(), "0xYOUR_ADDRESS")

# Get Sepolia ETH from faucet
# https://sepoliafaucet.com/
```

### Issue: Bridge not detecting BTC

**Symptoms:** Bridge shows balance 0

**Solutions:**
```bash
# Check Bitcoin RPC connection
bitcoin-cli -regtest getbalance

# Check bridge logs
tail -f bridge-service/logs/bridge.log

# Manually trigger check
curl http://localhost:3500/api/btc-wallet

# Verify Bitcoin wallet name
bitcoin-cli -regtest listwallets
```

### Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| "Connection refused" | Service not running | Check service is started |
| "Authentication failed" | Wrong RPC credentials | Check .env file |
| "Insufficient funds" | Not enough coins/gas | Mine blocks or get testnet ETH |
| "Invalid address" | Wrong format | Verify address format |
| "Transaction reverted" | Smart contract error | Check contract state and permissions |

---

## Next Steps

Congratulations! You now have a working MVP. Here's what to do next:

### Short Term (1-2 weeks)

1. **Improve Monitoring**
   - Set up Grafana dashboard
   - Add Prometheus metrics
   - Set up alerts

2. **Add Dashboard**
   - Build web UI to visualize:
     - Active miners
     - Hash rate
     - Blocks found
     - Token balances

3. **Testing**
   - Load testing with more miners
   - Stress testing with high hash rate
   - Security testing

4. **Documentation**
   - API documentation
   - Architecture diagrams
   - User guides

### Medium Term (1-2 months)

1. **Migrate to Real TEE**
   - Deploy Phat Contract on Phala testnet
   - Implement real remote attestation
   - Test with actual SGX hardware

2. **Automate Bridge**
   - Implement automated conversions
   - Add batching for efficiency
   - Set up monitoring

3. **Enhance Smart Contracts**
   - Add governance features
   - Implement timelocks
   - Add emergency mechanisms

4. **Testnet Deployment**
   - Move to Bitcoin testnet
   - Test with real network difficulty
   - Optimize for higher loads

### Long Term (3-6 months)

1. **Production Preparation**
   - Security audits
   - Performance optimization
   - Scalability improvements

2. **Mainnet Deployment**
   - Deploy to Ethereum mainnet
   - Use real WBTC
   - Launch with limited capacity

3. **Advanced Features**
   - DeFi integrations
   - Governance DAO
   - Mobile apps
   - Advanced analytics

---

## Success Metrics

Your MVP is successful if you can demonstrate:

✅ **Mining:** 3+ miners successfully mining blocks on regtest  
✅ **Pool:** Shares validated and recorded accurately  
✅ **TEE:** All operations cryptographically signed and auditable  
✅ **Bridge:** BTC successfully converted to WBTC  
✅ **Tokens:** Reward tokens minted and distributed to miners  
✅ **Redemption:** Tokens successfully redeemed for WBTC  
✅ **Transparency:** All operations visible on block explorers  
✅ **Decentralization:** No single point of failure (conceptually proven)  

---

## Additional Resources

### Documentation
- Each component has detailed README in its folder
- See `MVP_FEASIBILITY_ANALYSIS.md` for architecture details
- See original `README.md` for full system vision

### Community
- Bitcoin Core: https://bitcoin.org/en/developer-documentation
- Ethereum: https://ethereum.org/en/developers/
- Phala Network: https://docs.phala.network/
- OpenZeppelin: https://docs.openzeppelin.com/

### Support
- GitHub Issues: [Create issues for bugs/questions]
- Discord: [If you set up a community]
- Email: [Your contact]

---

## Conclusion

You've now built a working prototype of a decentralized TEE-based Bitcoin mining pool with tokenized rewards! This MVP demonstrates:

- Bitcoin mining pool operations
- TEE-based trust and verification
- Cross-chain interoperability (Bitcoin ↔ Ethereum)
- Tokenized reward system
- Smart contract integration

While this is an MVP and not production-ready, it proves the feasibility of the concept and provides a solid foundation for further development.

**Remember:** This is just the beginning. The full vision includes:
- Real Phala Network integration
- Mainnet deployment
- Decentralized governance
- Advanced DeFi features
- Global miner network

Good luck building the future of decentralized mining! 🚀

---

*For questions or issues, refer to individual component READMEs or create an issue in the project repository.*
