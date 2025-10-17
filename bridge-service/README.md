# Bridge Service Component

## Overview

The bridge service converts mined Bitcoin (BTC) into Wrapped Bitcoin (WBTC) on Ethereum and coordinates with smart contracts to mint reward tokens. This is the critical link between the Bitcoin mining rewards and the Ethereum-based tokenized reward system.

## What This Component Does

- **Monitors Bitcoin wallet** for newly mined BTC
- **Converts BTC to WBTC** via official WBTC protocol or testnet bridge
- **Deposits WBTC** into the vault smart contract
- **Triggers token minting** based on reward calculations
- **Tracks conversion status** for transparency

## Architecture Role

```
[Mining Pool finds block]
         ↓
[BTC sent to pool wallet] --monitored by--> [Bridge Service]
         ↓                                          ↓
[BTC Accumulated]                          [Converts to WBTC]
         ↓                                          ↓
[WBTC deposited] -----------------------> [WBTC Vault Contract]
                                                    ↓
                                          [Reward Tokens Minted]
```

## Technology Stack

**Recommended for MVP:**
- **Language:** Node.js (TypeScript) or Python
- **Bitcoin Library:** bitcoinjs-lib or python-bitcoinlib
- **Ethereum Library:** ethers.js or web3.py
- **Database:** PostgreSQL for transaction tracking
- **Queue:** Redis for async processing

## Prerequisites

- Node.js 18+ or Python 3.10+
- Bitcoin Core node (with wallet)
- Ethereum node (Infura/Alchemy or local)
- Private keys for Bitcoin and Ethereum wallets
- PostgreSQL for state tracking

## Installation

### Option A: Node.js/TypeScript

1. **Install dependencies:**
   ```bash
   cd bridge-service
   npm install
   ```

   **key packages (package.json):**
   ```json
   {
     "dependencies": {
       "bitcoinjs-lib": "^6.1.0",
       "ethers": "^6.7.0",
       "dotenv": "^16.3.0",
       "pg": "^8.11.0",
       "axios": "^1.5.0",
       "express": "^4.18.2"
     }
   }
   ```

2. **Set up environment:**
   ```bash
   cp .env.example .env
   nano .env
   ```

   ```env
   # Bitcoin Configuration
   BITCOIN_RPC_HOST=localhost
   BITCOIN_RPC_PORT=18443
   BITCOIN_RPC_USER=bitcoinrpc
   BITCOIN_RPC_PASSWORD=changeme123
   BITCOIN_WALLET_NAME=mining_pool_wallet
   BITCOIN_NETWORK=regtest
   
   # Ethereum Configuration
   ETHEREUM_RPC_URL=https://sepolia.infura.io/v3/YOUR_KEY
   ETHEREUM_CHAIN_ID=11155111
   VAULT_CONTRACT_ADDRESS=0x...
   TOKEN_CONTRACT_ADDRESS=0x...
   
   # Private Keys (KEEP SECURE!)
   BITCOIN_PRIVATE_KEY=cT...
   ETHEREUM_PRIVATE_KEY=0x...
   
   # WBTC Bridge
   WBTC_BRIDGE_TYPE=manual  # manual | testnet | mainnet
   WBTC_MERCHANT_ADDRESS=0x...
   
   # Service Configuration
   BRIDGE_SERVICE_PORT=3500
   CHECK_INTERVAL_SECONDS=60
   MIN_BTC_THRESHOLD=0.001
   MAX_GAS_PRICE_GWEI=50
   
   # Database
   DATABASE_URL=postgresql://postgres:password@localhost:5432/bridge_service
   ```

3. **Setup database:**
   ```bash
   psql -U postgres
   CREATE DATABASE bridge_service;
   ```

   ```sql
   -- Conversions tracking
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
   
   -- Token minting tracking
   CREATE TABLE minting_operations (
     id SERIAL PRIMARY KEY,
     conversion_id INTEGER REFERENCES conversions(id),
     block_hash VARCHAR(64),
     rewards JSONB NOT NULL,
     mint_txid VARCHAR(66),
     status VARCHAR(50) DEFAULT 'pending',
     created_at TIMESTAMPTZ DEFAULT NOW()
   );
   
   -- Status: pending, converting, converted, minting, completed, failed
   ```

