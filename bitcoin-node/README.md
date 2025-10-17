# Bitcoin Node Component

## Overview

This component runs a Bitcoin Core node in **regtest mode** (regression test network), which is a local private blockchain for development and testing. In regtest mode, you have complete control over block generation and difficulty, making it ideal for MVP development.

## What This Component Does

- Runs a local Bitcoin blockchain with instant block generation
- Provides RPC interface for the mining pool to fetch block templates
- Receives and validates mined blocks from the pool
- Manages wallet for receiving mining rewards
- Allows manual block generation for testing

## Architecture Role

```
[Bitcoin Node (regtest)] <--RPC--> [Mining Pool Server]
         ↑
         | Submits found blocks
         |
    [Miners via Pool]
```

## Prerequisites

- Docker and Docker Compose (recommended) OR
- Bitcoin Core 25.0+ installed locally

## Installation Options

### Option A: Docker (Recommended for MVP)

1. **Review the docker-compose.yml file:**
   ```bash
   cat docker-compose.yml
   ```

2. **Start the Bitcoin node:**
   ```bash
   docker-compose up -d
   ```

3. **Check if running:**
   ```bash
   docker-compose ps
   docker-compose logs -f bitcoin-node
   ```

4. **Access Bitcoin CLI:**
   ```bash
   docker-compose exec bitcoin-node bitcoin-cli -regtest [command]
   ```

### Option B: Local Installation

1. **Download Bitcoin Core:**
   - Visit: https://bitcoin.org/en/download
   - Or use package manager:
     ```bash
     # Ubuntu/Debian
     sudo apt-get update
     sudo apt-get install bitcoind
     
     # macOS
     brew install bitcoin
     ```

2. **Create configuration file:**
   ```bash
   mkdir -p ~/.bitcoin
   cp bitcoin.conf ~/.bitcoin/bitcoin.conf
   ```

3. **Start Bitcoin Core in regtest mode:**
   ```bash
   bitcoind -regtest -daemon
   ```

## Configuration

The `bitcoin.conf` file contains important settings:

```conf
# Network
regtest=1                    # Enable regtest mode
server=1                     # Enable RPC server

# RPC Settings
rpcuser=bitcoinrpc          # RPC username
rpcpassword=changeme123     # RPC password (CHANGE THIS!)
rpcallowip=0.0.0.0/0        # Allow RPC from any IP (development only)
rpcport=18443               # RPC port for regtest

# Mining
blockfilterindex=1          # Enable block filters
txindex=1                   # Index all transactions

# Performance
maxmempool=50               # Smaller mempool for testing
```

**⚠️ Security Note:** The default RPC password is for development only. Change it before any public deployment.

## Usage

### Starting the Node

**Docker:**
```bash
cd bitcoin-node
docker-compose up -d
```

**Local:**
```bash
bitcoind -regtest -daemon
```

### Basic Commands

1. **Generate initial blocks** (needed to make coins spendable):
   ```bash
   # Create a wallet
   bitcoin-cli -regtest createwallet "mining_wallet"
   
   # Generate address
   bitcoin-cli -regtest getnewaddress
   
   # Mine 101 blocks to that address (coins spendable after 100 confirmations)
   bitcoin-cli -regtest generatetoaddress 101 [your_address]
   ```

2. **Check blockchain info:**
   ```bash
   bitcoin-cli -regtest getblockchaininfo
   ```

3. **Get wallet balance:**
   ```bash
   bitcoin-cli -regtest getbalance
   ```

4. **Manual block generation** (useful for testing):
   ```bash
   bitcoin-cli -regtest generatetoaddress 1 [address]
   ```

5. **Get mining info:**
   ```bash
   bitcoin-cli -regtest getmininginfo
   ```

6. **Get block template** (what the pool will use):
   ```bash
   bitcoin-cli -regtest getblocktemplate '{"rules":["segwit"]}'
   ```

### Useful Scripts

**Start regtest with pre-mining:**
```bash
./scripts/start-regtest.sh
```

