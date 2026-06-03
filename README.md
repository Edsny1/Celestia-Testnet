# Celestia Mocha Testnet Node Installation Guide (`mocha-4`)

## 📋 Hardware Requirements

| Component | Minimum | Recommended |
| --- | --- | --- |
| **CPU** | 4 Cores | 8 Cores |
| **RAM** | 8 GB | 16 GB |
| **SSD** | 200 GB NVMe | 500 GB NVMe |
| **OS** | Ubuntu 22.04 | Ubuntu 22.04 |

## 🛠 Installation Steps

### 1. System Update & Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

### 2. Install Go

```bash
cd $HOME
VER="1.24.1"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
mkdir -p ~/go/bin
```

### 3. Configure Environment Variables

```bash
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export CELESTIA_CHAIN_ID="mocha-4"" >> $HOME/.bash_profile
echo "export CELESTIA_PORT="11"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### 4. Install Binary & Cosmovisor

We install and set up **Cosmovisor** to monitor the execution binary and seamlessly switch to upgraded code versions on chain hard forks.

```bash
# Install Cosmovisor
go install github.com/cosmos/cosmos-sdk/cosmovisor/cmd/cosmovisor@latest

# Clone and build Celestia app binary
cd $HOME
rm -rf celestia-app
git clone https://github.com/celestiaorg/celestia-app.git
cd celestia-app/
APP_VERSION=v9.0.1-mocha
git checkout tags/$APP_VERSION -b $APP_VERSION
make install

# Prepare Cosmovisor folder architecture
mkdir -p $HOME/.celestia-app/cosmovisor/genesis/bin
mkdir -p $HOME/.celestia-app/cosmovisor/upgrades
cp $(which celestia-appd) $HOME/.celestia-app/cosmovisor/genesis/bin/

# Bind universal production binary shortcut link
sudo ln -s $HOME/.celestia-app/cosmovisor/genesis/bin/celestia-appd /usr/local/bin/celestia-appd -f
```

### 5. Node Configuration & Initialization

> ⚠️ **Not:** `celestia-appd v9` sürümünde `config` komutu artık alt komut gerektiriyor.
> Eski `celestia-appd config node ...` sözdizimi çalışmıyor; `set client` kullanılmalıdır.

```bash
# Önce chain-id set edilmeli, sonra node adresi
celestia-appd config set client chain-id mocha-4
celestia-appd config set client node tcp://localhost:${CELESTIA_PORT}657
celestia-appd config set client keyring-backend os

celestia-appd init $MONIKER --chain-id mocha-4

# Download Genesis and Addrbook
wget -O $HOME/.celestia-app/config/genesis.json https://server-6.itrocket.net/testnet/celestia/genesis.json
wget -O $HOME/.celestia-app/config/addrbook.json https://server-6.itrocket.net/testnet/celestia/addrbook.json

# Configure Seeds and P2P Peers
SEEDS="b402fe40f3474e9e208840702e1b7aa37f2edc4b@celestia-testnet-seed.itrocket.net:14656"
PEERS="daf2cecee2bd7f1b3bf94839f993f807c6b15fbf@celestia-testnet-peer.itrocket.net:11656,2d4d3a97695a4cd85b2253b7417f85f2a5765f73@195.154.212.53:20656,4eeea98dd704ba43b2745f1041c81eb91b43d750@195.154.103.60:26656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.celestia-app/config/config.toml

# Map Custom Port Overlap Rules inside app.toml
sed -i.bak -e "s%:1317%:${CELESTIA_PORT}317%g; s%:8080%:${CELESTIA_PORT}080%g; s%:9090%:${CELESTIA_PORT}090%g; s%:9091%:${CELESTIA_PORT}091%g; s%:8545%:${CELESTIA_PORT}545%g; s%:8546%:${CELESTIA_PORT}546%g; s%:6065%:${CELESTIA_PORT}065%g" $HOME/.celestia-app/config/app.toml

# Map Custom Port Overlap Rules inside config.toml
sed -i.bak -e "s%:26658%:${CELESTIA_PORT}658%g; s%:26657%:${CELESTIA_PORT}657%g; s%:6060%:${CELESTIA_PORT}060%g; s%:26656%:${CELESTIA_PORT}656%g; s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${CELESTIA_PORT}656\"%; s%:26660%:${CELESTIA_PORT}660%g" $HOME/.celestia-app/config/config.toml