### Option B: Python

```bash
cd bridge-service
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

**requirements.txt:**
```
python-bitcoinrpc==1.0
web3==6.10.0
python-dotenv==1.0.0
psycopg2-binary==2.9.7
requests==2.31.0
fastapi==0.104.0
uvicorn==0.24.0
```

## Configuration

### Bridge Configuration (`config/bridge.json`)

```json
{
  "bitcoin": {
    "network": "regtest",
    "rpc": {
      "host": "localhost",
      "port": 18443,
      "user": "bitcoinrpc",
      "password": "changeme123"
    },
    "wallet": "mining_pool_wallet",
    "minConfirmations": 1
  },
  "ethereum": {
    "network": "sepolia",
    "rpcUrl": "https://sepolia.infura.io/v3/YOUR_KEY",
    "chainId": 11155111,
    "contracts": {
      "vault": "0x...",
      "token": "0x...",
      "testWBTC": "0x..."
    }
  },
  "bridge": {
    "mode": "manual",
    "checkInterval": 60,
    "minBTCThreshold": 0.001,
    "batchConversions": true,
    "autoConfirm": false
  },
  "fees": {
    "maxGasPriceGwei": 50,
    "priorityFeeGwei": 2
  }
}
```

## Bridge Modes

### Mode 1: Manual (Recommended for MVP)

**How it works:**
1. Service monitors Bitcoin wallet
2. When BTC threshold reached, notifies admin
3. Admin manually converts BTC to WBTC via official process
4. Admin manually deposits WBTC to vault
5. Service detects WBTC deposit and triggers token minting

**Pros:**
- ✅ Simple to implement
- ✅ Full control over conversions
- ✅ No risk of automated errors
- ✅ Good for testing

**Cons:**
- ❌ Requires manual intervention
- ❌ Slower process
- ❌ Not scalable

**Setup:**
```javascript
// Set in config
"bridge": {
  "mode": "manual",
  "autoConfirm": false
}
```

### Mode 2: Semi-Automated

**How it works:**
1. Service monitors Bitcoin wallet
2. Automatically initiates WBTC conversion request
3. Waits for admin approval
4. Executes conversion after approval
5. Auto-deposits to vault
6. Auto-triggers token minting

**Pros:**
- ✅ Faster than manual
- ✅ Still has human oversight
- ✅ Good middle ground

**Cons:**
- ⚠️ More complex
- ⚠️ Requires approval system

**Setup:**
```javascript
"bridge": {
  "mode": "semi-auto",
  "requireApproval": true
}
```

### Mode 3: Fully Automated (Future)

**How it works:**
1. Service monitors Bitcoin wallet
2. Automatically converts BTC to WBTC
3. Automatically deposits to vault
4. Automatically mints tokens

**Pros:**
- ✅ Fully automated
- ✅ Fast and scalable

**Cons:**
- ❌ Complex implementation
- ❌ Requires trusted setup
- ❌ Higher risk if bugs

**Not recommended for MVP**

## Usage

### Starting the Bridge Service

**Development:**
```bash
npm run dev
# or
python main.py --dev
```

**Production:**
```bash
npm start
# or
python main.py
```

**Docker:**
```bash
cd bridge-service
docker-compose up -d
```

### API Endpoints

**1. Check Status:**
```bash
GET /api/status

Response:
{
  "btc_balance": 0.0052,
  "btc_pending": 0.001,
  "wbtc_balance": 0.0042,
  "pending_conversions": 1,
  "last_check": "2025-10-17T10:30:00Z"
}
```

**2. Monitor BTC Wallet:**
```bash
GET /api/btc-wallet

Response:
{
  "address": "bcrt1q...",
  "balance": 0.0052,
  "unconfirmed": 0.001,
  "transactions": [...]
}
```

**3. Start Conversion (Manual):**
```bash
POST /api/convert
Content-Type: application/json