**Mine N blocks:**
```bash
./scripts/mine-blocks.sh 10  # Mines 10 blocks
```

## Testing the Node

1. **Verify node is running:**
   ```bash
   bitcoin-cli -regtest getnetworkinfo
   ```
   Expected: JSON output with network info

2. **Test RPC connectivity:**
   ```bash
   curl --user bitcoinrpc:changeme123 \
        --data-binary '{"jsonrpc":"1.0","id":"test","method":"getblockchaininfo","params":[]}' \
        -H 'content-type: text/plain;' \
        http://127.0.0.1:18443/
   ```

3. **Create and fund a test wallet:**
   ```bash
   bitcoin-cli -regtest createwallet "test"
   ADDR=$(bitcoin-cli -regtest getnewaddress)
   bitcoin-cli -regtest generatetoaddress 101 $ADDR
   bitcoin-cli -regtest getbalance
   ```
   Expected: Balance of 50 BTC (block reward)

## Integration with Mining Pool

The mining pool will communicate with this node via RPC:

1. **Pool requests block template:**
   ```
   GET getblocktemplate -> Returns block header, transactions, difficulty
   ```

2. **Miner finds valid block:**
   ```
   Pool submits via submitblock
   ```

3. **Node validates and adds to chain:**
   ```
   Returns null if successful, error if invalid
   ```

## Monitoring

### Check Node Status
```bash
# Uptime and blocks
bitcoin-cli -regtest uptime
bitcoin-cli -regtest getblockcount

# Mempool
bitcoin-cli -regtest getmempoolinfo

# Connections
bitcoin-cli -regtest getpeerinfo
```

### Watch Logs (Docker)
```bash
docker-compose logs -f bitcoin-node
```

### Watch Logs (Local)
```bash
tail -f ~/.bitcoin/regtest/debug.log
```

## Common Issues & Solutions

### Issue 1: "Could not connect to server"
**Cause:** Node not running or RPC not accessible  
**Solution:**
```bash
# Check if running
ps aux | grep bitcoind

# Check RPC connectivity
netstat -an | grep 18443

# Restart node
bitcoind -regtest -daemon
```

### Issue 2: "Authorization failed"
**Cause:** Incorrect RPC credentials  
**Solution:** Verify bitcoin.conf has correct rpcuser/rpcpassword

### Issue 3: "Insufficient funds"
**Cause:** Need to mine blocks to generate coins  
**Solution:**
```bash
bitcoin-cli -regtest generatetoaddress 101 $(bitcoin-cli -regtest getnewaddress)
```

## Stopping the Node

**Docker:**
```bash
docker-compose down
```

**Local:**
```bash
bitcoin-cli -regtest stop
```

## Data Persistence

- **Docker:** Data stored in Docker volume (persists between restarts)
- **Local:** Data in `~/.bitcoin/regtest/`

To reset blockchain (start fresh):
```bash
# Docker
docker-compose down -v

# Local
rm -rf ~/.bitcoin/regtest/
```

## Next Steps

Once your Bitcoin node is running:

1. ✅ Node is synced (block count > 0)
2. ➡️ Set up the **mining-pool** component to connect to this node
3. ➡️ Configure **miners** to connect to the pool
4. ➡️ Test block generation through the full pipeline

## Resources

- [Bitcoin Core Documentation](https://bitcoin.org/en/developer-documentation)
- [RPC API Reference](https://developer.bitcoin.org/reference/rpc/)
- [Regtest Mode Guide](https://developer.bitcoin.org/examples/testing.html#regtest-mode)
- [Bitcoin Core GitHub](https://github.com/bitcoin/bitcoin)

## Configuration Files Reference

- `bitcoin.conf` - Main configuration
- `docker-compose.yml` - Docker setup
- `scripts/start-regtest.sh` - Quick start script
- `scripts/mine-blocks.sh` - Block mining helper

## Support

For issues specific to this component, check:
1. Bitcoin Core debug logs
2. RPC connectivity (test with curl)
3. Configuration file syntax
4. Port availability (18443 for regtest)
