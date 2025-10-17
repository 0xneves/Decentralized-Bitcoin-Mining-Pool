# Mining Pool Server Component

## Overview

The mining pool server is the **central coordinator** that connects Bitcoin miners to the Bitcoin network. It implements the **Stratum protocol** for miner communication, distributes mining work, validates submitted shares, and tracks contributions for reward calculation.

## What This Component Does

- **Accepts miner connections** via Stratum protocol (TCP port 3333)
- **Fetches block templates** from Bitcoin Core node via RPC
- **Distributes mining jobs** to connected miners
- **Validates shares** (partial proofs-of-work) submitted by miners
- **Tracks contributions** for reward calculation
- **Submits found blocks** to Bitcoin network
- **Communicates with TEE** for verifiable share validation

## Architecture Role

```
[Bitcoin Node] <--RPC--> [Mining Pool Server] <--Stratum--> [Miners]
                              ↓
                         [TEE Component]
                         (Share Validation)
                              ↓
                      [Reward Calculation]
```

## Technology Stack

**Recommended for MVP:**
- **Language:** Node.js (TypeScript) or Python
- **Framework:** node-stratum-pool (Node.js) or custom implementation
- **Database:** PostgreSQL for share tracking
- **Cache:** Redis for performance
- **Communication:** HTTP/gRPC to TEE component

**Alternatives:**
- Go (performance-focused)
- Rust (for TEE integration)

## Prerequisites

- Node.js 18+ or Python 3.10+
- PostgreSQL 14+
- Redis 7+ (optional but recommended)
- Bitcoin Core node running and accessible
- TEE component running (or simulator)

## Installation

### Option A: Node.js Implementation

1. **Install dependencies:**
   ```bash
   cd mining-pool
   npm install
   ```

   **Key packages:**
   ```json
   {
     "dependencies": {
       "stratum-pool": "^0.1.7",
       "bitcoind-rpc": "^0.9.1",
       "pg": "^8.11.0",
       "redis": "^4.6.0",
       "express": "^4.18.2",
       "ws": "^8.13.0"
     }
   }
   ```

2. **Set up database:**
   ```bash
   psql -U postgres
   CREATE DATABASE mining_pool;
   \c mining_pool
   ```

   ```sql
   -- Create shares table
   CREATE TABLE shares (
     id SERIAL PRIMARY KEY,
     worker VARCHAR(255) NOT NULL,
     difficulty NUMERIC NOT NULL,
     timestamp TIMESTAMPTZ DEFAULT NOW(),
     is_valid BOOLEAN DEFAULT true,
     block_hash VARCHAR(64),
     reward_calculated BOOLEAN DEFAULT false
   );
   
   -- Create blocks table
   CREATE TABLE blocks (
     id SERIAL PRIMARY KEY,
     block_hash VARCHAR(64) UNIQUE NOT NULL,
     height INTEGER NOT NULL,
     finder VARCHAR(255),
     timestamp TIMESTAMPTZ DEFAULT NOW(),
     reward NUMERIC,
     confirmed BOOLEAN DEFAULT false
   );
   
   -- Create miners table
   CREATE TABLE miners (
     id SERIAL PRIMARY KEY,
     address VARCHAR(255) UNIQUE NOT NULL,
     total_shares INTEGER DEFAULT 0,
     last_active TIMESTAMPTZ,
     total_earned NUMERIC DEFAULT 0
   );
   ```

3. **Configure environment:**
   ```bash
   cp .env.example .env
   nano .env
   ```

   ```env
   # Bitcoin Node
   BITCOIN_RPC_HOST=localhost
   BITCOIN_RPC_PORT=18443
   BITCOIN_RPC_USER=bitcoinrpc
   BITCOIN_RPC_PASSWORD=changeme123
   
   # Pool Settings
   POOL_PORT=3333
   POOL_DIFFICULTY=1
   BLOCK_TEMPLATE_INTERVAL=10000
   
   # Database
   DATABASE_URL=postgresql://postgres:password@localhost:5432/mining_pool
   
   # Redis
   REDIS_URL=redis://localhost:6379
   
   # TEE Component
   TEE_VALIDATOR_URL=http://localhost:8080/validate
   ```

### Option B: Python Implementation

1. **Install dependencies:**
   ```bash
   cd mining-pool
   python -m venv venv
   source venv/bin/activate  # Windows: venv\Scripts\activate
   pip install -r requirements.txt
   ```

   **requirements.txt:**
   ```
   python-bitcoinrpc==1.0
   sqlalchemy==2.0.0
   redis==5.0.0
   asyncio==3.4.3
   websockets==11.0
   fastapi==0.104.0
   uvicorn==0.24.0
   ```

2. **Run database migrations:**
   ```bash
   python manage.py migrate
   ```

### Option C: Docker

```bash
cd mining-pool
docker-compose up -d
```

## Configuration

### Pool Configuration (`config/pool.json`)

