# Operating a private Ethereum network

## Setup
* 2 Ethereum node: node1 & node2, node1 being also a miner, 1 Bootnode
You can also use this tutorial for
* 1 Ethereum node: node1, node1 being also a miner, 1 Bootnode

## First Node:
We use the official Ethereum image `ethereum/client-go:alltools-latest`
```
docker run -it -p 8545:8545 -p 30303:30303 -v $PWD/node1:/root/.ethereum ethereum/client-go:alltools-latest
```
then inside that first container create as many account as needed:
```
geth account new
```
save passphrase and note public key, this is our account1, we will use them in genesis.json

And create one more for the miner collecting fees (even if low in a private network) if you want. You can reuse account1 as well
```
geth account new
```
save passphrase and note public key, e.g
```
Public address of the key:   0x252A02E3661b41De2A0cA1E6c870B1587E03a937
Path of the secret key file: /root/.ethereum/keystore/UTC--2020-01-15T16-33-31.575453200Z--252a02e3661b41de2a0ca1e6c870b1587e03a937

Public address of the key:   0x789B8680733C33A7797a49B7E52Fb7B10ED5b589
Path of the secret key file: /root/.ethereum/keystore/UTC--2020-01-15T16-34-14.489978700Z--789b8680733c33a7797a49b7e52fb7b10ed5b589
```

Edit file `node1/genesis.json` to `node2/genesis.json`

* adapt in "alloc" the accounts from before (in first node genesis.json is enough)
* set the balance in wei, use https://eth-converter.com/ for example. e.g 100000000000000000000 is 100 ETH
* changing the nonce to some random value so you prevent unknown remote nodes from being able to connect to you.
* chainId is an arbitrary integer value, e.g. 999

With the genesis state defined in the above JSON file, 
you'll need to initialize every geth node with it prior 
to starting it up to ensure all blockchain parameters
are correctly set:

```
geth init /root/.ethereum/genesis.json
```

Time to repeat in node 2 (optional)

```
docker run -it -p 8546:8545 -p 30303:30303 -v $PWD/node2:/root/.ethereum ethereum/client-go:alltools-latest

geth init /root/.ethereum/genesis.json
```

With all nodes that you want to run initialized to the 
desired genesis state, you'll need to start a bootstrap 
node that others can use to find each other in your 
network and/or over the internet. The clean way is 
to configure and run a dedicated bootnode:

```
docker run -it -p 30301:30301 -v $PWD/bootnode:/root/.ethereum ethereum/client-go:alltools-latest

bootnode --genkey=/root/.ethereum/boot.key

bootnode --nodekey=/root/.ethereum/boot.key
```

With the bootnode operational and externally reachable (you can try telnet <ip> <port> to ensure 
it's indeed reachable), start every subsequent geth node pointed to the bootnode for 
peer discovery via the --bootnodes flag. 

So we could start now node1 with, but read more first: we need a miner also.
```
geth --rpc --bootnodes=enode://4364896013735c849360d4a183c6e223c674c371c58433fe2f45da43e07e4fc506941fe24e81daf36e5ad0a848cb73879a0d240b4f94d910e86746b00ad92d69@127.0.0.1:0?discport=30301
```

Note: Since your network will be completely cut off from the main and test networks, 
we need now to configure a miner to process transactions and create new blocks for you.

## Running a private miner

In a private network setting, a single CPU miner instance 
is more than enough for practical purposes as it can produce a stable stream of blocks 
at the correct intervals without needing heavy resources (consider running on a single thread, 
no need for multiple ones either). 

We can dedicate another docker instance or just reuse node1 by appending, being here the first account created before

```
--mine --miner.threads=1 --etherbase=0x252A02E3661b41De2A0cA1E6c870B1587E03a937
```

this will create a new DAG and node1 will be a miner

Ethash PoW is memory hard, making it basically ASIC resistant. This basically means that calculating
the PoW requires choosing subsets of a fixed resource dependent on the nonce and block header. 
This resource (a few gigabyte size data) is called a DAG. 

# FINAL (recap)
Start bootnode first
```
docker run -it -p 30301:30301 -v $PWD/bootnode:/root/.ethereum ethereum/client-go:alltools-latest

bootnode --nodekey=/root/.ethereum/boot.key
```

then node1

```
docker run -it -p 8545:8545 -p 30303:30303 -v $PWD/node1:/root/.ethereum ethereum/client-go:alltools-latest 

geth --rpc --rpccorsdomain="chrome-extension://nkbihfbeogaeaoehlefnkodbefgpgknn,http://localhost:80" --rpcaddr "0.0.0.0" --rpcport 8545 --rpcapi="db,eth,net,web3,personal,web3" --bootnodes=enode://72e6656dd7bdf4e7bbcf4c283d03ac2491c595428be9caa081547fcbe9fbed6fd6a51278f79d2051f5546ec0eb83a06991eb36302751038edb300f27b594f4ac@127.0.0.1:0?discport=30301 --mine --miner.threads=1 --etherbase=0x252A02E3661b41De2A0cA1E6c870B1587E03a937 
```
NOTES: 
* --rpccorsdomain as explained at https://metamask.zendesk.com/hc/en-us/articles/360015290012-Using-a-Local-Node for metamask
* --rpccorsdomain with http://localhost:80 for the block explorer epirus-free 

# Metamask
## import geth account to your Metamask
Click on Metamask extension, open the account menu in the top right and click on Import Account.

* You can choose to import using Private Key and JSON File.
* If you choose to use Json File type then you need to import KeyFile stored in your 
   Keystore folder where your geth data is present.
* Choose the KeyFile and enter the password you entered when you created this account in geth.
* Click on Import button.
Now geth account is successfully imported into your Metamask.

see https://metamask.zendesk.com/hc/en-us/articles/360015489331-Importing-an-Account

# Export private keys of geth account using keystore
We use https://github.com/gochain/web3 for that
```
curl -LSs https://raw.githubusercontent.com/gochain/web3/master/install.sh | sh
```
then run
```
web3 account extract --keyfile node1/keystore/UTC-xxxxx   -password xxxxxx
```

# Block Explorer
it is quite challenging to find a running block explorer for a private network.
we will now use for now Epirus-Free  (commercial version https://www.web3labs.com/epirus)
```
cd explorer
git clone https://github.com/blk-io/epirus-free.git
cd epirus-free
# since we need the host machine from inside docker 
# we use the docker bridge network. ie docker0
NODE_ENDPOINT=http://docker0:8545 docker-compose up

# to stop, hit CTRL-C, use -v for deleting database and losing network history
docker-compose down -v
```

see https://github.com/blk-io/epirus-free/issues/14

browse http://localhost/

# useful GETH commands

eth.getBalance(eth.accounts[0]);

# links
* https://hub.docker.com/r/ethereum/client-go
* https://github.com/ethereum/go-ethereum/wiki/Command-Line-Options
* https://ethereum.stackexchange.com/questions/2919/block-explorer-running-on-private-network/79007#79007
