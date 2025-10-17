# Miners Component

## Overview

This component contains the mining software that performs the actual cryptographic work (SHA-256 hashing) to find valid Bitcoin blocks. For MVP testing, we use **CPU miners** which are sufficient for regtest/testnet but would be replaced by ASIC miners in production.

## What This Component Does

- Connects to the mining pool via Stratum protocol
- Receives mining jobs from the pool
- Performs SHA-256 hashing to find valid solutions
- Submits shares (partial proofs-of-work) back to the pool
- Simulates multiple independent miners for testing

## Architecture Role

```
[Mining Pool Server]
        ↓ (Stratum Protocol)
    [Miner 1] [Miner 2] [Miner 3]
        ↓       ↓       ↓
     SHA-256 Hashing Work
```

## Prerequisites

- C compiler (gcc or clang)
- OpenSSL development libraries
- libcurl development libraries
- For Docker: Docker installed

## Mining Software Options

For MVP, we recommend **cpuminer-multi** because:
- ✅ Supports Stratum protocol
- ✅ Works with regtest/testnet
- ✅ Easy to configure
- ✅ CPU-based (no special hardware needed)
- ✅ Open source and well-maintained

**Alternatives:**
- CGMiner: More features, more complex setup
- BFGMiner: FPGA/ASIC support, overkill for MVP
- Custom miner: Build from scratch (educational but time-consuming)

## Installation

### Option A: Docker (Recommended)

Each miner runs in its own container for isolation.

**Miner 1:**
```bash
cd miner1
docker build -t btc-miner .
docker run -d --name miner1 \
  -e POOL_URL="stratum+tcp://mining-pool:3333" \
  -e WALLET_ADDRESS="your_btc_address" \
  -e WORKER_NAME="miner1" \
  btc-miner
```

### Option B: Compile from Source

1. **Install dependencies:**
   ```bash
   # Ubuntu/Debian
   sudo apt-get update
   sudo apt-get install build-essential libcurl4-openssl-dev libssl-dev git autoconf automake
   
   # macOS
   brew install automake autoconf curl openssl
   ```

2. **Clone and compile cpuminer-multi:**
   ```bash
   git clone https://github.com/tpruvot/cpuminer-multi.git
   cd cpuminer-multi
   ./autogen.sh
   ./configure CFLAGS="-O3 -march=native"
   make
   sudo make install
   ```

3. **Verify installation:**
   ```bash
   cpuminer --version
   ```

### Option C: Pre-built Binary

Download from: https://github.com/tpruvot/cpuminer-multi/releases

```bash
wget [release-url]
tar -xvf cpuminer-[version].tar.gz
cd cpuminer-[version]
chmod +x cpuminer
```

## Configuration

### Basic Miner Configuration

Create a `miner-config.json` for each miner:

```json
{
  "pools": [
    {
      "url": "stratum+tcp://localhost:3333",
      "user": "your_btc_address",
      "pass": "worker1"
    }
  ],
  "algo": "sha256d",
  "threads": 2,
  "retries": 5,
  "quiet": false,
  "debug": false,
  "protocol": true
}
```

**Parameters:**
- `url`: Mining pool Stratum endpoint
- `user`: Your Bitcoin address (where rewards go)
- `pass`: Worker identifier (for pool statistics)
- `algo`: Algorithm (sha256d for Bitcoin)
- `threads`: CPU threads to use (2-4 for testing)
- `quiet`: Suppress output (false for debugging)

### Setting Up 3 Test Miners

For MVP testing, we simulate 3 independent miners:

**Miner 1 Configuration:**
```bash
cd miner1
cat > config.json << EOF
{
  "pools": [{
    "url": "stratum+tcp://localhost:3333",
    "user": "miner1_address",
    "pass": "miner1"
  }],
  "algo": "sha256d",
  "threads": 2
}
EOF
```

**Miner 2 Configuration:**
```bash
cd miner2
cat > config.json << EOF
{
  "pools": [{
    "url": "stratum+tcp://localhost:3333",
    "user": "miner2_address",
    "pass": "miner2"
  }],
  "algo": "sha256d",
  "threads": 2
}
EOF
```

