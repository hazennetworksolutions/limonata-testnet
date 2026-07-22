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
> **Node binary:** `limonatad`
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
- [Step 9 — State Sync](#step-9--state-sync)
- [Step 10 — Create a Wallet](#step-10--create-a-wallet)
- [Step 11 — Create Your Validator](#step-11--create-your-validator)
- [Step 12 — Foundation Grant](#step-12--foundation-grant)
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

---

## Network Endpoints

| Type | Endpoint |
|---|---|
| RPC | https://rpc.limonata.xyz |
| Explorer | https://explorer.limonata.xyz |
| Faucet | https://faucet.limonata.xyz |
| Genesis | https://limonata.xyz/genesis.json |
| Proving Grounds | https://grounds.limonata.xyz |
| Seed | `4b154368aab24cb5b31c927efd50c73d0f4f9799@142.127.103.79:26656` |
| Site / Docs | https://limonata.xyz |
| GitHub | https://github.com/Limonata-Blockchain/limonata |

---

## Step 1 — System Verification

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

```bash
go version
```

---

## Step 4 — Get the Binary and Install Cosmovisor

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0
cosmovisor version
```

### Option A — Prebuilt binary

```bash
cd $HOME
curl -sL https://github.com/Limonata-Blockchain/limonata/releases/latest/download/limonatad-linux-amd64.tar.gz | tar xz

mkdir -p $HOME/.limonatad/cosmovisor/genesis/bin
mv limonatad $HOME/.limonatad/cosmovisor/genesis/bin/

ln -s $HOME/.limonatad/cosmovisor/genesis $HOME/.limonatad/cosmovisor/current -f
sudo ln -s $HOME/.limonatad/cosmovisor/current/bin/limonatad /usr/local/bin/limonatad -f
```

> ⚠️ `GLIBC_2.3x not found`? Your OS glibc is older than the release build's. Use Option B instead.

### Option B — Build from source

```bash
cd $HOME
git clone https://github.com/Limonata-Blockchain/limonata.git
cd limonata

git fetch --all --tags
git checkout limonata-v0.3.3

sed -i 's/GetNodeHomeDirectory(".evmd")/GetNodeHomeDirectory(".limonatad")/' evmd/config/config.go

make install EXAMPLE_BINARY=limonatad

mkdir -p $HOME/.limonatad/cosmovisor/genesis/bin
mv $HOME/go/bin/evmd $HOME/.limonatad/cosmovisor/genesis/bin/limonatad

ln -s $HOME/.limonatad/cosmovisor/genesis $HOME/.limonatad/cosmovisor/current -f
sudo ln -s $HOME/.limonatad/cosmovisor/current/bin/limonatad /usr/local/bin/limonatad -f
```

> ⚠️ Build the release tag (`limonata-v0.3.3`), not `main` — `main` can reject the live genesis. `make install` outputs the binary as `evmd`, hence the `mv`. The `sed` patches the hardcoded `~/.evmd` home dir to `~/.limonatad` (source has it hardcoded, no build flag changes it).

```bash
limonatad version
```

---

## Step 5 — Initialize the Node and Fetch Genesis

```bash
MONIKER="YOUR_MONIKER"
limonatad init "$MONIKER" --chain-id limonata_10777-1
```

```bash
curl -s https://limonata.xyz/genesis.json -o $HOME/.limonatad/config/genesis.json
wc -c $HOME/.limonatad/config/genesis.json
sha256sum $HOME/.limonatad/config/genesis.json
```

```bash
curl -s -o $HOME/.limonatad/config/addrbook.json \
  https://raw.githubusercontent.com/hazennetworksolutions/limonata-testnet/main/addrbook.json
```

> ℹ️ No official addrbook exists yet — skip if it 404s, the seed is enough.

> ⚠️ **Skip `limonatad genesis validate-genesis`** — it fails with `failed to unmarshal vpcap genesis: unexpected end of JSON input` because the public genesis predates the `x/vpcap` module. Harmless: `InitGenesis` skips missing module keys at startup, the node runs fine. Don't hand-edit the genesis to "fix" it.

---

## Step 6 — Configure Peers, Mempool, Ports, Pruning and Gas Price

```bash
CFG=$HOME/.limonatad/config/config.toml
APP=$HOME/.limonatad/config/app.toml

sed -i 's#^persistent_peers =.*#persistent_peers = "4b154368aab24cb5b31c927efd50c73d0f4f9799@142.127.103.79:26656"#' "$CFG"
sed -i 's/^type = "flood"/type = "app"/' "$CFG"
sed -i 's/^minimum-gas-prices = .*/minimum-gas-prices = "0aLIMO"/' "$APP"

sed -i "s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):26656\"%" "$CFG"
```

> ⚠️ All three are required — the node won't sync or accept transactions without them.

### Custom ports (multiple nodes on one host)

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

### Pruning

```bash
sed -i -e 's/^pruning *=.*/pruning = "custom"/' "$APP"
sed -i -e 's/^pruning-keep-recent *=.*/pruning-keep-recent = "100"/' "$APP"
sed -i -e 's/^pruning-interval *=.*/pruning-interval = "10"/' "$APP"
```

> ⚠️ Skip pruning on any node serving state sync snapshots.

### Disable indexer (optional)

```bash
sed -i -e 's/^indexer *=.*/indexer = "null"/' "$CFG"
```

### Enable Prometheus (optional)

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

> ⚠️ `--evm.evm-chain-id 10777` is required every start, or `InitChain` aborts.

---

## Step 8 — Start the Node

```bash
sudo systemctl start limonatad
sudo journalctl -u limonatad -f --no-pager -o cat
```

```bash
sudo systemctl status limonatad --no-pager
limonatad status 2>&1 | jq .SyncInfo
```

> Wait for `catching_up: false` before creating a validator.

---

## Step 9 — State Sync

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

> Look for `Discovered new snapshot` → `Snapshot restored`.

---

## Step 10 — Create a Wallet

```bash
limonatad keys add operator
```

> ⚠️ Save the mnemonic. To recover: `limonatad keys add operator --recover`

```bash
limonatad keys show operator -a
limonatad query bank balances $(limonatad keys show operator -a)
```

### Get the 0x (hex) Address for the Faucet

The faucet asks for the EVM-style `0x...` address. Since this is an EVM chain (`evmd`), the same key has both a bech32 (`limo1...`) and a hex (`0x...`) representation — derived from the same public key.

```bash
limonatad debug addr $(limonatad keys show operator -a)
```

Look for the `Address (hex)` line in the output and use that value (with the `0x` prefix) on the faucet.

Fund from the faucet at https://limonata.xyz.

---

## Step 11 — Create Your Validator

```bash
limonatad comet show-validator
```

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

```bash
limonatad tx staking create-validator $HOME/validator.json \
  --from operator \
  --chain-id limonata_10777-1 \
  --gas auto --gas-adjustment 1.4 \
  --gas-prices 1000000000aLIMO \
  -y
```

```bash
limonatad query staking validator $(limonatad keys show operator --bech val -a)
```

> Node must be fully synced first. Gas is not sponsored for staking txs.

---

## Step 12 — Foundation Grant

Apply at **https://limonata.xyz/#validator** with two addresses:

1. Your existing `cosmosvaloper1...`
2. A brand-new, never-funded grant wallet:

```bash
limonatad keys add grant-wallet
limonatad keys show grant-wallet -a
```

> ⚠️ Never fund the grant wallet — grants only go to fresh addresses.

```bash
limonatad tx staking delegate <your cosmosvaloper1...> <granted-amount>aLIMO \
  --from grant-wallet \
  --chain-id limonata_10777-1 \
  --gas auto --gas-adjustment 1.4 \
  --gas-prices 1000000000aLIMO \
  -y
```

---

## Monitoring the Node

```bash
sudo journalctl -u limonatad -f --no-pager | grep "committed block"
sudo journalctl -u limonatad -f --no-pager
limonatad status 2>&1 | jq .SyncInfo
curl -s http://localhost:26657/net_info | jq .result.n_peers
```

```bash
sudo systemctl restart limonatad
sudo systemctl stop limonatad
sudo systemctl status limonatad
```

---

## Useful Commands

### Wallet

```bash
limonatad keys list
limonatad keys show operator -a
limonatad query bank balances $(limonatad keys show operator -a)
```

### Staking

```bash
limonatad tx staking delegate <valoper-address> <amount>aLIMO \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

limonatad tx staking redelegate <src-valoper> <dst-valoper> <amount>aLIMO \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

limonatad tx staking unbond <valoper-address> <amount>aLIMO \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y
```

### Rewards

```bash
limonatad tx distribution withdraw-all-rewards \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

limonatad tx distribution withdraw-rewards $(limonatad keys show operator --bech val -a) --commission \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y
```

### Governance

```bash
limonatad query gov proposals

limonatad tx gov vote 1 yes \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y
```

### Validator operations

```bash
limonatad tx staking edit-validator \
  --new-moniker "NEW_MONIKER" \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

limonatad tx slashing unjail \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

limonatad query slashing signing-info $(limonatad comet show-validator)
```

---

## Troubleshooting

### `failed to unmarshal vpcap genesis: unexpected end of JSON input`

Expected on `validate-genesis` — skip that command, doesn't affect the running node. See Step 5.

### `GLIBC_2.3x not found`

```bash
rm -f $HOME/.limonatad/cosmovisor/genesis/bin/limonatad
cd $HOME
git clone https://github.com/Limonata-Blockchain/limonata.git
cd limonata
make install
mv $HOME/go/bin/evmd $HOME/.limonatad/cosmovisor/genesis/bin/limonatad
limonatad version
```

Advanced (keep the release binary, patch just that ELF):

```bash
sudo apt install -y patchelf
strings $HOME/.limonatad/cosmovisor/genesis/bin/limonatad | grep -o 'GLIBC_2\.[0-9]*' | sort -V | uniq | tail -1

mkdir -p $HOME/.limonatad/glibc-compat && cd /tmp
apt-get download libc6:amd64
dpkg -x libc6_*.deb $HOME/.limonatad/glibc-compat

patchelf --set-interpreter $HOME/.limonatad/glibc-compat/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 \
  --set-rpath $HOME/.limonatad/glibc-compat/lib/x86_64-linux-gnu:$HOME/.limonatad/glibc-compat/usr/lib/x86_64-linux-gnu \
  $HOME/.limonatad/cosmovisor/genesis/bin/limonatad

limonatad version
```

---

## Firewall

```bash
sudo ufw allow 26656/tcp comment "limonatad P2P"
sudo ufw allow 26657/tcp comment "limonatad RPC"
```

---

## Staying Updated

- Discord: [discord.gg/vzbJ5u5Kex](https://discord.gg/vzbJ5u5Kex)
- GitHub Releases: [Limonata-Blockchain/limonata](https://github.com/Limonata-Blockchain/limonata/releases)
- Official validator guide: [limonata.xyz/VALIDATOR.md](https://limonata.xyz/VALIDATOR.md)
- Proving Grounds: [grounds.limonata.xyz](https://grounds.limonata.xyz)

### Upgrading (Cosmovisor)

```bash
mkdir -p $HOME/.limonatad/cosmovisor/upgrades/v0.4.0/bin
curl -sL https://github.com/Limonata-Blockchain/limonata/releases/download/v0.4.0/limonatad-linux-amd64.tar.gz \
  | tar xz -C $HOME/.limonatad/cosmovisor/upgrades/v0.4.0/bin
chmod +x $HOME/.limonatad/cosmovisor/upgrades/v0.4.0/bin/limonatad
```

Cosmovisor switches automatically at the upgrade height.

---

## Legal Note

Running a Limonata validator is operating open-source network infrastructure — a technical service, not an investment or security. `$LIMO` is a network-utility coin with no staking inflation currently. This guide is technical documentation only, not financial or legal advice.

---

## About the Author

This guide was prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
