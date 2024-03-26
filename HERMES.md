# Guide to Run Hermes Relayer

---
## What is Hermes?
Hermes is an IBC (Inter-Blockchain Communication) relayer implemented in Rust, used for enabling communication between different blockchains that support IBC. It allows for the transfer of data and tokens between compatible chains, facilitating interoperability in the blockchain ecosystem.

---
## Prerequisites
Before running Hermes, ensure you have the following:
- Two IBC-enabled blockchain nodes running and accessible.
- Native tokens and network access to both blockchain nodes (rpc, grpc available).
---
## Installation

The simplest way to install Hermes on Unix machine (Linux/macOS) is to download it from repo.
If you would like to build it from source or install over cargo you can follow by these 
steps described in [official docs](https://hermes.informal.systems/quick-start/installation.html). 

> We use here ubuntu 22.04, x86_64 arch, if you have another spec investigate 
> [binaries](https://github.com/informalsystems/hermes/releases) what suites to you.
```bash
VERSION=v1.8.2

mkdir -p $HOME/.hermes/bin
https://github.com/informalsystems/hermes/releases/download/v1.8.2/hermes-v1.8.2-x86_64-unknown-linux-gnu.tar.gz
wget https://github.com/informalsystems/hermes/releases/download/$VERSION/hermes-$VERSION-x86_64-unknown-linux-gnu.tar.gz
tar -xf hermes-$VERSION-x86_64-unknown-linux-gnu.tar.gz -C $HOME/.hermes/bin
rm hermes-$VERSION-x86_64-unknown-linux-gnu.tar.gz

PATH_INCLUDES_GO=$(grep "$HOME/.hermes/bin" $HOME/.bash_profile)
if [ -z "$PATH_INCLUDES_GO" ]; then
    echo "export PATH=\$PATH:$HOME/.hermes/bin" >> "$HOME/.bash_profile"
    source $HOME/.bash_profile
fi
```

Check hermes is executable and running without issues.
```bash
hermes version 
# 2024-03-24T17:39:37.086297Z  INFO ThreadId(01) running Hermes v1.8.2+06dfbaf
# hermes 1.8.2+06dfbaf
```

---
## Configuration
Hermes requires a configuration file to operate. Here's a basic template for the configuration file, config.toml.
Meaning of those parameters you can always read in [doc](https://hermes.informal.systems/documentation/configuration/description.html):

```bash
tee $HOME/.hermes/config.toml > /dev/null << EOF
[global]
log_level = 'info'

[mode]
[mode.clients]
enabled = true
refresh = true
misbehaviour = false

[mode.connections]
enabled = false

[mode.channels]
enabled = false

[mode.packets]
enabled = true
clear_interval = 200
clear_on_start = true
tx_confirmation = true
auto_register_counterparty_payee = false

[rest]
enabled = false
host = '0.0.0.0'
port = 3000

[telemetry]
enabled = false
host = '0.0.0.0'
port = 3001
EOF
```
Add networks you would like to relay in the end. Use `vim` or `nano` or any else editor to do this.
We will use working in testnet Elys<>Juno as example. 
> Pay attention to these parameters:
> - **memo_prefix** - should be your own, could be random names
> - **key_name** - some meaningful names, will be set during restore keys
> - **rpc_addr, grpc_addr, event_source.url** - working and accessible endpoints to your node.
> - information regarding channel, if you would like to use relay another path could be got here: [paths](https://docs.google.com/spreadsheets/u/3/d/1CuDdV2Rf-ph0HQ5ViUyNY_Z-ix-Rarcd1i4PJKhCuLw/htmlview)
```bash
nano $HOME/.hermes/config.toml
```
```toml
[[chains]]
id = 'elystestnet-1'

rpc_addr = 'http://127.0.0.1:38657'
grpc_addr = 'http://127.0.0.1:10290'
event_source = { mode = 'push', url = 'ws://127.0.0.1:38657/websocket', batch_delay = '500ms' }

rpc_timeout = '10s'

account_prefix = 'elys'
key_name = 'elys-relayer'
store_prefix = 'ibc'

default_gas = 800000
max_gas = 1600000
gas_price = { price = 0.00025, denom = 'uelys' }
gas_multiplier = 1.2

max_msg_num = 30
max_tx_size = 180000   # hermes default 2097152

clock_drift = '10s'
max_block_time = '30s'
trusting_period = '12days'
trust_threshold = { numerator = '1', denominator = '3' }
address_type = { derivation = 'cosmos' }

memo_prefix = 'relayer-memo'
[chains.packet_filter]
policy = 'allow'
list = [
    ['transfer', 'channel-10'], # juno testnet
]

[[chains]]
id = 'uni-6'

rpc_addr = 'http://127.0.0.1:33657'
grpc_addr = 'http://127.0.0.1:9790'
event_source = { mode = 'push', url = 'ws://127.0.0.1:33657/websocket', batch_delay = '500ms' }

rpc_timeout = '10s'

account_prefix = 'juno'
key_name = 'juno-relayer'
store_prefix = 'ibc'

default_gas = 800000
max_gas = 1600000
gas_price = { price = 0.025, denom = 'ujunox' }
gas_multiplier = 1.2

max_msg_num = 30
max_tx_size = 180000

clock_drift = '30s'
max_block_time = '30s'
trusting_period = '4days'
trust_threshold = { numerator = '1', denominator = '3' }
address_type = { derivation = 'cosmos' }

memo_prefix = 'relayer-memo'
[chains.packet_filter]
policy = 'allow'
list = [
    ['transfer', 'channel-641'], # elys
]
```

---
## Adding Keys
You need to add keys for both chains in Hermes, do not use wallets for anything else but relaying to avoid running into account sequence errors. Make sure wallets in both chains are funded and mnemonics are backed up.

>Execute the following:

```bash
ELYS_WALLET_MNEMONIC="24 word phrase"
JUNO_WALLET_MNEMONIC="24 word phrase"
echo $ELYS_WALLET_MNEMONIC >> $HOME/.hermes/elys-relayer.txt
echo $JUNO_WALLET_MNEMONIC >> $HOME/.hermes/juno-relayer.txt

hermes keys add --key-name elys-relayer --chain elystestnet-1 --mnemonic-file HOME/.hermes/elys-relayer.txt
hermes keys add --key-name juno-relayer --chain uni-6 --mnemonic-file $HOME/.hermes/juno-relayer.txt
```
## Verify configuration files 
After editing config.toml and adding wallet keys, it’s time to test the configurations and ensure the system is healthy. Run the following:

```bash
hermes health-check
hermes config validate
```

You should see similar output:

```bash
SUCCESS performed health check for all chains in the config
SUCCESS "configuration is valid"
```

## Starting the Relayer
With the configuration file in place and keys added, you can start the relayer

Create Hermes service file
```bash
sudo tee /etc/systemd/system/hermesd.service > /dev/null << EOF
[Unit]
Description=HERMES
After=network.target
[Service]
Type=simple
User=$USER
ExecStart=$HOME/.hermes/bin/hermes start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```
Enable added service file and start hermes.
```bash
sudo systemctl enable hermesd.service
sudo systemctl daemon-reload
sudo systemctl start hermesd.service
```

---
## Useful commands
Query hermes for unreceived packets and acknowledgements with the following:
```bash
hermes query packet pending --chain elystestnet-1 --port transfer --channel channel-10
hermes query packet pending --chain uni-6 --port transfer --channel channel-641
```

Clear the channel with the following:
```bash
hermes clear packets --chain elystestnet-1 --port transfer --channel channel-10
hermes clear packets --chain uni-6 --port transfer --channel channel-641
```

Update clients with the following:
```bash
hermes update client --host-chain elystestnet-1 --client 07-tendermint-12
hermes update client --host-chain uni-6 --client 07-tendermint-546
```

Check Hermes logs
```bash
 sudo journalctl -u hermesd.service -f -o cat
```

Stop Hermes
```bash
sudo systemctl stop hermesd.service
```

Restart Hermes
```bash
sudo systemctl restart hermesd.service
```

----
## Conclusion
This guide provides a basic overview of setting up and running the Hermes IBC relayer. For more advanced configurations and operations, refer to the official Hermes documentation.
Ensure you replace placeholder values with actual data from your blockchain nodes and configurations. This guide is a starting point, adjust configurations and commands as necessary for your specific setup and requirements.