**Miner 3 Configuration:**
```bash
cd miner3
cat > config.json << EOF
{
  "pools": [{
    "url": "stratum+tcp://localhost:3333",
    "user": "miner3_address",
    "pass": "miner3"
  }],
  "algo": "sha256d",
  "threads": 1
}
EOF
```

## Usage

### Starting Individual Miners

**Miner 1:**
```bash
cpuminer \
  --algo=sha256d \
  --url=stratum+tcp://localhost:3333 \
  --user=miner1_address \
  --pass=miner1 \
  --threads=2
```

**Miner 2:**
```bash
cpuminer \
  --algo=sha256d \
  --url=stratum+tcp://localhost:3333 \
  --user=miner2_address \
  --pass=miner2 \
  --threads=2
```

**Miner 3:**
```bash
cpuminer \
  --algo=sha256d \
  --url=stratum+tcp://localhost:3333 \
  --user=miner3_address \
  --pass=miner3 \
  --threads=1
```

### Starting All Miners (Script)

```bash
cd miners
./scripts/start-miners.sh
```

### Monitoring Miners

Each miner will output:
```
[2025-10-17 10:30:15] 5 miner threads started, using 'sha256d' algorithm.
[2025-10-17 10:30:16] Stratum session connected to localhost:3333
[2025-10-17 10:30:16] Stratum difficulty set to 1
[2025-10-17 10:30:20] CPU #0: 1.23 kH/s
[2025-10-17 10:30:20] accepted: 1/1 (100.00%), 1.23 kH/s yes!
```

**Key metrics:**
- `kH/s`: Hash rate (kilo-hashes per second)
- `accepted`: Shares accepted by pool
- `diff`: Current mining difficulty

### Stopping Miners

```bash
# Individual miner
pkill cpuminer

# All miners
killall cpuminer

# Docker
docker stop miner1 miner2 miner3
```

## Understanding Mining Output

### Successful Share Submission
```
[2025-10-17 10:30:20] accepted: 1/1 (100.00%), 1.23 kH/s yes!
```
- ✅ Share was valid and accepted by pool
- Contributes to your reward calculation

### Rejected Share
```
[2025-10-17 10:30:25] rejected: 0/2 (0.00%), 1.23 kH/s booooo
```
- ❌ Share was invalid (usually due to stale work)
- Does not contribute to rewards

### Block Found
```
[2025-10-17 10:31:00] *** BLOCK FOUND ***
```
- 🎉 Your miner found a valid block!
- Pool will submit to Bitcoin node
- Rewards distributed according to shares

## Performance Optimization

### CPU Mining Performance

For regtest testing, performance doesn't matter. For testnet:

**Increase threads:**
```bash
cpuminer --threads=4  # Use more CPU cores
```

**Optimize compilation:**
```bash
./configure CFLAGS="-O3 -march=native -funroll-loops"
```

**Expected hash rates:**
- Modern CPU: 1-5 MH/s per core
- Server CPU: 10-20 MH/s
- ASIC miner: 100+ TH/s (not needed for MVP)

### Resource Usage

Monitor CPU usage:
```bash
top -p $(pgrep cpuminer)
```

Limit CPU usage (Linux):
```bash
cpulimit -p $(pgrep cpuminer) -l 50  # Limit to 50%
```

## Testing & Validation

### Test 1: Connection to Pool
```bash
# Start miner and check logs
cpuminer --url=stratum+tcp://localhost:3333 --user=test --pass=test

# Expected: "Stratum session connected"
```

### Test 2: Share Submission
```bash
# Should see accepted shares within 30 seconds
# Expected: "accepted: 1/1"
```

### Test 3: Multiple Miners
```bash
# Start all 3 miners
./scripts/start-miners.sh

# Check all are running
ps aux | grep cpuminer
# Expected: 3 processes
```

