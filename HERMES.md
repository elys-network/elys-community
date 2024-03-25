# Guide to Running Hermes Relayer

---
## What is Hermes?
Hermes is an IBC (Inter-Blockchain Communication) relayer implemented in Rust, used for enabling communication between different blockchains that support IBC. It allows for the transfer of data and tokens between compatible chains, facilitating interoperability in the blockchain ecosystem.

---
## Prerequisites
Before running Hermes, ensure you have the following:
- Two IBC-enabled blockchain nodes running and accessible
- Network access to both blockchain nodes (rpc, grpc available)

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

Check hermes is executable and running without issues
```bash
hermes version 
# 2024-03-24T17:39:37.086297Z  INFO ThreadId(01) running Hermes v1.8.2+06dfbaf
# hermes 1.8.2+06dfbaf
```

---
## Configuration
Hermes requires a configuration file to operate. Here's a basic template for the configuration file, config.toml.
You can read about the meaning of these parameters in the [official documentation](https://hermes.informal.systems/documentation/configuration/description.html).

**Warning**: Informal's official documentation to set up Hermes includes creating new clients, channels and connections. This is generally not necessary or wanted, as relay paths often exist already. If you follow this documentation, make sure that it is relevant to create a new path entirely. 

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
**Tip**: if you relay multiple chains, Hermes can take a long while to start because it will scan each chain and channel. You can skip this step and make it start much faster by setting the following parameter:
```
[mode.clients]
enabled = false
```

Add networks you would like to relay in the end. Use `vim` or `nano` or any other editor to do this.
We will set up the relayer between the Elys and Juno testnets as an example. **Note**: in this instance, we'll assume that Hermes and the two nodes are installed locally, on the same server. If the nodes are remote, their actual IP addresses must be used, and the firewalls configured to allow incoming connnections from Hermes to their RPC and GRPC ports.
> Pay attention to these parameters:
> - **memo_prefix** - should be your own, could be random name
> - **key_name** - some meaningful name, will be set during restore keys
> - **rpc_addr, grpc_addr, event_source.url** - working and accessible endpoints to your node.
> - **list** - the channel types and numbers that are being relayed. Each channel on a chain has its _counterpart_ on the other chain. If you would like to relay others paths, find the existing ones here: [paths](https://docs.google.com/spreadsheets/u/3/d/1CuDdV2Rf-ph0HQ5ViUyNY_Z-ix-Rarcd1i4PJKhCuLw/htmlview)
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
The gas configuration should be carefully set:

`gas_price = { price = 0.025, denom = 'ujunox' }` should match the `minimum-gas-prices` value in the node's `app.toml`.
`max_gas`shouldn't exceed the value configured in the chain (the maximum amount of gas used in a single block), which can be obtained by querying an API endpoint, e.g. for Juno: `https://rest.testcosmos.directory/junotestnet/cosmos/consensus/v1/params`.
`default_gas` is the initial value that Hermes will set when trying to relay a packed, multiplying it by the `gas_multiplier` parameter. If you aren't sure or couldn't find the information, you can set tentative values and monitor the Hermes logs for errors saying `insufficient fees`: the log will tell what was used against what was needed, allowing you to adjust the parameters accordingly.
The objective here is to be able to relay packets while keeping the costs to a minimum -- as a reminder, the transaction fees are typically paid by the relayer.

---
## Adding Keys
You need to add keys for both chains in Hermes (the chains _must_ be set up in the configuration file for the command to succeed):

>Before execute the next step, you should have created wallets in both chains, backed up their mnemonics, and funded them.

```bash
ELYS_WALLET_MNEMONIC="24 word phrase"
JUNO_WALLET_MNEMONIC="24 word phrase"
echo $ELYS_WALLET_MNEMONIC >> $HOME/.hermes/elys-relayer.txt
echo $JUNO_WALLET_MNEMONIC >> $HOME/.hermes/juno-relayer.txt

hermes keys add --key-name elys-relayer --chain elystestnet-1 --mnemonic-file HOME/.hermes/elys-relayer.txt
hermes keys add --key-name juno-relayer --chain uni-6 --mnemonic-file $HOME/.hermes/juno-relayer.txt
```

**Important**: make sure that the imported wallet address is correct. In some cases, you will need to specify the _derivation path_ by adding a flag to the command, e.g. for Injective:`--hd-path "m/44'/60'/0'/0/0"`. Find the information about the chain's derivation path at https://cosmos.directory.

## Starting the Relayer
With the configuration file in place and keys added, you can start the relayer.
Add a service file first to run it through `systemd`.
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
Enable the service and start hermes.
```bash
sudo systemctl enable hermesd.service
sudo systemctl start hermesd.service
sudo journalctl -fu hermesd -n 100 #to monitor the activity of the relayer and verify that it works as intended.
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
**Important**: clients have a "lifespan", which is automatically reset each time they relay a packet. However if there is little to no traffic this lifespan can be reached, in which case the client will expire and the relay path will be closed. A client can only be reactivated through governance. Therefore, to prevent this it is advised to regularly pass this update command, for example as a cron job.  

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
Ensure you replace the placeholder values with actual data from your blockchain nodes and configurations. This guide is a starting point, adjust configurations and commands as necessary for your specific setup and requirements.
