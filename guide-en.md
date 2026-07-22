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

- [Hardware Requirements](#hardware-requirements)
- [Network Endpoints](#network-endpoints)
- [Step 1 — System Verification](#step-1--system-verification)
- [Step 2 — System Update and Dependencies](#step-2--system-update-and-dependencies)
- [Step 3 — Install Go](#step-3--install-go)
- [Step 4 — Get the Binary and Install Cosmovisor](#step-4--get-the-binary-and-install-cosmovisor)
- [Step 5 — Initialize the Node and Fetch Genesis](#step-5--initialize-the-node-and-fetch-genesis)
- [Step 6 — Configure Peers, Mempool, Ports, Pruning and Gas Price](#step-6--configure-peers-mempool-ports-pruning-and-gas-price)
- [Step 7 — Create Systemd Service](#step-7--create-systemd-service)
- [Step 8 — Start the Node](#step-8--start-the-node)
- [Step 9 — Faster Sync with State Sync (Recommended)](#step-9--faster-sync-with-state-sync-recommended)
- [Step 10 — Create a Wallet](#step-10--create-a-wallet)
- [Step 11 — Create Your Validator](#step-11--create-your-validator)
- [Step 12 — Get Scored and Apply for the Foundation Grant](#step-12--get-scored-and-apply-for-the-foundation-grant)
- [Monitoring the Node](#monitoring-the-node)
- [Useful Commands](#useful-commands)
- [Troubleshooting](#troubleshooting)
- [Firewall](#firewall)
- [Staying Updated](#staying-updated)
- [About the Author](#about-the-author)

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

> ℹ️ Cosmovisor (Step 4) is installed via `go install`, so keep Go even if you plan to use the prebuilt `limonatad` binary.

---

## Step 4 — Get the Binary and Install Cosmovisor

We run `limonatad` under **Cosmovisor**, the same way we run every other Cosmos SDK chain we operate (see our AtomOne guide). This lets future upgrades swap the binary automatically at the target block height, with no manual downtime.

### Install Cosmovisor

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0
```

Verify:

```bash
cosmovisor version
```

The node binary itself is `limonatad` (an alias of cosmos/evm's `evmd`). Pick **one** of the two options below, then place it in Cosmovisor's `genesis` folder.

### Option A — Prebuilt binary (fastest, the exact build the live network runs)

```bash
cd $HOME
curl -sL https://github.com/Limonata-Blockchain/limonata/releases/latest/download/limonatad-linux-amd64.tar.gz | tar xz

mkdir -p $HOME/.limonatad/cosmovisor/genesis/bin
mv limonatad $HOME/.limonatad/cosmovisor/genesis/bin/

ln -s $HOME/.limonatad/cosmovisor/genesis $HOME/.limonatad/cosmovisor/current -f
sudo ln -s $HOME/.limonatad/cosmovisor/current/bin/limonatad /usr/local/bin/limonatad -f
```

### Option B — Build from source (Go 1.26+, CGO enabled)

> ℹ️ **Four gotchas with a naive build:**
> 1. **Build the release tag, not `main`.** `main` is the team's active development branch and can be ahead of what the live network actually runs — genesis validation rules can diverge (we hit this ourselves: a `main` build rejected the live genesis with `burn_bps 0 out of allowed range [1000,5000]` because the `x/squeeze` param validation on `main` had moved on from what the network's genesis was created with). Always `git checkout` the tag matching the network's current release (check [limonata.xyz/VALIDATOR.md](https://limonata.xyz/VALIDATOR.md) or the [latest release](https://github.com/Limonata-Blockchain/limonata/releases/latest) for the current tag — `limonata-v0.3.3` at the time of writing).
> 2. `make install` builds and installs the binary as **`evmd`** (its upstream cosmos/evm package name), not `limonatad` — the `mv` below renames the file so every other command in this guide keeps working unchanged.
> 3. `EXAMPLE_BINARY=limonatad` (used below) only fixes the string printed by `limonatad version` — it does **not** change the default home directory.
> 4. The default home directory is **hardcoded in Go source**, not derived from any build flag: `evmd/config/config.go` calls `clienthelpers.GetNodeHomeDirectory(".evmd")` literally. Left as-is, `limonatad init` (and every other command) silently reads/writes `~/.evmd`, never `~/.limonatad` — no flag fixes this, you have to patch that one line **before** building. The official release binary is built from a tree where the team has already made this change; we replicate it with a one-line `sed` below.

```bash
cd $HOME
git clone https://github.com/Limonata-Blockchain/limonata.git
cd limonata

# build the exact tag the live network runs, NOT main — check the current tag first
git fetch --all --tags
git checkout limonata-v0.3.3

# patch the hardcoded default home dir from ~/.evmd to ~/.limonatad
sed -i 's/GetNodeHomeDirectory(".evmd")/GetNodeHomeDirectory(".limonatad")/' evmd/config/config.go

make install EXAMPLE_BINARY=limonatad

mkdir -p $HOME/.limonatad/cosmovisor/genesis/bin
mv $HOME/go/bin/evmd $HOME/.limonatad/cosmovisor/genesis/bin/limonatad

ln -s $HOME/.limonatad/cosmovisor/genesis $HOME/.limonatad/cosmovisor/current -f
sudo ln -s $HOME/.limonatad/cosmovisor/current/bin/limonatad /usr/local/bin/limonatad -f
```

> ⚠️ If you already built without the `sed` patch and/or built from `main` instead of the release tag, and ran commands against it — e.g. `limonatad init` printed `~/.evmd/...`, or `genesis validate-genesis` failed with a param-range error like `burn_bps ... out of allowed range` — reset and rebuild from the correct tag (safe if you haven't created keys yet):
> ```bash
> rm -rf $HOME/.evmd $HOME/.limonatad/config $HOME/.limonatad/data
> cd $HOME/limonata && git checkout -- . && git fetch --all --tags && git checkout limonata-v0.3.3
> sed -i 's/GetNodeHomeDirectory(".evmd")/GetNodeHomeDirectory(".limonatad")/' evmd/config/config.go
> make install EXAMPLE_BINARY=limonatad
> rm -f $HOME/.limonatad/cosmovisor/genesis/bin/limonatad
> mv $HOME/go/bin/evmd $HOME/.limonatad/cosmovisor/genesis/bin/limonatad
> ```
> then redo Step 5.

Verify:

```bash
limonatad version
```

> ⚠️ If Option A fails with `version 'GLIBC_2.3x' not found`, the release binary was built against a newer glibc than your OS ships. Switch to **Option B** — see [Troubleshooting](#troubleshooting) below.

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
wc -c $HOME/.limonatad/config/genesis.json   # sanity check: ~34 KB, non-zero
```

Verify the checksum (compare against another operator's output or a re-download to catch a corrupted/truncated fetch):

```bash
sha256sum $HOME/.limonatad/config/genesis.json
```

Download the addrbook (speeds up initial peer discovery):

```bash
curl -s -o $HOME/.limonatad/config/addrbook.json \
  https://raw.githubusercontent.com/hazennetworksolutions/limonata-testnet/main/addrbook.json
```

> ℹ️ Limonata doesn't publish an official addrbook yet (early testnet, network still small). The URL above follows the same convention as our [AtomOne guide](https://guides.hazennetworksolutions.com/atomone-mainnet/) — **TODO for us:** export `$HOME/.limonatad/config/addrbook.json` from our own running node and push it to `hazennetworksolutions/limonata-testnet` so this link resolves. Until then it 404s — skip the download if so; the seed configured in Step 6 is enough to bootstrap peer discovery on its own.

> ⚠️ **Do NOT run `limonatad genesis validate-genesis` — it is expected to fail on this network, and that's fine.** The published genesis predates the `x/vpcap` module that v0.3.3 registers, so its `app_state` has no `vpcap` key. The `validate-genesis` CLI passes `nil` to every registered module's validator and fails with `failed to unmarshal vpcap genesis: unexpected end of JSON input`. The node itself is unaffected: at startup, the SDK's `InitGenesis` explicitly **skips** modules whose key is missing from `app_state`, so the node starts and syncs normally. Do **not** hand-edit the genesis to add a `vpcap` key — injecting state the original network never had at height 0 causes an AppHash mismatch. (Reported to the team; if they republish `genesis.json` with the key, validation will pass again.)

---

## Step 6 — Configure Peers, Mempool, Ports, Pruning and Gas Price

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

### Custom ports (running more than one node on the same host)

If `limonatad` is the only chain on this server, skip this — the defaults are fine. If you're running several validators/full nodes side by side, pick a 3-digit port prefix per node (e.g. `119`) and remap every port in one shot instead of editing files by hand:

```bash
echo "export LIMO_PORT=\"119\"" >> $HOME/.bash_profile
source $HOME/.bash_profile

sed -i.bak -e "s%:1317%:${LIMO_PORT}17%g;
s%:8080%:${LIMO_PORT}80%g;
s%:9090%:${LIMO_PORT}90%g;
s%:9091%:${LIMO_PORT}91%g;
s%:8545%:${LIMO_PORT}45%g;
s%:8546%:${LIMO_PORT}46%g;
s%:6065%:${LIMO_PORT}65%g" "$APP"

sed -i.bak -e "s%:26658%:${LIMO_PORT}58%g;
s%:26657%:${LIMO_PORT}57%g;
s%:6060%:${LIMO_PORT}60%g;
s%:26656%:${LIMO_PORT}56%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${LIMO_PORT}56\"%;
s%:26660%:${LIMO_PORT}61%g" "$CFG"
```

> ⚠️ If you change the P2P port (`26656` → `${LIMO_PORT}56`), update the [Firewall](#firewall) rule and any `--node` flags you pass to `limonatad` accordingly. This also means the seed/peer strings from Step 6.1 keep their `:26656` port as-is — that's the **remote** node's port, not yours.

### Pruning (disk space)

By default the node keeps every historical version of the state store, which grows fast. Unless you specifically need full history (e.g. for an archive/RPC node), prune old versions:

```bash
sed -i -e 's/^pruning *=.*/pruning = "custom"/' "$APP"
sed -i -e 's/^pruning-keep-recent *=.*/pruning-keep-recent = "100"/' "$APP"
sed -i -e 's/^pruning-interval *=.*/pruning-interval = "10"/' "$APP"
```

> ⚠️ Do **not** enable custom pruning if you plan to serve state sync snapshots to other operators from this node — snapshot heights must stay available. Leave `pruning = "default"` on any node you intend to run as a public snapshot/RPC provider.

### Disable the tx indexer (optional, saves disk + CPU)

If you don't need to query historical transactions by hash/event through this node's own RPC (most validators don't — they query the public explorer/RPC instead), turn the indexer off:

```bash
sed -i -e 's/^indexer *=.*/indexer = "null"/' "$CFG"
```

### Enable Prometheus (optional)

Turn this on if you're feeding node metrics into Grafana/Prometheus for monitoring:

```bash
sed -i -e 's/prometheus = false/prometheus = true/' "$CFG"
```

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
ExecStart=$(which cosmovisor) run start \
  --chain-id limonata_10777-1 \
  --evm.evm-chain-id 10777 \
  --minimum-gas-prices 0aLIMO
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.limonatad"
Environment="DAEMON_NAME=limonatad"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.limonatad/cosmovisor/current/bin"

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

## Step 9 — Faster Sync with State Sync (Recommended)

State sync restores a recent snapshot instead of replaying full history — minutes instead of hours. Requires binary **v0.3.3+** (older builds diverge or lack the `squeeze` store).

> ℹ️ On this network state sync is more than a speed-up: the chain has gone through on-chain binary upgrades since genesis, so replaying from height 0 with the current v0.3.3 binary can hit an AppHash mismatch at a past upgrade height. State sync joins at a recent height and sidesteps both that and the missing-`vpcap`-key genesis issue entirely. This is how other operators joined the network.

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

### Rewards

```bash
# Withdraw all rewards
limonatad tx distribution withdraw-all-rewards \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

# Withdraw commission
limonatad tx distribution withdraw-rewards $(limonatad keys show operator --bech val -a) --commission \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y
```

> ℹ️ `x/mint` inflation is off on this testnet, so there's no staking reward stream yet — this is fee-share only, and fees are close to zero while most transactions are gas-sponsored (see Step 6, item 3). Rewards become meaningful once real fee volume or a mint schedule goes live.

### Governance

```bash
# List proposals
limonatad query gov proposals

# Vote on a proposal
limonatad tx gov vote 1 yes \
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

## Troubleshooting

### `failed to unmarshal vpcap genesis: unexpected end of JSON input`

This comes from `limonatad genesis validate-genesis` and is **expected** — the published `genesis.json` has no `app_state.vpcap` key while the v0.3.3 binary registers the `x/vpcap` module (see the warning in Step 5). It does **not** block the node: `InitGenesis` skips missing module keys at startup. Skip the validate command, keep the genesis file unmodified, and continue with Step 6 onward (use state sync in Step 9 to join the network).

### `version 'GLIBC_2.3x' not found (required by limonatad)`

You'll see this when running the **prebuilt release binary** (Step 4, Option A) on an OS whose glibc is older than the one the release was compiled against — for example Ubuntu 22.04 (glibc 2.35) against a binary built with glibc 2.38+.

**Recommended fix — build from source (Option B):** this compiles `limonatad` against your own server's glibc, so the mismatch can't happen. No system changes, no risk to other services on the box:

```bash
rm -f $HOME/.limonatad/cosmovisor/genesis/bin/limonatad

cd $HOME
git clone https://github.com/Limonata-Blockchain/limonata.git
cd limonata
make install

mv $HOME/go/bin/evmd $HOME/.limonatad/cosmovisor/genesis/bin/limonatad
limonatad version
```

**Advanced alternative — keep the exact release binary, isolated per-binary:** if you specifically need the release build (e.g. to match the live network's exact binary), you can patch just that one ELF file to use a newer glibc bundled in its own folder, without touching `/lib/x86_64-linux-gnu` or any other binary on the server — the same "don't pollute the system, scope it to one binary" principle we use for `LD_LIBRARY_PATH` tricks on other chains.

```bash
sudo apt install -y patchelf

# check which GLIBC version the binary actually needs
strings $HOME/.limonatad/cosmovisor/genesis/bin/limonatad | grep -o 'GLIBC_2\.[0-9]*' | sort -V | uniq | tail -1

# pull a matching glibc from a newer Ubuntu release WITHOUT installing it system-wide
mkdir -p $HOME/.limonatad/glibc-compat && cd /tmp
apt-get download libc6:amd64   # run this on a matching newer distro, or grab the .deb manually
dpkg -x libc6_*.deb $HOME/.limonatad/glibc-compat

# point ONLY the limonatad binary at the bundled glibc — system glibc is untouched
patchelf --set-interpreter $HOME/.limonatad/glibc-compat/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 \
  --set-rpath $HOME/.limonatad/glibc-compat/lib/x86_64-linux-gnu:$HOME/.limonatad/glibc-compat/usr/lib/x86_64-linux-gnu \
  $HOME/.limonatad/cosmovisor/genesis/bin/limonatad

limonatad version
```

> ⚠️ This only rewrites `limonatad`'s own ELF header (its interpreter + library search path) — every other binary and service on the server keeps using the system glibc unchanged. Still, treat it as an advanced workaround; building from source is simpler and has no failure modes.

---

## Firewall

```bash
# P2P — must be open to the public
sudo ufw allow 26656/tcp comment "limonatad P2P"

# RPC — open only if you serve public endpoints
sudo ufw allow 26657/tcp comment "limonatad RPC"
```

> ℹ️ If you set a custom port prefix in Step 6, open `${LIMO_PORT}56` (P2P) and, if needed, `${LIMO_PORT}57` (RPC) instead of the defaults above.

---

## Staying Updated

- Discord: [discord.gg/vzbJ5u5Kex](https://discord.gg/vzbJ5u5Kex)
- GitHub Releases: [Limonata-Blockchain/limonata](https://github.com/Limonata-Blockchain/limonata/releases)
- Official validator guide: [limonata.xyz/VALIDATOR.md](https://limonata.xyz/VALIDATOR.md)
- Proving Grounds (live leaderboard): [grounds.limonata.xyz](https://grounds.limonata.xyz)

### Upgrading the binary (Cosmovisor)

Since the node runs under Cosmovisor, you don't stop the service to upgrade — you **stage** the new binary in an `upgrades` folder ahead of time, and Cosmovisor swaps to it automatically at the target height (example: `v0.4.0`):

```bash
mkdir -p $HOME/.limonatad/cosmovisor/upgrades/v0.4.0/bin
curl -sL https://github.com/Limonata-Blockchain/limonata/releases/download/v0.4.0/limonatad-linux-amd64.tar.gz \
  | tar xz -C $HOME/.limonatad/cosmovisor/upgrades/v0.4.0/bin
chmod +x $HOME/.limonatad/cosmovisor/upgrades/v0.4.0/bin/limonatad
```

Cosmovisor detects the upgrade plan on-chain and switches the symlink at the correct block height, then restarts automatically. If you need to force-update outside of a governance upgrade (e.g. the team ships a patched build on this early testnet), stage the binary the same way, then:

```bash
ln -sfn $HOME/.limonatad/cosmovisor/upgrades/v0.4.0 $HOME/.limonatad/cosmovisor/current
sudo systemctl restart limonatad
sudo journalctl -u limonatad -f --no-pager -o cat
```

---

## Legal Note

Running a Limonata validator is operating open-source network infrastructure — a technical service, **not** an investment or security. The chain has no staking inflation (`x/mint` is off), so validators are not paid to hold a coin; they earn a share of network transaction fees plus their commission as compensation for operating a node — payment for a service, not a return on any purchase or investment. Testnet fees are small and testnet tokens are valueless. Stake delegated to your validator by the foundation remains the foundation's property and confers no ownership, claim, or interest to you. `$LIMO` is a network-utility coin. Any Genesis-network recognition for sustained, reliable operation is separate, discretionary, and is not a promised wage or return. Nothing in this guide is an offer, sale, or promise of any asset or investment return. This guide is technical documentation only and is not financial, investment, legal, or tax advice.

---

## About the Author

This guide was prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
