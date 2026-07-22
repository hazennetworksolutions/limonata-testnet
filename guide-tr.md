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
> **Node binary:** `limonatad` (cosmos/evm `evmd` takma adı)
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
- [Adım 6 — Peer, Mempool ve Gas Price Ayarları](#adım-6--peer-mempool-ve-gas-price-ayarları)
- [Adım 7 — Systemd Servisi Oluşturma](#adım-7--systemd-servisi-oluşturma)
- [Adım 8 — Node'un Başlatılması](#adım-8--nodeun-başlatılması)
- [Adım 9 — State Sync ile Hızlı Senkronizasyon (Önerilen)](#adım-9--state-sync-ile-hızlı-senkronizasyon-önerilen)
- [Adım 10 — Cüzdan Oluşturma](#adım-10--cüzdan-oluşturma)
- [Adım 11 — Validator Oluşturma](#adım-11--validator-oluşturma)
- [Adım 12 — Skor Kazanma ve Foundation Grant Başvurusu](#adım-12--skor-kazanma-ve-foundation-grant-başvurusu)
- [Node İzleme](#node-izleme)
- [Kullanışlı Komutlar](#kullanışlı-komutlar)
- [Sorun Giderme](#sorun-giderme)
- [Firewall](#firewall)
- [Güncel Kalmak](#güncel-kalmak)
- [Yasal Not](#yasal-not)
- [Hakkımızda](#hakkımızda)

---

## Donanım Gereksinimleri

| Bileşen | Minimum | Önerilen |
|---|---|---|
| İşletim Sistemi | Ubuntu 22.04+ | Ubuntu 24.04 |
| CPU | 2 çekirdek | 4 çekirdek |
| RAM | 4 GB | 8 GB |
| Disk | 50 GB SSD | 200 GB NVMe SSD |
| Ağ | 100 Mbps | 1 Gbps |

> ℹ️ Bu erken bir testnet — gereksinimler şu an düşük, ağ olgunlaştıkça artacaktır.

---

## Ağ Bilgileri

| Tür | Adres |
|---|---|
| RPC | https://rpc.limonata.xyz |
| Explorer | https://explorer.limonata.xyz |
| Faucet | https://faucet.limonata.xyz |
| Genesis | https://limonata.xyz/genesis.json |
| Proving Grounds (liderlik tablosu) | https://grounds.limonata.xyz |
| Seed node-id | `4b154368aab24cb5b31c927efd50c73d0f4f9799` |
| Seed adresi | `142.127.103.79:26656` |
| Site / Dokümantasyon | https://limonata.xyz |
| GitHub | https://github.com/Limonata-Blockchain/limonata |

---

## Adım 1 — Sistem Kontrolü

Sunucuya SSH ile bağlandıktan sonra sistemin gereksinimleri karşıladığını doğrulayın:

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

Limonata'yı kaynak koddan derlemek için **Go 1.24+** gerekir (ekip **Go 1.26+** öneriyor):

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

Kurulumu doğrulayın:

```bash
go version
```

> ℹ️ Cosmovisor (Adım 4) `go install` ile kurulur, dolayısıyla hazır (prebuilt) `limonatad` binary'sini kullanacak olsanız da Go'yu kurmanız gerekir.

---

## Adım 4 — Binary'nin Temin Edilmesi ve Cosmovisor Kurulumu

`limonatad`'ı, işlettiğimiz diğer tüm Cosmos SDK zincirlerinde olduğu gibi **Cosmovisor** altında çalıştırıyoruz (AtomOne rehberimize bakabilirsiniz). Bu sayede gelecekteki upgrade'lerde binary, hedef blok yüksekliğinde manuel kesinti gerektirmeden otomatik olarak değişir.

### Cosmovisor Kurulumu

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0
```

Doğrulama:

```bash
cosmovisor version
```

Node binary'si `limonatad`'dır (cosmos/evm'in `evmd`'sinin bir takma adı). Aşağıdaki iki seçenekten **birini** seçin, ardından binary'yi Cosmovisor'un `genesis` klasörüne yerleştirin.

### Seçenek A — Hazır binary (en hızlısı, canlı ağın çalıştırdığı build ile aynısı)

```bash
cd $HOME
curl -sL https://github.com/Limonata-Blockchain/limonata/releases/latest/download/limonatad-linux-amd64.tar.gz | tar xz

mkdir -p $HOME/.limonatad/cosmovisor/genesis/bin
mv limonatad $HOME/.limonatad/cosmovisor/genesis/bin/

ln -s $HOME/.limonatad/cosmovisor/genesis $HOME/.limonatad/cosmovisor/current -f
sudo ln -s $HOME/.limonatad/cosmovisor/current/bin/limonatad /usr/local/bin/limonatad -f
```

### Seçenek B — Kaynak koddan derleme (Go 1.26+, CGO aktif)

> ℹ️ **Basit bir build'de dört tuzak var:**
> 1. **`main` değil, release tag'ini derleyin.** `main`, ekibin aktif geliştirme branch'i ve canlı ağın çalıştırdığından ileride olabiliyor — genesis validasyon kuralları farklılaşabiliyor (biz bunu bizzat yaşadık: `main`'den derlenen binary, canlı genesis'i `burn_bps 0 out of allowed range [1000,5000]` diyerek reddetti, çünkü `main`'deki `x/squeeze` parametre validasyonu, ağın genesis'inin oluşturulduğu andan sonra değişmişti). Her zaman ağın o anki sürümüne denk gelen tag'i `git checkout` edin (güncel tag için [limonata.xyz/VALIDATOR.md](https://limonata.xyz/VALIDATOR.md) veya [en son release](https://github.com/Limonata-Blockchain/limonata/releases/latest)'e bakın — bu yazı yazıldığı sırada `limonata-v0.3.3`).
> 2. `make install`, binary'yi `limonatad` olarak değil, **`evmd`** (upstream cosmos/evm paket ismi) olarak derler ve kurar — aşağıdaki `mv` komutu dosyayı yeniden adlandırıyor ki rehberdeki diğer tüm komutlar değişmeden çalışsın.
> 3. Aşağıda kullanılan `EXAMPLE_BINARY=limonatad` **sadece** `limonatad version` çıktısındaki ismi düzeltir — varsayılan home dizinini **değiştirmez**.
> 4. Varsayılan home dizini herhangi bir build flag'inden gelmiyor, **doğrudan Go kaynak kodunda sabit yazılı**: `evmd/config/config.go`, `clienthelpers.GetNodeHomeDirectory(".evmd")`'yi düz metin olarak çağırıyor. Bu satır değiştirilmeden derlenirse `limonatad init` (ve diğer tüm komutlar) sessizce `~/.evmd`'yi okur/yazar, asla `~/.limonatad`'ı kullanmaz — hiçbir flag bunu düzeltmez, derlemeden **önce** bu tek satırı patch'lemeniz gerekir. Resmi release binary'si, ekibin bu değişikliği zaten yaptığı bir ağaçtan derleniyor; biz de aynısını aşağıdaki tek satır `sed` ile yapıyoruz.

```bash
cd $HOME
git clone https://github.com/Limonata-Blockchain/limonata.git
cd limonata

# canlı ağın çalıştırdığı tam tag'i derleyin, main DEĞİL — önce güncel tag'i kontrol edin
git fetch --all --tags
git checkout limonata-v0.3.3

# hardcoded varsayılan home dizinini ~/.evmd'den ~/.limonatad'a çeviriyoruz
sed -i 's/GetNodeHomeDirectory(".evmd")/GetNodeHomeDirectory(".limonatad")/' evmd/config/config.go

make install EXAMPLE_BINARY=limonatad

mkdir -p $HOME/.limonatad/cosmovisor/genesis/bin
mv $HOME/go/bin/evmd $HOME/.limonatad/cosmovisor/genesis/bin/limonatad

ln -s $HOME/.limonatad/cosmovisor/genesis $HOME/.limonatad/cosmovisor/current -f
sudo ln -s $HOME/.limonatad/cosmovisor/current/bin/limonatad /usr/local/bin/limonatad -f
```

> ⚠️ Eğer zaten `sed` patch'i olmadan ve/veya `main`'den (doğru tag yerine) derleyip binary'yi kullandıysanız — örneğin `limonatad init` `~/.evmd/...` yazdıysa, ya da `genesis validate-genesis` `burn_bps ... out of allowed range` gibi bir parametre aralığı hatasıyla başarısız olduysa — henüz key oluşturmadıysanız sıfırlayıp doğru tag'den yeniden derleyin:
> ```bash
> rm -rf $HOME/.evmd $HOME/.limonatad/config $HOME/.limonatad/data
> cd $HOME/limonata && git checkout -- . && git fetch --all --tags && git checkout limonata-v0.3.3
> sed -i 's/GetNodeHomeDirectory(".evmd")/GetNodeHomeDirectory(".limonatad")/' evmd/config/config.go
> make install EXAMPLE_BINARY=limonatad
> rm -f $HOME/.limonatad/cosmovisor/genesis/bin/limonatad
> mv $HOME/go/bin/evmd $HOME/.limonatad/cosmovisor/genesis/bin/limonatad
> ```
> ardından Adım 5'i tekrar çalıştırın.

Doğrulama:

```bash
limonatad version
```

> ⚠️ Seçenek A'da `version 'GLIBC_2.3x' not found` hatası alırsanız, hazır binary sisteminizden daha yeni bir glibc ile derlenmiştir. **Seçenek B**'ye geçin — detaylar için aşağıdaki [Sorun Giderme](#sorun-giderme) bölümüne bakın.

---

## Adım 5 — Node'un Başlatılması ve Genesis İndirilmesi

Moniker'ınızı belirleyip node'u başlatın (init):

```bash
MONIKER="KENDI_MONIKER_ADINIZ"
limonatad init "$MONIKER" --chain-id limonata_10777-1
```

Genesis dosyasını indirin:

```bash
curl -s https://limonata.xyz/genesis.json -o $HOME/.limonatad/config/genesis.json
wc -c $HOME/.limonatad/config/genesis.json   # kontrol: ~34 KB, sıfır olmamalı
```

> ⚠️ **`limonatad genesis validate-genesis` KOMUTUNU ÇALIŞTIRMAYIN — bu ağda başarısız olması beklenen bir durumdur ve sorun değildir.** Yayınlanan genesis dosyası, v0.3.3 binary'sinin kaydettiği `x/vpcap` modülünden daha eski; bu yüzden `app_state` içinde `vpcap` anahtarı yok. `validate-genesis` CLI komutu kayıtlı her modülün doğrulayıcısına genesis'teki anahtarı gönderir; anahtar yoksa `nil` gider ve `failed to unmarshal vpcap genesis: unexpected end of JSON input` hatası üretir. Node'un kendisi bundan etkilenmez: başlangıçtaki `InitGenesis`, `app_state`'te anahtarı olmayan modülleri açıkça **atlar**, yani node normal başlar ve senkronize olur. Genesis dosyasına elle `vpcap` anahtarı **eklemeyin** — orijinal ağda hiç var olmamış bir state'i height 0'a enjekte etmek AppHash uyuşmazlığına yol açar. (Ekibe bildirildi; `genesis.json`'u anahtarla birlikte yeniden yayınlarlarsa doğrulama tekrar geçer.)

---

## Adım 6 — Peer, Mempool ve Gas Price Ayarları

Bu build'de **opsiyonel olmayan**, zorunlu üç ayar var — bunlar yapılmadan node ne düzgün senkronize olur ne de işlemleri doğru kabul eder.

```bash
CFG=$HOME/.limonatad/config/config.toml
APP=$HOME/.limonatad/config/app.toml

# 1) ağa bağlanmak için p2p seed
sed -i 's#^persistent_peers =.*#persistent_peers = "4b154368aab24cb5b31c927efd50c73d0f4f9799@142.127.103.79:26656"#' "$CFG"

# 2) bu build ZORUNLU olarak app-side mempool kullanır
sed -i 's/^type = "flood"/type = "app"/' "$CFG"

# 3) sıfır ücretli EVM işlemlerini kabul et — gas protokol tarafından sponsorlanıyor
sed -i 's/^minimum-gas-prices = .*/minimum-gas-prices = "0aLIMO"/' "$APP"
```

Diğer peer'lerin sizi bulabilmesi için external address ayarlayın (opsiyonel, önerilir):

```bash
sed -i "s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):26656\"%" "$CFG"
```

> ℹ️ Aynı sunucuda birden fazla node çalıştırıyorsanız, çakışmaları önlemek için `config.toml` ve `app.toml` içindeki varsayılan port ön ekini (`26`) değiştirin.

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

> ⚠️ `--chain-id` (Cosmos) ve `--evm.evm-chain-id` (EVM, `10777`) parametrelerinin **her başlatmada** verilmesi zorunludur. EVM chain-id verilmezse node `InitChain` adımında `invalid chain-id on InitChain` hatasıyla durur ve tüm EVM işlem imza doğrulamaları başarısız olur.

---

## Adım 8 — Node'un Başlatılması

```bash
sudo systemctl start limonatad
sudo journalctl -u limonatad -f --no-pager -o cat
```

Servisin çalıştığını doğrulayın:

```bash
sudo systemctl status limonatad --no-pager
```

Senkronizasyon durumunu kontrol edin:

```bash
limonatad status 2>&1 | jq .SyncInfo
```

> ⚠️ Validator oluşturma adımına geçmeden önce `catching_up` değerinin `false` olmasını bekleyin.

---

## Adım 9 — State Sync ile Hızlı Senkronizasyon (Önerilen)

State sync, tüm geçmişi tekrar oynatmak yerine güncel bir snapshot'ı geri yükler — saatler yerine dakikalar sürer. **v0.3.3+** binary gerektirir (daha eski build'ler ayrışır ya da `squeeze` store'a sahip değildir).

> ℹ️ Bu ağda state sync sadece hız kazancı değil: zincir genesis'ten bu yana zincir üstü binary upgrade'lerinden geçti, dolayısıyla güncel v0.3.3 binary ile height 0'dan replay etmek geçmiş bir upgrade yüksekliğinde AppHash uyuşmazlığına takılabilir. State sync güncel bir yükseklikten katılır; hem bu sorunu hem de genesis'teki eksik `vpcap` anahtarı sorununu tamamen devre dışı bırakır. Diğer operatörler de ağa bu şekilde katıldı.

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

Loglarda önce `Discovered new snapshot`, ardından `Snapshot restored` görmelisiniz; node dakikalar içinde tip'e (en güncel bloğa) ulaşır.

---

## Adım 10 — Cüzdan Oluşturma

**Operator** hesabınızı oluşturun (validator'ınızı çalıştırmak ve fonlamak için kullanılır):

```bash
limonatad keys add operator
```

> ⚠️ **ÖNEMLİ:** Ekranda gösterilen mnemonic (kurtarma) ifadesini güvenli bir yere kaydedin. Bu ifade olmadan cüzdanınızı kurtaramazsınız.

Mevcut bir cüzdanı kurtarmak için:

```bash
limonatad keys add operator --recover
```

Adresinizi alın ve **https://limonata.xyz** üzerindeki faucet'ten az miktarda test `LIMO` alarak fonlayın:

```bash
limonatad keys show operator -a
```

Bakiyenizi kontrol edin:

```bash
limonatad query bank balances $(limonatad keys show operator -a)
```

---

## Adım 11 — Validator Oluşturma

> Validator oluşturmadan önce node **tamamen senkronize** olmalıdır (`catching_up: false`). Önce kendi validator'ınızı oluşturup kendi kendinizi fonlarsınız — foundation grant'i, iyi performansın ödülü olarak daha sonra gelir (Adım 12).

Validator pubkey'inizi alın:

```bash
limonatad comet show-validator
```

`validator.json` dosyasını oluşturun (pubkey nesnesini tam olarak yazdırıldığı gibi yapıştırın — tırnak içinde bırakın):

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

> ℹ️ `amount`, Adım 10'da faucet'ten fonladığınız self-bond miktarıdır. Aktif set'te yer var (`max_validators` değeri 100), yani pozitif herhangi bir self-bond sizi sete dahil eder.

İşlemi gönderin — staking işlemlerinde gas sponsorlanmaz, dolayısıyla bir gas price belirtmeniz gerekir (ağ tabanı 1 gwei = `1000000000aLIMO`; asla `--fees 0` kullanmayın):

```bash
limonatad tx staking create-validator $HOME/validator.json \
  --from operator \
  --chain-id limonata_10777-1 \
  --gas auto --gas-adjustment 1.4 \
  --gas-prices 1000000000aLIMO \
  -y
```

Validator'ınızı doğrulayın:

```bash
limonatad query staking validator $(limonatad keys show operator --bech val -a)
```

Artık **bonded ve imzalıyorsunuz**. Node'u çalışır tutun — independence (bağımsızlık) ve reliability (güvenilirlik) skorlarınız [grounds.limonata.xyz](https://grounds.limonata.xyz) üzerinde canlı olarak birikir.

---

## Adım 12 — Skor Kazanma ve Foundation Grant Başvurusu

Node'unuzu bir süre güvenilir şekilde çalıştırıp bir geçmiş (track record) oluşturun. Ardından **https://limonata.xyz/#validator** adresinden ("Apply to validate") **iki** adresle başvurun:

1. Zaten çalıştırdığınız `cosmosvaloper1...` validator adresiniz.
2. Sadece bu amaç için oluşturulmuş, **daha önce hiç fonlanmamış, yepyeni** bir `cosmos1...` grant cüzdanı:

```bash
limonatad keys add grant-wallet
limonatad keys show grant-wallet -a
```

> ⚠️ Grant cüzdanını **fonlamayın** — faucet kullanmayın, transfer yapmayın. Grant'ler sadece hiç fonlanmamış, temiz bir adrese verilebilir.

Onaylanırsanız, foundation grant cüzdanınıza **kilitli, devredilemez bir grant** (stake edilebilir ama asla satılamaz, geri alınabilir/clawback-able) ile birlikte küçük bir harcanabilir gas miktarı gönderir. Grant'i **mevcut** validator'ınıza delege edin — ikinci bir validator oluşturmayın, çünkü tekrarlanan bir validator seat'i bağımsızlık skorunuzu düşürür:

```bash
limonatad tx staking delegate <cosmosvaloper1... adresiniz> <grant-miktarı>aLIMO \
  --from grant-wallet \
  --chain-id limonata_10777-1 \
  --gas auto --gas-adjustment 1.4 \
  --gas-prices 1000000000aLIMO \
  -y
```

Validator'ınız artık gerçek oy gücüyle **ilk 16**'ya girmek için rekabet eder. Grant edilen stake foundation'ın malı olarak kalır ve size hiçbir sahiplik hakkı vermez — node'u çalıştırmanın karşılığı olarak komisyonunuzu ve fee payınızı elde edersiniz.

---

## Node İzleme

```bash
# Canlı loglar, sadece commit edilen bloklar
sudo journalctl -u limonatad -f --no-pager | grep "committed block"

# Tüm loglar
sudo journalctl -u limonatad -f --no-pager

# Senkronizasyon durumu
limonatad status 2>&1 | jq .SyncInfo

# Bağlı peer sayısı
curl -s http://localhost:26657/net_info | jq .result.n_peers
```

### Servis yönetimi

```bash
sudo systemctl restart limonatad
sudo systemctl stop limonatad
sudo systemctl status limonatad
```

---

## Kullanışlı Komutlar

### Cüzdan

```bash
# Cüzdanları listele
limonatad keys list

# Adresi göster
limonatad keys show operator -a

# Bakiyeyi kontrol et
limonatad query bank balances $(limonatad keys show operator -a)
```

### Staking

```bash
# Delege et
limonatad tx staking delegate <valoper-adresi> <miktar>aLIMO \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

# Redelege et
limonatad tx staking redelegate <kaynak-valoper> <hedef-valoper> <miktar>aLIMO \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

# Unbond
limonatad tx staking unbond <valoper-adresi> <miktar>aLIMO \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y
```

### Validator işlemleri

```bash
# Validator'ı düzenle
limonatad tx staking edit-validator \
  --new-moniker "YENI_MONIKER" \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

# Unjail
limonatad tx slashing unjail \
  --from operator --chain-id limonata_10777-1 --gas auto --gas-adjustment 1.4 --gas-prices 1000000000aLIMO -y

# İmzalama bilgisi
limonatad query slashing signing-info $(limonatad comet show-validator)
```

---

## Sorun Giderme

### `failed to unmarshal vpcap genesis: unexpected end of JSON input`

Bu hata `limonatad genesis validate-genesis` komutundan gelir ve **beklenen** bir durumdur — yayınlanan `genesis.json` dosyasında `app_state.vpcap` anahtarı yokken v0.3.3 binary'si `x/vpcap` modülünü kaydeder (Adım 5'teki uyarıya bakın). Node'u engellemez: `InitGenesis` başlangıçta eksik modül anahtarlarını atlar. Validate komutunu atlayın, genesis dosyasını değiştirmeden bırakın ve Adım 6'dan itibaren devam edin (ağa katılmak için Adım 9'daki state sync'i kullanın).

### `version 'GLIBC_2.3x' not found (required by limonatad)`

Bu hatayı, **hazır (prebuilt) release binary'sini** (Adım 4, Seçenek A) sisteminizden daha yeni bir glibc ile derlenmiş bir OS'ta çalıştırdığınızda görürsünüz — örneğin Ubuntu 22.04 (glibc 2.35), glibc 2.38+ ile derlenmiş bir binary'ye karşı.

**Önerilen çözüm — kaynak koddan derleme (Seçenek B):** Bu, `limonatad`'ı kendi sunucunuzun glibc'sine karşı derler, dolayısıyla uyumsuzluk hiç oluşmaz. Sistemde hiçbir değişiklik yapılmaz, sunucudaki diğer servisler için sıfır risk taşır:

```bash
rm -f $HOME/.limonatad/cosmovisor/genesis/bin/limonatad

cd $HOME
git clone https://github.com/Limonata-Blockchain/limonata.git
cd limonata
make install

mv $HOME/go/bin/evmd $HOME/.limonatad/cosmovisor/genesis/bin/limonatad
limonatad version
```

**İleri seviye alternatif — tam olarak resmi release binary'sini tutmak, sadece bu binary'ye özel izole şekilde:** Canlı ağın çalıştırdığı tam build'i özellikle kullanmanız gerekiyorsa, sistemdeki `/lib/x86_64-linux-gnu`'ya ya da sunucudaki başka herhangi bir binary'ye dokunmadan, sadece bu tek ELF dosyasını kendi klasöründeki daha yeni bir glibc'yi kullanacak şekilde patch'leyebilirsiniz — diğer zincirlerde `LD_LIBRARY_PATH` ile yaptığımız "sistemi bulaştırma, sadece bir binary'ye özel scope et" mantığının aynısı:

```bash
sudo apt install -y patchelf

# binary'nin gerçekte hangi GLIBC sürümünü istediğini kontrol edin
strings $HOME/.limonatad/cosmovisor/genesis/bin/limonatad | grep -o 'GLIBC_2\.[0-9]*' | sort -V | uniq | tail -1

# daha yeni bir Ubuntu sürümünden, sistem genelinde KURMADAN eşleşen glibc'yi indirin
mkdir -p $HOME/.limonatad/glibc-compat && cd /tmp
apt-get download libc6:amd64   # bunu eşleşen daha yeni bir dağıtımda çalıştırın, ya da .deb'i manuel indirin
dpkg -x libc6_*.deb $HOME/.limonatad/glibc-compat

# SADECE limonatad binary'sini bu paketlenmiş glibc'ye yönlendirin — sistem glibc'si dokunulmadan kalır
patchelf --set-interpreter $HOME/.limonatad/glibc-compat/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 \
  --set-rpath $HOME/.limonatad/glibc-compat/lib/x86_64-linux-gnu:$HOME/.limonatad/glibc-compat/usr/lib/x86_64-linux-gnu \
  $HOME/.limonatad/cosmovisor/genesis/bin/limonatad

limonatad version
```

> ⚠️ Bu işlem sadece `limonatad`'ın kendi ELF header'ını (interpreter + kütüphane arama yolu) değiştirir — sunucudaki diğer tüm binary ve servisler sistem glibc'sini değişmeden kullanmaya devam eder. Yine de bunu ileri seviye bir geçici çözüm olarak görün; kaynak koddan derlemek daha basittir ve hiçbir başarısızlık riski taşımaz.

---

## Firewall

```bash
# P2P — herkese açık olmalı
sudo ufw allow 26656/tcp comment "limonatad P2P"

# RPC — sadece public endpoint sunuyorsanız açın
sudo ufw allow 26657/tcp comment "limonatad RPC"
```

---

## Güncel Kalmak

- Discord: [discord.gg/vzbJ5u5Kex](https://discord.gg/vzbJ5u5Kex)
- GitHub Releases: [Limonata-Blockchain/limonata](https://github.com/Limonata-Blockchain/limonata/releases)
- Resmi validator rehberi: [limonata.xyz/VALIDATOR.md](https://limonata.xyz/VALIDATOR.md)
- Proving Grounds (canlı liderlik tablosu): [grounds.limonata.xyz](https://grounds.limonata.xyz)

### Binary Güncelleme (Cosmovisor)

Node Cosmovisor altında çalıştığı için upgrade sırasında servisi durdurmanız gerekmez — yeni binary'yi önceden `upgrades` klasörüne **stage** edersiniz ve Cosmovisor hedef blok yüksekliğinde otomatik olarak geçiş yapar (örnek: `v0.4.0`):

```bash
mkdir -p $HOME/.limonatad/cosmovisor/upgrades/v0.4.0/bin
curl -sL https://github.com/Limonata-Blockchain/limonata/releases/download/v0.4.0/limonatad-linux-amd64.tar.gz \
  | tar xz -C $HOME/.limonatad/cosmovisor/upgrades/v0.4.0/bin
chmod +x $HOME/.limonatad/cosmovisor/upgrades/v0.4.0/bin/limonatad
```

Cosmovisor, zincir üzerindeki upgrade planını algılar ve doğru blok yüksekliğinde symlink'i değiştirip otomatik olarak yeniden başlar. Bir governance upgrade'i dışında zorla güncelleme yapmanız gerekirse (örneğin ekip bu erken testnette yamalı bir build yayınlarsa), binary'yi aynı şekilde stage edip şunu çalıştırın:

```bash
ln -sfn $HOME/.limonatad/cosmovisor/upgrades/v0.4.0 $HOME/.limonatad/cosmovisor/current
sudo systemctl restart limonatad
sudo journalctl -u limonatad -f --no-pager -o cat
```

---

## Yasal Not

Limonata validator'ı çalıştırmak, açık kaynaklı ağ altyapısını işletmektir — bu bir yatırım veya menkul kıymet değil, **teknik bir hizmettir**. Zincirde staking enflasyonu yoktur (`x/mint` kapalı), yani validator'lar bir coin'i elinde tutmanın karşılığında ödeme almaz; node işletmenin karşılığı olarak network işlem ücretlerinden pay ve komisyon kazanırlar — bu bir hizmet ödemesidir, herhangi bir satın alma ya da yatırımın getirisi değildir. Testnet ücretleri düşüktür ve testnet token'ları değersizdir. Foundation tarafından validator'ınıza delege edilen stake, foundation'ın malı olarak kalır ve size herhangi bir sahiplik, hak talebi veya menfaat sağlamaz. `$LIMO` bir network-utility coin'dir. Sürdürülebilir, güvenilir operasyon için yapılan herhangi bir Genesis-network tanınırlığı ayrı, isteğe bağlıdır ve vaat edilmiş bir ücret veya getiri değildir. Bu rehberdeki hiçbir şey herhangi bir varlığın ya da yatırım getirisinin teklifi, satışı veya vaadi değildir. Bu rehber sadece teknik dokümantasyondur; finansal, yatırım, hukuki veya vergi tavsiyesi niteliği taşımaz.

---

## Hakkımızda

Bu rehber **HazenNetworkSolutions** tarafından hazırlanmıştır.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
