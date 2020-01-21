# Operating a private Ethereum network

## Setup
You can also use this tutorial for
* 1 Ethereum node: node1, node1 being also a miner
* 2+ Ethereum node: node1, ... ,nodeN, 1 Bootnode, 1..M Miner node

## Your first Node
We use the official Ethereum image `ethereum/client-go:alltools-stable` The official Golang implementation of the Ethereum protocol.

Ethereum currently maintain four different docker images for running the latest stable or develop versions of miscellaneous Ethereum tools.

* ethereum/client-go:alltools-latest is the latest develop version of the Ethereum tools
* ethereum/client-go:alltools-stable is the latest stable version of the Ethereum tools
* ethereum/client-go:alltools-{version} is the stable version of the Ethereum tools at a specific version number
* ethereum/client-go:alltools-release-{version} is the latest stable version of the Ethereum tools at a specific version family

The image has the following ports automatically exposed:

* 8545 TCP, used by the HTTP based JSON RPC API
* 8546 TCP, used by the WebSocket based JSON RPC API
* 30303 TCP and UDP, used by the P2P protocol running the network
* 30304 UDP, used by the P2P protocol's new peer discovery overlay

Just run:

```
docker run -it -p 8545:8545 -p 30303:30303 -name node1 \
       -v $PWD/node1:/root/.ethereum ethereum/client-go:alltools-stable
```

**Notes**
* We mount in a data volume the client's data directory (located at /root/.ethereum inside the container) 
to ensure that downloaded data is preserved between restarts and/or container life-cycles.

then inside that first container you can start creating as many account as needed:
```
geth account new
```
save passphrase and note public key, this is our account1, we will use all these account later in genesis.json
save passphrase and note public key, e.g
```
Public address of the key:   0x252A02E3661b41De2A0cA1E6c870B1587E03a937
Path of the secret key file: /root/.ethereum/keystore/UTC--2020-01-15T16-33-31.575453200Z--252a02e3661b41de2a0ca1e6c870b1587e03a937
```
You may want another account for the miner collecting fees. You can reuse account1 as well for this.

# Bootstraping your blockchain node with the genesis block

Edit the file `node1/genesis.json` provided or create a new file

* Adapt in "alloc" the accounts from before: these account will receive the amount of your ETH
* Set the balance in wei, use https://eth-converter.com/ for example. e.g 100000000000000000000 is 100 ETH
* Changing the nonce to some random value so you prevent unknown remote nodes from being able to connect to you.
* ChainId is an arbitrary integer value, e.g. 999
* You want all genesis node to be identical in all your node, so dont forget to sync them. You can either publish your genesis.json online like Rinkeby Testnet is doing https://www.rinkeby.io/rinkeby.json

## Explanation of genesis.json file
* **mixhash**

> A 256-bit hash which proves, combined with the nonce, that a sufficient amount of computation has been carried out on this block: the Proof-of-Work (PoF).
> The combination of nonce and mixhash must satisfy a mathematical condition described in the Yellowpaper, 4.3.4. Block Header Validity, (44). It allows to verify that the Block has really been cryptographically mined, thus, from this aspect, is valid.
> The value is a reduced representation (using a simple Fowler–Noll–Vo hash function) of the set of values selected from the DAG data file during mining calculation. This selection pick follows the implemented Estah' Hashimoto algorithm, which depends on the given Block header. The applied mixhash is re-determined for each hash operation that a Miner performs while searching for the correct Block nonce (cf. ASIC resistance, high IO). The final Block mixhash is the value leading to the valid Block. The reason why this value is part of the Block descriptor is that it becomes part of the //parentHash// of the next Block. By this, a potential attacker would need correct DAG data files to create illegal Blocks.

* **nonce**

> A scalar value equal to the number of transactions sent by the sender.
> A 64-bit hash which proves combined with the mix-hash that a sufficient amount of computation has been carried out on this block.
> The nonce is the cryptographically secure mining proof-of-work that proves beyond reasonable doubt that a particular amount of computation has been expended in the determination of this token value. (Yellowpager, 11.5. Mining Proof-of-Work).
> The final nonce value is the result of the the mining process iteration, in which the algorithm was able to discover a nonce value that satisfies the Mining Target. The Mining Target is a cryptographically described condition that strongly depends on the applied . Just by using the nonce Proof-of-Work, the validity of a Block can verified very quickly.

* **difficulty**

