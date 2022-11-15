# Nibiru public testnet phase 1 node tutorial 
<a id="anchor"></a>

[<img align="right" alt="Personal Website" width="22px" src="https://raw.githubusercontent.com/iconic/open-iconic/master/svg/globe.svg" />][nibiru-website]
[<img align="right" alt="Nibiru Discord" width="22px" src="https://cdn.jsdelivr.net/npm/simple-icons@v3/icons/discord.svg" />][nibiru-discord]
[<img align="right" alt="Nibiru Medium Blog" width="22px" src="https://cdn.jsdelivr.net/npm/simple-icons@3.13.0/icons/medium.svg"/>][nibiru-medium]

[nibiru-medium]: https://blog.nibiru.fi
[nibiru-website]: https://docs.nibiru.fi
[nibiru-discord]: https://discord.com/invite/pgArXgAxDD

[<img  width="200" alt="nibiru" src="https://user-images.githubusercontent.com/93165931/201940888-1c2d3833-3e40-4a1e-92e8-c865c4f948ea.png">
](https://docs.nibiru.fi/)

| Minimum configuration                                                                                |
|------------------------------------------------------------------------------------------------------|
- 2 CPU                                                                                                |
- 4 GB RAM (The requirements written in the official tutorial are too high, the actual 8GB+ is enough) |
- 100GB SSD                                                                                            |

--- 
### Install the basic environment
#### The system used in this tutorial is Ubuntu20.04, please adjust some commands of other systems by yourself. It is recommended to use a foreign VPS.

#### Install golang
```
sudo rm -rf /usr/local/go;
curl https://dl.google.com/go/go1.19.2.linux-amd64.tar.gz | sudo tar -C/usr/local -zxvf - ;
cat <<'EOF' >>$HOME/.profile
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
source $HOME/.profile
```
#### After the installation is complete, run the following command to check the version

```
go version
```
#### Install other necessary environments

```
cd
git clone https://github.com/NibiruChain/nibiru
cd nibiru
git checkout v0.15.0
make install
```
After the installation is complete, you can run `nibid version` to check whether the installation is successful.

Display should be v0.15.0
### Run node

#### Initialize node

```
moniker=YOUR_MONIKER_NAME
nibid init $moniker --chain-id=nibiru-testnet-1
nibid config chain-id nibiru-testnet-1
```

#### Download the Genesis file

```
curl -s https://rpc.testnet-1.nibiru.fi/genesis | jq -r .result.genesis >  ~/.nibid/config/genesis.json
```

#### Set peer and seed

```
PEERS="968472e8769e0470fadad79febe51637dd208445@65.108.6.45:60656"
seeds=""
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.nibid/config/config.toml
sed -i.bak -e "s/^seeds *=.*/seeds = \"$seeds\"/" ~/.nibid/config/config.toml
```

### Pruning settings
```
pruning="custom" && \
pruning_keep_recent="100" && \
pruning_keep_every="0" && \
pruning_interval="10" && \
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.nibid/config/app.toml && \
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.nibid/config/app.toml && \
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.nibid/config/app.toml && \
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.nibid/config/app.toml
```
#### State-sync fast synchronization
```
SNAP_RPC="https://rpc.nibiru-testnet-1.silentvalidator.com:443" \
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
BLOCK_HEIGHT=$((LATEST_HEIGHT - 500)); \
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash); \
echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH
#if there are no errors, then continue

peers="5eecfdf089428a5a8e52d05d18aae1ad8503d14c@65.108.141.109:19656,5c30c7e8240f2c4108822020ae95d7b5da727e54@65.108.75.107:19656"
sed -i 's|^persistent_peers *=.*|persistent_peers = "'$peers'"|' $HOME/.nibid/config/config.toml
sed -i -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" $HOME/.nibid/config/config.toml
```

#### Start node 
```
sudo tee <<EOF >/dev/null /etc/systemd/system/nibid.service
[Unit]
Description=nibid daemon
After=network-online.target
[Service]
User=$USER
ExecStart=$(which nibid) start
Restart=on-failure
RestartSec=3
LimitNOFILE=10000
[Install]
WantedBy=multi-user.target
EOF
```
```
sudo systemctl daemon-reload && \
sudo systemctl enable nibid && \
sudo systemctl start nibid
```
___

#### Show log
```
sudo journalctl -u nibid -f
```
#### Check sync status
```
curl -s localhost:26657/status | jq .result | jq .sync_info
```
The display `"catching_up":` shows `false` that it has been synchronized. Synchronization takes a while, maybe half an hour to an hour. If the synchronization has not started, it is usually because there are not enough peers. You can consider adding a Peer or using someone else's addrbook.

#### Replace addrbook
```
wget -O $HOME/.nibid/config/addrbook.json "https://raw.githubusercontent.com/sergiomateiko/addrbooks/main/nibiru/addrbook.json"
```
### Create a validator
#### Create wallet
```
nibid keys add WALLET_NAME
```
----
## `Note please save the mnemonic! If you don't save it, you won't be able to restore it later.`
----
### Receive test coins
#### Go to nibiru discord https://discord.gg/nsV3a5CdC9
#### Sent in #faucet channel
```
$request WALLET_ADDRESS
```
#### Can be used later
```
nibid query bank balances WALLET_ADDRESS
```
#### Query the test currency balance.
#### Create a validator
`After enough test coins are obtained and the node is synchronized, a validator can be created. Only validators whose pledge amount is in the top 100 are active validators.`
```
daemon=nibid
denom=unibi
moniker=MONIKER_NAME
chainid=nibiru-testnet-1
$daemon tx staking create-validator \
    --amount=1000000$denom \
    --pubkey=$($daemon tendermint show-validator) \
    --moniker=$moniker \
    --chain-id=$chainid \
    --commission-rate=0.05 \
    --commission-max-rate=0.1 \
    --commission-max-change-rate=0.1 \
    --min-self-delegation=1000000 \
    --fees 5000unibi \
    --from=WALLET_NAME
```
#### After that, you can go to the block [explorer](https://nibiru.explorers.guru/) to check whether your validator is created successfully.
And [other commands](https://github.com/cosmoswalk/Cosmos_tutorials/blob/main/Useful%20Tools/Other%20commands.md)
----

#### More information 

|[Official website](https://nibiru.fi/) |[Official twitter](https://twitter.com/NibiruChain) | [Discord](https://discord.gg/nsV3a5CdC9) | [Github](https://github.com/NibiruChain) | [Documentation](https://docs.nibiru.fi/)|
-----------------------------------------------------------

### [Up ↑](#anchor)