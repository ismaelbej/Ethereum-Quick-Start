# Ethereum Quick Start

This document describe how to create an Ethereum private test network,
deploy a contract, and interact with it.

Ethereum has a public main network, and a public test network. But we will
create our own private test network.

## Pre-requisites

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

## Running a private testnet

We create a file `genesis.json` with this content (this are the
parameters for our private testnet):

```json
{
  "config": {
    "chainId": 6000,
    "homesteadBlock": 0,
    "eip155Block": 0,
    "eip158Block": 0
  },
  "gasLimit": "0x8000000",
  "difficulty": "0x0400",
  "alloc": {
  }
}
```
**Note:** Since geth 1.6 it is required to include a `config` field for private testnets.

To initialize the private test network we should execute this command.

**Note:** This command should be run only once.

```sh
> geth --datadir node1 init genesis.jon
```

This command creates the _node1_ directory an initialize the blockchain.

To run the server we execute this command:

```sh
> geth --datadir node1 --rpc --rpcapi eth,web3,net,personal,ssh,db,debug --nodiscover --maxpeers 0 console
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
  network: "6000",
  node: "Geth//v1.6.1-stable//linux//go1.8.1",
  whisper: undefined,
  getEthereum: function(callback),
  getNetwork: function(callback),
  getNode: function(callback),
  getWhisper: function(callback)
}
```

We can exit from the geth console typing `exit` and pressing `Enter`.

## Creating an account

To create a new account. From the geth console we type:

```js
> personal.newAccount("abcd123")
```

Note: We should replace "abcd123" with our own password.

Now `eth.accounts[0]` contain the account recently created

```js
> eth.accounts[0]
```

To use this account to send transactions we need to unlock it.

```js
> personal.unlockAccount(eth.accounts[0], "abcd123", 10000)
```

The first parameter is the account to unlock, the second parameter is
the account's password, and the third parameter is the time to unlock the
account (in seconds).

We need to do this for operations that can modify our balance.
For example to send a transaction.

We can check our balance (which should be zero).

```js
> eth.getBalance(eth.accounts[0])
```

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

We can now display our balance. The balance is reported in _Wei_. But
a more convenient unit is _Ether_, to convert from _Wei_ to _Ether_ we
use the function `fromWei`.

```js
> web3.fromWei(eth.getBalance(eth.accounts[0]), "ether")
```

## Deploying a contract

This is our contract

```txt
contract hello {
    address owner;
    string greeting;
    function hello() {
        owner = msg.sender;
    }
    function kill() {
        if (msg.sender == owner) {
            suicide(owner);
        }
    }
    function setSalute(string _greeting) public {
        greeting = _greeting;
    }
    function greet() constant returns (string) {
        return greeting;
    }
}
```

We store the contract as a string in a variable

```js
> source = 'contract hello { address owner; string greeting; function hello() { owner = msg.sender; } function kill() { if (msg.sender == owner) { suicide(owner); } } function setSalute(string _greeting) public { greeting = _greeting; } function greet() constant returns (string) { return greeting; } }'
```

To compile the contract we use the command

```js
> compiled = eth.compile.solidity(source)
```

We can deploy the contract in our private the blockchain

```js
> helloContract = eth.contract(compiled['<stdin>:hello'].info.abiDefinition)
> hello = helloContract.new({from:eth.accounts[0], data:compiled['<stdin>:hello'].code, gas:3000000})
```

The transaction for our contract was created, but will not be available
until it is mined in the blockchain. The resulting object `hello` contains
a field `transactionHash`, that can be used to check if the transaction
was mined. The field `address` from the `hello` object should is undefined until
the contract is mined.

We get information about the transaction with `getTransactionReceipt`

```js
> eth.getTransactionReceipt(hello.transactionHash)
```

To mine the block we execute

```js
> miner.start(1); admin.sleepBlocks(2); miner.stop()
```

Now `eth.getTransactionReceipt` should return a non null object, and the
`address` field should be non null.

## Interacting with the contract

An Ethereum's contracts has two type of methods:

1.  The _constant methods_ those that do not change
    the contract's storage, their execution costs nothing
2.  The _non-constant_ methods are those that can modify
    the contract storage and they require
    to pay for their execution.

In our `hello` example `setSalute` is non-constant and `greet` is constant.

We use `sendTransaction` to execute a non-constant methods.

```js
> hello.setSalute.sendTransaction('Hi ', {from:eth.accounts[0], gas:1000000})
```

This returns a transaction hash. Again we mine a block to process the transaction

```js
> miner.start(1); admin.sleepBlocks(2); miner.stop()
```

Calling `getTransactionReceipt` with our transactionHash should be non-null.

And `call` to execute a constant method.

```js
> hello.greet.call()
```

Should print

```txt
"hi"
```

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

## Geth running in multiple instances

See [Running Multiple Geth Instances](Running Multiple Geth Instances.md).
Each instance has to have a separate datadir and use differents ports.

You can attach to a running instance of geth.

```sh
> geth --datadir node1 attach
```

After attach it is the whole path to the file geth.ipc inside the `datadir`
directoy for the running instance of geth.  

## Troubleshooting

### `Error: solc: exec: "solc": executable file not found in %!P(MISSING)ATH%!`

```txt
Error: solc: exec: "solc": executable file not found in %!P(MISSING)ATH%!
(MISSING)
    at web3.js:3104:20
    at web3.js:6191:15
    at web3.js:5004:36
    at <anonymous>:1:12
```

The _Solidity_ (`solc`) is missing. You should install the [Solidity Compiler](https://github.com/ethereum/solidity/releases)
and add it to the `PATH`, or specify the path with the `--solc` parameter
of `geth`.

## References

*   [Web3 JavaScript Dapp API](https://github.com/ethereum/wiki/wiki/JavaScript-API)
*   [Connecting to a private test net](https://ethereum.org/cli#connecting-to-a-private-test-net)