> A scalar value corresponding to the difficulty level applied during the nonce discovering of this block. It defines the Mining Target, which can be calculated from the previous block’s difficulty level and the timestamp. The higher the difficulty, the statistically more calculations a Miner must perform to discover a valid block. This value is used to control the Block generation time of a Blockchain, keeping the Block generation frequency within a target range. On the test network, we keep this value low to avoid waiting during tests since the discovery of a valid Block is required to execute a transaction on the Blockchain.

* **alloc**

> Allows to define a list of pre-filled wallets. That's a Ethereum specific functionality to handle the "Ether pre-sale" period. Since we can mine local Ether quickly, we don't use this option.

* **coinbase**

> The 160-bit address to which all rewards (in Ether) collected from the successful mining of this block have been transferred. They are a sum of the mining eward itself and the Contract transaction execution refunds. Often named "beneficiary" in the specifications, sometimes "etherbase" in the online documentation. This can be anything in the Genesis Block since the value is set by the setting of the Miner when a new Block is created.

* **timestamp**

> A scalar value equal to the reasonable output of Unix’ time() function at this block inception.
> This mechanism enforces a homeostasis in terms of the time between blocks. A smaller period between the last two blocks results in an increase in the difficulty level and thus additional computation required to find the next valid block. If the period is too large, the difficulty, and expected time to the next block, is reduced.
> The timestamp also allows to verify the order of block within the chain (Yellowpaper, 4.3.4. (43)).
> Note: Homeostasis is the property of a system in which variables are regulated so that internal conditions remain stable and relatively constant.

* **parentHash**

> The Keccak 256-bit hash of the entire parent block’s header (including its nonce and mixhash). Pointer to the parent block, thus effectively building the chain of blocks. In the case of the Genesis block, and only in this case, it's 0.

* **extraData**

> An optional free, but max. 32 byte long space to conserve smart things for ethernity on the Blockchain.

* **gasLimit**

> A scalar value equal to the current chain-wide limit of Gas expenditure per block. High in our case to avoid being limited by this threshold during tests. Note: this does not indicate that we should not pay attention to the Gas consumption of our Contracts.

With the genesis state defined in the above JSON file, you'll need to initialize EVERY geth node that will participate in your network 
to starting it up to ensure all blockchain parameters are correctly set:

```
geth init /root/.ethereum/genesis.json
```

Time to repeat in node 2... node N (optional)
```
docker run -it -p 8545:8545 -p 30303:30303 -v $PWD/node2:/root/.ethereum ethereum/client-go:alltools-stable
geth init /root/.ethereum/genesis.json
```

With all nodes that you want to run initialized to the 
desired genesis state, you'll need to start a bootstrap 
node that others node can use to find each other in your 
network and/or over the internet. The clean way is 
to configure and run a dedicated bootnode, so we fired again another container

```
docker run -it -p 30301:30301 -v $PWD/bootnode:/root/.ethereum ethereum/client-go:alltools-stable

# generate once the boot key only once
bootnode --genkey=/root/.ethereum/boot.key

# start the bootnode
bootnode --nodekey=/root/.ethereum/boot.key
```

