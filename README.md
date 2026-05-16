<div align="center">

# ⚛️ AtomOne Testnet Full Node & Validator Setup Guide

**A complete guide to running an AtomOne testnet full node and registering as a validator**  
*System preparation, binary installation, Cosmovisor setup, snapshot sync, and validator creation — step by step.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04+-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![AtomOne](https://img.shields.io/badge/AtomOne-Testnet-6B7CFF?style=flat-square)](https://atom.one)
[![Version](https://img.shields.io/badge/Node%20Version-v4.0.0-brightgreen?style=flat-square)](https://github.com/atomone-hub/atomone/releases)
[![Chain ID](https://img.shields.io/badge/Chain%20ID-atomone--testnet--1-blue?style=flat-square)](https://docs.atom.one)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions  
> **Network:** AtomOne Testnet (Chain ID: atomone-testnet-1)  
> **Version:** v4.0.0  
> **Last Updated:** May 2026

---

## Table of Contents

- [Hardware Requirements](#hardware-requirements)
- [Step 1 — System Verification](#step-1--system-verification)
- [Step 2 — System Update and Dependencies](#step-2--system-update-and-dependencies)
- [Step 3 — Install Go](#step-3--install-go)
- [Step 4 — Download and Build Binary](#step-4--download-and-build-binary)
- [Step 5 — Install Cosmovisor](#step-5--install-cosmovisor)
- [Step 6 — Create Systemd Service](#step-6--create-systemd-service)
- [Step 7 — Initialize the Node](#step-7--initialize-the-node)
- [Step 8 — Download Genesis and Addrbook](#step-8--download-genesis-and-addrbook)
- [Step 9 — Configure Ports, Gas Prices and Pruning](#step-9--configure-ports-gas-prices-and-pruning)
- [Step 10 — Configure Seeds and Peers](#step-10--configure-seeds-and-peers)
- [Step 11 — Download Snapshot](#step-11--download-snapshot)
- [Step 12 — Start the Node](#step-12--start-the-node)
- [Step 13 — Create a Wallet](#step-13--create-a-wallet)
- [Step 14 — Register as a Validator](#step-14--register-as-a-validator)
- [Monitoring the Node](#monitoring-the-node)
- [Useful Commands](#useful-commands)
- [Staying Updated](#staying-updated)

---

## Hardware Requirements

| Component | Minimum | Recommended |
|---|---|---|
| Operating System | Ubuntu 22.04+ | Ubuntu 24.04 |
| CPU | 4 cores | 8 cores |
| RAM | 16 GB | 32 GB |
| Disk | 500 GB NVMe SSD | 2 TB NVMe SSD |
| Network | 100 Mbps | 1 Gbps |

> ⚠️ Disk usage grows over time. Plan for long-term storage growth if running an archival node.

---

## Step 1 — System Verification

After SSH-ing into your server, verify the system meets requirements:

```bash
lsb_release -a          # Should be Ubuntu 22.04 or higher
uname -r                # Kernel version
lscpu | grep -E "Model name|CPU\(s\)|Thread|Socket|Core"
free -h                 # Minimum 16 GB RAM
df -h                   # Minimum 500 GB free disk
```

---

## Step 2 — System Update and Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git wget htop tmux build-essential jq make lz4 gcc unzip
```

---

## Step 3 — Install Go

AtomOne v4.0.0 requires **Go 1.22+**:

```bash
cd $HOME
VER="1.22.10"
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

Expected output: `go version go1.22.10 linux/amd64`

---

## Step 4 — Download and Build Binary

```bash
cd $HOME
rm -rf atomone
git clone https://github.com/atomone-hub/atomone
cd atomone
git checkout v4.0.0
make build

mkdir -p $HOME/.atomone/cosmovisor/genesis/bin
mv build/atomoned $HOME/.atomone/cosmovisor/genesis/bin/
rm -rf build

ln -s $HOME/.atomone/cosmovisor/genesis $HOME/.atomone/cosmovisor/current -f
sudo ln -s $HOME/.atomone/cosmovisor/current/bin/atomoned /usr/local/bin/atomoned -f
```

Verify:

```bash
atomoned version
```

Expected output: `v4.0.0`

> **Alternatively**, download the pre-built binary directly:
> ```bash
> wget -O atomoned https://github.com/atomone-hub/atomone/releases/download/v4.0.0/atomoned-v4.0.0-linux-amd64
> chmod +x atomoned
> mkdir -p $HOME/.atomone/cosmovisor/genesis/bin
> mv atomoned $HOME/.atomone/cosmovisor/genesis/bin/
> ln -s $HOME/.atomone/cosmovisor/genesis $HOME/.atomone/cosmovisor/current -f
> sudo ln -s $HOME/.atomone/cosmovisor/current/bin/atomoned /usr/local/bin/atomoned -f
> ```

---

## Step 5 — Install Cosmovisor

Cosmovisor handles automatic binary upgrades:

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0
```

Verify:

```bash
cosmovisor version
```

---

## Step 6 — Create Systemd Service

Set your moniker and port prefix before creating the service:

```bash
MONIKER="YOUR_MONIKER"
PORT="26"   # Default is 26. Change to avoid conflicts (e.g. 27, 28...)
```

Create the service file:

```bash
sudo tee /etc/systemd/system/atomoned.service > /dev/null << EOF
[Unit]
Description=AtomOne Node Service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start --home $HOME/.atomone
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.atomone"
Environment="DAEMON_NAME=atomoned"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.atomone/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable atomoned
```

---

## Step 7 — Initialize the Node

```bash
atomoned config set client chain-id atomone-testnet-1
atomoned config set client keyring-backend os
atomoned config set client node tcp://localhost:${PORT}657

atomoned init $MONIKER --chain-id atomone-testnet-1
```

Set environment variables permanently:

```bash
echo "export MONIKER=$MONIKER" >> $HOME/.bash_profile
echo "export ATOMONE_CHAIN_ID=\"atomone-testnet-1\"" >> $HOME/.bash_profile
echo "export ATOMONE_PORT=$PORT" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

---

## Step 8 — Download Genesis and Addrbook

```bash
wget -O $HOME/.atomone/config/genesis.json \
  https://atomone.fra1.digitaloceanspaces.com/atomone-testnet-1/genesis.json

wget -O $HOME/.atomone/config/addrbook.json \
  https://raw.githubusercontent.com/hazennetworksolutions/atomone-testnet/refs/heads/main/addrbook.json
```

Verify genesis checksum:

```bash
sha256sum $HOME/.atomone/config/genesis.json
```

---

## Step 9 — Configure Ports, Gas Prices and Pruning

### Custom Ports

If you changed the default port prefix, apply it here:

```bash
sed -i.bak -e "s%:1317%:${ATOMONE_PORT}317%g;
s%:8080%:${ATOMONE_PORT}080%g;
s%:9090%:${ATOMONE_PORT}090%g;
s%:9091%:${ATOMONE_PORT}091%g" $HOME/.atomone/config/app.toml

sed -i.bak -e "s%:26658%:${ATOMONE_PORT}658%g;
s%:26657%:${ATOMONE_PORT}657%g;
s%:6060%:${ATOMONE_PORT}060%g;
s%:26656%:${ATOMONE_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${ATOMONE_PORT}656\"%;
s%:26660%:${ATOMONE_PORT}660%g" $HOME/.atomone/config/config.toml
```

### Gas Prices

```bash
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.025uatone,0.025uphoton"|g' \
  $HOME/.atomone/config/app.toml
```

### Pruning

```bash
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.atomone/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.atomone/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.atomone/config/app.toml
```

### Enable Prometheus (optional)

```bash
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.atomone/config/config.toml
```

### Disable Indexer (saves disk space)

```bash
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.atomone/config/config.toml
```

---

## Step 10 — Configure Seeds and Peers

```bash
SEEDS="85e441cfe74b8c0f8b820beff46edab20e92716c@atomone-testnet-seed.itrocket.net:62657"
PEERS="bddf062a8328bcc50e37f83448e770e2e385c72f@atomone-testnet-peer.itrocket.net:62656,4adfce6fcbd3dc61109d8c67801272a162ecd29e@116.202.210.177:62656,14dbb758d8805a146497227caafe224a4ea29c2b@46.232.248.39:19656,dd27a23e0adc98d6dc53802d95ce581b06723845@185.252.233.217:26656,29d901d882d40048124670f6ea902cd933c9aa36@37.60.255.34:26656,2231b2285c3ba2f0dec145633d5bc90b8cf782bd@161.97.77.219:26656,843c14811951b44e7a55e7086d93f5425b549321@213.217.234.65:26656"

sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" \
       $HOME/.atomone/config/config.toml
```

---

## Step 11 — Download Snapshot

Syncing from genesis takes a very long time. Use a snapshot to speed up the process:

```bash
sudo systemctl stop atomoned

# Back up validator state
cp $HOME/.atomone/data/priv_validator_state.json $HOME/.atomone/priv_validator_state.json.backup

# Reset data directory
atomoned tendermint unsafe-reset-all --home $HOME/.atomone --keep-addr-book
```

Download and apply the snapshot:

```bash
SNAPSHOT_URL="https://server-2.itrocket.net/testnet/atomone/"

# List available snapshots
curl -s $SNAPSHOT_URL | grep -o '"atomone_[^"]*"' | head -5

# Download the latest snapshot (replace FILENAME with the latest one from the list above)
FILENAME="atomone_LATEST.tar.lz4"
curl -o - -L $SNAPSHOT_URL$FILENAME | lz4 -c -d - | tar -x -C $HOME/.atomone
```

Restore validator state:

```bash
mv $HOME/.atomone/priv_validator_state.json.backup $HOME/.atomone/data/priv_validator_state.json
```

---

## Step 12 — Start the Node

```bash
sudo systemctl start atomoned
sudo journalctl -u atomoned -f --no-pager -o cat
```

Verify all services are running:

```bash
sudo systemctl status atomoned --no-pager
```

The service should show `active (running)`.

Check sync status:

```bash
atomoned status 2>&1 | jq .SyncInfo
```

Wait until `catching_up` is `false` before proceeding to validator registration.

---

## Step 13 — Create a Wallet

```bash
atomoned keys add $WALLET
```

> ⚠️ **CRITICAL:** Save your mnemonic phrase in a secure location. Without it, you cannot recover your wallet.

To recover an existing wallet:

```bash
atomoned keys add $WALLET --recover
```

Set the wallet variable:

```bash
echo "export WALLET=\"YOUR_WALLET_NAME\"" >> $HOME/.bash_profile
source $HOME/.bash_profile
WALLET_ADDRESS=$(atomoned keys show $WALLET -a)
VALOPER_ADDRESS=$(atomoned keys show $WALLET --bech val -a)
```

Check your balance (after receiving testnet tokens):

```bash
atomoned query bank balances $WALLET_ADDRESS
```

---

## Step 14 — Register as a Validator

> The node must be **fully synced** before creating a validator.

### Get your pubkey:

```bash
atomoned comet show-validator
```

### Create validator JSON:

```bash
cat > $HOME/validator.json << EOF
{
  "pubkey": $(atomoned comet show-validator),
  "amount": "1000000uatone",
  "moniker": "$MONIKER",
  "identity": "",
  "website": "",
  "security": "",
  "details": "AtomOne Validator by HazenNetworkSolutions",
  "commission-rate": "0.05",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
EOF
```

### Submit the transaction:

```bash
atomoned tx staking create-validator $HOME/validator.json \
  --from $WALLET \
  --chain-id atomone-testnet-1 \
  --gas auto \
  --gas-adjustment 1.4 \
  --fees 500uatone \
  -y
```

### Verify your validator:

```bash
atomoned query staking validator $VALOPER_ADDRESS
```

---

## Monitoring the Node

### Watch live block commits:

```bash
sudo journalctl -u atomoned -f --no-pager | grep "committed block"
```

Expected output:

```
INF committed state app_hash=... height=XXXXXX
```

### Full logs:

```bash
sudo journalctl -u atomoned -f --no-pager
```

### Sync status:

```bash
atomoned status 2>&1 | jq .SyncInfo
```

### Service management:

```bash
# Restart service
sudo systemctl restart atomoned

# Stop service
sudo systemctl stop atomoned

# Check status
sudo systemctl status atomoned
```

---

## Useful Commands

### Wallet

```bash
# List wallets
atomoned keys list

# Show wallet address
atomoned keys show $WALLET -a

# Check balance
atomoned query bank balances $WALLET_ADDRESS
```

### Staking

```bash
# Delegate tokens
atomoned tx staking delegate $VALOPER_ADDRESS 1000000uatone \
  --from $WALLET --chain-id atomone-testnet-1 --gas auto --gas-adjustment 1.4 --fees 500uatone -y

# Redelegate tokens
atomoned tx staking redelegate $VALOPER_ADDRESS <NEW_VALOPER> 1000000uatone \
  --from $WALLET --chain-id atomone-testnet-1 --gas auto --gas-adjustment 1.4 --fees 500uatone -y

# Undelegate tokens
atomoned tx staking unbond $VALOPER_ADDRESS 1000000uatone \
  --from $WALLET --chain-id atomone-testnet-1 --gas auto --gas-adjustment 1.4 --fees 500uatone -y
```

### Rewards

```bash
# Withdraw all rewards
atomoned tx distribution withdraw-all-rewards \
  --from $WALLET --chain-id atomone-testnet-1 --gas auto --gas-adjustment 1.4 --fees 500uatone -y

# Withdraw commission
atomoned tx distribution withdraw-rewards $VALOPER_ADDRESS --commission \
  --from $WALLET --chain-id atomone-testnet-1 --gas auto --gas-adjustment 1.4 --fees 500uatone -y
```

### Governance

```bash
# List proposals
atomoned query gov proposals

# Vote on a proposal
atomoned tx gov vote 1 yes \
  --from $WALLET --chain-id atomone-testnet-1 --gas auto --gas-adjustment 1.4 --fees 500uatone -y
```

### Validator Operations

```bash
# Edit validator
atomoned tx staking edit-validator \
  --new-moniker "NEW_MONIKER" \
  --identity "" \
  --from $WALLET --chain-id atomone-testnet-1 --gas auto --gas-adjustment 1.4 --fees 500uatone -y

# Unjail validator
atomoned tx slashing unjail \
  --from $WALLET --chain-id atomone-testnet-1 --gas auto --gas-adjustment 1.4 --fees 500uatone -y

# Check validator signing info
atomoned query slashing signing-info $(atomoned comet show-validator)
```

---

## Staying Updated

Follow these channels to stay informed about upgrades and announcements:

- Discord: [AtomOne Developer Discord](https://discord.gg/atomone)
- GitHub Releases: [atomone-hub/atomone Releases](https://github.com/atomone-hub/atomone/releases)
- Official Docs: [docs.atom.one](https://docs.atom.one/guides/node-guide/)

### Upgrade with Cosmovisor (example: v5.0.0)

```bash
mkdir -p $HOME/.atomone/cosmovisor/upgrades/v5/bin
wget -O $HOME/.atomone/cosmovisor/upgrades/v5/bin/atomoned \
  https://github.com/atomone-hub/atomone/releases/download/v5.0.0/atomoned-v5.0.0-linux-amd64
chmod +x $HOME/.atomone/cosmovisor/upgrades/v5/bin/atomoned
```

Cosmovisor will automatically switch to the new binary at the upgrade block height.

---

## About the Author

This guide was prepared by **HazenNetworkSolutions**.  
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
