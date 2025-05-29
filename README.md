## 🚀 Deploy a NEAR Endorsements Validator Node (Mainnet)

> ⚠️ **Important**: To participate as a validator, you must have a NEAR account with **at least 4 NEAR** available. You will also need access to that account in order to sign transactions and generate your validator key. If you're creating a new account, make sure it is funded and you know its full address (account ID) before starting.

> This guide walks you through installing and running a NEAR endorsements validator node from scratch. No advanced knowledge is required, just basic familiarity with Linux CLI.

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
sudo apt install -y git curl wget jq build-essential clang python3
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

> ✅ Once the build completes, return to your home directory before continuing:
>
> ```bash
> cd ~
> ```

---

### 📡 Step 3 – Prepare NEAR Directory

Create the required directory structure:

```bash
mkdir -p ~/.near/data && cd ~/.near
```

---

### 🔑 Step 4 – Generate `node_key.json`

Use the following command to generate a `node_key.json`:

```bash
~/nearcore/target/release/neard init --chain-id mainnet
```

> ⚠️ This command will also create default versions of `config.json` and `genesis.json`. These files will be replaced in the next step with the official configuration from the NEAR network.

---

### 📡 Step 5 – Download Official Configuration Files

Download and overwrite the default configuration with the official mainnet files:

```bash
wget -O ~/.near/genesis.json https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/mainnet/genesis.json
wget -O ~/.near/config.json https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/mainnet/validator/config.json
```

---

### 🔐 Step 6 – Retrieve `validator_key.json`

To participate as a validator on NEAR, you'll need a `validator_key.json` tied to your validator account. This is automatically saved when you authenticate with NEAR CLI using your funded account.

First, install NEAR CLI:

```bash
sudo npm install -g near-cli
```

Before logging in, set the network environment to mainnet and make it persistent:

```bash
export NEAR_ENV=mainnet
echo 'export NEAR_ENV=mainnet' >> ~/.bashrc
source ~/.bashrc
```

Then, authenticate with your validator account:

```bash
near login
```

> 🔐 This will open a browser window where you can sign in with your NEAR Wallet (Meteor or any other wallet of your choice) and authorize the CLI.
>
> 💡 **If you're using a server without a GUI**, copy the URL and open it in a browser on another device. After signing the transaction, the CLI might return an error because it cannot open the local redirect (`127.0.0.1`). This is expected. Simply provide your NEAR account ID when prompted and confirm that the message `Logged in successfully` appears.
>
> ⚠️ Make sure the account you're logging in with is already funded with at least 4 NEAR, and that you know the full account ID before proceeding. If you're creating a new account during the login process, it may not be properly saved or funded, leading to errors.

Once logged in, your credentials will be saved locally in:

```
~/.near-credentials/mainnet/<your-validator-account>.json
```

Rename and move this file to the required location:

```bash
mv ~/.near-credentials/mainnet/<your-validator-account>.json ~/.near/validator_key.json
```

> ⚠️ This file contains your private key. Keep it secure and never share it.

---

### 🕝 Step 7 – Upload Existing Keys (if needed)

If you're restoring an existing validator identity, transfer your `node_key.json` and `validator_key.json` into `~/.near/`:

```bash
scp node_key.json username@your.server:/home/username/.near/
scp validator_key.json username@your.server:/home/username/.near/
```

> 💡 If you already generated `node_key.json` in Step 4, you can skip copying it manually.

> 🔐 These are critical for your validator identity. Never share them.

---

### ⚡ Step 8 – Sync via Snapshot

```bash
# Get list of boot nodes (filtering duplicate IPs)
BOOT_NODES=$(curl -s -X POST https://rpc.mainnet.near.org -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0", "method": "network_info", "params": [], "id": "sync"
}' | jq -r '.result.active_peers[] | "\(.addr) \(.id)"' | sort -u | awk '!seen[$1]++ {print $2 "@" $1}' | paste -sd "," -)

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

> 🔍 If you're applying the snapshot after starting the node, make sure to stop it first:
>
> ```bash
> sudo systemctl stop neard
> ```

---

### 🔧 Step 9 – Create systemd Service

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

### 🗺 Step 10 – Monitor the Logs

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

> 💡 These changes apply immediately and persist after reboot. **No need to stop the ************`neard`************ service to apply them.**

---

### ✅ Step 11 – Announce the Validator (Stake)

To activate your validator, you must deposit and stake NEAR from your validator account:

```bash
near call <your-validator-account> deposit_and_stake --accountId <signer-account> --amount "30"
```

* Replace `<your-validator-account>` with the account you want to stake for. This is the account whose validator key is in `validator_key.json`.
* Replace `<signer-account>` with the account that is executing and signing the transaction. If you're using the same account for both, these will be identical.

> 📅 **Example:**
>
> ```bash
> near call validator-account.near deposit_and_stake --accountId validator-account.near --amount "30"
> ```

* `--amount` is the amount of NEAR tokens to stake (minimum is 30 NEAR to enter the validator set).

> 🔐 This command signs a transaction. Ensure you’re logged in with the right account and have enough balance.

Check your validator status:

```bash
near validators current
```

Your account may appear in one of these states:

* **current**: actively validating
* **next**: will start validating in the next epoch
* **proposals**: pending approval
* **kicked out**: did not meet requirements or failed performance checks

You're now officially participating in NEAR's proof-of-stake consensus! 🎉

### 🔗 Extra Resources

* [NEAR Validator Docs](https://docs.near.org)
* [NEAR GitHub](https://github.com/near/nearcore)
* [NEAR Explorer](https://explorer.mainnet.near.org/)
* [Snapshot source](https://snapshots.mainnet.near.org/)

---

> ✅ Always validate each step as you go.
---

**Maintained by:** Meta Pool · [https://metapool.app](https://metapool.app)