With the bootnode operational and externally reachable (you can try telnet <ip> <port> to ensure 
it's indeed reachable), start every subsequent geth node pointed to the bootnode for 
peer discovery via the --bootnodes flag. 

So we could start now again node1 (hit CTRL-C to stop geth) and run
```
geth --rpc \
     --bootnodes=enode://4364896013735c849360d4a183c6e223c674c371c58433fe2f45da43e07e4fc506941fe24e81daf36e5ad0a848cb73879a0d240b4f94d910e86746b00ad92d69@127.0.0.1:0?discport=30301
```

Note: Since your network will be completely cut off from the main and test networks, 
we need now to configure a miner to process transactions and create new blocks for you.

## Running a private miner

In a private network setting, a single CPU miner instance 
is more than enough for practical purposes as it can produce a stable stream of blocks 
at the correct intervals without needing heavy resources (consider running on a single thread, 
no need for multiple ones either). 

We can dedicate another docker instance or just reuse node1 by appending:

```
--mine --miner.threads=1 --etherbase=0x252A02E3661b41De2A0cA1E6c870B1587E03a937
```
NOTE: 0x252A02E3661b41De2A0cA1E6c870B1587E03a937 being account1 the first account created before. You can also use any account.

Node 1 will now create a new DAG (size almost 2GB on disk) and start mining when completed.

Why a DAG of 2GB?
>Ethash PoW is memory hard, making it basically ASIC resistant. This basically means that calculating
>the PoW requires choosing subsets of a fixed resource dependent on the nonce and block header. 
>This resource (a few gigabyte size data) is called a DAG. 

**Notes**
* the dag DAG files are located at /root/.ethash and can be safely deleted


# FINAL (recap)
Start bootnode only if you have more than one node, aka node1, node2, ...nodeN
```
docker run -it -p 30301:30301 -v $PWD/bootnode:/root/.ethereum ethereum/client-go:alltools-stable
bootnode --nodekey=/root/.ethereum/boot.key
```

then node1 to nodeN

```
docker run -it -p 8545:8545 \ 
       -p 30303:30303 \
       -v $PWD/node1:/root/.ethereum ethereum/client-go:alltools-stable 

geth --rpc --rpccorsdomain="chrome-extension://nkbihfbeogaeaoehlefnkodbefgpgknn,http://localhost:80" \
     --rpcaddr "0.0.0.0" --rpcport 8545 --rpcapi="db,eth,net,web3,personal,web3" \
     --bootnodes=enode://72e6656dd7bdf4e7bbcf4c283d03ac2491c595428be9caa081547fcbe9fbed6fd6a51278f79d2051f5546ec0eb83a06991eb36302751038edb300f27b594f4ac@127.0.0.1:0?discport=30301 \
     --mine --miner.threads=1 --etherbase=0x252A02E3661b41De2A0cA1E6c870B1587E03a937 
```
NOTES: 
* `--rpccorsdomain `is required ONLY if you connect  to your node using Metamask
see https://metamask.zendesk.com/hc/en-us/articles/360015290012-Using-a-Local-Node
* `--rpccorsdomain` with http://localhost:80 is required the block explorer epirus-free 

# Metamask
## Import geth account to your Metamask
Click on Metamask extension, open the account menu in the top right and click on Import Account.

* You can choose to import using Private Key and JSON File.
* If you choose to use Json File type then you need to import KeyFile stored in your 
   Keystore folder where your geth data is present.
* Choose the KeyFile and enter the password you entered when you created this account in geth.
* Click on Import button.
Now geth account is successfully imported into your Metamask.

see [Support page of Metamask](https://metamask.zendesk.com/hc/en-us/articles/360015489331-Importing-an-Account) for more details.

# Export private keys of geth account using keystore
We use [Web3 CLI](https://github.com/gochain/web3) for that, install web3 command line interface first:
```
curl -LSs https://raw.githubusercontent.com/gochain/web3/master/install.sh | sh
```
then run
```
web3 account extract --keyfile node1/keystore/UTC-xxxxx --password xxxxxx
```

# Using a Block Explorer
it is quite challenging to find a running block explorer for a private network.
we will now use for now Epirus-Free  ([commercial version here](https://www.web3labs.com/epirus))

We run the block explorer on the same machine as node1. If not replace below `NODE_ENDPOINT=http://node1:8545` with the IP of your node
```
cd explorer
git clone https://github.com/blk-io/epirus-free.git
cd epirus-free
# since we need the host machine from inside docker 
# we use the docker bridge network. ie docker0
NODE_ENDPOINT=http://node1:8545 docker-compose up

# to stop, hit CTRL-C, add -v for deleting database and losing network history
docker-compose down -v
```

Stop Node1, `docker rm node1` and start it again by adding `--network Epirus-net --rpcvhost=*` if you run your block explorer on the same host as your private node1.
You need to start epirus first then your node 1.

Browse http://localhost/ to see your local explorer

**Notes**
* `--rpcvhost` Comma separated list of virtual hostnames from which to accept requests (server enforced). Accepts '*' wildcard. (default: "localhost")

# Useful GETH commands

`eth.getBalance(eth.accounts[0]);`

# links
* https://hub.docker.com/r/ethereum/client-go
* https://github.com/ethereum/go-ethereum/wiki/Command-Line-Options
* https://ethereum.stackexchange.com/questions/2919/block-explorer-running-on-private-network/79007#79007

![Disruptr](https://disruptr.ch/wp-content/uploads/2019/11/disruptr-logo.png)

Disruptr offer services regarding blockchain IT consulting & engineering, blockchain strategy and advisory services, including:
* Conception and realization of proof of concepts, complete projects.
* Project management for implementations
* Smart contract auditing
* Blockchain courses/trainings
* Technical content writing
* Development of software products using Corda/Ethereum
Would you like to learn more about how blockchain can help you automate processes, lower costs, create transparency and make your business more efficient? 
 [Contact us](https://disruptr.ch/?page_id=120)