{
  "btc_amount": 0.001,
  "btc_txid": "abc123..."
}

Response:
{
  "conversion_id": 42,
  "status": "pending",
  "btc_amount": 0.001,
  "estimated_wbtc": 0.001,
  "instructions": "Send BTC to merchant address: ..."
}
```

**4. Confirm WBTC Received:**
```bash
POST /api/confirm-wbtc
Content-Type: application/json

{
  "conversion_id": 42,
  "wbtc_txid": "0xabc123..."
}

Response:
{
  "conversion_id": 42,
  "status": "converted",
  "wbtc_amount": 0.001,
  "vault_deposit_pending": true
}
```

**5. Trigger Token Minting:**
```bash
POST /api/mint-tokens
Content-Type: application/json

{
  "conversion_id": 42,
  "block_hash": "00000000abc...",
  "rewards": [
    {"miner": "0x...", "amount": 0.0003},
    {"miner": "0x...", "amount": 0.0007}
  ]
}

Response:
{
  "mint_id": 15,
  "tx_hash": "0xdef456...",
  "gas_used": 150000,
  "status": "pending"
}
```

**6. Get Conversion History:**
```bash
GET /api/conversions?limit=10&status=completed

Response:
{
  "conversions": [
    {
      "id": 42,
      "btc_amount": 0.001,
      "wbtc_amount": 0.001,
      "status": "completed",
      "created_at": "2025-10-17T10:00:00Z",
      "completed_at": "2025-10-17T10:30:00Z"
    }
  ]
}
```

## Implementation Details

### 1. Bitcoin Wallet Monitor (`src/btc-monitor.js`)

```javascript
const Bitcoin = require('bitcoinjs-lib');
const bitcoinRPC = require('bitcoin-core');

class BTCMonitor {
  constructor(config) {
    this.rpc = new bitcoinRPC(config.bitcoin.rpc);
    this.wallet = config.bitcoin.wallet;
  }
  
  async checkBalance() {
    const balance = await this.rpc.getBalance(this.wallet);
    return balance;
  }
  
  async getNewTransactions() {
    const txs = await this.rpc.listTransactions(this.wallet, 100);
    return txs.filter(tx => tx.category === 'receive');
  }
  
  async monitorWallet() {
    setInterval(async () => {
      const balance = await this.checkBalance();
      const newTxs = await this.getNewTransactions();
      
      if (balance >= MIN_BTC_THRESHOLD) {
        await this.notifyConversionNeeded(balance);
      }
      
      for (const tx of newTxs) {
        await this.processTransaction(tx);
      }
    }, CHECK_INTERVAL_SECONDS * 1000);
  }
}
```

### 2. WBTC Converter (`src/wbtc-converter.js`)

**Manual Mode:**
```javascript
class WBTCConverter {
  async startManualConversion(btcAmount, btcTxid) {
    // 1. Create conversion record
    const conversion = await db.query(
      'INSERT INTO conversions (btc_txid, btc_amount, status) VALUES ($1, $2, $3) RETURNING *',
      [btcTxid, btcAmount, 'pending']
    );
    
    // 2. Generate instructions for admin
    const instructions = {
      step1: 'Send BTC to WBTC merchant',
      merchantAddress: WBTC_MERCHANT_ADDRESS,
      amount: btcAmount,
      step2: 'Wait for WBTC to be minted',
      step3: 'Call /api/confirm-wbtc with WBTC txid'
    };
    
    // 3. Send notification
    await this.notifyAdmin(instructions);
    
    return { conversion, instructions };
  }
  
  async confirmWBTCReceived(conversionId, wbtcTxid) {
    // Update conversion record
    await db.query(
      'UPDATE conversions SET wbtc_txid = $1, status = $2 WHERE id = $3',
      [wbtcTxid, 'converted', conversionId]
    );
    
    // Deposit to vault
    await this.depositToVault(conversionId);
  }
}
```

### 3. Vault Depositor (`src/vault-depositor.js`)

```javascript
const { ethers } = require('ethers');

