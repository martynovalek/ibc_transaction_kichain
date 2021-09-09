# IBC Relayer settings. 
## Intro
Here's a guide to setting up cross-chain transfer in networks built on the Cosmos SDK using IBC.

We need two nodes in different networks. The Relayer can be placed on a separate VPS or on VPS together with one of the nodes.

>I use kichain and rizon in my case. The relayer is installed together with the Rizon node. You can use any other networks that supports IBC transfer.

## Checking the network for IBC support.
Use the following command to make sure that IBC transfer is supported:
`[network command] q ibc-transfer params`

>In my case, the command is:
>`rizond q ibc-transfer params`

Output will be:
`receive_enabled: true send_enabled: true`

## Table of contents
1. Install and initializing the relayer on the Rizon node.
2. Create a config files for both networks.
3. Get wallets
4. Open a channel for relaying
5. Make a cross-chain transfer.
6. Open channel for transfers from any node.

### 1. Install and initializing the relayer on the Rizon node
#### 1.1 Download and install the relayer from the official source Relayer v.1.0.0
```
git clone https://github.com/cosmos/relayer.git
cd relayer
make install
cd
rly version
```

>Output:<br>
version: 1.0.0-rc1–152-g112205b

#### 1.2 Initializing
`rly config init`

### 2. Create a config files for both networks

#### 2.1 Create a folder for config files
```
mkdir rly_config
cd rly_config
```

#### 2.2 Add config for KiChain network 
Here we can use the Global RPC address in the config file.

Create json file using command:
`nano kichain-t-4.json`

Insert this text in the file you created:
```
{
 "chain-id": "kichain-t-4",
  "rpc-addr": "https://rpc-challenge.blockchain.ki:443",
  "account-prefix": "tki",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025utki",
  "trusting-period": "48h"
}
```

**Save it**: ctrl+X, Y, enter

#### 2.3 Add config for Rizon network
Instead of the Global RPC address we use the **localhost**, since the node Rizon is located on the same server as the relayer.

Create json file using command:
`nano groot-011.json`

Insert this text in the file you created:
```
{
 "chain-id": "kichain-t-4",
 "rpc-addr": "http://localhost:26657",
  "account-prefix": "tki",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025utki",
  "trusting-period": "48h"
}
```

**Save it**: ctrl+X, Y, enter

#### 2.4 Add settings to the Relay Config
```
rly chains add -f groot-011.json
rly chains add -f kichain-t-4.json
cd
```

### 3. Get wallets
#### 3.1 Create or restore wallets
>In my case, I restored the existing keys on both networks.

For **create** wallets, using:
```
rly keys add groot-011 name_of_your_wallet
rly keys add kichain-t-4 name_of_your_wallet
```

For **restore** wallets, using:
```
rly keys restore groot-011 name of your wallet "mnemonic"
rly keys restore kichain-t-4 name of your wallet "mnemonic"
```

#### 3.2 Add the keys to the config of the relay
```
rly chains edit groot-011 key name_of_your_wallet
rly chains edit kichain-t-4 key name_of_your_wallet
```

#### 3.3 Change the timeout of waiting for confirmation to 30s
Use following command to open the config file:

`nano ~/.relayer/config/config.yaml`

Set timeout to 30s.<br>
**Save it**: ctrl+X, Y, enter

#### 3.4 Wallets must be funded on both networks
Check the availability of coins with the command.
```
rly q balance groot-011
rly q balance kichain-t-4
```

The output must not be empty.

#### 3.5 Initialize the light client in both networks
```
rly light init groot-011 -f
rly light init kichain-t-4 -f
```

### 4. Open a channel for relaying
#### 4.1 Generate a channel between the networks
`rly paths generate groot-011 kichain-t-4 transfer --port=transfer`

>Im used **'transfer'** name for the path. You can use another one.<br>
>If successful, the output will be:<br>
>`Generated path(transfer), run 'rly paths show transfer --yaml' to see details`

#### 4.2 Open a channel for relaying
Use following command:<br>
`rly tx link transfer`