```json
{
  "pool": {
    "name": "TEE Mining Pool MVP",
    "host": "0.0.0.0",
    "port": 3333,
    "difficulty": 1,
    "varDiff": {
      "enabled": true,
      "minDiff": 0.5,
      "maxDiff": 16,
      "targetTime": 15,
      "retargetTime": 90
    }
  },
  "bitcoin": {
    "rpc": {
      "host": "localhost",
      "port": 18443,
      "user": "bitcoinrpc",
      "password": "changeme123"
    },
    "network": "regtest",
    "blockTemplateInterval": 10000
  },
  "database": {
    "host": "localhost",
    "port": 5432,
    "database": "mining_pool",
    "user": "postgres",
    "password": "password"
  },
  "tee": {
    "enabled": true,
    "validatorUrl": "http://localhost:8080/validate",
    "timeout": 5000
  },
  "rewards": {
    "method": "PPLNS",
    "fee": 0.01,
    "blockConfirmations": 100
  }
}
```

### Explanation of Key Settings

**Difficulty:**
- `difficulty: 1` - Easy for CPU miners on regtest
- Increase for testnet (256+)
- Dynamic difficulty (`varDiff`) adjusts per miner

**Variable Difficulty:**
- `targetTime: 15` - Target 15 seconds per share
- Pool auto-adjusts difficulty per miner

**Reward Method:**
- `PPLNS`: Pay Per Last N Shares (most fair)
- Alternative: PPS (Pay Per Share)

**Block Confirmations:**
- Regtest: 1-10 confirmations
- Testnet/Mainnet: 100+ confirmations

## Core Components

### 1. Stratum Server (`src/stratum-server.js`)

**Responsibilities:**
- Accept TCP connections from miners
- Handle Stratum protocol messages
- Manage miner sessions

**Key methods:**
- `handleSubscribe()` - Miner subscription
- `handleAuthorize()` - Miner authentication
- `handleSubmit()` - Share submission
- `broadcastJob()` - Send new work to miners

### 2. Block Template Manager (`src/block-template.js`)

**Responsibilities:**
- Fetch block templates from Bitcoin node
- Update on new blocks
- Manage coinbase transaction

**Key methods:**
- `getBlockTemplate()` - Fetch from Bitcoin Core
- `updateTemplate()` - Refresh on interval or new block
- `buildJob()` - Create Stratum job from template

### 3. Share Validator (`src/share-validator.js`)

**Responsibilities:**
- Validate submitted shares
- Check proof-of-work
- Communicate with TEE for verification

**Key methods:**
- `validateShare()` - Basic PoW validation
- `sendToTEE()` - TEE verification (optional for MVP)
- `recordShare()` - Store in database

### 4. Reward Calculator (`src/reward-calculator.js`)

**Responsibilities:**
- Calculate miner rewards
- Implement PPLNS/PPS algorithm
- Trigger token minting

**Key methods:**
- `calculatePPLNS()` - Reward distribution
- `getTotalShares()` - Share counting
- `notifyBridge()` - Inform bridge service

## Usage

### Starting the Pool

**Development mode:**
```bash
npm run dev
# or
python main.py --dev
```

**Production mode:**
```bash
npm start
# or
python main.py
```

**Docker:**
```bash
docker-compose up -d mining-pool
docker-compose logs -f mining-pool
```

### Verifying Pool is Running

1. **Check port is listening:**
   ```bash
   netstat -an | grep 3333
   # Expected: LISTEN on 0.0.0.0:3333
   ```

2. **Test Stratum connection:**
   ```bash
   telnet localhost 3333
   # Expected: Connection accepted
   ```

3. **Test miner connection:**
   ```bash
   cpuminer --url=stratum+tcp://localhost:3333 \
            --user=test_address \
            --pass=test
   # Expected: "Stratum session connected"
   ```

### Monitoring Pool Activity

**CLI Commands:**
```bash
# Connected miners
npm run cli miners

# Pool statistics
npm run cli stats

# Recent shares
npm run cli shares --limit 10

# Blocks found
npm run cli blocks
```

**Web Dashboard:**
```bash
# Access at http://localhost:3334
npm run dashboard
```

**Database Queries:**
```sql
-- Active miners
SELECT address, COUNT(*) as shares, MAX(timestamp) as last_seen
FROM shares
WHERE timestamp > NOW() - INTERVAL '1 hour'
GROUP BY address;

-- Pool hash rate (approximate)
SELECT SUM(difficulty) / 600 as hashrate_ths
FROM shares
WHERE timestamp > NOW() - INTERVAL '10 minutes';

-- Blocks found
SELECT * FROM blocks ORDER BY timestamp DESC LIMIT 10;
```

## API Endpoints

### Miner API (Stratum - TCP)

```json
// Subscribe
{ "id": 1, "method": "mining.subscribe", "params": ["cpuminer/1.0"] }

// Authorize
{ "id": 2, "method": "mining.authorize", "params": ["btc_address", "worker1"] }

// Submit share
{ "id": 3, "method": "mining.submit", "params": ["worker1", "job_id", "nonce", "result"] }
```

### Admin API (HTTP - Port 3334)