### Test 4: Hash Rate Monitoring
```bash
# Observe hash rate output
# Miner 1 (2 threads): ~2-4 MH/s
# Miner 2 (2 threads): ~2-4 MH/s
# Miner 3 (1 thread): ~1-2 MH/s
```

## Integration with Pool

### Stratum Protocol Flow

1. **Connection:**
   ```
   Miner -> Pool: TCP connection to port 3333
   ```

2. **Subscription:**
   ```
   Miner -> Pool: mining.subscribe
   Pool -> Miner: Session ID, extranonce
   ```

3. **Authorization:**
   ```
   Miner -> Pool: mining.authorize (user, pass)
   Pool -> Miner: true/false
   ```

4. **Job Assignment:**
   ```
   Pool -> Miner: mining.notify (new job)
   Miner: Starts hashing
   ```

5. **Share Submission:**
   ```
   Miner -> Pool: mining.submit (nonce, hash)
   Pool -> Miner: true/false (accepted/rejected)
   ```

### Debugging Connection Issues

**Enable protocol debugging:**
```bash
cpuminer --protocol-dump
```

**Use Stratum protocol monitor:**
```bash
tcpdump -i lo -A port 3333
```

## Common Issues & Solutions

### Issue 1: "Stratum connection failed"
**Cause:** Pool not running or wrong address  
**Solution:**
```bash
# Check pool is running
netstat -an | grep 3333

# Test connectivity
telnet localhost 3333

# Check firewall
sudo ufw allow 3333
```

### Issue 2: "All shares rejected"
**Cause:** Wrong algorithm or difficulty too high  
**Solution:**
```bash
# Verify algorithm
cpuminer --algo=sha256d

# Check pool difficulty settings
```

### Issue 3: "CPU usage too high"
**Cause:** Too many threads  
**Solution:**
```bash
# Reduce threads
cpuminer --threads=1
```

### Issue 4: "Segmentation fault"
**Cause:** Incompatible CPU or bad compilation  
**Solution:**
```bash
# Recompile without optimizations
./configure CFLAGS="-O2"
make clean && make
```

## Scripts Reference

### `scripts/start-miners.sh`
Starts all 3 miners in background:
```bash
#!/bin/bash
cd miner1 && cpuminer -c config.json &
cd miner2 && cpuminer -c config.json &
cd miner3 && cpuminer -c config.json &
echo "All miners started"
```

### `scripts/stop-miners.sh`
Stops all miners:
```bash
#!/bin/bash
killall cpuminer
echo "All miners stopped"
```

### `scripts/monitor-miners.sh`
Shows miner statistics:
```bash
#!/bin/bash
watch -n 5 'ps aux | grep cpuminer | grep -v grep'
```

## Production Considerations

For production (not MVP):

1. **Use ASIC Miners:** CPU mining is 100,000x slower than ASICs
2. **Dedicated Hardware:** Separate physical machines for each miner
3. **Monitoring:** Prometheus + Grafana for metrics
4. **Auto-restart:** Systemd services for reliability
5. **Load Balancing:** Multiple pool servers for redundancy

## Next Steps

Once miners are running:

1. ✅ Miners connect to pool
2. ✅ Shares are being submitted
3. ➡️ Verify pool is receiving shares
4. ➡️ Check TEE is validating shares
5. ➡️ Monitor reward calculations

## Resources

- [cpuminer-multi GitHub](https://github.com/tpruvot/cpuminer-multi)
- [Stratum Protocol Specification](https://braiins.com/stratum-v1/docs)
- [Bitcoin Mining Guide](https://en.bitcoin.it/wiki/Mining)
- [SHA-256 Algorithm](https://en.wikipedia.org/wiki/SHA-2)

## Troubleshooting Checklist

- [ ] cpuminer installed and in PATH
- [ ] Pool server is running
- [ ] Port 3333 is accessible
- [ ] Valid Bitcoin address configured
- [ ] Correct algorithm (sha256d)
- [ ] Sufficient CPU resources available
- [ ] No firewall blocking connections