Output:<br>
>★ Channel created: [groot-011]chan{channel-11}port{transfer} -> [kichain-t-4]chan{channel-41}port{transfer}

If it didn't work, try again or change the **config.yaml** file

Use command to open config.yaml<br>
`nano /root/.relayer/config/config.yaml`

Remove the following lines in both networks:
>client-id: 07-tendermint-18<br>
>connection-id: connection-25<br>
>channel-id: channel-23

**Save it**: ctrl+X, Y, enter

Follow step 3.5 and 4.2 again

#### 4.3 Check the channel
To make sure everything is OK, use the following command:<br>
`rly paths list -d`

Output:<br>
>0: transfer -> chns(✔) clnts(✔) conn(✔) chan(✔) (groot-011:transfer<>kichain-t-3:transfer)

### 5. Make a cross-chain token transfer

Template for cross-chain transation:<br>
`rly transact transfer [src-chain-id] [dst-chain-id] [amount] [dst-addr] [flags]`

#### 5.1 Transfer from Rizon to Kichain
In my case, the command looks like this:<br>
`rly tx transfer groot-011 kichain-t-4 5000000uatolo tki1cahyp4gs70w55ppnvneulpjjqv2ppc5xx53m2u --path transfer`

Output:<br>
>✔ [groot-011]@{427471} - msg(0:transfer) hash(405B95613CD768EC971E82F133288F5033F5193B425AF70E1D59A0BBC5C71775)

#### 5.2 Transfer from Kichain to Rizon
In my case, the command looks like this:<br>
`rly tx transfer kichain-t-4 groot-011 10000000utki rizon1gdrat404pvj4hxc9nvu65apq6nte5hyk862crs --path transfer`

Output:
>✔ [kichain-t-4]@{246212} - msg(0:transfer) hash(D26C3B00BFBFAA63FFAD7754304133A26AC81848820CC20C15B634154D8B7A02)

You can check your wallet keys using following commands:<br>
`rly keys list groot-011`<br>
`rly keys list kichaint-t-4`

### 6. Open channel for transfers from any node
#### 6.1 Create a service file for relayer
```
sudo tee /etc/systemd/system/rlyd.service > /dev/null <<EOF
[Unit]
Description=relayer client
After=network-online.target

[Service]
User=$USER
ExecStart=$(which rly) start transfer
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

#### 6.2 Start our service file
```
sudo systemctl daemon-reload
sudo systemctl enable rlyd
sudo systemctl start rlyd
```
#### 6.3 Transfer
Transfer from Kichain to Rizon.
Template:
`kid tx ibc-transfer transfer <path_name> <channel-N> rizon_WALLET_address 1000utki --from <name_OF_wallet> --chain-id kichain-t-4`

In my case, the command looks like this:<br>
````
kid tx ibc-transfer transfer transfer channel-26 rizon1gdrat404pvj4hxc9nvu65apq6nte5hyk862crs 1000utki --from my_rizon_wallet --chain-id kichain-t-4
````

Transfer from Rizon to kichain.
In my case, the command looks like this:<br>
```
rizond tx ibc-transfer transfer <path_name> <channel-N> tki_wallet_adress 1000uatolo --from <name_OF_wallet> --chain-id groot-011
```

>Use following command to find channel nummbers:<br>
>`rly paths show transfer`

## HASH of my transactions

### Rizon > Kichain:
86022EC9CF803473772C2BCBB65388415566B7FC6309B145374AE8574C18F71D
405B95613CD768EC971E82F133288F5033F5193B425AF70E1D59A0BBC5C71775

Use following URL to check HASH:
https://testnet.mintscan.io/rizon/txs/XXXXXXX

### Kichain > Rizon:
C2793A55CAE1F7F603823E317C888048CFB710E971BCC9781D1AEF8DA512ECA4
6772AE3630F8181B084954DD737B951F426DB7F704AE80EBE342A15BF23FEAA3
D26C3B00BFBFAA63FFAD7754304133A26AC81848820CC20C15B634154D8B7A02

Use following URL to check HASH:
https://api-challenge.blockchain.ki/txs/XXXXXX
