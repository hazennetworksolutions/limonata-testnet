<div align="center">

# 🍋 Limonata Testnet Full Node & Validator Setup Guide

**A complete guide to running a Limonata testnet full node and registering as a validator**
*Binary installation, genesis sync, systemd service, state sync, validator creation, and the foundation grant flow — step by step.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04+-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Limonata](https://img.shields.io/badge/Limonata-Testnet-FFD100?style=flat-square)](https://limonata.xyz)
[![EVM Chain ID](https://img.shields.io/badge/EVM%20Chain%20ID-10777-blue?style=flat-square)](https://limonata.xyz)
[![Cosmos Chain ID](https://img.shields.io/badge/Cosmos%20Chain%20ID-limonata__10777--1-6B7CFF?style=flat-square)](https://limonata.xyz)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions
> **Network:** Limonata Testnet (Cosmos chain-id: `limonata_10777-1` / EVM chain-id: `10777`)
> **Node binary:** `limonatad` (alias of cosmos/evm `evmd`)
> **Last Updated:** July 2026

---

## Table of Contents

- [About Limonata](#about-limonata)
- [Hardware Requirements](#hardware-requirements)
- [Network Endpoints](#network-endpoints)
- [Step 1 — System Verification](#step-1--system-verification)
- [Step 2 — System Update and Dependencies](#step-2--system-update-and-dependencies)
- [Step 3 — Install Go](#step-3--install-go)
- [Step 4 — Get the Binary](#step-4--get-the-binary)
- [Step 5 — Initialize the Node and Fetch Genesis](#step-5--initialize-the-node-and-fetch-genesis)
- [Step 6 — Configure Peers, Mempool and Gas Price](#step-6--configure-peers-mempool-and-gas-price)
- [Step 7 — Create Systemd Service](#step-7--create-systemd-service)
- [Step 8 — Start the Node](#step-8--start-the-node)
- [Step 9 — Faster Sync with State Sync (Optional)](#step-9--faster-sync-with-state-sync-optional)
- [Step 10 — Create a Wallet](#step-10--create-a-wallet)
- [Step 11 — Create Your Validator](#step-11--create-your-validator)
- [Step 12 — Get Scored and Apply for the Foundation Grant](#step-12--get-scored-and-apply-for-the-foundation-grant)
- [Monitoring the Node](#monitoring-the-node)
- [Useful Commands](#useful-commands)
- [Firewall](#firewall)
- [Staying Updated](#staying-updated)
- [About the Author](#about-the-author)

---

## About Limonata

Limonata is an independent EVM Layer 1 built on the **Cosmos SDK + cosmos/evm**, with CometBFT single-slot BFT finality (~2s blocks) and a full standard EVM (MetaMask, Hardhat, Foundry, Viem all connect unchanged). The native coin is `LIMO` (base denom `aLIMO`, 18 decimals) and it is a pure network-utility coin — gas and staking only, no yield.

The chain runs a protocol-level gas sponsor (`x/gassponsor`) that pays EVM gas for users out of a self-refilling on-chain pool, and validating on Limonata is designed as **access, not capital**: you don't buy a validator stake, you self-bond a small faucet amount, build a track record, and then apply for a locked, non-transferable grant from the foundation to grow your voting power. See the [legal note](#legal-note) at the end of this guide.

- Website: [limonata.xyz](https://limonata.xyz)
- Official validator guide: [limonata.xyz/VALIDATOR.md](https://limonata.xyz/VALIDATOR.md)
- Chain source: [github.com/Limonata-Blockchain/limonata](https://github.com/Limonata-Blockchain/limonata)
- Discord: [discord.gg/vzbJ5u5Kex](https://discord.gg/vzbJ5u5Kex)

---

## Hardware Requirements

| Component | Minimum | Recommended |
|---|---|---|
| Operating System | Ubuntu 22.04+ | Ubuntu 24.04 |
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Disk | 50 GB SSD | 200 GB NVMe SSD |
| Network | 100 Mbps | 1 Gbps |

> ℹ️ This is an early testnet — requirements are light today and will grow as the network matures.

---

## Network Endpoints

| Type | Endpoint |
|---|---|
| RPC | https://rpc.limonata.xyz |
| Explorer | https://explorer.limonata.xyz |
| Faucet | https://faucet.limonata.xyz |
| Genesis | https://limonata.xyz/genesis.json |
| Proving Grounds (leaderboard) | https://grounds.limonata.xyz |
| Seed node-id | `4b154368aab24cb5b31c927efd50c73d0f4f9799` |
| Seed address | `142.127.103.79:26656` |
| Site / Docs | https://limonata.xyz |
| GitHub | https://github.com/Limonata-Blockchain/limonata |

---

## Step 1 — System Verification

After SSH-ing into your server, verify the system meets requirements:

```bash
lsb_release -a
uname -r
lscpu | grep -E "Model name|CPU\(s\)|Thread|Socket|Core"
free -h
df -h
```

---

## Step 2 — System Update and Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git wget htop tmux build-essential jq make lz4 gcc unzip
```

---

## Step 3 — Install Go

Limonata requires **Go 1.24+** to build from source (the team recommends **Go 1.26+**):

```bash
cd $HOME
VER="1.26.0"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"

echo "export PATH=\$PATH:/usr/local/go/bin:\$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

Verify the installation:

```bash
go version
```

> ℹ️ Skip this step entirely if you plan to use the prebuilt binary in Step 4.

---

## Step 4 — Get the Binary

The node binary is `limonatad` (an alias of cosmos/evm's `evmd`). Pick **one** of the two options below.

### Option A — Prebuilt binary (fastest, the exact build the live network runs)

```bash
cd $HOME
curl -sL https://github.com/Limonata-Blockchain/limonata/releases/latest/download/limonatad-linux-amd64.tar.gz | tar xz
sudo install limonatad /usr/local/bin/
```

### Option B — Build from source (Go 1.26+, CGO enabled)

```bash
cd $HOME
git clone https://github.com/Limonata-Blockchain/limonata.git
cd limonata
make install
sudo install $HOME/go/bin/limonatad /usr/local/bin/
```

Verify:

```bash
limonatad version
```

---

## Step 5 — Initialize the Node and Fetch Genesis

Set your moniker and initialize:

```bash
MONIKER="YOUR_MONIKER"
limonatad init "$MONIKER" --chain-id limonata_10777-1
```

Download the genesis file:

```bash
curl -s https://limonata.xyz/genesis.json -o $HOME/.limonatad/config/genesis.json
```

Validate it — **it must print "is a valid genesis file"**:

```bash
limonatad genesis validate-genesis
```

> ⚠️ Do not continue if validation fails. Re-download the genesis file and try again.

---

## Step 6 — Configure Peers, Mempool and Gas Price

This build has three settings that are **required**, not optional — the node will not sync or accept transactions correctly without them.

```bash
CFG=$HOME/.limonatad/config/config.toml
APP=$HOME/.limonatad/config/app.toml

# 1) connect to the network — the p2p seed
sed -i 's#^persistent_peers =.*#persistent_peers = "4b154368aab24cb5b31c927efd50c73d0f4f9799@142.127.103.79:26656"#' "$CFG"

# 2) this build REQUIRES the app-side mempool
sed -i 's/^type = "flood"/type = "app"/' "$CFG"

# 3) accept zero-fee EVM transactions — gas is sponsored by the protocol
sed -i 's/^minimum-gas-prices = .*/minimum-gas-prices = "0aLIMO"/' "$APP"
```

Set your external address so peers can reach you (optional, but recommended):

```bash
sed -i "s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):26656\"%" "$CFG"
```

> ℹ️ If you are running multiple nodes on the same host, change the default port prefix (`26`) throughout `config.toml` and `app.toml` to avoid conflicts.

---

## Step 7 — Create Systemd Service

```bash
sudo tee /etc/systemd/system/limonatad.service > /dev/null << EOF
[Unit]
Description=Limonata Testnet Node
After=network-online.target
Wants=network-online.target

[Service]
User=$USER
ExecStart=$(which limonatad) start \
  --chain-id limonata_10777-1 \
  --evm.evm-chain-id 10777 \
  --minimum-gas-prices 0aLIMO
Restart=on-failure
RestartSec=10
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable limonatad
```

> ⚠️ Both `--chain-id` (Cosmos) and `--evm.evm-chain-id` (EVM, `10777`) are **required on every start**. If the EVM chain-id is missing, the node aborts at `InitChain` with `invalid chain-id on InitChain`, and every EVM transaction signature check will fail.

---

## Step 8 — Start the Node

```bash
sudo systemctl start limonatad
sudo journalctl -u limonatad -f --no-pager -o cat
```

Verify the service is running:

```bash
sudo systemctl status limonatad --no-pager
```

Check sync status:

```bash
limonatad status 2>&1 | jq .SyncInfo
```

> ⚠️ Wait until `catching_up` is `false` before proceeding to validator creation.

---

## Step 9 — Faster Sync with State Sync (Optional)

State sync restores a recent snapshot instead of replaying full history — minutes instead of hours. Requires binary **v0.3.3+** (older builds diverge or lack the `squeeze` store).

```bash
sudo systemctl stop limonatad

CFG=$HOME/.limonatad/config/config.toml
RPC=https://rpc.limonata.xyz:443

LATEST=$(curl -s $RPC/block | jq -r .result.block.header.height)
TRUST=$(( (LATEST - 2000) / 1000 * 1000 ))
HASH=$(curl -s "$RPC/commit?height=$TRUST" | jq -r .result.signed_header.commit.block_id.hash)

sed -i "/^\[statesync\]/,/^\[/ { \
  s/^enable *=.*/enable = true/; \
  s#^rpc_servers *=.*#rpc_servers = \"$RPC,$RPC\"#; \
  s/^trust_height *=.*/trust_height = $TRUST/; \
  s/^trust_hash *=.*/trust_hash = \"$HASH\"/; \
  s/^trust_period *=.*/trust_period = \"168h0m0s\"/ }" "$CFG"

limonatad tendermint unsafe-reset-all --home $HOME/.limonatad --keep-addr-book

sudo systemctl start limonatad
sudo journalctl -u limonatad -f --no-pager -o cat
```

You should see `Discovered new snapshot` followed by `Snapshot restored`, and the node should catch up to the tip within minutes.

---

## Step 10 — Create a Wallet

Create your **operator** account (used to run and fund your validator):

```bash
limonatad keys add operator
```

> ⚠️ **CRITICAL:** Save the mnemonic phrase shown in a secure location. Without it, you cannot recover your wallet.

To recover an existing wallet:

```bash
limonatad keys add operator --recover
```

Get your address and fund it with a little test `LIMO` from the faucet at **https://limonata.xyz**:

```bash
limonatad keys show operator -a
```

Check your balance:

```bash
limonatad query bank balances $(limonatad keys show operator -a)
```

---

## Step 11 — Create Your Validator

> The node must be **fully synced** (`catching_up: false`) before creating a validator. You create and self-fund your own validator first — the foundation grant comes later, as a reward for good performance (Step 12).

Get your validator pubkey:

```bash
limonatad comet show-validator
```

Create `validator.json` (paste the pubkey object exactly as printed — keep it quoted):

```bash
cat > $HOME/validator.json << EOF
{
  "pubkey": $(limonatad comet show-validator),
  "amount": "1000000000000000000aLIMO",
  "moniker": "YOUR_MONIKER",
  "identity": "",
  "website": "",
  "security": "",
  "details": "",
  "commission-rate": "0.10",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
EOF
```

> ℹ️ `amount` is your self-bond, funded from the faucet in Step 10. The active set has room (`max_validators` is 100), so any positive self-bond puts you in the set.

Submit the transaction — gas is **not** sponsored for staking transactions, so a gas price is required (network floor is 1 gwei = `1000000000aLIMO`; never pass `--fees 0`):

```bash
limonatad tx staking create-validator $HOME/validator.json \
  --from operator \
  --chain-id limonata_10777-1 \
  --gas auto --gas-adjustment 1.4 \
  --gas-prices 1000000000aLIMO \
  -y
```

Verify your validator:

```bash
limonatad query staking validator $(limonatad keys show operator --bech val -a)
```

You are now **bonded and signing**. Keep the node running — your independence and reliability scores accrue live on [grounds.limonata.xyz](https://grounds.limonata.xyz).

---

## Step 12 — Get Scored and Apply for the Foundation Grant

Run your node reliably for a while and build a track record. Then apply at **https://limonata.xyz/#validator** ("Apply to validate") with **two** addresses:

1. Your existing `cosmosvaloper1...` — the validator you already run.
2. A **brand-new, never-funded** `cosmos1...` grant wallet, created just for this purpose:

```bash
limonatad keys add grant-wallet
limonatad keys show grant-wallet -a
```

> ⚠️ Do **not** fund the grant wallet — no faucet, no transfers. Grants can only be issued to a fresh, never-funded address.

If approved, the foundation issues a **locked, non-transferable grant** (stake-but-never-sell, clawback-able) to your grant wallet, plus a small spendable gas dust. Delegate the grant to your **existing** validator — do not create a second validator, as a duplicate seat hurts your independence score:

```bash
limonatad tx staking delegate <your cosmosvaloper1...> <granted-amount>aLIMO \
  --from grant-wallet \
  --chain-id limonata_10777-1 \
  --gas auto --gas-adjustment 1.4 \
  --gas-prices 1000000000aLIMO \
  -y
```

Your validator now competes for the **top 16** with real voting power. The granted stake remains foundation property and gives you no ownership — you keep your commission and fee share as compensation for operating the node.

---

## Monitoring the Node

```bash
# Live logs, filtered on committed blocks
sudo journalctl -u limonatad -f --no-pager | grep "committed block"

# Full logs
sudo journalctl -u limonatad -f --no-pager

# Sync status
limonatad status 2>&1 | jq .SyncInfo

# Connected peers
curl -s http://localhost:26657/net_info | jq .result.n_peers
```

### Service management

```bash
sudo systemctl restart limonatad
sudo systemctl stop limonatad
sudo systemctl status limonatad
```

---

## Useful Commands

### Wallet

```bash
# List wallets
limonatad keys list

# Show address
limonatad keys show operator -a

# Check balance
limonatad query bank balances $(limonatad keys show operator -a)
```

### Staking

```bash
# Delegate
limonatad tx staking delegate <valoper-address> <amount>aLIMO \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

# Redelegate
limonatad tx staking redelegate <src-valoper> <dst-valoper> <amount>aLIMO \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

# Unbond
limonatad tx staking unbond <valoper-address> <amount>aLIMO \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y
```

### Validator operations

```bash
# Edit validator
limonatad tx staking edit-validator \
  --new-moniker "NEW_MONIKER" \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

# Unjail
limonatad tx slashing unjail \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

# Signing info
limonatad query slashing signing-info $(limonatad comet show-validator)
```

---

## Firewall

```bash
# P2P — must be open to the public
sudo ufw allow 26656/tcp comment "limonatad P2P"

# RPC — open only if you serve public endpoints
sudo ufw allow 26657/tcp comment "limonatad RPC"
```

---

## Staying Updated

- Discord: [discord.gg/vzbJ5u5Kex](https://discord.gg/vzbJ5u5Kex)
- GitHub Releases: [Limonata-Blockchain/limonata](https://github.com/Limonata-Blockchain/limonata/releases)
- Official validator guide: [limonata.xyz/VALIDATOR.md](https://limonata.xyz/VALIDATOR.md)
- Proving Grounds (live leaderboard): [grounds.limonata.xyz](https://grounds.limonata.xyz)

### Upgrading the binary

```bash
sudo systemctl stop limonatad

cd $HOME/limonata
git fetch --all --tags
git checkout <new-tag>
make install
sudo install $HOME/go/bin/limonatad /usr/local/bin/

sudo systemctl start limonatad
sudo journalctl -u limonatad -f --no-pager -o cat
```

---

## Legal Note

Running a Limonata validator is operating open-source network infrastructure — a technical service, **not** an investment or security. The chain has no staking inflation (`x/mint` is off), so validators are not paid to hold a coin; they earn a share of network transaction fees plus their commission as compensation for operating a node — payment for a service, not a return on any purchase or investment. Testnet fees are small and testnet tokens are valueless. Stake delegated to your validator by the foundation remains the foundation's property and confers no ownership, claim, or interest to you. `$LIMO` is a network-utility coin. Any Genesis-network recognition for sustained, reliable operation is separate, discretionary, and is not a promised wage or return. Nothing in this guide is an offer, sale, or promise of any asset or investment return. This guide is technical documentation only and is not financial, investment, legal, or tax advice.

---

## About the Author

This guide was prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
