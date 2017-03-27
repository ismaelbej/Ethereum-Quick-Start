# Ethereum Quick Start

This document will describe how to create an Ethereum private test network,
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

The ethereum package will install geth which is one of the Official
Ethereum nodes (it is programmed in Go). The package solc is the compiler
for Solidity, one of the programming language for Ethereum's contracts.

## Running a private testnet

Create a file genesis.json with the following content, this are the
parameters for our private testnet.

```json
{
  "nonce": "0xdeadbeefdeadbeef",
  "timestamp": "0x00",
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "extraData": "0x00",
  "gasLimit": "0x80000000",
  "difficulty": "0x00",
  "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "coinbase": "0x3333333333333333333333333333333333333333",
  "alloc": {
  }
}
```

We can initialize the private testnet, this command will create the _node1_
directory with the ethereum blockchain info within.

**This command should be run only once**.

```sh
> geth --datadir node1 init genesis.jon
```

Now to run the server we use the following command

```sh
> geth --networkid 6000 --datadir node1 --rpc --rpcapi eth,web3,net,personal,ssh,db --nodiscover console
```

This will leave us in command line environment, this is a limited
javascript prompt (has a simple autocomplete).

For example the following command

```js
> web3.version
```

Should output something like this

```js
{
  api: "0.15.3",
  ethereum: "0x3f",
  network: "6000",
  node: "Geth/v1.4.5-stable/linux/go1.6.1",
  whisper: undefined,
  getEthereum: function(callback),
  getNetwork: function(callback),
  getNode: function(callback),
  getWhisper: function(callback)
}
```

We can exit from the geth console typing Ctrl+D.

## Creating an account

The first thing we need to do is create a new account. From the geth console
(replacing "abcd123" with our own password).

```js
> personal.newAccount("abcd123")
```

Now `eth.accounts[0]` should contain the account recently created

```js
> eth.accounts[0]
```

To use this account we need to unlock it.

```js
> personal.unlockAccount(eth.accounts[0], "abcd123", 10000)
```

The first parameter is the account to unlock, the second parameter is
the account's password, and the third parameter is the time to unlock the
account for (in seconds).

We need to do this for every operation that will modify our balance.
For example send a transaction.

We can check our balance (which should be zero).

```js
> eth.getBalance(eth.accounts[0])
```

To deploy a contract, or any transaction we need to have some ether,
in the testnet we can mine some blocks.

```js
> miner.start(1); admin.sleepBlocks(2); miner.stop()
```

This will create one miner thread, and will wait to have mined at least
two blocks.

The **first time** that mining is started it will create the DAG,
which may take **several minutes**.

The DAG is a dataset used by the Ethereum proof of work to secure
its blockchain.

We can now display our balance in ethers

```js
> web3.fromWei(eth.getBalance(eth.accounts[0]), "ether")
```

## Deploying a contract

We will use the following contract

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

Store the contract in a variable

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
> hello = helloContract.new({from:eth.accounts[0], data:compiled.hello.code, gas:3000000})
```

The transaction for our contract was created, but will not be available
until it is mined in the blockchain. The resulting object `hello` contains
a field `transactionHash`, that can be used to check if the transaction
was mined. Also the field `address` from `hello` should be undefined until
the contract is mined.

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

An Ethereum's contracts has two type of methods: constant methods, and
non-constant methods. The constant methods those that do not change
the contract's storage, their execution costs nothing, and the non-constant
methods are those that can modify the contract storage and they require
to pay for their execution.

In our `hello` example `setSalute` is non-constant and `greet` is constant.

We use `sendTransaction` to execute a non-constant methods.

```js
> hello.setSalute.sendTransaction('Hi ', {from:eth.accounts[0], gas:1000000})
```

This will return a transaction hash, and the method will not be executed
until it is mined. Again we mine a block with

```js
> miner.start(1); admin.sleepBlocks(2); miner.stop()
```

from the console.

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