```bash
# Pool stats
GET /api/stats

# Miner list
GET /api/miners

# Shares
GET /api/shares?limit=100&offset=0

# Blocks
GET /api/blocks?confirmed=true

# Trigger payout calculation (manual for MVP)
POST /api/calculate-rewards
```

## Integration with Other Components

### Bitcoin Node Integration

```javascript
// Fetch block template
const template = await bitcoinRPC.getBlockTemplate();

// Submit found block
const result = await bitcoinRPC.submitBlock(blockHex);
```

### TEE Component Integration

```javascript
// Send share to TEE for validation
const teeResponse = await fetch(TEE_VALIDATOR_URL, {
  method: 'POST',
  body: JSON.stringify({
    share: shareData,
    miner: minerAddress,
    timestamp: Date.now()
  })
});

const { valid, attestation } = await teeResponse.json();
```

### Bridge Service Integration

```javascript
// Notify bridge when block is found
await fetch(BRIDGE_SERVICE_URL + '/new-block', {
  method: 'POST',
  body: JSON.stringify({
    blockHash: block.hash,
    reward: block.reward,
    shares: minerShares
  })
});
```

## Testing

### Unit Tests

```bash
npm test
# or
pytest tests/
```

### Integration Tests

```bash
# Start test environment
docker-compose -f docker-compose.test.yml up -d

# Run integration tests
npm run test:integration
```

### Manual Testing

1. **Test miner connection:**
   ```bash
   cpuminer --url=stratum+tcp://localhost:3333 --user=test --pass=test
   ```

2. **Check share recording:**
   ```sql
   SELECT * FROM shares ORDER BY timestamp DESC LIMIT 10;
   ```

3. **Simulate block find:**
   ```bash
   # Lower difficulty temporarily
   bitcoin-cli -regtest setgenerate false
   # Mine through pool
   ```

## Performance Optimization

### For MVP (3 miners):
- Single server instance
- Basic database indexing
- No caching needed

### For Production:
- Redis caching for block templates
- Connection pooling for database
- Load balancing across multiple pool servers
- Database read replicas

**Benchmarks:**
- Single instance: 1000-10000 miners
- With Redis: 10000-50000 miners
- Clustered: 100000+ miners

## Security Considerations

### MVP Security:
- ✅ Input validation on all Stratum messages
- ✅ Rate limiting per IP
- ✅ SQL injection prevention (parameterized queries)
- ⚠️ No TLS (development only)

### Production Security:
- ✅ TLS encryption for Stratum
- ✅ DDoS protection
- ✅ Multi-signature block submission
- ✅ TEE attestation verification

## Common Issues & Solutions

### Issue 1: "Failed to connect to Bitcoin RPC"
**Solution:**
```bash
# Verify Bitcoin node is running
bitcoin-cli -regtest getblockchaininfo

# Check RPC credentials in config
# Ensure firewall allows connection
```

### Issue 2: "All shares rejected"
**Solution:**
```bash
# Check difficulty setting
# Verify block template is fresh
# Ensure time sync between pool and Bitcoin node
```

### Issue 3: "Database connection failed"
**Solution:**
```bash
# Check PostgreSQL is running
sudo systemctl status postgresql

# Test connection
psql -U postgres -d mining_pool -c "SELECT 1;"
```

### Issue 4: "High latency for miners"
**Solution:**
```bash
# Enable Redis caching
# Reduce block template fetch interval
# Optimize database queries with indexes
```

## Logging & Debugging

**Enable debug logging:**
```javascript
// In config
"logging": {
  "level": "debug",
  "file": "./logs/pool.log"
}
```

**Watch logs:**
```bash
tail -f logs/pool.log | grep -i error
```

**Enable Stratum protocol dump:**
```javascript
// In stratum-server.js
const DEBUG_PROTOCOL = true;
```

## Next Steps

Once pool is running:

1. ✅ Pool accepts miner connections
2. ✅ Shares are validated and recorded
3. ➡️ Integrate with TEE component
4. ➡️ Test reward calculation
5. ➡️ Connect to bridge service

## Resources

- [Stratum Mining Protocol](https://braiins.com/stratum-v1/docs)
- [Bitcoin RPC Documentation](https://developer.bitcoin.org/reference/rpc/)
- [node-stratum-pool](https://github.com/zone117x/node-stratum-pool)
- [Mining Pool Algorithms](https://en.bitcoin.it/wiki/Comparison_of_mining_pools)

## Configuration Files

- `config/pool.json` - Main pool configuration
- `.env` - Environment variables
- `src/stratum-server.js` - Stratum protocol handler
- `src/share-validator.js` - Share validation logic
- `src/reward-calculator.js` - Reward distribution

## Troubleshooting Checklist

- [ ] Bitcoin node is running and accessible
- [ ] PostgreSQL database is set up
- [ ] Port 3333 is not in use
- [ ] RPC credentials are correct
- [ ] Miners can reach pool server (no firewall)
- [ ] Database tables are created
- [ ] TEE component is running (if enabled)