class VaultDepositor {
  constructor(config) {
    this.provider = new ethers.JsonRpcProvider(config.ethereum.rpcUrl);
    this.wallet = new ethers.Wallet(ETHEREUM_PRIVATE_KEY, this.provider);
    this.vaultContract = new ethers.Contract(
      VAULT_CONTRACT_ADDRESS,
      vaultABI,
      this.wallet
    );
  }
  
  async depositWBTC(amount, conversionId) {
    try {
      // 1. Approve WBTC spending
      const wbtcContract = new ethers.Contract(WBTC_ADDRESS, erc20ABI, this.wallet);
      const approveTx = await wbtcContract.approve(
        VAULT_CONTRACT_ADDRESS,
        ethers.parseUnits(amount.toString(), 8)
      );
      await approveTx.wait();
      
      // 2. Deposit to vault
      const depositTx = await this.vaultContract.deposit(
        ethers.parseUnits(amount.toString(), 8)
      );
      const receipt = await depositTx.wait();
      
      // 3. Update database
      await db.query(
        'UPDATE conversions SET status = $1, eth_block_number = $2 WHERE id = $3',
        ['deposited', receipt.blockNumber, conversionId]
      );
      
      return receipt;
    } catch (error) {
      console.error('Vault deposit failed:', error);
      throw error;
    }
  }
}
```

### 4. Token Minter (`src/token-minter.js`)

```javascript
class TokenMinter {
  async mintRewardTokens(conversionId, blockHash, rewards) {
    try {
      // 1. Calculate USD value
      const btcPrice = await this.getBTCPrice();
      const totalBTC = rewards.reduce((sum, r) => sum + r.amount, 0);
      const usdValue = totalBTC * btcPrice;
      
      // 2. Prepare minting data
      const recipients = rewards.map(r => r.miner);
      const amounts = rewards.map(r => {
        const minerUSDValue = r.amount * btcPrice;
        return ethers.parseUnits(minerUSDValue.toString(), 18);
      });
      
      // 3. Call mint function on token contract
      const tokenContract = new ethers.Contract(
        TOKEN_CONTRACT_ADDRESS,
        tokenABI,
        this.wallet
      );
      
      const mintTx = await tokenContract.batchMint(recipients, amounts);
      const receipt = await mintTx.wait();
      
      // 4. Record minting operation
      await db.query(
        'INSERT INTO minting_operations (conversion_id, block_hash, rewards, mint_txid, status) VALUES ($1, $2, $3, $4, $5)',
        [conversionId, blockHash, JSON.stringify(rewards), receipt.transactionHash, 'completed']
      );
      
      return receipt;
    } catch (error) {
      console.error('Token minting failed:', error);
      throw error;
    }
  }
  
  async getBTCPrice() {
    // For MVP: use fixed price or simple API
    // Production: use Chainlink oracle
    const response = await fetch('https://api.coinbase.com/v2/prices/BTC-USD/spot');
    const data = await response.json();
    return parseFloat(data.data.amount);
  }
}
```

## Testing the Bridge

### Test 1: Bitcoin Wallet Monitoring

```bash
# Mine some blocks to pool address
bitcoin-cli -regtest generatetoaddress 10 $(bitcoin-cli -regtest getnewaddress)

# Check bridge detected it
curl http://localhost:3500/api/btc-wallet

# Expected: balance > 0
```

### Test 2: Manual Conversion

```bash
# 1. Get BTC transaction
TXID=$(bitcoin-cli -regtest listtransactions | jq -r '.[0].txid')

# 2. Start conversion
curl -X POST http://localhost:3500/api/convert \
  -H "Content-Type: application/json" \
  -d "{\"btc_amount\": 0.001, \"btc_txid\": \"$TXID\"}"

# 3. Simulate WBTC receipt (for testing, use testnet WBTC)
# In production, this would be real WBTC merchant process

# 4. Confirm WBTC
curl -X POST http://localhost:3500/api/confirm-wbtc \
  -H "Content-Type: application/json" \
  -d '{"conversion_id": 1, "wbtc_txid": "0x123abc..."}'
