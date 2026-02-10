# AgentWallet

AgentWallet is a command-line Ethereum wallet built for AI agents and automation. It provides a scriptable, cross-platform interface for managing Ethereum accounts, sending transactions, and interacting with dApps via WalletConnect.

## Features

- **Script-friendly**: All operations accessible via CLI flags with structured JSON output (`--json`)
- **Daemon architecture**: Persistent background server with lightweight CLI client
- **Cross-platform**: Single binary for Windows, macOS, Linux, and Android
- **WalletConnect v2**: Connect to any dApp supporting WalletConnect
- **Session Keys**: Delegate limited permissions to sub-accounts (spending limits, contract restrictions, expiry)
- **Multi-network**: Support for any EIP-155 compatible Ethereum network
- **Clear exit codes**: Machine-parseable success/failure indicators

## Installation

### From Source

```bash
git clone https://github.com/BlockEase/AgentWallet.git
cd agentwallet
make build
```

The binary will be at `bin/agentwallet`.

### Cross-compilation

```bash
# Linux
GOOS=linux GOARCH=amd64 make build

# Windows
GOOS=windows GOARCH=amd64 make build

# Android
GOOS=android GOARCH=arm64 make build
```

## Quick Start

```bash
# Start the daemon (auto-started on first command if not running)
agentwallet --daemon

# Create a new wallet
agentwallet account create --password "your-password"

# List accounts
agentwallet account list

# Check balance (works for any address)
agentwallet balance --address 0x... --network ethereum

# Send transaction
agentwallet send --network ethereum --from 0x... --to 0x... --value 0.1 --password "your-password"

# Check transaction status
agentwallet tx status --guid <guid>

# Connect to a dApp via WalletConnect
agentwallet wc connect --network ethereum --address 0x... --uri "wc:..."
```

## Architecture

AgentWallet runs as two components:

1. **Daemon** (`agentwallet --daemon`): A background HTTP server on `localhost:6756` that manages keys, signs transactions, and maintains WalletConnect sessions.

2. **CLI Client** (`agentwallet <command>`): A lightweight client that communicates with the daemon via HTTP. If the daemon is not running, it is started automatically.

```
┌──────────────┐         HTTP/JSON         ┌──────────────────┐
│  CLI Client   │ ◄──────────────────────► │     Daemon       │
│  (one-shot)   │    localhost:6756        │  (persistent)    │
└──────────────┘                           │                  │
                                           │  ┌─────────────┐ │
                                           │  │  Keystore    │ │
                                           │  ├─────────────┤ │
                                           │  │  Tx Manager  │ │
                                           │  ├─────────────┤ │
                                           │  │  WC Sessions │ │
                                           │  └─────────────┘ │
                                           └────────┬─────────┘
                                                    │
                                           Ethereum RPC / WalletConnect Relay
```

## Commands

### General

| Command | Description |
|---------|-------------|
| `agentwallet version` | Print version number |
| `agentwallet build-info` | Print build information (platform, time, git commit) |

### Network

| Command | Description |
|---------|-------------|
| `agentwallet networks` | List configured networks |

### Account

| Command | Description |
|---------|-------------|
| `agentwallet account list` | List wallet accounts with supported networks |
| `agentwallet account create --password <pass>` | Create new account |

### Transaction

| Command | Description |
|---------|-------------|
| `agentwallet balance --address <addr> --network <name>` | Get ETH balance for any address |
| `agentwallet send --network <name> --from <addr> [flags]` | Send a transaction, returns a GUID |
| `agentwallet tx status --guid <guid>` | Poll transaction status by GUID |

#### `send` flags

| Flag | Required | Description |
|------|----------|-------------|
| `--network` | Yes | Network name, chain ID, or `eip155:<id>` |
| `--from` | Yes | Sender wallet address |
| `--to` | No | Recipient address (omit for contract creation) |
| `--value` | No | ETH amount to send (e.g., `0.1`) |
| `--data` | No | Hex-encoded calldata |
| `--nonce` | No | Override nonce |
| `--gas-price` | No | Gas price in wei |
| `--password` | Yes | Wallet password |

### WalletConnect

| Command | Description |
|---------|-------------|
| `agentwallet wc status` | List active WalletConnect sessions |
| `agentwallet wc connect --network <n> --address <a> --uri <wc_uri>` | Connect to a dApp |
| `agentwallet wc switch --session <id> [--address <a>] [--network <n>]` | Switch session params |
| `agentwallet wc disconnect --session <id>` | Disconnect a session |

## Global Flags

| Flag | Description |
|------|-------------|
| `--json` | Output in JSON format (machine-readable) |
| `--config <path>` | Path to configuration file |
| `--addr <host:port>` | Server address (default: `127.0.0.1:6756`) |
| `--daemon` | Run as background server daemon |

## Configuration

AgentWallet uses a JSON configuration file. Default location: `~/.agentwallet/wallet.conf`

```json
{
  "networks": [
    {
      "name": "ethereum",
      "chain_id": 1,
      "rpc_url": "https://eth.llamarpc.com"
    },
    {
      "name": "sepolia",
      "chain_id": 11155111,
      "rpc_url": "https://rpc.sepolia.org"
    },
    {
      "name": "base",
      "chain_id": 8453,
      "rpc_url": "https://mainnet.base.org"
    }
  ],
  "keystore_dir": "",
  "server_addr": "127.0.0.1:6756"
}
```

Network names follow [EIP-155](https://eips.ethereum.org/EIPS/eip-155) chain IDs from [chainid.network](https://chainid.network/). You can reference networks by name, numeric chain ID, or `eip155:<chain_id>` format.

## JSON Output

All commands support `--json` for structured output suitable for piping and automation:

```bash
$ agentwallet --json balance --address 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 --network ethereum
{
  "address": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
  "network": "ethereum",
  "chain_id": 1,
  "balance_wei": "1000000000000000000",
  "balance_eth": "1.000000000000000000"
}
```

Errors are written to stderr (or included in the JSON envelope with `"success": false`).

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Invalid arguments |

## Data Directory

```
~/.agentwallet/
├── wallet.conf          # Configuration file
├── keystore/            # Encrypted account keys (go-ethereum keystore format)
└── logs/                # Daemon logs
```

## Session Keys (Planned)

AgentWallet will support delegated sub-accounts with configurable permissions:

- Time-limited validity (e.g., expires in N days)
- Maximum spending limits per token
- Whitelist of allowed contract addresses
- Network restrictions

This enables safe delegation of wallet access to AI agents with bounded capabilities.

## Building

```bash
make build        # Build binary
make install      # Install to GOPATH/bin
make test         # Run tests
make clean        # Remove build artifacts
```

Build flags are injected automatically:

```bash
$ agentwallet build-info
Version:    v0.1.0
Git Commit: abc1234
Build Time: 2026-02-10T00:00:00Z
Go Version: go1.22.0
Platform:   darwin/arm64
```

## License

See [LICENSE](LICENSE) for details.
