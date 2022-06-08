# Gerta testnet

This guide assumes you're running on Ubuntu 22.04 LTS - the commands will probably work just fine on 20.04 LTS or Debian.

## Basic Machine setup

1. ssh into your node
2. update your machine `sudo apt update && sudo apt dist-upgrade -y`. Answer yes / ok to the prompts
3. install required tools tools `sudo apt install build-essential git unzip curl wget`

### Golang

First off we install go 1.18.x
1. `wget https://go.dev/dl/go1.18.2.linux-amd64.tar.gz`
2. extract the runtime `sudo tar -C /usr/local -xzf go1.18.2.linux-amd64.tar.gz`

## Get the testnet up and running

1. create the gerta user and switch to it
```
sudo useradd -m gerta
sudo su -s /bin/bash -l gerta
```
2. Add go to your path (add these lines to `~/.profile` or `~/.bashrc`)
```
export PATH=$PATH:/usr/local/go/bin
export PATH=$PATH:$(go env GOPATH)/bin
```
4. Run `source ~/.profile` and/or `source ~/.bashrc`
3. Time to grab the code
```bash
git clone https://github.com/gerta-Chain/core $HOME/gerta-core
```

## Now we build!

Time to ignite the build sequence (using the `gerta` user created above)

1. move into the source folder 
```
cd $HOME/gerta-core
```
2. build and install `gertad`
```
make install
```
3. verify your binary is working
```
gertad version
```

## Running the test net

If the build succeed you should now have the `gertad` cli in your path (of the `gerta` user).

Try invoking with `gertad` and you should see output like

```
Stargate CosmosHub App

Usage:
  gertad [command]

Available Commands:
  add-genesis-account Add a genesis account to genesis.json
  collect-gentxs      Collect genesis txs and output a genesis.json file
  config              Create or query an application CLI configuration file
  debug               Tool for helping with debugging your application
  export              Export state to JSON
  gentx               Generate a genesis tx carrying a self delegation
  help                Help about any command
....
```

## Join the test net

Now we can initialize and join the network

1. Initialize

```
export CHAIN_ID=stella-1
export MONIKER_NAME="<moniker name>"
gertad init "${MONIKER_NAME}" --chain-id ${CHAIN_ID}
```

Replacing `<moniker name>` with your desired name.

2. Fetch the genesis `genesis.json` file

```
wget https://raw.githubusercontent.com/gerta-Chain/networks/master/testnet/genesis.json -O $HOME/.gerta/config/genesis.json
```
3. Download the `addrbook.json` file

```
wget https://raw.githubusercontent.com/gerta-Chain/networks/master/testnet/addrbook.json -O $HOME/.gerta/config/addrbook.json
```

Now try to start the network

```bash
$ gertad start

gerta@fsn1-gerta-testnet-01:~$ gertad start
12:22PM INF starting node with ABCI Tendermint in-process
12:22PM INF Starting multiAppConn service impl=multiAppConn module=proxy
12:22PM INF Starting localClient service connection=query impl=localClient module=abci-client
12:22PM INF Starting localClient service connection=snapshot impl=localClient module=abci-client
12:22PM INF Starting localClient service connection=mempool impl=localClient module=abci-client
12:22PM INF Starting localClient service connection=consensus impl=localClient module=abci-client
```

And then a whole bunch of log messages while your node is catching up. If this is now verified working it's time to install it as a system level service so it always starts with the machine

## Registering as a service

Drop out of the `gerta` user if you're still in that terminal session. Write `exit` or type `ctrl+d` on your keyboard.

1. Create a service definition file in `/etc/systemd/system/gertad.service`. Example file that fits with our kuijrad install and gerta runtime user:

```
[Unit]
Description=gerta Daemon
After=network.target

[Service]
Type=simple
User=gerta
ExecStart=/home/gerta/go/bin/gertad start --log_level error 
Restart=on-abort

[Install]
WantedBy=multi-user.target

[Service]
LimitNOFILE=65535  
```

1. Reload your systemctl `sudo systemctl daemon-reload` and enable the service `sudo systemctl enable gertad`
2. And finally start the service with `sudo systemctl start gertad`
3. Check the status of the service with `systemctl status gertad.service` - it should return something like

```
● gertad.service - gerta Daemon
     Loaded: loaded (/etc/systemd/system/gertad.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2022-05-24 12:34:26 UTC; 5s ago
   Main PID: 18490 (gertad)
      Tasks: 11 (limit: 4541)
     Memory: 65.4M
        CPU: 4.430s
     CGroup: /system.slice/gertad.service
             └─18490 /home/gerta/go/bin/gertad start
...
```

Once your node is synced. it is time to make it a validator.
this can be done on a seperate machine if preferred.

create a key that will be the validators key. I have chosen the creative name of 'validator'
```
gertad keys add validator
```
copy the seed phrase and put it somewhere safe.
you will need to also make note of the address "gerta..." and use that in the faucet to get some coins.
you can check your balance via
```
gertad query bank balances gerta....
```

The next part is associating your node with your account, creating the validator
```
export PUBKEY=$( gertad tendermint show-validator)
export CHAIN_ID=
export MONIKER_NAME="<your moniker>"
gertad tx staking create-validator --moniker="${MONIKER_NAME}" \
 --amount=1000000ugerta \
        --gas-prices=1ugerta \
        --pubkey=$PUBKEY \
         --from=validator \
        --yes \
        --node=tcp://localhost:26657 \
        --chain-id=${CHAIN_ID} \
        --commission-max-change-rate=0.01 \
        --commission-max-rate=0.20 \
        --commission-rate=0.10 \
        --min-self-delegation=1
```
now your node should be present. 
```
gertad query staking validators|grep details
```
please remember to also back up  $HOME/.gerta/config/priv_validator_key.json
if you lose this, you are toast.


## Tips
### disk utilization
to help manage the disk size you can prune the blocks being kept. for this I use prime numbers pick your own 
in app.toml
```
pruning = "custom"

# These are applied if and only if the pruning strategy is custom.
pruning-keep-recent = "809"
pruning-keep-every = "0"
pruning-interval = "43"
```
you should also check what you are indexing
```
index-events = ["tx.hash", "tx.height"]
```
### adding more peers.
you should modify /etc/security/limits.conf 
and add
```
*                soft    nofile          65535
*                hard    nofile          65535
```
you can then modify the config.yaml to increase connections. This may cost you more in ingress/egress charges.
```
max_open_connections = 1900
max_num_inbound_peers = 50
max_num_outbound_peers = 50
```
