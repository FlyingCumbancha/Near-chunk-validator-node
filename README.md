## 🚀 Deploy a NEAR Chunk Validator Node (Mainnet)

> This guide walks you through installing and running a NEAR chunk validator node from scratch. No advanced knowledge is required, just basic familiarity with Linux CLI.

---

### 📋 Recommended Server Specs

| Component | Minimum Requirement     |
| --------- | ----------------------- |
| CPU       | 4 vCPUs (Intel/AMD)     |
| RAM       | 8 GB                    |
| Disk      | 160 GB SSD              |
| OS        | Debian 12 (recommended) |

> 💡 **Note**: Ubuntu 22.04 LTS also works fine.

---

### 📦 Step 1 – Install Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl wget jq build-essential clang python3 awscli
```

---

### 🦀 Step 2 – Install Rust and NEAR Core

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

git clone https://github.com/near/nearcore.git
cd nearcore
export NEAR_RELEASE_VERSION=2.6.3
git checkout $NEAR_RELEASE_VERSION
make release
```

> 🔧 This may take a few minutes. The `neard` binary will be built in `target/release/`.

---

### 📡 Step 3 – Download Configuration Files

```bash
mkdir -p ~/.near/data && cd ~/.near

wget -c https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/mainnet/genesis.json
wget -c https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/mainnet/validator/config.json
```

---

### 🔑 Step 4 – Upload Your Keys

Transfer your `node_key.json` and `validator_key.json` into `~/.near/`:

```bash
scp node_key.json username@your.server:/home/username/.near/
scp validator_key.json username@your.server:/home/username/.near/
```

> 🔐 These are critical for your validator identity. Never share them.

---

### ⚡ Step 5 – Sync via Snapshot

```bash
# Get list of boot nodes
BOOT_NODES=$(curl -s -X POST https://rpc.mainnet.near.org -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0", "method": "network_info", "params": [], "id": "sync"
}' | jq -r '.result.active_peers[] | "\(.id)@\(.addr)"' | paste -sd "," -)

# Inject into config.json
jq --arg nodes "$BOOT_NODES" '.network.boot_nodes = $nodes' ~/.near/config.json > ~/.near/config.json.tmp && mv ~/.near/config.json.tmp ~/.near/config.json
```

> 🚀 Optional: use snapshot for faster sync.

```bash
cd ~/.near
wget https://snapshots.mainnet.near.org/latest.tar.gz
rm -rf data/*
tar -xvf latest.tar.gz --strip-components=1 -C data/
```

---

### 🔧 Step 6 – Create systemd Service

```bash
sudo tee /etc/systemd/system/neard.service > /dev/null <<EOF
[Unit]
Description=NEAR Validator Node
After=network.target

[Service]
User=$USER
ExecStart=$HOME/nearcore/target/release/neard run
WorkingDirectory=$HOME/.near
Restart=always
RestartSec=30
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reexec
sudo systemctl enable --now neard
```

---

### 🧪 Step 7 – Monitor the Logs

```bash
journalctl -u neard -f -n 100
```

---

### ⚙️ (Optional) Network Performance Tuning

If you're running on a dedicated server and want to tune TCP parameters for better validator performance:

```bash
MaxExpectedPathBDP=8388608

sudo sysctl -w net.core.rmem_max=$MaxExpectedPathBDP
sudo sysctl -w net.core.wmem_max=$MaxExpectedPathBDP
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 $MaxExpectedPathBDP"
sudo sysctl -w net.ipv4.tcp_wmem="4096 16384 $MaxExpectedPathBDP"
sudo sysctl -w net.ipv4.tcp_slow_start_after_idle=0

sudo bash -c "cat > /etc/sysctl.d/99-near-validator.conf" <<EOL
net.core.rmem_max = $MaxExpectedPathBDP
net.core.wmem_max = $MaxExpectedPathBDP
net.ipv4.tcp_rmem = 4096 87380 $MaxExpectedPathBDP
net.ipv4.tcp_wmem = 4096 16384 $MaxExpectedPathBDP
net.ipv4.tcp_slow_start_after_idle = 0
EOL

sudo sysctl --system
```

---

### 🔗 Resources

* [NEAR Validator Docs](https://docs.near.org)
* [NEAR GitHub](https://github.com/near/nearcore)
* [NEAR Explorer](https://explorer.mainnet.near.org/)
* [Snapshot source](https://snapshots.mainnet.near.org/)

---

> ✅ Validate each step as you go. We'll incorporate corrections live based on your testing.

---

**Maintained by:** Meta Pool · [https://metapool.app](https://metapool.app)
