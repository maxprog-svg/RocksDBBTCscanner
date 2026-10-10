# RocksDBBTCscanner
A high-performance, multi-strategy Bitcoin private key scanner that searches for valid Bitcoin addresses stored in a RocksDB key-value database.

## Overview

This tool generates candidate private keys using 30+ different strategies (sequential, brainwallet, Coldcard vulnerabilities, CryptoJS weaknesses, etc.), derives their Bitcoin addresses, and checks if they exist in a pre-built RocksDB database of known addresses.

When a match is found, the private key and derived address are saved to an output file.

### Key Features

- **30+ scanning strategies** covering weak key generation patterns from various wallets and libraries
- **Multi-threaded** scanning with configurable thread count
- **RocksDB backend** for fast address lookups (supports databases of billions of entries)
- **External dictionary support** for custom wordlist attacks
- **Parallel strategy execution** — all strategies run simultaneously per key index

## Supported Strategies

| Strategy | Description |
|---|---|
| `sequential` | Sequential 64-bit keys (big-endian) |
| `little_endian` | Sequential keys in little-endian byte order |
| `be_front` | Sequential keys with big-endian at front |
| `reversed` | Reversed byte order keys |
| `self_hash` | SHA256(key) as key |
| `brainwallet` | SHA256(known brainwallet phrases) |
| `dictionary` | SHA256(external wordlist entries) |
| `brain_combo` | Combinations of brainwallet words |
| `hex_strings` | Hex-encoded strings as keys |
| `num_strings` | Numeric strings as keys |
| `mini_keys` | Bitcoin mini-key format |
| `puzzle` | Puzzle transaction public keys |
| `weak_openssl` | Weak OpenSSL RNG keys |
| `reused_nonce` | ECDSA nonce-reused signatures |
| `pollard_kangaroo` | Pollard's kangaroo algorithm |
| `android_rng` | Weak Android RNG keys |
| `blockchain_info` | blockchain.info wallet keys |
| `bitcoind_old` | Old Bitcoin Core wallet keys |
| `armory_weak` | Armory wallet weak keys |
| `electrum_old` | Old Electrum wallet keys |
| `cryptojs_weak_rng` | CryptoJS weak RNG (CVE-2020-36732) |
| `cryptojs_weak_pbkdf2` | CryptoJS weak PBKDF2 (CVE-2023-46233) |
| `cryptojs_weak_rng_fast` | Optimized CryptoJS weak RNG |
| `cryptojs_weak_pbkdf2_fast` | Optimized CryptoJS weak PBKDF2 |
| `cryptojs_math_random_fast` | Math.random() based keys |
| `cryptojs_pbkdf2_weak_fast` | Fast PBKDF2 weak keys |
| `coldcard_mk3` | Coldcard Mk3 LFSR PRNG keys |
| `coldcard_mk3_fast` | Optimized Coldcard Mk3 keys |
| `coldcard_bip39` | Coldcard BIP-39 seed phrases |

## Prerequisites

```bash
# Ubuntu/Debian
sudo apt install build-essential pkg-config rocksdb-dev libsecp256k1-dev

# Verify RocksDB is available
ldconfig -p | grep rocksdb
```

## Building

```bash
cargo build --release
```

The binary will be at `target/release/rocksdb_scanner`.

## Database Creation

The scanner requires a **RocksDB database** containing Bitcoin addresses as keys. The database can be built from a text file of addresses.

### Step 1: Prepare Address List

Create a plain text file with one Bitcoin address per line:

```
1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa
1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2
...
```

### Step 2: Build the RocksDB Database

```bash
./target/release/rocksdb_scanner --build <db_path> <address_file>
```

**Example:**

```bash
# Build database from addresses.txt, store in my_addresses.db
./target/release/rocksdb_scanner --build my_addresses.db addresses.txt
```

The build process:
- Reads each line from the input file
- Stores each address as a key in RocksDB (value: "1")
- Shows progress: count, percentage, and write speed
- Creates a directory at `<db_path>` containing the RocksDB data

### Step 3: Verify Database

```bash
ls -la my_addresses.db/
# Should contain MANIFEST, OPTIONS, SST files, etc.
```

## Launch Parameters

### Basic Syntax

```
rocksdb_scanner <command> [arguments]
```

