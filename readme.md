# Operating a private Ethereum network

# we will setup 3 docker container
2 Ethereum node: node1 & node2, node1 being also a miner
1 Bootnode


## First Node:
We use the official Ethereum image `ethereum/client-go:alltools-latest`
`
docker run -it -p 8545:8545 -p 30303:30303 -v node1:/root/.ethereum ethereum/client-go:alltools-latest
`
then inside that first container create as many account as needed

`geth account new`
save passphrase and note public kegity, this is our account1, we will use them in genesis.json

And create one more for the miner collecting fees (even if low in a private network) if you want. You can reuse account1 as well
`geth account new`
save passphrase and note public key

Public address of the key:   0x252A02E3661b41De2A0cA1E6c870B1587E03a937
Path of the secret key file: /root/.ethereum/keystore/UTC--2020-01-15T16-33-31.575453200Z--252a02e3661b41de2a0ca1e6c870b1587e03a937

Public address of the key:   0x789B8680733C33A7797a49B7E52Fb7B10ED5b589
Path of the secret key file: /root/.ethereum/keystore/UTC--2020-01-15T16-34-14.489978700Z--789b8680733c33a7797a49b7e52fb7b10ed5b589

Edit file `node1/genesis.json` to `node2/genesis.json`

* adapt in "alloc" the accounts from before
* changing the nonce to some random value so you prevent unknown remote nodes from being able to connect to you.
* chainId is an arbitrary integer value

With the genesis state defined in the above JSON file, 
you'll need to initialize every geth node with it prior 
to starting it up to ensure all blockchain parameters
are correctly set:

`geth init /root/.ethereum/genesis.json`

we do node 2

```
docker run -it -p 8545:8545 -p 30303:30303 -v node2:/root/.ethereum ethereum/client-go:alltools-latest
geth init /root/.ethereum/genesis.json
```

With all nodes that you want to run initialized to the 
desired genesis state, you'll need to start a bootstrap 
node that others can use to find each other in your 
network and/or over the internet. The clean way is 
to configure and run a dedicated bootnode:

```
docker run -it -p 30301:30301 -v bootnode:/root/.ethereum ethereum/client-go:alltools-latest
bootnode --genkey=/root/.ethereum/boot.key
bootnode --nodekey=/root/.ethereum/boot.key
```

With the bootnode operational and externally reachable (you can try telnet <ip> <port> to ensure 
it's indeed reachable), start every subsequent geth node pointed to the bootnode for 
peer discovery via the --bootnodes flag. 

`
geth --rpc --bootnodes=enode://4364896013735c849360d4a183c6e223c674c371c58433fe2f45da43e07e4fc506941fe24e81daf36e5ad0a848cb73879a0d240b4f94d910e86746b00ad92d69@127.0.0.1:0?discport=30301
`

Note: Since your network will be completely cut off from the main and test networks, 
we need now to configure a miner to process transactions and create new blocks for you.

## Running a private miner

In a private network setting, a single CPU miner instance 
is more than enough for practical purposes as it can produce a stable stream of blocks 
at the correct intervals without needing heavy resources (consider running on a single thread, 
no need for multiple ones either). 

We can dedicate another docker instance or just reuse node1 by appending 

`--mine --miner.threads=1 --etherbase=0x252A02E3661b41De2A0cA1E6c870B1587E03a937`

this will create a new DAG and node1 will be a miner

Ethash PoW is memory hard, making it basically ASIC resistant. This basically means that calculating
the PoW requires choosing subsets of a fixed resource dependent on the nonce and block header. 
This resource (a few gigabyte size data) is called a DAG. 

# FINAL
start bootnode
docker run -it -p 30301:30301 -v bootnode:/root/.ethereum ethereum/client-go:alltools-latest
bootnode --nodekey=/root/.ethereum/boot.key



# node1
as explained at https://metamask.zendesk.com/hc/en-us/articles/360015290012-Using-a-Local-Node

docker run --network host -it -p 8545:8545 -p 30303:30303 -v node1:/root/.ethereum ethereum/client-go:alltools-latest
geth --rpc --rpccorsdomain="chrome-extension://nkbihfbeogaeaoehlefnkodbefgpgknn" -rpcport 8545 --rpcapi="db,eth,net,web3,personal,web3" --bootnodes=enode://4364896013735c849360d4a183c6e223c674c371c58433fe2f45da43e07e4fc506941fe24e81daf36e5ad0a848cb73879a0d240b4f94d910e86746b00ad92d69@127.0.0.1:0?discport=30301 --allow-insecure-unlock 

--mine --miner.threads=1 --etherbase=0x252A02E3661b41De2A0cA1E6c870B1587E03a937



# import geth account to your Metamask
Click on Metamask extension, open the account menu in the top right and click on Import Account.

* You can choose to import using Private Key and JSON File.
* If you choose to use Json File type then you need to import KeyFile stored in your 
   Keystore folder where your geth data is present.
* Choose the KeyFile and enter the password you entered when you created this account in geth.
* Click on Import button.
Now geth account is successfully imported into your Metamask.

# useful commands

eth.getBalance(eth.accounts[0]);

# links
* https://hub.docker.com/r/ethereum/client-go
