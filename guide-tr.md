<div align="center">

# 🍋 Limonata Testnet Full Node & Validator Kurulum Rehberi

**Limonata testnet full node kurulumu ve validator kaydı için eksiksiz rehber**
*Binary kurulumu, genesis senkronizasyonu, systemd servisi, state sync, validator oluşturma ve foundation grant süreci — adım adım.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04+-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Limonata](https://img.shields.io/badge/Limonata-Testnet-FFD100?style=flat-square)](https://limonata.xyz)
[![EVM Chain ID](https://img.shields.io/badge/EVM%20Chain%20ID-10777-blue?style=flat-square)](https://limonata.xyz)
[![Cosmos Chain ID](https://img.shields.io/badge/Cosmos%20Chain%20ID-limonata__10777--1-6B7CFF?style=flat-square)](https://limonata.xyz)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Hazırlayan:** HazenNetworkSolutions
> **Ağ:** Limonata Testnet (Cosmos chain-id: `limonata_10777-1` / EVM chain-id: `10777`)
> **Node binary:** `limonatad`
> **Son Güncelleme:** Temmuz 2026

---

## İçindekiler

- [Donanım Gereksinimleri](#donanım-gereksinimleri)
- [Ağ Bilgileri](#ağ-bilgileri)
- [Adım 1 — Sistem Kontrolü](#adım-1--sistem-kontrolü)
- [Adım 2 — Sistem Güncelleme ve Bağımlılıklar](#adım-2--sistem-güncelleme-ve-bağımlılıklar)
- [Adım 3 — Go Kurulumu](#adım-3--go-kurulumu)
- [Adım 4 — Binary'nin Temin Edilmesi ve Cosmovisor Kurulumu](#adım-4--binarynin-temin-edilmesi-ve-cosmovisor-kurulumu)
- [Adım 5 — Node'un Başlatılması ve Genesis İndirilmesi](#adım-5--nodeun-başlatılması-ve-genesis-indirilmesi)
- [Adım 6 — Peer, Mempool, Port, Pruning ve Gas Price Ayarları](#adım-6--peer-mempool-port-pruning-ve-gas-price-ayarları)
- [Adım 7 — Systemd Servisi Oluşturma](#adım-7--systemd-servisi-oluşturma)
- [Adım 8 — Node'un Başlatılması](#adım-8--nodeun-başlatılması)
- [Adım 9 — State Sync](#adım-9--state-sync)
- [Adım 10 — Cüzdan Oluşturma](#adım-10--cüzdan-oluşturma)
- [Adım 11 — Validator Oluşturma](#adım-11--validator-oluşturma)
- [Adım 12 — Foundation Grant](#adım-12--foundation-grant)
- [Node İzleme](#node-izleme)
- [Kullanışlı Komutlar](#kullanışlı-komutlar)
- [Sorun Giderme](#sorun-giderme)
- [Firewall](#firewall)
- [Güncel Kalmak](#güncel-kalmak)
- [Hazırlayan](#hazırlayan)

---

## Donanım Gereksinimleri

| Bileşen | Minimum | Önerilen |
|---|---|---|
| İşletim Sistemi | Ubuntu 22.04+ | Ubuntu 24.04 |
| CPU | 2 çekirdek | 4 çekirdek |
| RAM | 4 GB | 8 GB |
| Disk | 50 GB SSD | 200 GB NVMe SSD |
| Ağ | 100 Mbps | 1 Gbps |

---

## Ağ Bilgileri

| Tür | Endpoint |
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

## Adım 1 — Sistem Kontrolü

```bash
lsb_release -a
uname -r
lscpu | grep -E "Model name|CPU\(s\)|Thread|Socket|Core"
free -h
df -h
```

---

## Adım 2 — Sistem Güncelleme ve Bağımlılıklar

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git wget htop tmux build-essential jq make lz4 gcc unzip
```

---

## Adım 3 — Go Kurulumu

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

## Adım 4 — Binary'nin Temin Edilmesi ve Cosmovisor Kurulumu

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0
cosmovisor version
```

### Seçenek A — Hazır binary

```bash
cd $HOME
curl -sL https://github.com/Limonata-Blockchain/limonata/releases/latest/download/limonatad-linux-amd64.tar.gz | tar xz

mkdir -p $HOME/.limonatad/cosmovisor/genesis/bin
mv limonatad $HOME/.limonatad/cosmovisor/genesis/bin/

ln -s $HOME/.limonatad/cosmovisor/genesis $HOME/.limonatad/cosmovisor/current -f
sudo ln -s $HOME/.limonatad/cosmovisor/current/bin/limonatad /usr/local/bin/limonatad -f
```

> ⚠️ `GLIBC_2.3x not found` hatası alırsanız sisteminizin glibc'si release build'inkinden eski — Seçenek B'ye geçin.

### Seçenek B — Kaynak koddan derleme

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

> ⚠️ Release tag'ini derleyin (`limonata-v0.3.3`), `main`'i değil — `main` canlı genesis'i reddedebilir. `make install` binary'yi `evmd` olarak çıkarır, bu yüzden `mv`. `sed` komutu hardcoded `~/.evmd` home dizinini `~/.limonatad`'a çeviriyor (kaynakta sabit, hiçbir build flag'i değiştirmiyor).

```bash
limonatad version
```

---

## Adım 5 — Node'un Başlatılması ve Genesis İndirilmesi

```bash
MONIKER="KENDI_MONIKER_ADINIZ"
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

> ℹ️ Henüz resmi bir addrbook yok — 404 verirse atlayın, seed yeterli.

> ⚠️ **`limonatad genesis validate-genesis`'i atlayın** — `failed to unmarshal vpcap genesis: unexpected end of JSON input` hatası verir çünkü public genesis, `x/vpcap` modülünden önceki bir tarihte oluşturulmuş. Zararsız: `InitGenesis` başlangıçta eksik modül anahtarlarını atlar, node normal çalışır. Genesis'i elle "düzeltmeye" çalışmayın.

---

## Adım 6 — Peer, Mempool, Port, Pruning ve Gas Price Ayarları

```bash
CFG=$HOME/.limonatad/config/config.toml
APP=$HOME/.limonatad/config/app.toml

sed -i 's#^persistent_peers =.*#persistent_peers = "4b154368aab24cb5b31c927efd50c73d0f4f9799@142.127.103.79:26656"#' "$CFG"
sed -i 's/^type = "flood"/type = "app"/' "$CFG"
sed -i 's/^minimum-gas-prices = .*/minimum-gas-prices = "0aLIMO"/' "$APP"

sed -i "s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):26656\"%" "$CFG"
```

> ⚠️ Üçü de zorunlu — bunlar olmadan node ne senkronize olur ne de işlem kabul eder.

### Özel portlar (aynı sunucuda birden fazla node)

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

> ⚠️ State sync snapshot servis edecek node'larda pruning'i atlayın.

### Indexer'ı kapatma (opsiyonel)

```bash
sed -i -e 's/^indexer *=.*/indexer = "null"/' "$CFG"
```

### Prometheus'u açma (opsiyonel)

```bash
sed -i -e 's/prometheus = false/prometheus = true/' "$CFG"
```

---

## Adım 7 — Systemd Servisi Oluşturma

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

> ⚠️ `--evm.evm-chain-id 10777` her başlatmada zorunlu, yoksa `InitChain` durur.

---

## Adım 8 — Node'un Başlatılması

```bash
sudo systemctl start limonatad
sudo journalctl -u limonatad -f --no-pager -o cat
```

```bash
sudo systemctl status limonatad --no-pager
limonatad status 2>&1 | jq .SyncInfo
```

> Validator oluşturmadan önce `catching_up: false` olmasını bekleyin.

---

## Adım 9 — State Sync

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

> `Discovered new snapshot` → `Snapshot restored` görmelisiniz.

---

## Adım 10 — Cüzdan Oluşturma

```bash
limonatad keys add operator
```

> ⚠️ Mnemonic'i kaydedin. Kurtarmak için: `limonatad keys add operator --recover`

```bash
limonatad keys show operator -a
limonatad query bank balances $(limonatad keys show operator -a)
```

### Faucet İçin 0x (Hex) Adres Alma

Faucet, EVM tarzı `0x...` adresi ister. Bu chain EVM uyumlu olduğu (`evmd`) için aynı key'in hem bech32 (`limo1...`) hem de hex (`0x...`) karşılığı vardır — ikisi de aynı public key'den türetilir.

```bash
limonatad debug addr $(limonatad keys show operator -a)
```

Çıktıda `Address (hex)` satırını bulun ve o değeri (`0x` ön eki ile) faucet'e girin.

Faucet'ten fonlayın: https://limonata.xyz

---

## Adım 11 — Validator Oluşturma

```bash
limonatad comet show-validator
```

```bash
cat > $HOME/validator.json << EOF
{
  "pubkey": $(limonatad comet show-validator),
  "amount": "1000000000000000000aLIMO",
  "moniker": "KENDI_MONIKER_ADINIZ",
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

> Node önce tam senkronize olmalı. Staking işlemlerinde gas sponsorlu değil.

---

## Adım 12 — Foundation Grant

**https://limonata.xyz/#validator** üzerinden iki adresle başvurun:

1. Mevcut `cosmosvaloper1...` adresiniz
2. Yeni, hiç fonlanmamış bir grant cüzdanı:

```bash
limonatad keys add grant-wallet
limonatad keys show grant-wallet -a
```

> ⚠️ Grant cüzdanını asla fonlamayın — grant'ler sadece yeni adreslere verilir.

```bash
limonatad tx staking delegate <cosmosvaloper1-adresiniz> <verilen-miktar>aLIMO \
  --from grant-wallet \
  --chain-id limonata_10777-1 \
  --gas auto --gas-adjustment 1.4 \
  --gas-prices 1000000000aLIMO \
  -y
```

---

## Node İzleme

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

## Kullanışlı Komutlar

### Cüzdan

```bash
limonatad keys list
limonatad keys show operator -a
limonatad query bank balances $(limonatad keys show operator -a)
```

### Staking

```bash
limonatad tx staking delegate <valoper-adresi> <miktar>aLIMO \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

limonatad tx staking redelegate <kaynak-valoper> <hedef-valoper> <miktar>aLIMO \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

limonatad tx staking unbond <valoper-adresi> <miktar>aLIMO \
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

### Validator işlemleri

```bash
limonatad tx staking edit-validator \
  --new-moniker "YENI_MONIKER" \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

limonatad tx slashing unjail \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

limonatad query slashing signing-info $(limonatad comet show-validator)
```

---

## Sorun Giderme

### `failed to unmarshal vpcap genesis: unexpected end of JSON input`

`validate-genesis` komutunda beklenen bir hata — bu komutu atlayın, çalışan node'u etkilemez. Bkz. Adım 5.

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

İleri seviye (release binary'yi koru, sadece o ELF'i patch'le):

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

## Güncel Kalmak

- Discord: [discord.gg/vzbJ5u5Kex](https://discord.gg/vzbJ5u5Kex)
- GitHub Releases: [Limonata-Blockchain/limonata](https://github.com/Limonata-Blockchain/limonata/releases)
- Resmi validator guide: [limonata.xyz/VALIDATOR.md](https://limonata.xyz/VALIDATOR.md)
- Proving Grounds: [grounds.limonata.xyz](https://grounds.limonata.xyz)

### Upgrade (Cosmovisor)

```bash
mkdir -p $HOME/.limonatad/cosmovisor/upgrades/v0.4.0/bin
curl -sL https://github.com/Limonata-Blockchain/limonata/releases/download/v0.4.0/limonatad-linux-amd64.tar.gz \
  | tar xz -C $HOME/.limonatad/cosmovisor/upgrades/v0.4.0/bin
chmod +x $HOME/.limonatad/cosmovisor/upgrades/v0.4.0/bin/limonatad
```

Cosmovisor upgrade yüksekliğinde otomatik geçiş yapar.

---

## Yasal Not

Limonata validator işletmek açık kaynak ağ altyapısı işletmektir — bir yatırım veya menkul kıymet değil, teknik bir hizmettir. `$LIMO` bir ağ-fayda coin'idir, şu an staking inflation'ı yok. Bu rehber sadece teknik dokümantasyondur, finansal veya yasal tavsiye değildir.

---

## Hazırlayan

Bu rehber **HazenNetworkSolutions** tarafından hazırlanmıştır.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
