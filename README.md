# IBC-transfers on KiChain.
Hello everyone. :wave:

In this article, I will teach you how to independently make cross-chain transfers in networks built on the Cosmos SDK with using IBC.

For this we need two nodes of different networks and a repeater. The relay can be installed on a separate server, but I will install it on one of the nodes.
> For example, I will use the nodes of two networks: KiChain and Umee with a repeater installed on it. 

____

## Checking the network for IBC support.
You can use any other networks built on the Cosmos, **the main thing is that they support IBC transfers**.
> You can check this with the following command: ```< command for your network > q ibc-transfer params```  
> The answer should be like this: receive_enabled: **true** send_enabled: **true**
____

# What do we need to do?
- :white_check_mark: Install and initializing the repeater on the UMEE node.
- :white_check_mark: Create a settings file for both networks.
- :white_check_mark: Open a channel for relaying.
- :white_check_mark: Make a cross-chain money transfer.
- :white_check_mark: Use an open channel for transfers from any node.


## Install and initializing the repeater on the UMEE node.

Download and install the repeater from the official source Relayer **v.1.0.0**. 

1) #### Download  
```
git clone https://github.com/cosmos/relayer.git
cd relayer
```
2) #### Install  
```
make install
cd
```  
3) #### Checking version  
```
rly version
```  
Output shoud be like: **version: 1.0.0-rc1–152-g112205b**  

4) #### Initializing  
```
rly config init
```  

## Create a settings file for both networks.

1) #### Create a folder with network configuration and go to it.
```
mkdir rly_config 
cd rly_config
```  
2) #### Create a settings file for KiChain network.  
```
nano kichain-t-4.json
```  
Copy the config below and paste it into the file.  
``` 
{  "chain-id": "kichain-t-4",   "rpc-addr": "https://rpc-challenge.blockchain.ki:443",    "account-prefix": "tki",   "gas-adjustment": 1.5,   "gas-prices": "0.025utki",   "trusting-period": "48h" } 
```  
Then press **CTRL + O**, then **ENTER** and at the end **CTRL + X**.  
``` 
I am using the global RPC, because there is no KiChain client on this node. 
```
3) #### Create a settings file for Umee network.
```
nano groot-011.json
```
Copy the config below and paste it into the file.  
``` 
{  "chain-id": "umee-betanet-1",   "rpc-addr": ""http://localhost:26657",    "account-prefix": "umee",   "gas-adjustment": 1.5,   "gas-prices": "0.025uumee",   "trusting-period": "48h" } 
```  
Then press **CTRL + O**, then **ENTER** and at the end **CTRL + X**.  
``` I am using the global RPC, because there is no KiChain client on this node. ```

4) #### We parse these settings into the config of the relay
```
rly chains add -f groot-011.json  
rly chains add -f kichain-t-4.json  
cd  
```  
5) #### Create or restore wallets.
For create wallets, using: 
```
rly keys add umee-betanet-1 name of your wallet 

rly keys add kichain-t-4 name of your wallet
```  
For restore wallets, using: 
```
rly keys restore umee-betanet-1 name of your wallet "seed phrase"

rly keys restore kichain-t-4 name of your wallet "seed phrase"
```
6) #### Add the newly created keys to the config of the relay.
```
rly chains edit umee-betanet-1 key name of your wallet

rly chains edit kichain-t-4 key name of your wallet
```
7) #### Let's change the timeout of waiting for confirmation to 30s by changing the standard 10s.
```
nano ~/.relayer/config/config.yaml
```  
Change timeout: **30s**

8) #### Wallets must be funded on both networks, check the availability of coins with the command.
```
rly q balance umee-betanet-1

rly q balance kichain-t-4
```
9) #### Change the config.yaml file
```
nano /root/.relayer/config/config.yaml
```
Erase the following lines in the path section on both networks: **client-id**, **connection-id**, **channel-id**.

10) ####  Initialize the light client in both networks.
```
rly light init umee-betanet-1 -f

rly light init kichain-t-4 -f
```
## Open a channel for relaying.

1) #### We try to generate a channel between the networks with the command.
```
rly paths generate umee-betanet-1 kichain-t-4 transfer  -- port=transfer
```
2) #### Open a channel for relaying.
```
rly tx link transfer  -- debug
```
Output should be like this:  
**Channel created: [umee-betanet-1]chan{channel-0}port{transfer} -> [kichain-t-4]chan{channel-61}port{transfer}**
3) ### Checking channel.
```
rly paths list -d
```
Output shoud be like this:  
**0: transfer -> chns(✔) clnts(✔) conn(✔) chan(✔) (umee-betanet-1:transfer<>kichain-t-3:transfer)**  

Congrats! Channel opened :tada:

## Make a cross-chain money transfer.
Template for cross-chain transation : 
```
rly transact transfer [src-chain-id] [dst-chain-id] [amount] [dst-addr] [flags] 
```  
#### Now try to make a transaction from Umee to Kichain: 
```
rly tx transfer umee-betanet-1 kichain-t-4 5000000uumee tki1zqsujj7prvjhqghzl6mkwfda4rr86t5nadreyf --path transfer
```
Output should be like this: **✔ [umee-betanet-1]@{201561} - msg(0:transfer) hash(39784B4643CE00FC57618D6C4084D232A284EEB88D876B2F9B39D9CA305EB47E)**

#### You can also make transfers in the opposite direction.
```
rly tx transfer kichain-t-4 umee-betanet-1 700000utki umee1vukdvkf7cygz42kal0ksw23utnlvzw332crl8a --path transfer
```
Output should be like this: **✔ [kichain-t-4]@{222196} - msg(0:transfer) hash(B50DABEF493370B39FF0B8C6933B2622A78641DE8659054D522DBA38B682B4FF)**

## Use an open channel for transfers from any node.

1) #### Let's create a service file for relayer: 
```
sudo tee /etc/systemd/system/rlyd.service > /dev/null <<EOF
[Unit]
Description=relayer client
After=network-online.target,starsd.service

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

2) #### Let's start this service file.
```
sudo systemctl daemon-reload

sudo systemctl enable rlyd

sudo systemctl start rlyd
```

3) ### Let's make some transfer.
Find channel numbers here or check config.yaml file: 
```
rly paths show transfer --yaml
```
Transfer from Kichain to Umee:
```
kid tx ibc-transfer transfer transfer channel-N umee_WALLET_address 1000utki --from name_OF_wallet --fees=5000utki --gas=auto --chain-id kichain-t-4 --home $HOME/kichain/kid
```

Transfer from Umee to Kichain:
```
umeed tx ibc-transfer transfer transfer channel-N tki_wallet_adress 1000uumee --from your_wallet_name --fees=5000uumee --gas=auto --chain-id umee-betanet-1
```
____

# Thank you all for reading. :tada: :tada: :tada:  
I hope this will help you cross-chain transfers using IBC! 