### Commands

#### 1. Build Database

```bash
./target/release/rocksdb_scanner --build <db_path> <input_file>
```

| Argument | Description |
|---|---|
| `<db_path>` | Path to create/store the RocksDB database |
| `<input_file>` | Text file with one Bitcoin address per line |

#### 2. Single Strategy Scan

```bash
./target/release/rocksdb_scanner <db_path> --strategy <name> <start> <count> [output] [threads]
```

| Argument | Description |
|---|---|
| `<db_path>` | Path to the RocksDB database |
| `<name>` | Strategy name (see Supported Strategies above) |
| `<start>` | Starting key index (u64) |
| `<count>` | Number of keys to scan |
| `[output]` | Output file path (default: `found.txt`) |
| `[threads]` | Number of parallel threads (default: 20) |

**Examples:**

```bash
# Scan 1 million sequential keys
./target/release/rocksdb_scanner my_addresses.db --strategy sequential 1 1000000 found.txt 16

# Scan brainwallet phrases
./target/release/rocksdb_scanner my_addresses.db --strategy brainwallet 1 1000 brain_results.txt

# Scan Coldcard Mk3 vulnerability
./target/release/rocksdb_scanner my_addresses.db --strategy coldcard_mk3_fast 1 10000 coldcard_results.txt 8
```

#### 3. Default Sequential Scan

```bash
./target/release/rocksdb_scanner <db_path> <start> <count> [output] [threads]
```

Uses the `sequential` strategy by default.

#### 4. All Strategies

```bash
./target/release/rocksdb_scanner <db_path> --all <start> <count> [output_dir] [threads]
```

Runs all 30+ strategies in parallel for the given key range. Results are saved in `<output_dir>`.

#### 5. External Dictionary Attack

```bash
./target/release/rocksdb_scanner <db_path> --dict <wordlist> --strategy dictionary <start> <count>
```

| Argument | Description |
|---|---|
| `<wordlist>` | Path to a wordlist file (one word per line) |
| `<start>` | Starting index in the wordlist |
| `<count>` | Number of words to scan |

**Example:**

```bash
./target/release/rocksdb_scanner my_addresses.db --dict rockyou.txt --strategy dictionary 1 100000 dict_results.txt 16
```

#### 6. List Available Strategies

```bash
./target/release/rocksdb_scanner --list-strategies
```

#### 7. Help

```bash
./target/release/rocksdb_scanner --help
```

## Output Format

Found keys are saved in text files with the format:

```
<address>:<private_key_hex>
```

Example:
```
1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa:0000000000000000000000000000000000000000000000000000000000000001
```

Each strategy produces its own output file named `found_<strategy_name>.txt`.

## Optimization Tips

### Environment Variables

```bash
# Increase RocksDB cache (adjust based on available RAM)
export ROCKSDB_CACHE_SIZE=$((4 * 1024 * 1024 * 1024))  # 4 GB

# Increase max open files
export ROCKSDB_MAX_OPEN_FILES=10000

# Reduce malloc arenas for better memory usage
export MALLOC_ARENA_MAX=2
```

### I/O Priority

Run with lower I/O priority to avoid system slowdown:

```bash
ionice -c2 -n7 ./target/release/rocksdb_scanner ...
```

### Thread Count

For I/O-bound scanning, fewer threads may perform better:
- SSD/NVMe: 8-16 threads
- HDD: 4-8 threads
- Network storage: 2-4 threads

## File Structure

```
rocksdb_scanner/
├── Cargo.toml              # Rust package config
├── build.rs                # Links against rocksdb and secp256k1
├── src/
│   ├── main.rs             # Main program with all strategies
│   ├── coldcard_optimized.rs    # Optimized Coldcard strategies
│   ├── coldcard_strategy.rs     # Coldcard Mk3 vulnerability
│   ├── coldcard_wrappers.rs     # Strategy wrappers
│   ├── cryptojs_fast.rs         # Optimized CryptoJS strategies
│   ├── rocksdb_multiget.rs      # MultiGet optimization module
│   └── main_multiget.rs         # MultiGet integration tests
├── *.sh                    # Helper scripts
├── *.md                    # Documentation
└── test_all.db/            # Sample RocksDB database
```

https://satoshidisk.com/pay/CSaoN5
