---
title: EVM
weight: 90
---

{{< alert >}}
**Note**

XPLA Chain's evm module inherits from Evmos's [`evm`](https://docs.evmos.org/protocol/modules/evm) module. This document is a stub and covers mainly important XPLA Chain-specific notes about how it is used.
{{< /alert >}}

This document defines the specification of the Ethereum Virtual Machine (EVM) as a Cosmos SDK module.

Since the introduction of Ethereum in 2015, the ability to control digital assets through [**smart contracts**](https://www.fon.hum.uva.nl/rob/Courses/InformationInSpeech/CDROM/Literature/LOTwinterschool2006/szabo.best.vwh.net/idea.html) has attracted a large community of developers to build decentralized applications on the Ethereum Virtual Machine (EVM). This community is continuously creating extensive tooling and introducing standards, which are further increasing the adoption rate of EVM compatible technology.

The growth of EVM-based chains (e.g. Ethereum), however, has uncovered several scalability challenges that are often referred to as the [Trilemma of decentralization, security, and scalability](https://vitalik.ca/general/2021/04/07/sharding.html). Developers are frustrated by high gas fees, slow transaction speed & throughput, and chain-specific governance that can only undergo slow change because of its wide range of deployed applications. A solution is required that eliminates these concerns for developers, who build applications within a familiar EVM environment.

The `x/evm` module provides this EVM familiarity on a scalable, high-throughput Proof-of-Stake blockchain. It is built as a [Cosmos SDK module](https://docs.cosmos.network/v0.45/building-modules/intro.html) which allows for the deployment of smart contracts, interaction with the EVM state machine (state transitions), and the use of EVM tooling. It can be used on Cosmos application-specific blockchains, which alleviate the aforementioned concerns through high transaction throughput via [Tendermint Core](https://github.com/tendermint/tendermint), fast transaction finality, and horizontal scalability via [IBC](https://ibcprotocol.org/).

The `x/evm` is part of the [ethermint library](https://pkg.go.dev/github.com/evmos/ethermint).

This module will be used in the XPLA Chain.

## Concepts

### EVM

The Ethereum Virtual Machine (EVM) is a computation engine which can be thought of as one single entity maintained by thousands of connected computers (nodes) running an Ethereum client. As a virtual machine ([VM](https://en.wikipedia.org/wiki/Virtual_machine)), the EVM is responisble for computing changes to the state deterministically regardless of its environment (hardware and OS). This means that every node has to get the exact same result given an identical starting state and transaction (tx).

The EVM is considered to be the part of the Ethereum protocol that handles the deployment and execution of [smart contracts](https://ethereum.org/en/developers/docs/smart-contracts/). To make a clear distinction:

* The Ethereum protocol describes a blockchain, in which all Ethereum accounts and smart contracts live. It has only one canonical state (a data structure, which keeps all accounts) at any given block in the chain.
* The EVM, however, is the [state machine](https://en.wikipedia.org/wiki/Finite-state_machine) that defines the rules for computing a new valid state from block to block. It is an isolated runtime, which means that code running inside the EVM has no access to network, filesystem, or other processes (not external APIs).

The `x/evm` module implements the EVM as a Cosmos SDK module. It allows users to interact with the EVM by submitting Ethereum txs and executing their containing messages on the given state to evoke a state transition.

#### State

The Ethereum state is a data structure, implemented as a [Merkle Patricia Trie](https://en.wikipedia.org/wiki/Merkle_tree), that keeps all accounts on the chain. The EVM makes changes to this data structure resulting in a new state with a different State Root. Ethereum can therefore be seen as a state chain that transitions from one state to another by executing transations in a block using the EVM. A new block of txs can be described through its Block header (parent hash, block number, time stamp, nonce, receipts,...).

#### Accounts

There are two types of accounts that can be stored in state at a given address:

* **Externally Owned Account (EOA)**: Has nonce (tx counter) and balance
* **Smart Contract**: Has nonce, balance, (immutable) code hash, storage root (another Merkle Patricia Trie)

Smart contracts are just like regular accounts on the blockchain, which additionally store executable code in an Ethereum-specific binary format, known as **EVM bytecode**. They are typically written in an Ethereum high level language such as Solidity which is compiled down to EVM bytecode and deployed on the blockchain, by submitting a tx using an Ethereum client.

#### Architecture

The EVM operates as a stack-based machine. It's main architecture components consist of:

* Virtual ROM: contract code is pulled into this read only memory when processing txs
* Machine state (volatile): changes as the EVM runs and is wiped clean after processing each tx
    * Program counter (PC)
    * Gas: keeps track of how much gas is used
    * Stack and Memory: compute state changes
* Access to account storage (persistent)

#### State Transitions with Smart Contracts

Typically smart contracts expose a public ABI, which is a list of supported ways a user can interact with a contract. To interact with a contract and invoke a state transition, a user will submit a tx carrying any amount of gas and a data payload formatted according to the ABI, specifying the type of interaction and any additional parameters. When the tx is received, the EVM executes the smart contracts's EVM bytecode using the tx payload.

#### Executing EVM bytecode

A contract's EVM bytecode consists of basic operations (add, multiply, store, etc...), called **Opcodes**. Each Opcode execution requires gas that needs to be payed with the tx. The EVM is therefore considered quasi-turing complete, as it allows any arbitrary computation, but the amount of computations during a contract execution is limited to the amount of gas provided in the tx. Each Opcode's [**gas cost**](https://www.evm.codes/) reflects the cost of running these operations on actual computer hardware (e.g. `ADD = 3gas` and `SSTORE = 100gas`). To calculate the gas consumption of a tx, the gas cost is multiplied by the **gas price**, which can change depending on the demand of the network at the time. If the network is under heavy load, you might have to pay a highter gas price to get your tx executed. If the gas limit is hit (out of gas execption) no changes to the Ethereum state are applied, except that the sender's nonce increments and their balance goes down to pay for wasting the EVM's time.

Smart contracts can also call other smart contracts. Each call to a new contract creates a new instance of the EVM (including a new stack and memory). Each call passes the sandbox state to the next EVM. If the gas runs out, all state changes are discareded. Otherwise they are kept.

For further reading, please refer to:

* [EVM](https://eth.wiki/concepts/evm/evm)
* [EVM Architecture](https://cypherpunks-core.github.io/ethereumbook/13evm.html#evm_architecture)
* [What is Ethereum](https://ethdocs.org/en/latest/introduction/what-is-ethereum.html#what-is-ethereum)
* [Opcodes](https://www.ethervm.io/)

### Ethermint as Geth implementation

Ethermint is an implementation of the [Etherum protocal in Golang](https://geth.ethereum.org/docs/getting-started) (Geth) as a Cosmos SDK module. Geth includes an implementation of the EVM to compute state transitions. Have a look at the [go-etheruem source code](https://github.com/ethereum/go-ethereum/blob/master/core/vm/instructions.go) to see how the EVM opcodes are implemented. Just as Geth can be run as an Ethereum node, Ethermint can be run as a node to compute state transitions with the EVM. Ethermint supports Geth's standard [Ethereum JSON-RPC APIs](https://docs.evmos.org/develop/api/ethereum-json-rpc/methods#endpoints) in order to be Web3 and EVM compatible.

#### JSON-RPC

JSON-RPC is a stateless, lightweight remote procedure call (RPC) protocol. Primarily this specification defines several data structures and the rules around their processing. It is transport agnostic in that the concepts can be used within the same process, over sockets, over HTTP, or in many various message passing environments. It uses JSON (RFC 4627) as a data format.

##### JSON-RPC Example: `eth_call`

The JSON-RPC method [`eth_call`](https://docs.evmos.org/develop/api/ethereum-json-rpc#json-rpc-over-http) allows you to execute messages against contracts. Usually, you need to send a transaction to a Geth node to include it in the mempool, then nodes gossip between each other and eventually the transaction is included in a block and gets executed. `eth_call` however lets you send data to a contract and see what happens without commiting a transaction.

In the Geth implementation, calling the endpoint roughly goes through the following steps:

1. The `eth_call` request is transformed to call the `func (s *PublicBlockchainAPI) Call()` function using the `eth` namespace
2. [`Call()`](https://github.com/ethereum/go-ethereum/blob/master/internal/ethapi/api.go#L982) is given the transaction arguments, the block to call against and optional overides that modify the state to call against. It then calls `DoCall()`
3. [`DoCall()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/internal/ethapi/api.go#L891) transforms the arguments into a `ethtypes.message`, instantiates an EVM and applies the message with `core.ApplyMessage`
4. [`ApplyMessage()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/state_transition.go#L180) calls the state transition `TransitionDb()`
5. [`TransitionDb()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/state_transition.go#L275) either `Create()`s a new contract or `Call()`s a contract
6. [`evm.Call()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/vm/evm.go#L168) runs the interpreter `evm.interpreter.Run()` to execute the message. If the execution fails, the state is reverted to a snapshot taken before the execution and gas is consumed.
7. [`Run()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/vm/interpreter.go#L116) performs a loop to execute the opcodes.

The ethermint implementatiom is similar and makes use of the gRPC query client which is included in the Cosmos SDK:

1. `eth_call` request is transformed to call the `func (e *PublicAPI) Call` function using the `eth` namespace
2. [`Call()`](https://github.com/evmos/ethermint/blob/main/rpc/namespaces/ethereum/eth/api.go#L639) calls `doCall()`
3. [`doCall()`](https://github.com/evmos/ethermint/blob/main/rpc/namespaces/ethereum/eth/api.go#L656) transforms the arguments into a `EthCallRequest` and calls `EthCall()` using the query client of the evm module.
4. [`EthCall()`](https://github.com/evmos/ethermint/blob/main/x/evm/keeper/grpc_query.go#L212) transforms the arguments into a `ethtypes.message` and calls `ApplyMessageWithConfig()
5. [`ApplyMessageWithConfig()`](https://github.com/evmos/ethermint/blob/d5598932a7f06158b7a5e3aa031bbc94eaaae32c/x/evm/keeper/state_transition.go#L341) instantiates an EVM and either `Create()`s a new contract or `Call()`s a contract using the Geth implementation.

#### StateDB

The `StateDB` interface from [go-ethereum](https://github.com/ethereum/go-ethereum/blob/master/core/vm/interface.go) represents an EVM database for full state querying. EVM state transitions are enabled by this interface, which in the `x/evm` module is implemented by the `Keeper`. The implementation of this interface is what makes Ethermint EVM compatible.

### Consensus Engine

The application using the `x/evm` module interacts with the Tendermint Core Consensus Engine over an Application Blockchain Interface (ABCI). Together, the application and Tendermint Core form the programs that run a complete blockchain and combine business logic with decentralized data storage.

Ethereum transactions that are submitted to the `x/evm` module take part in a this consensus process before being executed and changing the application state. We encourage to understand the basics of the [Tendermint consensus engine](https://docs.tendermint.com/v0.34/introduction/what-is-tendermint.html#intro-to-abci) in order to understand state transitions in detail.

### Transaction Logs

On every `x/evm` transaction, the result contains the Ethereum `Log`s from the state machine execution that are used by the JSON-RPC Web3 server for filter querying and for processing the EVM Hooks.

The tx logs are stored in the transient store during tx execution and then emitted through cosmos events after the transaction has been processed. They can be queried via gRPC and JSON-RPC.

### Block Bloom

Bloom is the bloom filter value in bytes for each block that can be used for filter queries. The block bloom value is stored in the transient store and then emitted through a cosmos event during `EndBlock` processing. They can be queried via gRPC and JSON-RPC.

{{< alert >}}
👉 **Note**: Since they are not stored on state, Transaction Logs and Block Blooms are not persisted after upgrades. A user must use an archival node after upgrades in order to obtain legacy chain events.
{{< /alert >}}

## State

This section gives you an overview of the objects stored in the `x/evm` module state, functionalities that are derived from the go-ethereum `StateDB` interface, and its implementation through the Keeper as well as the state implementation at genesis.

### State Objects

The `x/evm` module keeps the following objects in state:

#### State

|             | Description                                                  | Key                           | Value               | Store     |
| ----------- | ------------------------------------------------------------ | ----------------------------- | ------------------- | --------- |
| Code        | Smart contract bytecode                                      | `[]byte{1} + []byte(address)` | `[]byte{code}`      | KV        |
| Storage     | Smart contract storage                                       | `[]byte{2} + [32]byte{key}`   | `[32]byte(value)`   | KV        |
| Block Bloom | Block bloom filter, used to accumulate the bloom filter of current block, emitted to events at end blocker. | `[]byte{1} + []byte(tx.Hash)` | `protobuf([]Log)`   | Transient |
| Tx Index    | Index of current transaction in current block.               | `[]byte{2}`                   | `BigEndian(uint64)` | Transient |
| Log Size    | Number of the logs emitted so far in current block. Used to decide the log index of following logs. | `[]byte{3}`                   | `BigEndian(uint64)` | Transient |
| Gas Used    | Amount of gas used by ethereum messages of current cosmos-sdk tx, it's necessary when cosmos-sdk tx contains multiple ethereum messages. | `[]byte{4}`                   | `BigEndian(uint64)` | Transient |

### StateDB

The `StateDB` interface is implemented by the `StateDB` in the `x/evm/statedb` module to represent an EVM database for full state querying of both contracts and accounts. Within the Ethereum protocol, `StateDB`s are used to store anything within the IAVL tree and take care of caching and storing nested states.

```go
// github.com/ethereum/go-ethereum/core/vm/interface.go
type StateDB interface {
 CreateAccount(common.Address)

 SubBalance(common.Address, *big.Int)
 AddBalance(common.Address, *big.Int)
 GetBalance(common.Address) *big.Int

 GetNonce(common.Address) uint64
 SetNonce(common.Address, uint64)

 GetCodeHash(common.Address) common.Hash
 GetCode(common.Address) []byte
 SetCode(common.Address, []byte)
 GetCodeSize(common.Address) int

 AddRefund(uint64)
 SubRefund(uint64)
 GetRefund() uint64

 GetCommittedState(common.Address, common.Hash) common.Hash
 GetState(common.Address, common.Hash) common.Hash
 SetState(common.Address, common.Hash, common.Hash)

 Suicide(common.Address) bool
 HasSuicided(common.Address) bool

 // Exist reports whether the given account exists in state.
 // Notably this should also return true for suicided accounts.
 Exist(common.Address) bool
 // Empty returns whether the given account is empty. Empty
 // is defined according to EIP161 (balance = nonce = code = 0).
 Empty(common.Address) bool

 PrepareAccessList(sender common.Address, dest *common.Address, precompiles []common.Address, txAccesses types.AccessList)
 AddressInAccessList(addr common.Address) bool
 SlotInAccessList(addr common.Address, slot common.Hash) (addressOk bool, slotOk bool)
 // AddAddressToAccessList adds the given address to the access list. This operation is safe to perform
 // even if the feature/fork is not active yet
 AddAddressToAccessList(addr common.Address)
 // AddSlotToAccessList adds the given (address,slot) to the access list. This operation is safe to perform
 // even if the feature/fork is not active yet
 AddSlotToAccessList(addr common.Address, slot common.Hash)

 RevertToSnapshot(int)
 Snapshot() int

 AddLog(*types.Log)
 AddPreimage(common.Hash, []byte)

 ForEachStorage(common.Address, func(common.Hash, common.Hash) bool) error
}
```

The `StateDB` in the `x/evm` provides the following functionalities:

#### CRUD of Ethereum accounts

You can create `EthAccount` instances from the provided address and set the value to store on the  `AccountKeeper`with `createAccount()`. If an account with the given address already exists, this function also resets any preexisting code and storage associated with that address.

An account's coin balance can be is managed through the `BankKeeper` and can be read with `GetBalance()` and updated with `AddBalance()` and `SubBalance()`.

- `GetBalance()` returns the EVM denomination balance of the provided address. The denomination is obtained from the module parameters.
- `AddBalance()` adds the given amount to the address balance coin by minting new coins and transferring them to the address. The coin denomination is obtained from the module parameters.
- `SubBalance()` subtracts the given amount from the address balance by transferring the coins to an escrow account and then burning them. The coin denomination is obtained from the module parameters. This function performs a no-op if the amount is negative or the user doesn't have enough funds for the transfer.

The nonce (or transaction sequence) can be obtained from the Account `Sequence` via the auth module `AccountKeeper`.

- `GetNonce()` retrieves the account with the given address and returns the tx sequence (i.e nonce). The function performs a no-op if the account is not found.
- `SetNonce()` sets the given nonce as the sequence of the address' account. If the account doesn't exist, a new one will be created from the address.

The smart contract bytecode containing arbitrary contract logic is stored on the `EVMKeeper` and it can be queried with `GetCodeHash()` ,`GetCode()` & `GetCodeSize()`and updated with `SetCode()`.

- `GetCodeHash()` fetches the account from the store and returns its code hash. If the account doesn't exist or is not an EthAccount type, it returns the empty code hash value.
- `GetCode()` returns the code byte array associated with the given address. If the code hash from the account is empty, this function returns nil.
- `SetCode()` stores the code byte array to the application KVStore and sets the code hash to the given account. The code is deleted from the store if it is empty.
- `GetCodeSize()` returns the size of the contract code associated with this object, or zero if none.

Gas refunded needs to be tracked and stored in a separate variable in
order to add it subtract/add it from/to the gas used value after the EVM
execution has finalized. The refund value is cleared on every transaction and at the end of every block.

- `AddRefund()` adds the given amount of gas to the in-memory refund value.
- `SubRefund()` subtracts the given amount of gas from the in-memory refund value. This function will panic if gas amount is greater than the current refund.
- `GetRefund()` returns the amount of gas available for return after the tx execution finalizes. This value is reset to 0 on every transaction.

The state is stored on the `EVMKeeper`. It can be queried with `GetCommittedState()`, `GetState()` and updated with `SetState()`.

- `GetCommittedState()` returns the value set in store for the given key hash. If the key is not registered this function returns the empty hash.
- `GetState()` returns the in-memory dirty state for the given key hash, if not exist load the committed value from KVStore.
- `SetState()` sets the given hashes (key, value) to the state. If the value hash is empty, this function deletes the key from the state, the new value is kept in dirty state at first, and will be committed to KVStore in the end.

Accounts can also be set to a suicide state. When a contract commits suicide, the account is marked as suicided, when committing the code, storage and account are deleted (from the next block and forward).

- `Suicide()` marks the given account as suicided and clears the account balance of the EVM tokens.
- `HasSuicided()` queries the in-memory flag to check if the account has been marked as suicided in the current transaction. Accounts that are suicided will be returned as non-nil during queries and "cleared" after the block has been committed.

To check account existence use `Exist()` and `Empty()`.

- `Exist()` returns true if the given account exists in store or if it has been
marked as suicided.
- `Empty()` returns true if the address meets the following conditions:
    - nonce is 0
    - balance amount for evm denom is 0
    - account code hash is empty

#### EIP2930 functionality

Supports a transaction type that contains an [access list](https://eips.ethereum.org/EIPS/eip-2930), a list of addresses, and storage keys that the transaction plans to access. The access list state is kept in memory and discarded after the transaction committed.

- `PrepareAccessList()` handles the preparatory steps for executing a state transition with regards to both EIP-2929 and EIP-2930. This method should only be called if Yolov3/Berlin/2929+2930 is applicable at the current number.
    - Add sender to access list (EIP-2929)
    - Add destination to access list (EIP-2929)
    - Add precompiles to access list (EIP-2929)
    - Add the contents of the optional tx access list (EIP-2930)
- `AddressInAccessList()` returns true if the address is registered.
- `SlotInAccessList()` checks if the address and the slots are registered.
- `AddAddressToAccessList()` adds the given address to the access list. If the address is already in the access list, this function performs a no-op.
- `AddSlotToAccessList()` adds the given (address, slot) to the access list. If the address and slot are already in the access list, this function performs a no-op.

#### Snapshot state and Revert functionality

The EVM uses state-reverting exceptions to handle errors. Such an exception will undo all changes made to the state in the current call (and all its sub-calls), and the caller could handle the error and don't propagate. You can use `Snapshot()` to identify the current state with a revision and revert the state to a given revision with `RevertToSnapshot()` to support this feature.

- `Snapshot()` creates a new snapshot and returns the identifier.
- `RevertToSnapshot(rev)` undo all the modifications up to the snapshot identified as `rev`.

This module adapted the [go-ethereum journal implementation](https://github.com/ethereum/go-ethereum/blob/master/core/state/journal.go#L39) to support this, it uses a list of journal logs to record all the state modification operations done so far,
snapshot is consists of a unique id and an index in the log list, and to revert to a snapshot it just undo the journal logs after the snapshot index in reversed order.

#### Ethereum Transaction logs

With `AddLog()` you can append the given ethereum `Log` to the list of Logs associated with the transaction hash kept in the current state. This function also fills in the tx hash, block hash, tx index and log index fields before setting the log to store.

### Keeper

The EVM module `Keeper` grants access to the EVM module state and implements `statedb.Keeper` interface to support the `StateDB` implementation. The Keeper contains a store key that allows the DB to write to a concrete subtree of the multistore that is only accessible to the EVM module. Instead of using a trie and database for querying and persistence (the `StateDB` implementation on Ethermint), use the Cosmos `KVStore` (key-value store) and Cosmos SDK `Keeper` to facilitate state transitions.

To support the interface functionality, it imports 4 module Keepers:

- `auth`: CRUD accounts
- `bank`: accounting (supply) and CRUD of balances
- `staking`: query historical headers
- `fee market`: EIP1559 base fee for processing `DynamicFeeTx` after the `London` hard fork has been activated on the `ChainConfig` parameters

```go
type Keeper struct {
 // Protobuf codec
 cdc codec.BinaryCodec
 // Store key required for the EVM Prefix KVStore. It is required by:
 // - storing account's Storage State
 // - storing account's Code
 // - storing Bloom filters by block height. Needed for the Web3 API.
 // For the full list, check the module specification
 storeKey sdk.StoreKey

 // key to access the transient store, which is reset on every block during Commit
 transientKey sdk.StoreKey

 // module specific parameter space that can be configured through governance
 paramSpace paramtypes.Subspace
 // access to account state
 accountKeeper types.AccountKeeper
 // update balance and accounting operations with coins
 bankKeeper types.BankKeeper
 // access historical headers for EVM state transition execution
 stakingKeeper types.StakingKeeper
 // fetch EIP1559 base fee and parameters
 feeMarketKeeper types.FeeMarketKeeper

 // chain ID number obtained from the context's chain id
 eip155ChainID *big.Int

 // Tracer used to collect execution traces from the EVM transaction execution
 tracer string
 // trace EVM state transition execution. This value is obtained from the `--trace` flag.
 // For more info check https://geth.ethereum.org/docs/dapp/tracing
 debug bool

 // EVM Hooks for tx post-processing
 hooks types.EvmHooks
}
```

### Genesis State

The `x/evm` module `GenesisState` defines the state necessary for initializing the chain from a previous exported height. It contains the `GenesisAccounts` and the module parameters

```go
type GenesisState struct {
  // accounts is an array containing the ethereum genesis accounts.
  Accounts []GenesisAccount `protobuf:"bytes,1,rep,name=accounts,proto3" json:"accounts"`
  // params defines all the parameters of the module.
  Params Params `protobuf:"bytes,2,opt,name=params,proto3" json:"params"`
}
```

### Genesis Accounts

The `GenesisAccount` type corresponds to an adaptation of the Ethereum `GenesisAccount` type. It defines an account to be initialized in the genesis state.

Its main difference is that the one on Ethermint uses a custom `Storage` type that uses a slice instead of maps for the evm `State` (due to non-determinism), and that it doesn't contain the private key field.

It is also important to note that since the `auth` module on the Cosmos SDK manages the account state,  the `Address` field must correspond to an existing `EthAccount` that is stored in the `auth`'s module `Keeper` (i.e `AccountKeeper`). Addresses use the **[EIP55](https://eips.ethereum.org/EIPS/eip-55)** hex **[format](https://docs.evmos.org/protocol/concepts/accounts#address-formats-for-clients)** on `genesis.json`.

```go
type GenesisAccount struct {
  // address defines an ethereum hex formated address of an account
  Address string `protobuf:"bytes,1,opt,name=address,proto3" json:"address,omitempty"`
  // code defines the hex bytes of the account code.
  Code string `protobuf:"bytes,2,opt,name=code,proto3" json:"code,omitempty"`
  // storage defines the set of state key values for the account.
  Storage Storage `protobuf:"bytes,3,rep,name=storage,proto3,castrepeated=Storage" json:"storage"`
}
```

## State Transitions

The `x/evm` module allows for users to submit Ethereum transactions (`Tx`) and execute their containing messages to evoke state transitions on the given state.

Users submit transactions client-side to broadcast it to the network. When the transaction is included in a block during consensus, it is executed server-side. We highly recommend to understand the basics of the [Tendermint consensus engine](https://docs.tendermint.com/v0.34/introduction/what-is-tendermint.html#intro-to-abci) to understand the State Transitions in detail.

### Client-Side

{{< alert >}}
👉 This is based on the `eth_sendTransaction` JSON-RPC
{{< /alert >}}

1. A user submits a transaction via one of the available JSON-RPC endpoints using an Ethereum-compatible client or wallet (eg Metamask, WalletConnect, Ledger, etc):
 a. eth (public) namespace:
     - `eth_sendTransaction`
     - `eth_sendRawTransaction`
 b. personal (private) namespace:
     - `personal_sendTransaction`
2. An instance of `MsgEthereumTx` is created after populating the RPC transaction using `SetTxDefaults` to fill missing tx arguments with  default values
3. The `Tx` fields are validated (stateless) using `ValidateBasic()`
4. The `Tx` is **signed** using the key associated with the sender address and the latest ethereum hard fork (`London`, `Berlin`, etc) from the `ChainConfig`
5. The `Tx` is **built** from the msg fields using the Cosmos Config builder
6. The `Tx` is **broadcasted** in [sync mode](https://docs.cosmos.network/v0.45/run-node/txs.html#broadcasting-a-transaction-3) to ensure to wait for a [`CheckTx`](https://docs.tendermint.com/v0.34/introduction/what-is-tendermint.html#intro-to-abci) execution response. Transactions are validated by the application using `CheckTx()`, before being added to the mempool of the consensus engine.
7. JSON-RPC user receives a response with the [`RLP`](https://eth.wiki/en/fundamentals/rlp) hash of the transaction fields. This hash is different from the default hash used by SDK Transactions that calculates the `sha256` hash of the transaction bytes.

### Server-Side

Once a block (containing the `Tx`) has been committed during consensus, it is applied to the application in a series of ABCI msgs server-side.

Each `Tx` is handled by the application by calling [`RunTx`](https://docs.cosmos.network/v0.45/core/baseapp.html#runtx). After a stateless validation on each `sdk.Msg` in the `Tx`, the `AnteHandler` confirms whether the `Tx` is an Ethereum or SDK transaction. As an Ethereum transaction it's containing msgs are then handled by the `x/evm` module to update the application's state.

#### AnteHandler

The `anteHandler` is run for every transaction. It checks if the `Tx` is an Ethereum transaction and routes it to an internal ante handler. Here, `Tx`s are handled using EthereumTx extension options to process them differently than normal Cosmos SDK transactions. The `antehandler` runs through a series of options and their `AnteHandle` functions for each `Tx`:

- `EthSetUpContextDecorator()` is adapted from SetUpContextDecorator from cosmos-sdk, it ignores gas consumption by setting the gas meter to infinite
- `EthValidateBasicDecorator(evmKeeper)` validates the fields of a Ethereum type Cosmos `Tx` msg
- `EthSigVerificationDecorator(evmKeeper)` validates that the registered chain id is the same as the one on the message, and that the signer address matches the one defined on the message. It's not skipped for RecheckTx, because it set `From` address which is critical from other ante handler to work. Failure in RecheckTx will prevent tx to be included into block, especially when CheckTx succeed, in which case user won't see the error message.
- `EthAccountVerificationDecorator(ak, bankKeeper, evmKeeper)` that the sender balance is greater than the total transaction cost. The account will be set to store if it doesn't exist, i.e cannot be found on store. This AnteHandler decorator will fail if:
    - any of the msgs is not a MsgEthereumTx
    - from address is empty
    - account balance is lower than the transaction cost
- `EthNonceVerificationDecorator(ak)` validates that the transaction nonces are valid and equivalent to the sender account’s current nonce.
- `EthGasConsumeDecorator(evmKeeper)` validates that the Ethereum tx message has enough to cover intrinsic gas (during CheckTx only) and that the sender has enough balance to pay for the gas cost. Intrinsic gas for a transaction is the amount of gas that the transaction uses before the transaction is executed. The gas is a constant value plus any cost incurred by additional bytes of data supplied with the transaction. This AnteHandler decorator will fail if:
    - the transaction contains more than one message
    - the message is not a MsgEthereumTx
    - sender account cannot be found
    - transaction's gas limit is lower than the intrinsic gas
    - user doesn't have enough balance to deduct the transaction fees (gas_limit * gas_price)
    - transaction or block gas meter runs out of gas
- `CanTransferDecorator(evmKeeper, feeMarketKeeper)` creates an EVM from the message and calls the BlockContext CanTransfer function to see if the address can execute the transaction.
- `EthIncrementSenderSequenceDecorator(ak)`  handles incrementing the sequence of the signer (i.e sender). If the transaction is a contract creation, the nonce will be incremented during the transaction execution and not within this AnteHandler decorator.

The options `authante.NewMempoolFeeDecorator()`, `authante.NewTxTimeoutHeightDecorator()` and `authante.NewValidateMemoDecorator(ak)` are the same as for a Cosmos `Tx`. Click [here](https://docs.cosmos.network/v0.45/basics/gas-fees.html#antehandler) for more on the `anteHandler`.

#### EVM module

After authentication through the `antehandler`, each `sdk.Msg` (in this case `MsgEthereumTx`) in the `Tx` is delivered to the Msg Handler in the `x/evm` module and runs through the following the steps:

1. Convert `Msg` to an ethereum `Tx` type
2. Apply `Tx` with `EVMConfig` and attempt to perform a state transition, that will only be persisted (committed) to the underlying KVStore if the transaction does not fail:
    1. Confirm that `EVMConfig` is created
    2. Create the ethereum signer using chain config value from `EVMConfig`
    3. Set the ethereum transaction hash to the (impermanent) transient store so that it's also available on the StateDB functions
    4. Generate a new EVM instance
    5. Confirm that EVM params for contract creation (`EnableCreate`) and contract execution (`EnableCall`) are enabled
    6. Apply message. If `To` address is `nil`, create new contract using code as deployment code. Else call contract at given address with the given input as parameters
    7. Calculate gas used by the evm operation
3. If `Tx` applied sucessfully
    1. Execute EVM `Tx` postprocessing hooks. If hooks return error, revert the whole `Tx`
    2. Refund gas according to Ethereum gas accounting rules
    3. Update block bloom filter value using the logs generated from the tx
    4. Emit SDK events for the transaction fields and tx logs

## Message Types

This section defines the `sdk.Msg` concrete types that result in the state transitions defined on the previous section.

### MsgEthereumTx

An EVM state transition can be achieved by using the `MsgEthereumTx`. This message encapsulates an Ethereum transaction data (`TxData`) as a `sdk.Msg`. It contains the necessary transaction data fields. Note, that the `MsgEthereumTx` implements both the [`sdk.Msg`](https://github.com/cosmos/cosmos-sdk/blob/v0.39.2/types/tx_msg.go#L7-L29) and [`sdk.Tx`](https://github.com/cosmos/cosmos-sdk/blob/v0.39.2/types/tx_msg.go#L33-L41) interfaces. Normally,  SDK messages only implement the former, while the latter is a group of messages bundled together.

```go
type MsgEthereumTx struct {
 // inner transaction data
 Data *types.Any `protobuf:"bytes,1,opt,name=data,proto3" json:"data,omitempty"`
 // DEPRECATED: encoded storage size of the transaction
 Size_ float64 `protobuf:"fixed64,2,opt,name=size,proto3" json:"-"`
 // transaction hash in hex format
 Hash string `protobuf:"bytes,3,opt,name=hash,proto3" json:"hash,omitempty" rlp:"-"`
 // ethereum signer address in hex format. This address value is checked
 // against the address derived from the signature (V, R, S) using the
 // secp256k1 elliptic curve
 From string `protobuf:"bytes,4,opt,name=from,proto3" json:"from,omitempty"`
}
```

This message field validation is expected to fail if:

- `From` field is defined and the address is invalid
- `TxData` stateless validation fails

The transaction execution is expected to fail if:

- Any of the custom `AnteHandler` Ethereum decorators checks fail:
    - Minimum gas amount requirements for transaction
    - Tx sender account doesn't exist or hasn't enough balance for fees
    - Account sequence doesn't match the transaction `Data.AccountNonce`
    - Message signature verification fails
- EVM contract creation (i.e `evm.Create`) fails, or `evm.Call` fails

#### Conversion

The `MsgEthreumTx` can be converted to the go-ethereum `Transaction` and `Message` types in order to create and call evm contracts.

```go
// AsTransaction creates an Ethereum Transaction type from the msg fields
func (msg MsgEthereumTx) AsTransaction() *ethtypes.Transaction {
 txData, err := UnpackTxData(msg.Data)
 if err != nil {
  return nil
 }

 return ethtypes.NewTx(txData.AsEthereumData())
}

// AsMessage returns the transaction as a core.Message.
func (tx *Transaction) AsMessage(s Signer, baseFee *big.Int) (Message, error) {
 msg := Message{
  nonce:      tx.Nonce(),
  gasLimit:   tx.Gas(),
  gasPrice:   new(big.Int).Set(tx.GasPrice()),
  gasFeeCap:  new(big.Int).Set(tx.GasFeeCap()),
  gasTipCap:  new(big.Int).Set(tx.GasTipCap()),
  to:         tx.To(),
  amount:     tx.Value(),
  data:       tx.Data(),
  accessList: tx.AccessList(),
  isFake:     false,
 }
 // If baseFee provided, set gasPrice to effectiveGasPrice.
 if baseFee != nil {
  msg.gasPrice = math.BigMin(msg.gasPrice.Add(msg.gasTipCap, baseFee), msg.gasFeeCap)
 }
 var err error
 msg.from, err = Sender(s, tx)
 return msg, err
}
```

#### Signing

In order for the signature verification to be valid, the  `TxData` must contain the `v | r | s` values from the `Signer`. Sign calculates a secp256k1 ECDSA signature and signs the transaction. It takes a keyring signer and the chainID to sign an Ethereum transaction according to EIP155 standard. This method mutates the transaction as it populates the V, R, S fields of the Transaction's Signature. The function will fail if the sender address is not defined for the msg or if the sender is not registered on the keyring.

```go
// Sign calculates a secp256k1 ECDSA signature and signs the transaction. It
// takes a keyring signer and the chainID to sign an Ethereum transaction according to
// EIP155 standard.
// This method mutates the transaction as it populates the V, R, S
// fields of the Transaction's Signature.
// The function will fail if the sender address is not defined for the msg or if
// the sender is not registered on the keyring
func (msg *MsgEthereumTx) Sign(ethSigner ethtypes.Signer, keyringSigner keyring.Signer) error {
 from := msg.GetFrom()
 if from.Empty() {
  return fmt.Errorf("sender address not defined for message")
 }

 tx := msg.AsTransaction()
 txHash := ethSigner.Hash(tx)

 sig, _, err := keyringSigner.SignByAddress(from, txHash.Bytes())
 if err != nil {
  return err
 }

 tx, err = tx.WithSignature(ethSigner, sig)
 if err != nil {
  return err
 }

 msg.FromEthereumTx(tx)
 return nil
}
```

### TxData

The `MsgEthereumTx` supports the 3 valid Ethereum transaction data types from go-ethereum: `LegacyTx`, `AccessListTx`  and `DynamicFeeTx`. These types are defined as protobuf messages and packed into a `proto.Any` interface type in the `MsgEthereumTx` field.

- `LegacyTx`: [EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md) transaction type
- `DynamicFeeTx`: [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559) transaction type. Enabled by London hard fork block
- `AccessListTx`: [EIP-2930](https://eips.ethereum.org/EIPS/eip-2930) transaction type. Enabled by Berlin hard fork block

#### LegacyTx

The transaction data of regular Ethereum transactions.

```go
type LegacyTx struct {
 // nonce corresponds to the account nonce (transaction sequence).
 Nonce uint64 `protobuf:"varint,1,opt,name=nonce,proto3" json:"nonce,omitempty"`
 // gas price defines the value for each gas unit
 GasPrice *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,2,opt,name=gas_price,json=gasPrice,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_price,omitempty"`
 // gas defines the gas limit defined for the transaction.
 GasLimit uint64 `protobuf:"varint,3,opt,name=gas,proto3" json:"gas,omitempty"`
 // hex formatted address of the recipient
 To string `protobuf:"bytes,4,opt,name=to,proto3" json:"to,omitempty"`
 // value defines the unsigned integer value of the transaction amount.
 Amount *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,5,opt,name=value,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"value,omitempty"`
 // input defines the data payload bytes of the transaction.
 Data []byte `protobuf:"bytes,6,opt,name=data,proto3" json:"data,omitempty"`
 // v defines the signature value
 V []byte `protobuf:"bytes,7,opt,name=v,proto3" json:"v,omitempty"`
 // r defines the signature value
 R []byte `protobuf:"bytes,8,opt,name=r,proto3" json:"r,omitempty"`
 // s define the signature value
 S []byte `protobuf:"bytes,9,opt,name=s,proto3" json:"s,omitempty"`
}
```

This message field validation is expected to fail if:

- `GasPrice` is invalid (`nil` , negaitve or out of int256 bound)
- `Fee` (gasprice * gaslimit) is invalid
- `Amount` is invalid (negaitve or out of int256 bound)
- `To` address is invalid (non valid ethereum hex address)

#### DynamicFeeTx

The transaction data of EIP-1559 dynamic fee transactions.

```go
type DynamicFeeTx struct {
 // destination EVM chain ID
 ChainID *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,1,opt,name=chain_id,json=chainId,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"chainID"`
 // nonce corresponds to the account nonce (transaction sequence).
 Nonce uint64 `protobuf:"varint,2,opt,name=nonce,proto3" json:"nonce,omitempty"`
 // gas tip cap defines the max value for the gas tip
 GasTipCap *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,3,opt,name=gas_tip_cap,json=gasTipCap,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_tip_cap,omitempty"`
 // gas fee cap defines the max value for the gas fee
 GasFeeCap *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,4,opt,name=gas_fee_cap,json=gasFeeCap,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_fee_cap,omitempty"`
 // gas defines the gas limit defined for the transaction.
 GasLimit uint64 `protobuf:"varint,5,opt,name=gas,proto3" json:"gas,omitempty"`
 // hex formatted address of the recipient
 To string `protobuf:"bytes,6,opt,name=to,proto3" json:"to,omitempty"`
 // value defines the the transaction amount.
 Amount *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,7,opt,name=value,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"value,omitempty"`
 // input defines the data payload bytes of the transaction.
 Data     []byte     `protobuf:"bytes,8,opt,name=data,proto3" json:"data,omitempty"`
 Accesses AccessList `protobuf:"bytes,9,rep,name=accesses,proto3,castrepeated=AccessList" json:"accessList"`
 // v defines the signature value
 V []byte `protobuf:"bytes,10,opt,name=v,proto3" json:"v,omitempty"`
 // r defines the signature value
 R []byte `protobuf:"bytes,11,opt,name=r,proto3" json:"r,omitempty"`
 // s define the signature value
 S []byte `protobuf:"bytes,12,opt,name=s,proto3" json:"s,omitempty"`
}
```

This message field validation is expected to fail if:

- `GasTipCap` is invalid (`nil` , negative or overflows int256)
- `GasFeeCap` is invalid (`nil` , negative or overflows int256)
- `GasFeeCap` is less than `GasTipCap`
- `Fee` (gas price * gas limit) is invalid (overflows int256)
- `Amount` is invalid (negative or overflows int256)
- `To` address is invalid (non-valid ethereum hex address)
- `ChainID` is `nil`

#### AccessListTx

The transaction data of EIP-2930 access list transactions.

```go
type AccessListTx struct {
 // destination EVM chain ID
 ChainID *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,1,opt,name=chain_id,json=chainId,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"chainID"`
 // nonce corresponds to the account nonce (transaction sequence).
 Nonce uint64 `protobuf:"varint,2,opt,name=nonce,proto3" json:"nonce,omitempty"`
 // gas price defines the value for each gas unit
 GasPrice *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,3,opt,name=gas_price,json=gasPrice,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_price,omitempty"`
 // gas defines the gas limit defined for the transaction.
 GasLimit uint64 `protobuf:"varint,4,opt,name=gas,proto3" json:"gas,omitempty"`
 // hex formatted address of the recipient
 To string `protobuf:"bytes,5,opt,name=to,proto3" json:"to,omitempty"`
 // value defines the unsigned integer value of the transaction amount.
 Amount *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,6,opt,name=value,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"value,omitempty"`
 // input defines the data payload bytes of the transaction.
 Data     []byte     `protobuf:"bytes,7,opt,name=data,proto3" json:"data,omitempty"`
 Accesses AccessList `protobuf:"bytes,8,rep,name=accesses,proto3,castrepeated=AccessList" json:"accessList"`
 // v defines the signature value
 V []byte `protobuf:"bytes,9,opt,name=v,proto3" json:"v,omitempty"`
 // r defines the signature value
 R []byte `protobuf:"bytes,10,opt,name=r,proto3" json:"r,omitempty"`
 // s define the signature value
 S []byte `protobuf:"bytes,11,opt,name=s,proto3" json:"s,omitempty"`
}
```

This message field validation is expected to fail if:

- `GasPrice` is invalid (`nil` , negative or overflows int256)
- `Fee` (gas price * gas limit) is invalid (overflows int256)
- `Amount` is invalid (negative or overflows int256)
- `To` address is invalid (non-valid ethereum hex address)
- `ChainID` is `nil`

## Transitions

### InitGenesis

`InitGenesis` initializes the EVM module genesis state by setting the `GenesisState` fields to the store. In particular it sets the parameters and genesis accounts (state and code).

### ExportGenesis

The `ExportGenesis` ABCI function exports the genesis state of the EVM module. In particular, it retrieves all the accounts with their bytecode, balance and storage, the transaction logs, and the EVM parameters and chain configuration.

### BeginBlock

The EVM module `BeginBlock` logic is executed prior to handling the state transitions from the transactions. The main objective of this function is to:

- Set the context for the current block so that the block header, store, gas meter, etc are available to the `Keeper` once one of the `StateDB` functions are called during EVM state transitions.
- Set the EIP155 `ChainID` number (obtained from the full chain-id), in case it hasn't been set before during `InitChain`

### EndBlock

The EVM module `EndBlock` logic occurs after executing all the state transitions from the transactions. The main objective of this function is to:

- Emit Block bloom events
    - This is due for Web3 compatibility as the Ethereum headers contain this type as a field. The JSON-RPC service uses this event query to construct an Ethereum Header from a Tendermint Header.
    - The block Bloom filter value is obtained from the Transient Store and then emitted

### Hooks

The `x/evm` module implements an `EvmHooks` interface that extend and customize the `Tx` processing logic externally.

This supports EVM contracts to call native cosmos modules by

1. defining a log signature and emitting the specific log from the smart contract,
2. recognizing those logs in the native tx processing code, and
3. converting them to native module calls.

To do this, the interface includes a  `PostTxProcessing` hook that registers custom `Tx` hooks in the `EvmKeeper`. These  `Tx` hooks are processed after the EVM state transition is finalized and doesn't fail. Note that there are no default hooks implemented in the EVM module.

```go
type EvmHooks interface {
 // Must be called after tx is processed successfully, if return an error, the whole transaction is reverted.
 PostTxProcessing(ctx sdk.Context, msg core.Message, receipt *ethtypes.Receipt) error
}
```

#### PostTxProcessing

 `PostTxProcessing` is only called after a EVM transaction finished successfully and delegates the call to underlying hooks.  If no hook has been registered, this function returns with a `nil` error.

```go
func (k *Keeper) PostTxProcessing(ctx sdk.Context, msg core.Message, receipt *ethtypes.Receipt) error {
 if k.hooks == nil {
  return nil
 }
 return k.hooks.PostTxProcessing(k.Ctx(), msg, receipt)
}
```

It's executed in the same cache context as the EVM transaction, if it returns an error, the whole EVM transaction is reverted, if the hook implementor doesn't want to revert the tx, they can always return `nil` instead.

The error returned by the hooks is translated to a VM error `failed to process native logs`, the detailed error message is stored in the return value. The message is sent to native modules asynchronously, there's no way for the caller to catch and recover the error.

## Parameters

The evm module contains the following parameters:

### Params

```go
// Params defines the EVM module parameters
type Params struct {
	// evm_denom represents the token denomination used to run the EVM state
	// transitions.
	EvmDenom string `protobuf:"bytes,1,opt,name=evm_denom,json=evmDenom,proto3" json:"evm_denom,omitempty" yaml:"evm_denom"`
	// enable_create toggles state transitions that use the vm.Create function
	EnableCreate bool `protobuf:"varint,2,opt,name=enable_create,json=enableCreate,proto3" json:"enable_create,omitempty" yaml:"enable_create"`
	// enable_call toggles state transitions that use the vm.Call function
	EnableCall bool `protobuf:"varint,3,opt,name=enable_call,json=enableCall,proto3" json:"enable_call,omitempty" yaml:"enable_call"`
	// extra_eips defines the additional EIPs for the vm.Config
	ExtraEIPs []int64 `protobuf:"varint,4,rep,packed,name=extra_eips,json=extraEips,proto3" json:"extra_eips,omitempty" yaml:"extra_eips"`
	// chain_config defines the EVM chain configuration parameters
	ChainConfig ChainConfig `protobuf:"bytes,5,opt,name=chain_config,json=chainConfig,proto3" json:"chain_config" yaml:"chain_config"`
	// allow_unprotected_txs defines if replay-protected (i.e non EIP155
	// signed) transactions can be executed on the state machine.
	AllowUnprotectedTxs bool `protobuf:"varint,6,opt,name=allow_unprotected_txs,json=allowUnprotectedTxs,proto3" json:"allow_unprotected_txs,omitempty"`
}
```

### EVM denom

The evm denomination parameter defines the token denomination used on the EVM state transitions and gas consumption for EVM messages.

For example, on Ethereum, the `evm_denom` would be `Xpla`. In the case of Ethermint, the default denomination is the **atto photon**(used on the default denom). In terms of precision, the `PHOTON` and `Xpla` share the same value, *i.e* `1 Xpla = 10^18 axpla` and `1 ETH = 10^18 wei`.

{{< alert >}}
Note: SDK applications that want to import the EVM module as a dependency will need to set their own `evm_denom` (i.e not `"aphoton"`).
{{< /alert >}}

### Enable Create

The enable create parameter toggles state transitions that use the `vm.Create` function. When the parameter is disabled, it will prevent all contract creation functionality.

### Enable Transfer

The enable transfer toggles state transitions that use the `vm.Call` function. When the parameter is disabled, it will prevent transfers between accounts and executing a smart contract call.

### Extra EIPs

The extra EIPs parameter defines the set of activateable Ethereum Improvement Proposals (**[EIPs](https://ethereum.org/en/eips/)**)
on the Ethereum VM `Config` that apply custom jump tables.

{{< alert >}}
NOTE: some of these EIPs are already enabled by the chain configuration, depending on the hard fork number.
{{< /alert >}}

The supported activateable EIPS are:

- **[EIP 1344](https://eips.ethereum.org/EIPS/eip-1344)**
- **[EIP 1884](https://eips.ethereum.org/EIPS/eip-1884)**
- **[EIP 2200](https://eips.ethereum.org/EIPS/eip-2200)**
- **[EIP 2315](https://eips.ethereum.org/EIPS/eip-2315)**
- **[EIP 2929](https://eips.ethereum.org/EIPS/eip-2929)**
- **[EIP 3198](https://eips.ethereum.org/EIPS/eip-3198)**
- **[EIP 3529](https://eips.ethereum.org/EIPS/eip-3529)**

### Chain Config

The `ChainConfig` is a protobuf wrapper type that contains the same fields as the go-ethereum `ChainConfig` parameters, but using `*sdk.Int` types instead of `*big.Int`.

By default, all block configuration fields but `ConstantinopleBlock`, are enabled at genesis (height 0).

```go
// ChainConfig defines the Ethereum ChainConfig parameters using *sdk.Int values
// instead of *big.Int.
type ChainConfig struct {
	// homestead_block switch (nil no fork, 0 = already homestead)
	HomesteadBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,1,opt,name=homestead_block,json=homesteadBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"homestead_block,omitempty" yaml:"homestead_block"`
	// dao_fork_block corresponds to TheDAO hard-fork switch block (nil no fork)
	DAOForkBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,2,opt,name=dao_fork_block,json=daoForkBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"dao_fork_block,omitempty" yaml:"dao_fork_block"`
	// dao_fork_support defines whether the nodes supports or opposes the DAO hard-fork
	DAOForkSupport bool `protobuf:"varint,3,opt,name=dao_fork_support,json=daoForkSupport,proto3" json:"dao_fork_support,omitempty" yaml:"dao_fork_support"`
	// eip150_block: EIP150 implements the Gas price changes
	// (https://github.com/ethereum/EIPs/issues/150) EIP150 HF block (nil no fork)
	EIP150Block *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,4,opt,name=eip150_block,json=eip150Block,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"eip150_block,omitempty" yaml:"eip150_block"`
	// eip150_hash: EIP150 HF hash (needed for header only clients as only gas pricing changed)
	EIP150Hash string `protobuf:"bytes,5,opt,name=eip150_hash,json=eip150Hash,proto3" json:"eip150_hash,omitempty" yaml:"byzantium_block"`
	// eip155_block: EIP155Block HF block
	EIP155Block *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,6,opt,name=eip155_block,json=eip155Block,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"eip155_block,omitempty" yaml:"eip155_block"`
	// eip158_block: EIP158 HF block
	EIP158Block *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,7,opt,name=eip158_block,json=eip158Block,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"eip158_block,omitempty" yaml:"eip158_block"`
	// byzantium_block: Byzantium switch block (nil no fork, 0 = already on byzantium)
	ByzantiumBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,8,opt,name=byzantium_block,json=byzantiumBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"byzantium_block,omitempty" yaml:"byzantium_block"`
	// constantinople_block: Constantinople switch block (nil no fork, 0 = already activated)
	ConstantinopleBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,9,opt,name=constantinople_block,json=constantinopleBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"constantinople_block,omitempty" yaml:"constantinople_block"`
	// petersburg_block: Petersburg switch block (nil same as Constantinople)
	PetersburgBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,10,opt,name=petersburg_block,json=petersburgBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"petersburg_block,omitempty" yaml:"petersburg_block"`
	// istanbul_block: Istanbul switch block (nil no fork, 0 = already on istanbul)
	IstanbulBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,11,opt,name=istanbul_block,json=istanbulBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"istanbul_block,omitempty" yaml:"istanbul_block"`
	// muir_glacier_block: Eip-2384 (bomb delay) switch block (nil no fork, 0 = already activated)
	MuirGlacierBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,12,opt,name=muir_glacier_block,json=muirGlacierBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"muir_glacier_block,omitempty" yaml:"muir_glacier_block"`
	// berlin_block: Berlin switch block (nil = no fork, 0 = already on berlin)
	BerlinBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,13,opt,name=berlin_block,json=berlinBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"berlin_block,omitempty" yaml:"berlin_block"`
	// london_block: London switch block (nil = no fork, 0 = already on london)
	LondonBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,17,opt,name=london_block,json=londonBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"london_block,omitempty" yaml:"london_block"`
	// arrow_glacier_block: Eip-4345 (bomb delay) switch block (nil = no fork, 0 = already activated)
	ArrowGlacierBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,18,opt,name=arrow_glacier_block,json=arrowGlacierBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"arrow_glacier_block,omitempty" yaml:"arrow_glacier_block"`
	// gray_glacier_block: EIP-5133 (bomb delay) switch block (nil = no fork, 0 = already activated)
	GrayGlacierBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,20,opt,name=gray_glacier_block,json=grayGlacierBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gray_glacier_block,omitempty" yaml:"gray_glacier_block"`
	// merge_netsplit_block: Virtual fork after The Merge to use as a network splitter
	MergeNetsplitBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,21,opt,name=merge_netsplit_block,json=mergeNetsplitBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"merge_netsplit_block,omitempty" yaml:"merge_netsplit_block"`
	// Shanghai switch block (nil = no fork, 0 = already on shanghai)
	ShanghaiBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,22,opt,name=shanghai_block,json=shanghaiBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"shanghai_block,omitempty" yaml:"shanghai_block"`
	// Cancun switch block (nil = no fork, 0 = already on cancun)
	CancunBlock *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,23,opt,name=cancun_block,json=cancunBlock,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"cancun_block,omitempty" yaml:"cancun_block"`
}
```

#### HomesteadBlock

- type: `sdk.Int`
- `"homestead_block": "0"`

#### DAOForkBlock

- type: `sdk.Int`
- `"dao_fork_block": "0"`

#### DAOForkSupport

- type : `bool`
- `"dao_fork_support": true`

#### EIP150Block

- type : `sdk.Int`
- `"eip150_block": "0"`

#### EIP150Hash

- type : `string`
- `"eip150_hash": "0x0000000000000000000000000000000000000000000000000000000000000000"`

#### EIP155Block

- type : `sdk.Int`
- `"eip155_block": "0"`
#### EIP158Block

- type : `sdk.Int`
- `"eip158_block": "0"`
#### ByzantiumBlock

- type : `sdk.Int`
- `"byzantium_block": "0"`
#### ConstantinopleBlock

- type : `sdk.Int`
- `"constantinople_block": "0"`
#### PetersburgBlock

- type : `sdk.Int`
- `"petersburg_block": "0"`
#### IstanbulBlock

- type : `sdk.Int`
- `"istanbul_block": "0"`
#### MuirGlacierBlock

- type : `sdk.Int`
- `"muir_glacier_block": "0"`
#### BerlinBlock

- type : `sdk.Int`
- `"berlin_block": "0"`
#### LondonBlock

- type : `sdk.Int`
- `"london_block": "0"`

#### ArrowGlacierBlock

- type : `sdk.Int`
- `"arrow_glacier_block": "0"`

#### GrayGlacierBlock

- type : `sdk.Int`
- `"gray_glacier_block": "0"`

#### MergeNetsplitBlock

- type : `sdk.Int`
- `"merge_netsplit_block": "0"`

#### ShanghaiBlock

- type : `sdk.Int`
- `"shanghai_block": "0"`

#### CancunBlock

- type : `sdk.Int`
- `"cancun_block": "0"`