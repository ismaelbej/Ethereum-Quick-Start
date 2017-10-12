# Ethereum Quick Start

This document describe how to create an Ethereum private test network,
deploy a contract, and interact with it.

Ethereum has a public main network, and a public test network. But we will
create our own private test network.

## Pre-requisites

### Windows

Download geth from https://geth.ethereum.org/downloads/

To make it a global available:

  * Use the shortcut ("Windows or meta key" + "Pause") to launch Windows system setting
  * Advanced system settings, Environment Variables, Path, Edit.
  * Add the geth path to the Path variable.

### Ubuntu

All the examples were tested with the latest Ubuntu LTS 16.04.

We enable Ethereum's PPA, and install the required applications.

```sh
sudo apt-get install software-properties-common
sudo add-apt-repository -y ppa:ethereum/ethereum
sudo apt-get update
sudo apt-get install ethereum
sudo apt-get install solc
```

The ethereum package will install `geth` which is one of the Ethereum Official nodes.
The package `solc` is the compiler for _Solidity_, a programming language for Ethereum's contracts.

## Running a private network

Create a new work folder, e.g: `/blockchain`, then create two additional folders,
e.g: `/blockchain/node1` and `/blockchain/node2`.

Use a command prompt to go to the blockchain directory:

```sh
> cd /blockchain
```

### Genesis file

Create a file `genesis.json` in the blockchain foler with this content:

```json
{
  "config": {
    "chainId": 100,
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip155Block": 0,
    "eip158Block": 0
  },
  "nonce": "0x00",
  "difficulty": "0x0400",
  "mixhash": "0x00",
  "coinbase": "0x00",
  "timestamp": "0x00",
  "parentHash": "0x00",
  "extraData": "0x00",
  "gasLimit": "0x8000000",
  "alloc": {
    @account: {
      "balance": @amount
    }
  }
}
```

We set up two variables (`@account` and `@money`) in the `genesis.json` file

### Create an account

```
> geth --datadir=node1 account new
```

Provide a passphrase and the console will give you an address like:

> Address: {dca126ad1248c2e8914f82d5c8d6ece5c49f4f00}

Go to Genesis.json and replace `@account` for that address and enter the number of wei you want for your account, e.g: "1000000000000000000". Remember 1 ether = 10<sup>18</sup> wei . Replace `@amount` with this number and save the file. The account will have funds.

```json
{
  ...
  "alloc": {
    "dca126ad1248c2e8914f82d5c8d6ece5c49f4f00": {
      "balance": "1000000000000000000"
    }
  }
}
```

Repeat the same step for each node. You can have multiple entries in the genesis file.

### Create genesis block

To initialize the private test network we should execute this command.

```sh
> geth --datadir=node1 init genesis.json
```

This command initialize the blockchain creates in the _node1_ folder.

### Balance

We start the geth console:

```sh
> geth --datadir=node1 --nodiscover --maxpeers=0 --nat=none console
```

This command will leave us in a command line environment. In this is prompt
we can type javascript to send commands to the node.

For example the command

```js
> web3.version
```

Will output information about our node

```txt
{
  api: "0.18.1",
  ethereum: "0x3f",
  network: "100",
  node: "Geth/v1.7.1-stable-05101641/linux-amd64/go1.9",
  whisper: undefined,
  getEthereum: function(callback),
  getNetwork: function(callback),
  getNode: function(callback),
  getWhisper: function(callback)
}
```

We can display our balance. The balance is reported in _Wei_.

```js
> eth.getBalance(eth.accounts[0])
```

A more convenient unit is _Ether_, to convert from _Wei_ to _Ether_ we
use the function `fromWei`.

```js
> web3.fromWei(eth.getBalance(eth.accounts[0]), "ether")
```

We can exit from the geth console typing `exit` and pressing `Enter`.

### Mining

To deploy a contract, or any transaction we need to have some ether,
in out test network we can mine some blocks.

```js
> miner.start(1); admin.sleepBlocks(2); miner.stop()
```

This will create a single miner thread, and will wait to have mined at least
two blocks.

**Note:** Only the _first time_ that mining is started it will create the DAG,
which may take _several minutes_ to complete.

The DAG is a dataset used by the Ethereum to secure its blockchain.

### Connecting nodes

In order to connect one `geth` instance to another, we need its _enode_.
To get it, type the following in the first console:

```js
> admin.nodeInfo.enode
```

It will return something like:

> "enode://28b9d6f5836ce3c7f20752d119c3066bf79b1e12f1dc0cdc8d8a73429f3d867d8c10ffdf9f1e58f2ce954c83b0944e2c110ac98502e3f0a7d159728122e28703@[::]:30802?discport=0"

Copy the output in a texteditor and replace "[::]" with "127.0.0.1" to get the following:

> "enode://28b9d6f5836ce3c7f20752d119c3066bf79b1e12f1dc0cdc8d8a73429f3d867d8c10ffdf9f1e58f2ce954c83b0944e2c110ac98502e3f0a7d159728122e28703@127.0.0.1:30802?discport=0"

Next, type the following in the second console:

```js
> admin.addPeer(<copy the enode here>)
```

To verify that the nodes are connected type the following in either:

```js
> web3.net.peerCount
```

The result should be "1"/"true"


## Contracts, Ether, EVM and Gas

*   _Ether_ is denomination of the cryptocurrency used by the Ethereum Project,
and it is used to pay for contracts execution and transactions.

*   _EVM_ refers to the Ethereum Virtual Machine, which is the environment
where the contracts will execute. The instructions executed by this
virtual machine are called opcodes.

*   _Contracts_ are the pieces of code that are secured in the Ethereum
blockchain and will be run inside the EVM. The contracts are compiled into
bytecode that will be interpreted by the EVM. A contract has a private
storage, can hold funds in ether. The public interface of the contract
can be called contract's ABI. A contract has limited access to the
ethereum blockchain, and can call other contracts public interface.

*   _Gas_ measure the cost of execution of EVM opcodes. The total cost of
execution of a contract's method is the sum of all the opcodes that
were executed. The gas is pay with ether, the price is determined
when one submit a transaction. A higher price usually means that
the transaction will be mined faster.


## References

*   [Web3 JavaScript Dapp API](https://github.com/ethereum/wiki/wiki/JavaScript-API)
*   [Connecting to a private test net](https://ethereum.org/cli#connecting-to-a-private-test-net)