# Performance Tweaks: Pruning, Prometheus, and Null Indexer
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.celestia-app/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.celestia-app/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.celestia-app/config/app.toml
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.002utia"|g' $HOME/.celestia-app/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.celestia-app/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.celestia-app/config/config.toml
```

### 6. Enable BBR Congestion Control (Önerilen)

BBR, Google tarafından geliştirilen bir TCP sıkışıklık kontrolü algoritmasıdır. P2P bağlantılarında (blockchain node'ları) performansı artırır. Sunucudaki diğer node'lara zarar vermez, aksine onlara da fayda sağlar.

```bash
sudo modprobe tcp_bbr
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Doğrula
sysctl net.ipv4.tcp_congestion_control
# Çıktı: net.ipv4.tcp_congestion_control = bbr olmalı
```

### 7. Create Systemd Service (Using Cosmovisor)

```bash
sudo tee /etc/systemd/system/celestia-appd.service > /dev/null <<EOF
[Unit]
Description=Celestia Testnet Node (Cosmovisor)
After=network-online.target

[Service]
User=$USER
WorkingDirectory=$HOME/.celestia-app
ExecStart=$(which cosmovisor) run start --home $HOME/.celestia-app
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_NAME=celestia-appd"
Environment="DAEMON_HOME=$HOME/.celestia-app"
Environment="DAEMON_ALLOW_STRUCTURAL_ADDITIONS=true"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable celestia-appd
```

### 8. Apply Snapshot & Start Node

> ⚠️ **Not:** Snapshot içinde upgrade geçmişi (v8→v9) bulunabilir. Cosmovisor başlamadan önce v8 binary klasörünü oluşturmanız gerekir.

```bash
# v8 upgrade binary klasörünü oluştur
mkdir -p $HOME/.celestia-app/cosmovisor/upgrades/v8/bin
cp $HOME/.celestia-app/cosmovisor/genesis/bin/celestia-appd $HOME/.celestia-app/cosmovisor/upgrades/v8/bin/celestia-appd

celestia-appd tendermint unsafe-reset-all --home $HOME/.celestia-app

# Download and unpack snapshot
curl https://server-6.itrocket.net/testnet/celestia/celestia_2026-06-03_11626921_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.celestia-app

# Start Node Service
sudo systemctl restart celestia-appd && sudo journalctl -u celestia-appd -fo cat
```

---

## 📊 Live Monitoring Telemetry

Run this loop to continuously calculate the remaining synchronized height index gap between your local target runtime and live nodes:

```bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.celestia-app/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://celestia-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Missing response data from network endpoints. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"
  sleep 5
done
```

---

## 🔑 Wallet & Validator Operations

### Key Operations

```bash
# Register keys locally
celestia-appd keys add $WALLET

# Or recover seed backup
celestia-appd keys add $WALLET --recover
```

Keep profile addresses pinned for immediate validation calls:

```bash
WALLET_ADDRESS=$(celestia-appd keys show $WALLET -a)
VALOPER_ADDRESS=$(celestia-appd keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### Launching the Validator Instance

Ensure the sync loop yields exactly `0` blocks remaining, and that your testnet addresses are sufficiently funded before initiating compilation rules:

```bash
celestia-appd query bank balances $WALLET_ADDRESS --node tcp://localhost:${CELESTIA_PORT}657
```

> ⚠️ **Not:** `celestia-appd v9` sürümünde `create-validator` komutu artık flag değil **JSON dosyası** kabul ediyor.

**Adım 1: validator.json dosyasını oluştur**

```bash
# Önce pubkey'i al
celestia-appd tendermint show-validator

# validator.json oluştur (PUBKEY kısmını yukarıdaki çıktıyla değiştir)
cat > $HOME/validator.json << EOF
{
  "pubkey": BURAYA_SHOW_VALIDATOR_CIKTISINI_YAPISTIR,
  "amount": "1000000utia",
  "moniker": "$MONIKER",
  "identity": "",
  "website": "",
  "details": "",
  "commission-rate": "0.2",
  "commission-max-rate": "0.2",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
EOF
```

**Adım 2: Validator oluştur**

```bash
celestia-appd tx staking create-validator $HOME/validator.json \
--from $WALLET \
--chain-id mocha-4 \
--fees 21000utia \
--gas 220000 \
--node tcp://localhost:${CELESTIA_PORT}657 \
--keyring-backend test \
-y
```

Başarılı çıktı şöyle görünür:

```
code: 0
txhash: <tx-hash>
```

---

## 🗑 Total Wipe Clean Protocol

```bash
sudo systemctl stop celestia-appd
sudo systemctl disable celestia-appd
sudo rm -rf /etc/systemd/system/celestia-appd.service
sudo rm /usr/local/bin/celestia-appd
sudo rm -rf $HOME/.celestia-app
sed -i "/CELESTIA_/d" $HOME/.bash_profile
```

Thanks itrocket
