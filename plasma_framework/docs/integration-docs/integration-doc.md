# Introduction
This document describes how to interact with the Plasma Abstract Layer Design (ALD) framework, and provides details about the components relevant to an integration, including:

- PlasmaFramework contract

  The entry point contract. Any integration with the system will interact with this contract.

- Block submission

  The child chain integration requires the data submitted in the blocks.

- Transactions

  Transactions are a fundamental part of the system. Any integration will require transaction-related information.

> ***Note**: For more information on the concepts relevant to this process, see the following documentation:*
- *[High level design of the Plasma Abstract Layer](https://docs.google.com/document/d/1PSxLnMskjqje4MksmW2msSSg-GtZoBSMNYey9nEDvt8)*
- *[Tesuji Plasma Blockchain Design](https://github.com/omisego/elixir-omg/blob/master/docs/tesuji_blockchain_design.md)*
- *[Solidity contract documentation](https://github.com/omisego/plasma-contracts/blob/master/plasma_framework/docs/PlasmaFramework.md)*


# PlasmaFramework
The PlasmaFramework contract may be viewed as the top-level contract. It contains the following components (described in more detail in this document):

- BlockController
- ExitGameController
- ExitGameRegistry
- VaultRegistry


The PlasmaFramework contract provides access to components in the system. For example, to get the Payment ExitGame, call the following:

 `PlasmaFramework.exitGames(txType)`

The PlasmaFramework also provides the means for the `maintainer` to upgrade certain components in the system. Since this functionality has important security considerations, the PlasmaFramework emits events whenever a component is added. The Watchers monitor these events and alert users. More information is provided below.


# Block submission
Only the operator can submit blocks. The data submitted is the root of the Merkle tree containing all the transactions in the block. To submit a block, call the following:

```
PlasmaFramework.submitBlock(blockRoot);
```

On success, the PlasmaFramework contract emits a `BlockSubmitted` event:

```
event BlockSubmitted(
    uint256 blknum
);
```

 > ***Note**: For more information about parameter types, return types, valid values, and so on, see the [Plasma Contract API documentation](../contracts/BlockController.md#submitblock).*

# Block retrieval
To learn what blocks need to be retrieved during sync, read `PlasmaFramework.nextChildBlock()` and `PlasmaFramework.childBlockInterval()`. Compute the block numbers by iterating every `childBlockInterval` from `0 + childBlockInterval` up to `nextChildBlock - childBlockInterval`, inclusive. Next, read the Merkle tree root of each block by calling `PlasmaFramework.blocks(blknum)`.

# Transactions
Transactions are composed of inputs and outputs. An input is simply a pointer to the output of another transaction.

Transactions included in a block have a position, which is the number of the block it's in, and its index in that block. For example, the fourth transaction in block number 5000 has this position: `(5000, 3)`

To determine the position of the outputs of a transaction, you will include the index of the output in the transaction. So, the position of the second output of the transaction, in the example above, would be: `(5000, 3, 1)`



## Transaction type, output type, and spending condition

ALD introduces the concepts of transaction type and transaction output type.
Each transaction output type can define its own rules about how funds may be spent.
Output type white-lists transaction type and defines how exactly it can spend.
Each output type can white-list many transaction types.

For example, consider an output tx type that represents client funds being held by a venue.
This output type can limit the transaction types that can spend it to either a "withdrawal" transaction that returns the funds to the client, or a "trade settlement" transaction that contains information about orders signed by clients.
In this way the output tx type prevents spending the funds via a "payment" transaction - ensuring that the venue can't pay its bills using the clients' money.

Rules are encoded in the [mapping](../contracts/src/exits/registries/SpendingConditionRegistry.sol) from `hash(tx_type, output_type)` to an address.
If address is 0x0, spending is forbidden.
Otherwise, the address is that of a contract that implements the [ISpendingCondition](../contracts/src/exits/interfaces/ISpendingCondition.sol) interface and its `verify(...) returns (bool)` function. An example is in [PaymentOutputToPaymentTxCondition](../contracts/src/exits/payment/spendingConditions/PaymentOutputToPaymentTxCondition.sol).

For a valid transaction, `verify(...)` will return `true` for each of the transaction's inputs.

## Transaction balance validation

Plasma transactions must not mint tokens. The sum of inputs must not be smaller than the sum of outputs. ALD defines `IStateTransitionVerifier.verify(...) returns (bool)` [interface](../contracts/src/exits/interfaces/IStateTransitionVerifier.sol) where such checks must be implemented. An [example](../contracts/src/exits/payment/PaymentTransactionStateTransitionVerifier.sol) is in `PaymentTransactionStateTransitionVerifier`.

## Generic transaction format
All Transactions follow the same basic format [GenericTransaction](../contracts/GenericTransaction.md) and can be extended.
GenericTransaction is based on [Wire Transaction format](https://docs.google.com/document/d/1ETAO5ZUO7S_A8sXUK5cyAN6yMMotRDbJphAa2hPJIyU/edit) but has diverged somewhat from the original design.

A GenericTransaction is:

```
transaction::= txType [input] [output] txData metaData
```

Where
```
txType ::= uint256
input ::= outputId | outputPosition
outputId ::= hash of the transaction that produced the output concatenated with the outputIndex
outputPosition ::= 32 byte string that is (blknum * BLOCK_OFFSET + txIndex * TX_OFFSET + outputIndex)
output ::= outputType outputData
outputType ::= uint256
outputData ::= undefined, to be defined by concrete transaction types
txData ::= undefined, to be defined by concrete transaction types
metaData ::= bytes32
```

GenericTransaction [performs](../contracts/src/transactions/GenericTransaction.sol#L42) a range of checks when decoding:

* Is the binary a proper RLP encoding?
* Is the decoded list of required length? See `TX_NUM_ITEMS`
* Is txType an integer? Is it `> 0`?
* Is each of the outputs correctly formed? See `decodeOutput(...)`

The current implementation supports only the `Payment` transaction type.

Support for additional transaction types, such as ERC721, is reserved for future development.

### Signed transaction format (child chain and watcher only)
A note on encoding of signed transactions (transactions including witnesses) when handled in the [child chain and watcher implementations](github.com/omisego/elixir-omg).

Signed transactions are:

```
signedTransaction ::= [signature] rawTransaction
```

Where
```
signature ::= bytes
rawTransaction ::= transaction
```

and are transferred and stored RLP-encoded.

Child chain and watcher both expect, for a signed transaction to be valid, that:
 - every input has a corresponding signature by the respective output's owner (`outputGuard`)
 - every signature is a 65-byte long binary

**NOTE** Only signature witnesses are operative now, because only payment transactions are supported.
A signature is a specific form of a `witness` and is called as such throughout this document.

## Payment transaction format
Payment transactions are used to transfer fungible tokens, such as ETH and ERC20 tokens. A Payment transaction's output is as described in [FungibleTokenOutputModel](../contracts/FungibleTokenOutputModel.md).

[PaymentTransactionModel](../contracts/PaymentTransactionModel.md) does few more checks when decoding comparing to GenericTransaction above:

* Is the number of inputs too large? See `_MAX_INPUT_NUM`.
* Is the number outputs in the range `(0, _MAX_OUTPUT_NUM]`?
* None of the input pointers is null.

### Payment transaction RLP encoding
A Payment transaction must be RLP encoded as follows:
```
[txType, inputs, outputs, txData, metaData]

txType ::= uint256
inputs ::= [input]
input ::= bytes32
outputs ::= [output]
output ::= [outputType, outputData]
outputType ::= uint256
outputData ::= [outputGuard, token, amount]
outputGuard ::= bytes20
token ::= bytes20
amount ::= uint256
txData ::= uint256 (must be 0)
metadata ::= bytes32
```

Example transaction with two inputs and two outputs:
```
[
    1,
    [
        "0x0000000000000000000011111111111111100001",
        "0x0000000000000000000011111111111111100002",
    ],
    [
        [
            1,
            [
                "0xc5fdf4076b8f3a5357c5e395ab970b5b54098fef",
                "0x0000000000000000000000000000000000000000",
                100
            ]
        ],
        [
            1,
            [
                "0xc5fdf4076b8f3a5357c5e395ab970b5b54098fef",
                "0x0000000000000000000000000000000000000000",
                900
            ]
        ]
    ],
    0,
    "0x0000000000000000000000000000000000000000000000000000000000000000"
]
```
Note that:

1. `txType` must not be `0`
2. `inputs` must be a list
3. `input` must be padded to 32 bytes long
4. A null `input` (`"0x0...000"`) is invalid
5. `outputs` must be a list
6. `outputType` must not be `0`
7. `outputData` must be a list containing 3 items
8. `outputData.outputGuard` must be 20 bytes long
9. A null `outputData.outputGuard` (`"0x0...000"`) is invalid
10. An `outputData.amount` of `0` is invalid
11. `txData` is unused and must be set to `0`
12. `metadata` must be padded to 32 bytes long

### Payment transaction fees
Operator collects fees for including a transaction in a block. The transaction fee is implicit i.e. it is the difference between the transaction inputs and outputs, grouped by the token. Operator requires fee to be covered completely with a single currency, but not necessarily by a single input.

For example if the transaction inputs are as follows:
```
[
  {
    'outputType': 1,
    'outputGuard': '...',
    'token': '0x000000000000000000000000000000000000000a',
    'amount': 2
  },
  {
    'outputType': 1,
    'outputGuard': '...',
    'token': '0x000000000000000000000000000000000000000b',
    'amount': 1
  }
]
```
and the only transaction output is:
```
{
'outputType': 1,
'outputGuard': '...',
'token': '0x000000000000000000000000000000000000000a',
'amount': 1
}
```
and operator requires to pay 1 of token '0x000000000000000000000000000000000000000a` to include a transaction in a block, then the fee collected by the operator is:
```
1 of token '0x000000000000000000000000000000000000000a' (amount 2 in transaction inputs, 1 in outputs)
1 of token '0x000000000000000000000000000000000000000b' (amount 1 in transaction inputs, 0 in outputs)
```
Notice that although fees were covered completely by the first input, the second input was also charged as a fee.

Specification for sending a transaction to the child chain is available [here](https://developer.omisego.co/elixir-omg/docs-ui/?url=master%2Foperator_api_specs.yaml&urls.primaryName=master%2Foperator_api_specs#/ChildChain/submit).

When transaction is submitted with insufficient fees error code in the response is `transaction.submit:fees_not_covered`.

## Deposit transactions
Deposit transactions are special transactions that have no inputs. The transaction inputs should be encoded as an empty array. Deposit transactions are created by the vault contracts, and do not need to be explicitly submitted.


## Payment transction EIP-712 signing
The **witness** field of a transaction contains the data that proves its inputs can be spent. For a standard payment transaction, this data is the signatures of the owners of the inputs. Transactions are signed using the [EIP-712](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md) method.

The EIP-712 typed data structure is as follows:
 > ***Important**! This is likely to change in future.*

```
{
  types: {
    EIP712Domain: [
        { name: 'name', type: 'string' },
        { name: 'version', type: 'string' },
        { name: 'verifyingContract', type: 'address' },
        { name: 'salt', type: 'bytes32' }
    ],
    Transaction: [
        { name: 'txType', type: 'uint256' },
        { name: 'input0', type: 'Input' },
        { name: 'input1', type: 'Input' },
        { name: 'input2', type: 'Input' },
        { name: 'input3', type: 'Input' },
        { name: 'output0', type: 'Output' },
        { name: 'output1', type: 'Output' },
        { name: 'output2', type: 'Output' },
        { name: 'output3', type: 'Output' },
        { name: 'txData', type: 'uint256' },
        { name: 'metadata', type: 'bytes32' }
    ],
    Input: [
        { name: 'blknum', type: 'uint256' },
        { name: 'txindex', type: 'uint256' },
        { name: 'oindex', type: 'uint256' }
    ],
    Output: [
        { name: 'outputType', type: 'uint256' },
        { name: 'outputGuard', type: 'bytes20' },
        { name: 'currency', type: 'address' },
        { name: 'amount', type: 'uint256' }
    ]
  },
  domain: {
        name: 'OMG Network',
        version: '1',
        verifyingContract: '',
        salt: '0xfad5c7f626d80f9256ef01929f3beb96e058b8b4b0e3fe52d84f054c0e2a7a83'
    },
  primaryType: 'Transaction'
}
```

# Vaults
Vaults are used to deposit funds and, indirectly, to withdraw funds via the exit game.

A deposit to Plasma Framework means moving funds from the Root chain (Ethereum) to the Child chain (or Plasma chain). A withdrawal means the opposite, moving funds from the Child chain back to the Root chain. A withdrawal must only be indirectly initiated by a canonical exit.

## Vault contract
A vault is a contract that holds custody of tokens transferred to the Plasma Framework. Each vault has a unique `vaultId` as an identifier used in the Plasma Framework. You can retrieve a vault's contract address from the  `PlasmaFramework`:

`PlasmaFramework.vaults(vaultId)`

### Vault Id
These are the `vaultId`s of the currently deployed vault contracts:
- ETH vault: 1
- ERC20 vault: 2

## Depositing funds
You must use the appropriate vault to deposit funds from the root chain (Ethereum) into the child chain. Check the [vault contract](#vault-contract) section above to see how to get the vault address.

### Depositing ETH
1. Create the RLP-encoded deposit transaction: `depositTx`
2. Call `EthVault.deposit(depositTx)`
3. Along with the transaction, send the amount of ETH specified in the deposit transaction.
4. The ETHVault creates a deposit block, and submits it to the PlasmaFramework.
5. The ETHVault emits the `DepositCreated` event.
6. The child chain receives the `DepositCreated`, and creates the corresponding UTXO.
7. The UTXO is spendable by the user after a certain number of blocks are submitted to the PlasmaFramework (specified in `deposit_finality_margin`).



### Depositing ERC20 tokens

1. To transfer the amount of tokens you wish to deposit, approve the ERC20Vault contract.
2. Create the RLP-encoded deposit transaction: `depositTx`
3. Call `ERC20Vault.deposit(depositTx)`.
4. The ERCVault calls `ERC20.transferFrom()` to transfer the tokens to itself.
5. The ERC20Vault creates a deposit block, and submits it to the PlasmaFramework.
6. The ERC20Vault emits the `DepositCreated` event.
7. The child chain receives the `DepositCreated`, and creates the corresponding UTXO.
8. The UTXO is spendable by the user after a certain number of blocks are submitted to the PlasmaFramework (specified in `deposit_finality_margin`).



## Vault events
Vaults emit events on deposit, and on withdrawal. Each event is from the specific vault contract.

### ETH Vault

```
    event DepositCreated(
        address indexed depositor,
        uint256 indexed blknum,
        address indexed token,
        uint256 amount
    );

    event EthWithdrawn(
        address indexed receiver,
        uint256  amount
    );
```

For all events of the vault, see contract documentation: [ETH vault doc](https://github.com/omisego/plasma-contracts/blob/master/plasma_framework/docs/contracts/EthVault.md#contract-members)


### ERC20 Vault
```
    event DepositCreated(
        address indexed depositor,
        uint256 indexed blknum,
        address indexed token,
        uint256 amount
    );

    event Erc20Withdrawn(
        address payable indexed receiver,
        address indexed token,
        uint256 amount
    );
```

For all events of the vault, see contract documentation: [ERC20 vault doc](https://github.com/omisego/plasma-contracts/blob/master/plasma_framework/docs/contracts/Erc20Vault.md#erc20vaultsol)

# Exit Game
Exit Games handle all the actions around exits, challenges, etc. Each transaction type has one and only one exit game contract that provides the means of exiting funds from a transaction of that type.

Once an exit is canonical to exit after the challenge period, it can be processed according to the exit priority and withdraws the funds back to the Root chain from the Child chain.

## Exit Game contracts
An exit game is a contract that holds the logic and states of a current exit game status. Each exit game is bound to a `txType` and uses it as an identifier in the Plasma Framework. You can retrieve an exit game's contract address from the `PlasmaFramework`:

`PlasmaFramework.exitGames(txType)`

### Transaction types
The following are the currently deployed exit game contracts, mapped to `txType`:
- Payment (v1): 1
- Fee: 3

Transaction type 2 is reserved for Payment v2, although it does not have a specific exit game contract registered in the framework yet. For more detail on why we are reserving a transaction type, please see the "Adding new transaction type" section of the [high level design doc](https://docs.google.com/document/d/1PSxLnMskjqje4MksmW2msSSg-GtZoBSMNYey9nEDvt8/edit#heading=h.yu9haziuwf6b)

### Payment Exit Game (V1)
This contract is the main exit game contract in the first Plasma Framework iteration. It handles the exit game logic for payment transactions, including both standard exit and in-flight exit.

Most of the following documentation in the Exit Game section refers to the `PaymentExitGame`.

### Fee Exit Game

The fee is designed to be exited indirectly from the fee transaction. Operator must first spend the fee transaction output to a payment transaction and then exit the fee value using the normal process of exiting a payment transaction.

As a result, the fee exit game contract is just an empty contract.


## Payment Exit game bonds
Exit games are associated with various bonds. The values of these bonds may change over time. The current value of a bond can be retrieved from the `PaymentExitGame` contract.

To get the `PaymentExitGame` contract (js):
```
address = PlasmaFramework.exitGames(1)
PaymentExitGame = PaymetExitGame.at(address);
```


### Standard exit bond
A standard exit bond is used to start a standard exit:

```
    PaymentExitGame.startStandardExitBondSize()
```

### In-flight exit bonds

There are two types of in-flight exit bonds:

- In-flight exit bond for starting an in-flight exit
```
    PaymentExitGame.startIFEBondSize()
```
- In-flight exit bond for piggybacking on an in-flight exit's input or output:
```
    PaymentExitGame.piggybackBondSize()
```


## Playing the payment exit game

### Starting a standard exit

To start a standard exit, perform these steps:

1. To exit a UTXO you need to retrieve the corresponding transaction type. Currently, the platform supports only payment transactions, for which the ID is 1.

2. Now retrieve the exit game contract address from the PlasmaFramework contract:

```
address = PlasmaFramework.exitGames(1)
PaymentExitGame = PaymetExitGame.at(address);
```

3. To start a standard exit, send the appropriate amount of ETH to cover the bond. To retrieve the actual amount, call the following function:

```
PaymentExitGame.startStandardExitBondSize()
```

```
PaymentExitGame.startStandardExit({
  utxoPos,
  rlpOutputTx,
  outputTxInclusionProof,
})
```

### Parameters
This section describes the parameters in the function for starting a standard exit.

#### utxoPos (uint256)
The position of the exiting output. The formula is as follows:

```
block number * the block offset (defaults: `1000000000`) + transaction position * transaction offset (defaults to `10000`) + the index of the UTXO in the list of outputs of the transaction
```

For example, if we have a deposit transaction in block 160000 at index 0, and we want the utxoPos of the output at index 0 (deposit transactions only have one output, and no inputs):

```
160000 * 10000 + 0 * 10000 + 0 = 1600000000
```

#### rlpOutputTx (bytes)
The RLP-encoded transaction that creates the exiting output.

This example is a deposit transaction of 1,000,000,000,000,000 Wei, sent from address `0xa013debd703e28af78c2ffd0264ef70f978c5465`, to the `EthVault` contract:

```
[
  1, # Transaction type, 1 for Payment
  [], # inputs, empty for deposit transactions
  [ # list of outputs, only one for deposit transactions
    [
      1, # transaction output type, 1 for payment transaction output
      # owner of the output, for deposit txs, owner = sender
      "0xa013debd703e28af78c2ffd0264ef70f978c5465",

      # The currency, 0x0000000000000000000000000000000000000000 = ether
      "0x0000000000000000000000000000000000000000",

      # The amount to deposit
      1000000000000000
    ]
  ],
  # metadata, in this case, no metadata are sent
  "0x0000000000000000000000000000000000000000000000000000000000000000"
]
```

This transaction must be RLP-encoded, using a library of your choice. Before RLP-encoding, ensure you decode the owner address, currency and metadata, from hexadecimal to bytes. Once the transaction is RLP-encoded, encode the result to hexadecimal.

#### outputTxInclusionProof (bytes)
A Merkle proof showing that the transaction was included. This Merkle proof, which is used to prove the inclusion of a specific hash in a Merkle tree, is a string containing each sibling hash for each level of the Merkle tree, concatenated together.

 > ***Note**: To learn more about Merkle tree implementation, see [Basic Merkle Tree understanding](https://en.wikipedia.org/wiki/Merkle_tree)
[Implementation example](https://github.com/omisego/plasma-contracts/blob/master/plasma_framework/test/helpers/merkle.js)*

### Troubleshooting
If starting the standard exit fails with an error that refers to a missing queue, use the following function to check that the exit queue is registered for the given vault ID and token:


```
PlasmaFramework.hasExitQueue(vaultId, tokenAddress)
```

If no exit queue is registered, you’ll need to register it using the following function:
```
PlasmaFramework.addExitQueue(vaultId, tokenAddress)
```

#### Example:

```
PaymentExitGame.startStandardExit([
  1600000000,
  0xf85801c0f4f3019441777dc7bdcc6b58be1c25eb3df7df52d1bfecbd94000000000000000000000000000000000000000087038d7ea4c68000a00000000000000000000000000000000000000000000000000000000000000000,
  0x290decd9548b62a8d60345a988386fc84ba6bc95484008f6362f93160ef3e563633dc4d7da7256660a892f8f1604a44b5432649cc8ec5cb3ced4c4e6ac94dd1d890740a8eb06ce9be422cb8da5cdafc2b58c0a5e24036c578de2a433c828ff7d3b8ec09e026fdc305365dfc94e189a81b38c7597b3d941c279f042e8206e0bd8ecd50eee38e386bd62be9bedb990706951b65fe053bd9d8a521af753d139e2dadefff6d330bb5403f63b14f33b578274160de3a50df4efecf0e0db73bcdd3da5617bdd11f7c0a11f49db22f629387a12da7596f9d1704d7465177c63d88ec7d7292c23a9aa1d8bea7e2435e555a4a60e379a5a35f3f452bae60121073fb6eeade1cea92ed99acdcb045a6726b2f87107e8a61620a232cf4d7d5b5766b3952e107ad66c0a68c72cb89e4fb4303841966e4062a76ab97451e3b9fb526a5ceb7f82e026cc5a4aed3c22a58cbd3d2ac754c9352c5436f638042dca99034e836365163d04cffd8b46a874edf5cfae63077de85f849a660426697b06a829c70dd1409cad676aa337a485e4728a0b240d92b3ef7b3c372d06d189322bfd5f61f1e7203ea2fca4a49658f9fab7aa63289c91b7c7b6c832a6d0e69334ff5b0a3483d09dab4ebfd9cd7bca2505f7bef59cc1c12ecc708fff26ae4af19abe852afe9e20c8622def10d13dd169f550f578bda343d9717a138562e0093b380a1120789d53cf10,
])
```

### Challenging a standard exit
To challenge an exit, retrieve payment exit game as in [step 2 for starting a standard exit](#starting-a-standard-exit).

Then obtain `exitId` (if you don’t yet have it):

```
PaymentExitGame.getStandardExitId(
  true, # true if the exiting output is from a deposit transaction, false otherwise
  "0xf85801c0f4f3019441777dc7bdcc6b58be1c25eb3df7df52d1bfecbd94000000000000000000000000000000000000000087038d7ea4c68000a00000000000000000000000000000000000000000000000000000000000000000", # RLP-encoded transaction sent when startStandardExit was called
  1600000000, # utxoPos
)
```

Once you have the exit game and `exitId`, call `challengeStandardExit`:

```
PaymentExitGame.challengeStandardExit({
  exitId,
  exitingTx,
  challengeTx,
  inputIndex,
  witness,
  senderData
});
```

### Parameters
This section describes arguments provided to `PaymentExitGame.challengeStandardExit`

#### exitId (uint160)
Identifier of the exit to challenge.

#### exitingTx (bytes)
RLP-encoded transaction that created the exiting output.

#### challengeTx (bytes)
RLP-encoded transaction that spends the exiting output (challenging transaction).

#### inputIndex (uint16)
Input index of exiting UTXO in the challenging transaction.

#### witness (bytes)
Data proving that exiting output was spent. A signature of exiting output owner on challenging transaction.

#### senderData (bytes32)
A keccak256 hash of the sender's address.

#### Example:

```
PaymentExitGame.challengeStandardExit([
  1590417156246506823621774965204183050418265227,
  0xf85401c0f0ef01949c7fc8601655b4e1ef395107217e6ed600f7ba48940000000000000000000000000000000000000000830f4240a00000000000000000000000000000000000000000000000000000000000000000,
  0xf88a01c685e9103fda00f85fee0194821aea9a577a9b44299b9c15c88cf3087f3b55449400000000000000000000000000000000000000008203e8ef01949c7fc8601655b4e1ef395107217e6ed600f7ba48940000000000000000000000000000000000000000830f3e58a00000000000000000000000000000000000000000000000000000000000000000,
  0,
  0xc8fafc7490868b372863778cd2c7928c835e66c59d7bc44b912d14ca574732434f928004b680d9a231c3a688fe1c1f62bac47c663695c8287d779ff2658626c81b,
  0x984f41eb67eb23e42139410eade1008a90b01a018ce2577fea44262c6fc16ce3
])
```

### Processing a standard exit

Once the exit period is over, an exit can be processed to release the funds on the root chain. An end user can perform this action, or the operator can do it for everyone.

To process a standard exit:
1. Obtain your `exitId` as described [here](#challenging-a-standard-exit)

2. Process your exit.

```
PlasmaFramework.processExits({
  uint256 vaultId,
  address token,
  uint160 topExitId,
  uint256 maxExitsToProcess
})
```

### Parameters
This section describes the parameters included in the function called for processing a standard exit.

#### vaultId (uint256)
The vault ID of the vault that stores exiting funds.

Use `1` for Ether, `2` for ERC-20.

#### token (address)
The token type to process.

ETH: `0x0000000000000000000000000000000000000000`

The contract address for ERC-20 tokens.

#### topExitId (uint160)
The unique priority of the first exit that should be processed. Set to zero to skip the check.

If you're trying to process only your own exit, set your exitId here.

#### maxExitsToProcess (uint256)
Defines the maximum number of exits you wish to process. Set to `1` to process only your own exit.


### Example: Processing a standard exit

```
PlasmaFramework.processExits([
  1, # vaultId
  0x0000000000000000000000000000000000000000, # token, ETH
  707372774235521271159305957085057710072500938, # topExitId
  1 # maxExitsToProcess
])
```

### Starting an in-flight exit

To start an in-flight exit, follow these steps:

1. Obtain `PaymentExitGame` address as described in [step 2 for starting a standard exit](#starting-a-standard-exit).

2. Get the amount of ETH to cover the bond for starting in-flight exit as described [here](#in-flight-exit-bonds).

3. Call
```
PaymentExitGame.startInFlightExit({
  inFlightTx,
  inputTxs,
  inputUtxosPos
  inputTxsInclusionProofs,
  inFlightTxWitnesses
})
```
with appropriate amount of ETH to cover the bond.


### Parameters

#### inFlightTx (bytes)
RLP encoded in-flight transaction.

#### inputTxs (bytes[])
RLP encoded transactions that created the inputs to the in-flight transaction. In the same order as in-flight transaction inputs.

#### inputUtxosPos (uint256[])
Utxo positions that represent in-flight transaction inputs. In the same order as input transactions.

#### inputTxsInclusionProofs (bytes[])
Merkle proofs that show the input-creating transactions are valid. In the same order as input transactions.

#### inFlightTxWitnesses (bytes[])
Witnesses for in-flight transaction. In the same order as input transactions. Depends on transaction type, for example the signatures of input owners.

#### Example:

```
PaymentExitGame.startInFlightExit([
  0xf87701e1a0000000000000000000000000000000000000000000000000000000003b9aca00f1f001ee947a809718aec76d8ac282a825be98e6ba4fc01fb89400000000000000000000000000000000000000008307a12080a00000000000000000000000000000000000000000000000000000000000000000,[0xf85601c0f1f001ee949c7fc8601655b4e1ef395107217e6ed600f7ba48940000000000000000000000000000000000000000830f424080a00000000000000000000000000000000000000000000000000000000000000000],[0xf39a869f62e75cf5f0bf914688a6b289caf2049435d8e68c5c5e6d05e44913f34ed5c02d6d48c8932486c99d3ad999e5d8949dc3be3b3058cc2979690c3e3a621c792b14bf66f82af36f00f5fba7014fa0c1e2ff3c7c273bfe523c1acf67dc3f5fa080a686a5a0d05c3d4822fd54d632dc9cc04b1616046eba2ce499eb9af79f5eb949690a0404abf4cebafc7cfffa382191b7dd9e7df778581e6fb78efab35fd364c9d5dadad4569b6dd47f7feabafa3571f842434425548335ac6e690dd07168d8bc5b77979c1a6702334f529f5783f79e942fd2cd03f6e55ac2cf496e849fde9c446fab46a8d27db1e3100f275a777d385b44e3cbc045cabac9da36cae040ad516082324c96127cf29f4535eb5b7ebacfe2a1d6d3aab8ec0483d32079a859ff70f9215970a8beebb1c164c474e82438174c8eeb6fbc8cb4594b88c9448f1d40b09beaecac5b45db6e41434a122b695c5a85862d8eae40b3268f6f37e414337be38eba7ab5bbf303d01f4b7ae07fd73edc2f3be05e43948a34418a3272509c43c2811a821e5c982ba51874ac7dc9dd79a80cc2f05f6f664c9dbb2e454435137da06ce44de45532a56a3a7007a2d0c6b435f726f95104bfa6e707046fc154bae91898d03a1a0ac6f9b45e471646e2555ac79e3fe87eb1781e26f20500240c379274fe91096e60d1545a8045571fdab9b530d0d6e7e8746e78bf9f20f4e86f06],
  [0x7876f3035bddd396f99ac58a5da145687e1dae2dff38d6c6e4fbdd76d548ed945ccc4b9e8226c87dbf8648cf09e8bd89926fdfb84e632b211e27a6a33f2882d01c]
])
```

### Piggybacking on an in-flight exit input / output

To exit an input or output of an in-flight transaction the user has to piggyback on it.

When in-flight transaction exits, the child chain removes the inputs of the in-flight transaction from the spendable set and they can no longer be spent. Calls to the child chain to spend the inputs or outputs of an in-flight transaction result in `submit:utxo_not_found` error. The effect of an input or output not being available to spend is not visible in contract's state until the exit is processed. After the in-flight exit is processed `PlasmaFramework.isOutputFinalized(piggybackedInputOrOutputPosition)` is set to true.

1. Get the amount of ETH to cover the bond for piggybacking in-flight exit as described [here](#in-flight-exit-bonds).

2. To piggyback on in-flight exit input call:
```
PaymentExitGame.piggybackInFlightExitOnInput({
  inFlightTx,
  inputIndex
})
```
To piggyback on in-flight exit output call:
```
PaymentExitGame.piggybackInFlightExitOnOutput({
  inFlightTx,
  outputIndex
})
```
Appropriate amount of ETH needs to be provided to cover the bond.

### Parameters

#### inFlightTx (bytes)
RLP encoded in-flight transaction.

#### inputIndex (uint16)
Index of the input to piggyback on.

#### outputIndex (uint16)
Index of the output to piggyback on.


#### Examples:

```
PaymentExitGame.piggybackInFlightExitOnInput([
  0xf87701e1a0000000000000000000000000000000000000000000000000000000003b9aca00f1f001ee947a809718aec76d8ac282a825be98e6ba4fc01fb89400000000000000000000000000000000000000008307a12080a00000000000000000000000000000000000000000000000000000000000000000,
  0
])
```

```
PaymentExitGame.piggybackInFlightExitOnOutput([
  0xf87701e1a0000000000000000000000000000000000000000000000000000000003b9aca00f1f001ee947a809718aec76d8ac282a825be98e6ba4fc01fb89400000000000000000000000000000000000000008307a12080a00000000000000000000000000000000000000000000000000000000000000000,
  0
])
```

### Challenging canonicity of an in-flight exit

To challenge canonicity of an in-flight exit show a competing transaction:
```
PaymentExitGame.challengeInFlightExitNotCanonical({
    inputTx,
    inputUtxosPos,
    inFlightTx,
    inFlightTxInputIndex,
    competingTx,
    competingTxInputIndex,
    competingTxPos,
    competingTxInclusionProof,
    competingTxWitness
})
```

### Parameters

#### inputTx (bytes)
RLP encoded transaction that created the input shared by in-flight transaction and its competitor.

#### inFlightTx (bytes)
RLP encoded in-flight transaction.

#### inFlightTxInputIndex (uint16)
Index of shared input in in-flight transaction.

#### competingTx (bytes)
RLP-encoded competing transaction that spends the shared input.

#### competingTxInputIndex (uint16)
Index of shared input in competing transaction

#### competingTxPos (uint256)
Transaction position of competing tx.
If competing transaction is not included in any Plasma block provide value `0`.

#### competingTxInclusionProof (bytes)
Merkle proof showing that the competing transaction is included in a Plasma block.
Send empty bytes if competing transaction is not included in a Plasma block.

#### competingTxWitness (bytes)
Witness for competing transaction

#### Example:

```
PaymentExitGame.challengeInFlightExitNotCanonical([
  0xf85601c0f1f001ee949c7fc8601655b4e1ef395107217e6ed600f7ba48940000000000000000000000000000000000000000830f424080a00000000000000000000000000000000000000000000000000000000000000000,
  2000000000,
  0xf89901f842a00000000000000000000000000000000000000000000000000000000077359400a000000000000000000000000000000000000000000000000000000000b2d05e00f1f001ee947a809718aec76d8ac282a825be98e6ba4fc01fb89400000000000000000000000000000000000000008307a12080a00000000000000000000000000000000000000000000000000000000000000000,
  0,
  0xf87701e1a00000000000000000000000000000000000000000000000000000000077359400f1f001ee94821aea9a577a9b44299b9c15c88cf3087f3b55449400000000000000000000000000000000000000008307a12080a00000000000000000000000000000000000000000000000000000000000000000,
  0,
  0,
  0x,
  0xb5bb3fff130c206bb550e969964c673e53fd5f15edc9e20046ea504389bf7ba324b81adc8a91e01ef49384d86d71ae00b9ac092bf88156a67f1e202806b61a221c
])
```

### Responding to canonicity challenge

To respond to a canonicity challenge show that in-flight transaction is included in a Plasma block:
```
PaymentExitGame.respondToNonCanonicalChallenge({
    inFlightTx,
    inFlightTxPos,
    inFlightTxInclusionProof
})
```

### Parameters

#### inFlightTx (bytes)
RLP encoded transaction that is in-flight exiting

#### inFlightTxPos (uint256)
Transaction position of in-flight exiting transaction.

#### inFlightTxInclusionProof (bytes)
Proof that in-flight exiting transaction is included in a Plasma block.

#### Example:

```
PaymentExitGame.respondToNonCanonicalChallenge([
  0xf87701e1a0000000000000000000000000000000000000000000000000000000012a05f200f1f001ee947a809718aec76d8ac282a825be98e6ba4fc01fb89400000000000000000000000000000000000000008307a12080a00000000000000000000000000000000000000000000000000000000000000000,
  1000000000000,
  0xf39a869f62e75cf5f0bf914688a6b289caf2049435d8e68c5c5e6d05e44913f34ed5c02d6d48c8932486c99d3ad999e5d8949dc3be3b3058cc2979690c3e3a621c792b14bf66f82af36f00f5fba7014fa0c1e2ff3c7c273bfe523c1acf67dc3f5fa080a686a5a0d05c3d4822fd54d632dc9cc04b1616046eba2ce499eb9af79f5eb949690a0404abf4cebafc7cfffa382191b7dd9e7df778581e6fb78efab35fd364c9d5dadad4569b6dd47f7feabafa3571f842434425548335ac6e690dd07168d8bc5b77979c1a6702334f529f5783f79e942fd2cd03f6e55ac2cf496e849fde9c446fab46a8d27db1e3100f275a777d385b44e3cbc045cabac9da36cae040ad516082324c96127cf29f4535eb5b7ebacfe2a1d6d3aab8ec0483d32079a859ff70f9215970a8beebb1c164c474e82438174c8eeb6fbc8cb4594b88c9448f1d40b09beaecac5b45db6e41434a122b695c5a85862d8eae40b3268f6f37e414337be38eba7ab5bbf303d01f4b7ae07fd73edc2f3be05e43948a34418a3272509c43c2811a821e5c982ba51874ac7dc9dd79a80cc2f05f6f664c9dbb2e454435137da06ce44de45532a56a3a7007a2d0c6b435f726f95104bfa6e707046fc154bae91898d03a1a0ac6f9b45e471646e2555ac79e3fe87eb1781e26f20500240c379274fe91096e60d1545a8045571fdab9b530d0d6e7e8746e78bf9f20f4e86f06
])
```

### Challenging in-flight transaction input exit

To challenge a piggyback on in-flight transaction input show a transaction that spends the same input as the in-flight transaction:
```
PaymentExitGame.challengeInFlightExitInputSpent({
    inFlightTx,
    inFlightTxInputIndex,
    challengingTx,
    challengingTxInputIndex,
    challengingTxWitness,
    inputTx,
    inputUtxoPos,
    senderData
})
```

### Parameters

#### inFlightTx (bytes)
RLP encoded in-flight transaction.

#### inFlightTxInputIndex (uint16)
Index of spent input in the in-flight transaction.

#### challengingTx (bytes)
RLP-encoded challenging transaction.

#### challengingTxInputIndex (uint16)
Index of spent input in the challenging transaction.

#### challengingTxWitness (bytes)
Witness for challenging transactions. For payment transaction it's a signature of input's owner.

#### inputTx (bytes)
RLP encoded input transaction.

#### inputUtxoPos (uint256)
 UTXO position of input transaction's output.

#### senderData (bytes32)
A keccak256 hash of the sender's address.

#### Example:

```
PaymentExitGame.challengeInFlightExitInputSpent([
  0xf89901f842a00000000000000000000000000000000000000000000000000000000077359400a000000000000000000000000000000000000000000000000000000000b2d05e00f1f001ee947a809718aec76d8ac282a825be98e6ba4fc01fb89400000000000000000000000000000000000000008307a12080a00000000000000000000000000000000000000000000000000000000000000000,
  0,
  0xf87701e1a00000000000000000000000000000000000000000000000000000000077359400f1f001ee94821aea9a577a9b44299b9c15c88cf3087f3b55449400000000000000000000000000000000000000008307a12080a00000000000000000000000000000000000000000000000000000000000000000,
  0,
  0xb5bb3fff130c206bb550e969964c673e53fd5f15edc9e20046ea504389bf7ba324b81adc8a91e01ef49384d86d71ae00b9ac092bf88156a67f1e202806b61a221c,
  0xf85601c0f1f001ee949c7fc8601655b4e1ef395107217e6ed600f7ba48940000000000000000000000000000000000000000830f424080a00000000000000000000000000000000000000000000000000000000000000000,
  2000000000,
  0x984f41eb67eb23e42139410eade1008a90b01a018ce2577fea44262c6fc16ce3
])
```

### Challenging in-flight transaction output exit

To challenge a piggyback on in-flight transaction output show that the in-flight exit transaction is included in a Plasma block and another transaction that spends the output:
```
PaymentExitGame.challengeInFlightExitOutputSpent({
    inFlightTx,
    inFlightTxInclusionProof,
    outputUtxoPos,
    challengingTx,
    challengingTxInputIndex,
    challengingTxWitness,
    senderData
})
```

### Parameters

#### inFlightTx (bytes)
RLP encoded in-flight exit transaction.

#### inFlightTxInclusionProof (bytes)
Proof that `inFlightTx` is included in a Plasma block.

#### outputUtxoPos (uint256)
UTXO position of exiting output.

#### challengingTx (bytes)
RLP encoded challenging transaction.

#### challengingTxInputIndex (uint16)
Input index of challenged output in the challenging transaction.

#### challengingTxWitness (bytes)
Witness for challenging transaction.

#### senderData (bytes32)
A keccak256 hash of the sender's address.

#### Example:

```
PaymentExitGame.challengeInFlightExitOutputSpent([
  0xf8a301e1a00000000000000000000000000000000000000000000000000000000000000001f85ced01eb94f17f52151ebef6c7334fad080c5704d77216b73294000000000000000000000000000000000000000001ed01eb94f17f52151ebef6c7334fad080c5704d77216b7329400000000000000000000000000000000000000000180a00000000000000000000000000000000000000000000000000000000000000000,
  0xf39a869f62e75cf5f0bf914688a6b289caf2049435d8e68c5c5e6d05e44913f34ed5c02d6d48c8932486c99d3ad999e5d8949dc3be3b3058cc2979690c3e3a621c792b14bf66f82af36f00f5fba7014fa0c1e2ff3c7c273bfe523c1acf67dc3f,
  1000000000000,
  0xf87401e1a06e5de68fadce7ffe6ebf92546a4e6d4aedfa3c325929eebfa8dd1f8e6918c4b7eeed02eb94f17f52151ebef6c7334fad080c5704d77216b7329400000000000000000000000000000000000000000180a00000000000000000000000000000000000000000000000000000000000000000,
  0,
  0xb833e902b1f76a9ece793891fdae542c01b742ea83dfbfb721df1e82413fb5fc,
  0x984f41eb67eb23e42139410eade1008a90b01a018ce2577fea44262c6fc16ce3
])
```

### Delete an in-flight exit

For in-flight exit, an exit is only enqueued in the PlasmaFramework when the exit is piggybacked. As a consequence, if an in-flight exit is never piggybacked then the in-flight exit bond would end up being locked.

As a mitigation, we have the `deleteNonPiggybackedInFlightExit` api on the exit game contract. One can call this to release the bond back to its owner and clean up the state of in-flight exit. This can only be called after the first phase has been passed and nobody has piggybacked the exit.

For some more detail, see the original issue: [here](https://github.com/omisego/plasma-contracts/issues/440)

```
PaymentExitGame.deleteNonPiggybackedInFlightExit(exitId)
```

### Parameters

#### exitId (uint160)
Exit Id of the in-flight exit.

You can get your `exitId` by `getInFlightExitId` helper function of the `PaymentExitGame`, see doc: [here](https://github.com/omisego/plasma-contracts/blob/master/plasma_framework/docs/contracts/PaymentExitGame.md#getinflightexitid)


#### Example:

```
PaymentExitGame.deleteNonPiggybackedInFlightExit(707372774235521271159305957085057710072500938)
```


### Processing an in-flight exit

Once the exit period is over, an exit can be processed to release the funds on the root chain. An end user can perform this action, or the operator can do it for everyone.

Be aware that in-flight exit is only put on a token's exit queue if a piggyback of that token exists. In other words, if an in-flight exit ends up with no piggybacks of a certain token, eg. ETH, then user will not find the exit on the priority queue for ETH. Please make sure piggyback step is done before process exit.

To process a in-flight exit:
1. (Optional) Obtain your `exitId` by `getInFlightExitId` helper function of the `PaymentExitGame`, see doc: [here](https://github.com/omisego/plasma-contracts/blob/master/plasma_framework/docs/contracts/PaymentExitGame.md#getinflightexitid)

```
PaymentExitGame.getInFlightExitId(
  "0xf85801c0f4f3019441777dc7bdcc6b58be1c25eb3df7df52d1bfecbd94000000000000000000000000000000000000000087038d7ea4c68000a00000000000000000000000000000000000000000000000000000000000000000", # RLP-encoded transaction sent when startInFlightExit was called
)
```

2. Process your exit.

Exits are processed in the order of the priority queue (each vault has its own priority queue). To find out which exit is at the head of a queue, you can call `getNextExit(vaultId, token)`. The return value of this call is binary data about the exit and its priority; the exitId is the 160 least significant bits of this data

```
PlasmaFramework.processExits({
  uint256 vaultId,
  address token,
  uint160 topExitId,
  uint256 maxExitsToProcess
})
```

### Parameters
This section describes the parameters included in the function called for processing a standard exit.

#### vaultId (uint256)
The vault ID of the vault that stores exiting funds.

Use `1` for Ether, `2` for ERC-20.

#### token (address)
The token type to process.

ETH: `0x0000000000000000000000000000000000000000`

The contract address for ERC-20 tokens.

#### topExitId (uint160)
The unique priority of the first exit that should be processed. Set to zero to skip the check.

The purpose of this parameter is to prevent you from inadvertently processing another exit that has jumped to the head of the queue because it has a higher priority. If your exit is at the head of the queue and you want to make sure that you process _only_ your exit then you should set this parameter to your exitId. If another exit with higher priority jumps to the head of the queue, then the processExits() call will fail and you won't spend the gas to process the other exit. You can then wait until your exit is at the head of the queue before trying again.

#### maxExitsToProcess (uint256)
Defines the maximum number of exits you wish to process. Set to `1` to process only your own exit.

***Note**: `processExits()` will only process exits that have completed their exit period. You can find out which (if any) exits were processed via the `ExitFinalized` or `ProcessedExitsNum` events *

### Example: Processing an in-flight exit

```
PlasmaFramework.processExits([
  1, # vaultId
  0x0000000000000000000000000000000000000000, # token, ETH
  707372774235521271159305957085057710072500938, # topExitId
  1 # maxExitsToProcess
])
```

## Payment Exit game events
When listening for events related to the exit game, it's important to remember that there will be only one exit game per transaction type. One should listen the events from the corresponding exit game contract. See [Exit Game contract section](#exit-game-contracts) for details of each contract and transaction types.

Following are lists of events from `PaymentExitGame` contract:

### Standard Exit Events
- A standard exit has started:
```
   event ExitStarted(
        address indexed owner,
        uint160 exitId,
        uint256 utxoPos,
        bytes outputTx
    );
```
- A standard exit is successfully challenged:
```
    event ExitChallenged(
        uint256 indexed utxoPos
    );
```
- An exit is successfully processed; that is, funds sent back to owner:
```
    event ExitFinalized(
        uint160 indexed exitId
    );
```
- An exit was in the exit queue but was not processed; for example, because it was already processed:
```
    event ExitOmitted(
        uint160 indexed exitId
    );
```

### In-flight exit events
This section describes the events for an in-flight exit.


- An in-flight exit has started:
```
    event InFlightExitStarted(
        address indexed initiator,
        bytes32 indexed txHash,
        bytes inFlightTx,
        uint256[] inputUtxosPos,
        bytes[] inFlightTxWitnesses,
        bytes[] inputTxs
    );
```
- An input has been piggybacked on an in-flight exit:
```
    event InFlightExitInputPiggybacked(
        address indexed exitTarget,
        bytes32 indexed txHash,
        uint16 inputIndex
    );
```
- An output has been piggybacked on an in-flight exit:
```
    event InFlightExitOutputPiggybacked(
        address indexed exitTarget,
        bytes32 indexed txHash,
        uint16 outputIndex
    );
```
- An in-flight exit has been successfully challenged as non-canonical:
```
    event InFlightExitChallenged(
        address indexed challenger,
        bytes32 indexed txHash,
        uint256 challengeTxPosition,
        uint16 inFlightTxInputIndex,
        bytes challengeTx,
        uint16 challengeTxInputIndex,
        bytes challengeTxWitness
    );
```
- An in-flight exit has been proved canonical in response to a non-canonical challenge:
```
    event InFlightExitChallengeResponded(
        address indexed challenger,
        bytes32 indexed txHash,
        uint256 challengeTxPosition
    );
```
- A piggybacked input on an in-flight exit has been shown to have been spent:
```
    event InFlightExitInputBlocked(
        address indexed challenger,
        bytes32 indexed txHash,
        uint16 inputIndex
    );
```
- A piggybacked output on an in-flight exit has been shown to have been spent:
```
    event InFlightExitOutputBlocked(
        address indexed challenger,
        bytes32 indexed txHash,
        uint16 inputIndex
    );
```
- A piggybacked input on an in-flight exit has been successfully withdrawn; that is, funds sent back to owner:
```
    event InFlightExitInputWithdrawn(
        uint160 indexed exitId,
        uint16 inputIndex
    );
```
- A piggybacked output on an in-flight exit has been successfully withdrawn; that is, funds sent back to owner:
```
    event InFlightExitOutputWithdrawn(
        uint160 indexed exitId,
        uint16 outputIndex
    );
```
- An exit was in the exit queue, but was not processed; for example, because it was already processed:
```
    event InFlightExitOmitted(
        uint160 indexed exitId,
        address token
    );
```

# Upgrading the PlasmaFramework
The PlasmaFramework is designed to be upgraded, either to fix bugs, or to add new functionality. Upgrades are done by adding new contracts to the framework. Any upgrade takes some time to come into effect. This keeps the framework trustless; that is, if a new malicious or vulnerable contract is added, users have an opportunity to exit before the problematic contract is activated.


## Upgrade the deposit transaction type of a vault
In the current implementation, a vault comes with a single transaction type, which it accepts to be the deposit transaction. However, as time goes, we might want to use as the deposit transaction, the later version of transaction type or a completely new transaction type. For instance, the ETH vault and ERC20 vault accepts payment transaction as the deposit transaction.

 > ***Note**: In future, when a new payment transaction type is available, it will be changed to payment transaction v2.*

The vault is designed with the concept of `depositVerifier`. This is a predicate contract with the ability to check whether a deposit transaction is the right transaction type, with the data required for a deposit. As a result, to upgrade to new deposit transaction type, it is necessary to implement the new `depositVerifier` contract and set the new verifier to the vault.

### Configure a new deposit verifier
1. Implement new deposit verifier that fullfils the interface of the certain vault.

   For details on the deposit verifier code and interface, see the documentation, [here](https://github.com/omisego/plasma-contracts/tree/master/plasma_framework/contracts/src/vaults/verifiers)

2. Call the `setDepositVerifier` function by `maintainer`.

   For more information, see [setDepositVerifier doc](https://github.com/omisego/plasma-contracts/blob/master/plasma_framework/docs/contracts/Vault.md#setdepositverifier).

3. After the call, you should recieve the `setDepositVerifierCalled` event:

    ```
    event SetDepositVerifierCalled(address  nextDepositVerifier);
    ```
4. Wait **two** `minExitPeriod` for the new deposit verifier to take effect. In production, it should be two weeks.


### Security analysis

The waiting period of two weeks when upgrading the deposit verifier provides protection for deposit transactions that are sent on the root chain, but are still in mempool before the `setDepositVerifier` is called. Also, it gives a one week buffer for the user to perform a standard exit if an invalid deposit verifier is set.

For more information, see the description in the following issues:
- https://github.com/omisego/plasma-contracts/issues/174
- https://github.com/omisego/plasma-contracts/issues/412

Users/watchers should listen to the following event to understand when the new deposit verifier takes effect: `setDepositVerifierCalled`

In case the new deposit verifier is not trustworthy, immediately stop any deposit action to the vault, and exit all outputs from the PlasmaFramework.


## Add a new vault
Although it's possible to upgrade the deposit transaction type of a vault, one vault supports only one ERC protocol. For this reason, a new vault must be added if support is required for a new ERC protocol.

Additionally, if the existing vault code contains any bugs, a new vault is required to replace it.

 > ***Important**! The process of moving funds from one vault to another is relatively complex, involving a full exit and re-deposit, and should be done with caution.*

Perform these steps to add a new vault:

1. Design and implement a new vault contract.

  Unless some feature breaks the abstraction, otherwise please inheritence the existing abstract vault contract. For more
  information, see: [Vault.sol](https://github.com/omisego/plasma-            contracts/blob/master/plasma_framework/contracts/src/vaults/Vault.sol)

2. The maintainer registers the new vault to the PlasmaFramework, using the following function: `registerVault`

   For more information, see the documentation, [here](https://github.com/omisego/plasma-contracts/blob/master/plasma_framework/docs/contracts/VaultRegistry.md#registervault).

3. Wait for the following event to be returned: `event VaultRegistered(uint256  vaultId, address  vaultAddress);`

5. Wait **two** `minExitPeriod` (two weeks in production).

6. After the waiting period completes, the user is able to deposit to the new vault.

### Security analysis
As with the process of configuring a new deposit verifier, a period of two weeks waiting time is chosen to protect the deposit transactions that are still in the mempool when the transaction of `registerVault` is sent. Also, it gives a one week buffer for the user to perform a standard exit if an invalid deposit verifier is set.

For more information, see the description in the following issues:
- https://github.com/omisego/plasma-contracts/issues/173
- https://github.com/omisego/plasma-contracts/issues/412

Users/watchers should listen to the follwing event to understand when the new vault is registered: `VaultRegistered`

In case the new vault is not truthworthy, do not perform any deposit to the new vault, and exit all outputs from the PlasmaFramework.


## Add a new exit game
Adding a new exit game is the main method for adding new features to the PlasmaFramework.

For each transaction type, there is a corresponding exit game. This means that when a feature is added, a new transaction type is also added, and the new transaction type is registered to the corresponding exit game contract in the PlasmaFramework contract.

For example, to support a new DEX feature, a new DEX transaction may be added for spending a payment transaction.

Perform these steps to add a new exit game:
1. Design a new transaction type and implement the exit game.
2. The maintainer registers the new exit game contract to the PlasmaFramework, using the following function:  `registerExitGame`

 For more information, see the [documentation](https://github.com/omisego/plasma-contracts/blob/389_add_priority_queue_test/plasma_framework/docs/contracts/ExitGameRegistry.md#registerexitgame)

3. The following event is returned: `event ExitGameRegistered(uint256  txType, address  exitGameAddress, uint8  protocol);`

4. Wait **three** `minExitPeriod` (three weeks in production) for the new exit game to take effect.


### Security analysis
A waiting time is required when adding a new exit game. This provides protection for existing users, since the exit game contract has access to several components in the PlasmaFramework. For example, the exit game can insert an exit with the wrong order, flag a random output as used, or ask the vault to withdraw funds directly. For this reason, it has been determined that a minimum period of three weeks is required before allowing any new exit game contracts to take effect.

**Why a three week waiting period?**

The three week period is determined as follows:
* It takes two weeks for a newly mined transaction to exit once a mass exit scenario is detected; thus, `mined_block_time + 2 weeks`; or, two weeks after it's mined.
* One extra week provides users with an opportunity to clean up the exit queue (`processExits`) before a bad exit game contract can potentially insert a bad exit, with the wrong priority.

For details, see this issue: https://github.com/omisego/plasma-contracts/issues/172

Users/Watchers should listen to the following event to be alerted about a new exit game: `ExitGameRegistered`

If new exit game contract is not trustworthy, it is important to exit immediately.


# Ensuring the correctness of the Plasma network
Plasma is designed to be somewhat optimistic; that is, it assumes everything is correct unless proven otherwise.

Exit games allow users to participate in ensuring the correctness of the system. However, there must be a way to alert users so they know when to engage in the exit games. This role is performed by the Watchers.


## Watchers
Watchers monitor the network and send alerts so that users know when they need to react to events.

### Event BlockSubmitted (emitted in PlasmaFramework contract)

A new Plasma Block has been submitted. Watcher should fetch block data from child chain and verify that it matches block data published to the Plasma contract on the root chain.
```
event BlockSubmitted(
    uint256 blknum
);
```

Fetching block data from `PlasmaFramework`:
```
block = PlasmaFramework.blocks(blknum)
```
where block is:
```
struct Block {
    bytes32 root; // root of Merkle Tree built from transactions
    uint256 timestamp; // Ethereum block timestamp for Ethereum transaction that submitted the Plasma block
}
```

### Event VaultRegistered (emitted in PlasmaFramework contract)

A new vault has been registered. After a quarantine period, a vault can be used for managing funds: storing deposits and making withdrawals.
```
event VaultRegistered(
    uint256 vaultId, // id of the vault in the Plasma Framework
    address vaultAddress // address of the vault
);
```

### Event SetDepositVerifierCalled (emitted in vault contracts)

The Chain maintainer can change deposit verification for a vault by switching to a new deposit verifier, which is a contract that verifies the validity of a Plasma deposit. Change of a verifier takes effect after 2 exit periods. User should inspect the source code of the new deposit verifier to examine whether it is safe to use.
```
event VaultRegistered(
    uint256 vaultId, // id of the vault in the Plasma Framework
    address vaultAddress // address of the vault
);
```

### Event DepositCreated (emitted in EthVault contract)

After a successful deposit of ETH, `EthVault` contract emits `DepositCreated` event. This event notifies that deposited funds are available to spend.
```
event DepositCreated(
    address indexed depositor, // address that deposited the funds
    uint256 indexed blknum, // Plasma block number that contains the deposit
    address indexed token, // ETH
    uint256 amount // amount of ETH deposited
);
```

### Event DepositCreated (emitted in Erc20Vault contract)

After a successful deposit of ERC20 tokens, `Erc20Vault` contract emits `DepositCreated` event. This event notifies that deposited funds are available to spend.
```
event DepositCreated(
    address indexed depositor, // address that deposited the funds
    uint256 indexed blknum, // Plasma block number that contains the deposit
    address indexed token, // deposited ERC20 token
    uint256 amount // amount of tokens deposited
);
```

### Event EthWithdrawn (emitted in EthVault contract)

After a successful withdawal of ETH, `EthVault` contract emits `EthWithdrawn` event. This event notifies that withdrawn funds are no longer available to spend on Plasma chain and have been transfered to the receiver.
```
event EthWithdrawn(
    address indexed receiver, // address that receives the funds
    uint256 amount // amount withdrawn
);
```

### Event Erc20Withdrawn (emitted in Erc20Vault contract)

After a successful withdawal of ERC20 tokens, `Erc20Vault` contract emits `Erc20Withdrawn` event. This event notifies that withdrawn funds are no longer available to spend on Plasma chain and have been transfered to the receiver.
```
event Erc20Withdrawn(
    address indexed receiver, // address that receives the funds
    address indexed token, // transferred ERC20 token
    uint256 amount // amount withdrawn
);
```

### Event WithdrawFailed (emitted in EthVault contract)

If a withdrawal fails, the `EthVault` contract emits a `WithdrawFailed` event. This event notifies that a withdrawal, which is a result of an exit, failed. This can happen because of malicious behaviour or the vault not having enough funds, most probably due to a malicious attack.
```
event WithdrawFailed(
    address indexed receiver, // address that is the target of the withdrawal
    uint256 amount // exiting amount
);
```

### Event ExitGameRegistered (emitted in PlasmaFramework contract)

Event is emitted when maintainer registers a new exit game. The new exit game can be used after a quarantine period which is `4 * MIN_EXIT_PERIOD`.
```
event ExitGameRegistered(
    uint256 txType, // transaction type handled by the registered exit gmae
    address exitGameAddress, // address of the exit game contract
    uint8 protocol // 1 for MVP, 2 for MORE_VP
);
```

### Event ExitQueueAdded (emitted in PlasmaFramework contract)

Event is emitted when a user adds an exit queue. Exit queue stores all exits for given vaultId and token pair. Before exiting funds stored in a vault it is necessary to register an exit queue for that vault and exiting token.
```
event ExitQueueAdded(
    uint256 vaultId, // id of the vault
    address token // token
);
```

### Event ExitQueued (emitted in PlasmaFramework contract)

Event notifies that exit has been queued in an exit queue.
```
event ExitQueued(
    uint160 indexed exitId, // id of the exit
    uint256 priority // exit's priority
);
```

### Event ProcessedExitsNum (emitted in PlasmaFramework contract)

Event is emitted after exits have been processed.
```
event ProcessedExitsNum(
    uint256 processedNum, // number of processed exits
    uint256 vaultId, // vault id of the vault that stored the funds
    address token // token that was withdrawn
);
```