```

### Test 3: Token Minting

```bash
curl -X POST http://localhost:3500/api/mint-tokens \
  -H "Content-Type: application/json" \
  -d '{
    "conversion_id": 1,
    "block_hash": "00000000abc123",
    "rewards": [
      {"miner": "0xMiner1Address", "amount": 0.0003},
      {"miner": "0xMiner2Address", "amount": 0.0007}
    ]
  }'

# Check Ethereum explorer for mint transaction
```

## Integration with Other Components

### Mining Pool Integration

```javascript
// Mining pool notifies bridge when block is found
await fetch('http://bridge-service:3500/api/new-block', {
  method: 'POST',
  body: JSON.stringify({
    blockHash: '00000000abc...',
    reward: 6.25,
    shares: minerShares
  })
});
```

### Smart Contract Integration

The bridge must interact with two contracts:

**WBTC Vault:**
- `deposit(uint256 amount)` - Deposit WBTC
- `getBalance()` - Check vault balance

**Reward Token:**
- `mint(address to, uint256 amount)` - Mint tokens
- `batchMint(address[] to, uint256[] amounts)` - Mint to multiple

## Monitoring & Alerts

### Health Checks

```bash
# Service health
curl http://localhost:3500/health

# Bitcoin connection
curl http://localhost:3500/api/btc-status

# Ethereum connection
curl http://localhost:3500/api/eth-status
```

### Logging

```javascript
// All operations logged
[2025-10-17 10:30:00] INFO: BTC wallet check - Balance: 0.005
[2025-10-17 10:30:15] INFO: New BTC transaction detected: abc123...
[2025-10-17 10:30:30] INFO: Conversion started: ID 42
[2025-10-17 10:35:00] INFO: WBTC deposited to vault: 0xdef456...
[2025-10-17 10:35:15] INFO: Tokens minted: 0xghi789...
```

## Common Issues & Solutions

### Issue 1: "Bitcoin RPC connection failed"
```bash
# Check Bitcoin node
bitcoin-cli -regtest getblockchaininfo

# Verify RPC credentials in .env
```

### Issue 2: "Ethereum transaction failed"
```bash
# Check gas price
# Check wallet has ETH for gas
# Check contract addresses are correct
```

### Issue 3: "WBTC balance mismatch"
```bash
# Verify WBTC was actually received
# Check Ethereum explorer
# Verify contract ABI is correct
```

## Security Considerations

### MVP Security (Acceptable Risks):
- ⚠️ Manual bridge (centralized)
- ⚠️ Single admin key
- ✅ Testnet only
- ✅ Transaction logging

### Production Security:
- ✅ Multi-sig wallets
- ✅ Time locks on large transfers
- ✅ Automated monitoring
- ✅ Insurance fund
- ✅ Hardware wallet integration

## Next Steps

Once bridge is running:

1. ✅ BTC wallet monitoring active
2. ✅ Conversion process tested
3. ➡️ Test full flow: BTC → WBTC → Vault → Tokens
4. ➡️ Integrate with coordinator service
5. ➡️ Set up monitoring dashboard

## Resources

- [WBTC Official Site](https://wbtc.network/)
- [WBTC Merchant Process](https://wbtc.network/dashboard/merchants)
- [bitcoinjs-lib](https://github.com/bitcoinjs/bitcoinjs-lib)
- [ethers.js](https://docs.ethers.org/)
- [ERC-20 Standard](https://eips.ethereum.org/EIPS/eip-20)

## Configuration Files

- `.env` - Environment variables
- `config/bridge.json` - Bridge configuration
- `src/btc-monitor.js` - Bitcoin monitoring
- `src/wbtc-converter.js` - Conversion logic
- `src/vault-depositor.js` - Vault interaction
- `src/token-minter.js` - Token minting

## Troubleshooting Checklist

- [ ] Bitcoin node is running
- [ ] Ethereum node/Infura is accessible
- [ ] Private keys are correctly configured
- [ ] Smart contracts are deployed
- [ ] Database is set up
- [ ] BTC wallet has funds
- [ ] ETH wallet has gas
- [ ] WBTC testnet tokens available
