# PaymentInFlightExitRouter.sol

View Source: [contracts/src/exits/payment/routers/PaymentInFlightExitRouter.sol](../../contracts/src/exits/payment/routers/PaymentInFlightExitRouter.sol)

**↗ Extends: [IExitProcessor](IExitProcessor.md), [OnlyFromAddress](OnlyFromAddress.md), [OnlyWithValue](OnlyWithValue.md), [FailFastReentrancyGuard](FailFastReentrancyGuard.md)**
**↘ Derived Contracts: [PaymentExitGame](PaymentExitGame.md)**

**PaymentInFlightExitRouter**

## Contract Members
**Constants & Variables**

```js
//public members
uint128 public constant INITIAL_IFE_BOND_SIZE;
uint128 public constant INITIAL_PB_BOND_SIZE;
uint16 public constant BOND_LOWER_BOUND_DIVISOR;
uint16 public constant BOND_UPPER_BOUND_MULTIPLIER;
uint128 public constant INITIAL_IFE_EXIT_BOUNTY_SIZE;
uint128 public constant INITIAL_PB_BOUNTY_SIZE;

//internal members
struct PaymentExitDataModel.InFlightExitMap internal inFlightExitMap;
struct PaymentStartInFlightExit.Controller internal startInFlightExitController;
struct PaymentPiggybackInFlightExit.Controller internal piggybackInFlightExitController;
struct PaymentChallengeIFENotCanonical.Controller internal challengeCanonicityController;
struct PaymentChallengeIFEInputSpent.Controller internal challengeInputSpentController;
struct PaymentChallengeIFEOutputSpent.Controller internal challengeOutputSpentController;
struct PaymentDeleteInFlightExit.Controller internal deleteNonPiggybackIFEController;
struct PaymentProcessInFlightExit.Controller internal processInflightExitController;
struct BondSize.Params internal startIFEBond;
struct BondSize.Params internal piggybackBond;

//private members
contract PlasmaFramework private framework;
bool private bootDone;

```

**Events**

```js
event IFEBondUpdated(uint128  bondSize);
event PiggybackBondUpdated(uint128  bondSize, uint128  exitBountySize);
event InFlightExitStarted(address indexed initiator, bytes32 indexed txHash, bytes inFlightTx, uint256[] inputUtxosPos, bytes[] inFlightTxWitnesses bytes[] inputTxs);
event InFlightExitInputPiggybacked(address indexed exitTarget, bytes32 indexed txHash, uint16  inputIndex);
event InFlightExitOmitted(uint168 indexed exitId, address  token);
event InFlightBondReturnFailed(address indexed receiver, uint256  amount);
event InFlightExitOutputWithdrawn(uint168 indexed exitId, uint16  outputIndex);
event InFlightExitInputWithdrawn(uint168 indexed exitId, uint16  inputIndex);
event InFlightExitOutputPiggybacked(address indexed exitTarget, bytes32 indexed txHash, uint16  outputIndex);
event InFlightExitChallenged(address indexed challenger, bytes32 indexed txHash, uint256  challengeTxPosition);
event InFlightExitChallengeResponded(address indexed challenger, bytes32 indexed txHash, uint256  challengeTxPosition);
event InFlightExitInputBlocked(address indexed challenger, bytes32 indexed txHash, uint16  inputIndex);
event InFlightExitOutputBlocked(address indexed challenger, bytes32 indexed txHash, uint16  outputIndex);
event InFlightExitDeleted(uint168 indexed exitId);
event InFlightExitFinalized(uint168 indexed exitId);
```

## Functions

- [boot(struct PaymentExitGameArgs.Args paymentExitGameArgs)](#boot)
- [inFlightExits(uint168[] exitIds)](#inflightexits)
- [startInFlightExit(struct PaymentInFlightExitRouterArgs.StartExitArgs args)](#startinflightexit)
- [piggybackInFlightExitOnInput(struct PaymentInFlightExitRouterArgs.PiggybackInFlightExitOnInputArgs args)](#piggybackinflightexitoninput)
- [piggybackInFlightExitOnOutput(struct PaymentInFlightExitRouterArgs.PiggybackInFlightExitOnOutputArgs args)](#piggybackinflightexitonoutput)
- [challengeInFlightExitNotCanonical(struct PaymentInFlightExitRouterArgs.ChallengeCanonicityArgs args)](#challengeinflightexitnotcanonical)
- [respondToNonCanonicalChallenge(bytes inFlightTx, uint256 inFlightTxPos, bytes inFlightTxInclusionProof)](#respondtononcanonicalchallenge)
- [challengeInFlightExitInputSpent(struct PaymentInFlightExitRouterArgs.ChallengeInputSpentArgs args)](#challengeinflightexitinputspent)
- [challengeInFlightExitOutputSpent(struct PaymentInFlightExitRouterArgs.ChallengeOutputSpent args)](#challengeinflightexitoutputspent)
- [deleteNonPiggybackedInFlightExit(uint168 exitId)](#deletenonpiggybackedinflightexit)
- [processInFlightExit(uint168 exitId, address token, address payable processExitInitiator)](#processinflightexit)
- [startIFEBondSize()](#startifebondsize)
- [updateStartIFEBondSize(uint128 newBondSize)](#updatestartifebondsize)
- [piggybackBondSize()](#piggybackbondsize)
- [updatePiggybackBondSize(uint128 newBondSize, uint128 newExitBountySize)](#updatepiggybackbondsize)
- [processInFlightExitBountySize()](#processinflightexitbountysize)

### boot

⤾ overrides [PaymentStandardExitRouter.boot](PaymentStandardExitRouter.md#boot)

```js
function boot(struct PaymentExitGameArgs.Args paymentExitGameArgs) internal nonpayable
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|
| paymentExitGameArgs | struct PaymentExitGameArgs.Args |  |

### inFlightExits

Getter functions to retrieve in-flight exit data of the PaymentExitGame

```js
function inFlightExits(uint168[] exitIds) external view
returns(struct PaymentExitDataModel.InFlightExit[])
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|
| exitIds | uint168[] | The exit IDs of the in-flight exits |

### startInFlightExit

Starts withdrawal from a transaction that may be in-flight

```js
function startInFlightExit(struct PaymentInFlightExitRouterArgs.StartExitArgs args) public payable nonReentrant onlyWithValue
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|
| args | struct PaymentInFlightExitRouterArgs.StartExitArgs | Input argument data to challenge (see also struct 'StartExitArgs') |

### piggybackInFlightExitOnInput

Piggyback on an input of an in-flight exiting tx. Processed only if the in-flight exit is non-canonical.

```js
function piggybackInFlightExitOnInput(struct PaymentInFlightExitRouterArgs.PiggybackInFlightExitOnInputArgs args) public payable nonReentrant onlyWithValue
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|
| args | struct PaymentInFlightExitRouterArgs.PiggybackInFlightExitOnInputArgs | Input argument data to piggyback (see also struct 'PiggybackInFlightExitOnInputArgs') |

### piggybackInFlightExitOnOutput

Piggyback on an output of an in-flight exiting tx. Processed only if the in-flight exit is canonical.

```js
function piggybackInFlightExitOnOutput(struct PaymentInFlightExitRouterArgs.PiggybackInFlightExitOnOutputArgs args) public payable nonReentrant onlyWithValue
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|
| args | struct PaymentInFlightExitRouterArgs.PiggybackInFlightExitOnOutputArgs | Input argument data to piggyback (see also struct 'PiggybackInFlightExitOnOutputArgs') |

### challengeInFlightExitNotCanonical

Challenges an in-flight exit to be non-canonical

```js
function challengeInFlightExitNotCanonical(struct PaymentInFlightExitRouterArgs.ChallengeCanonicityArgs args) public nonpayable nonReentrant
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|
| args | struct PaymentInFlightExitRouterArgs.ChallengeCanonicityArgs | Input argument data to challenge (see also struct 'ChallengeCanonicityArgs') |

### respondToNonCanonicalChallenge

Respond to a non-canonical challenge by providing its position and by proving its correctness

```js
function respondToNonCanonicalChallenge(bytes inFlightTx, uint256 inFlightTxPos, bytes inFlightTxInclusionProof) public nonpayable nonReentrant
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|
| inFlightTx | bytes | The RLP-encoded in-flight transaction |
| inFlightTxPos | uint256 | The position of the in-flight exiting transaction. The output index within the position is unused and should be set to 0 |
| inFlightTxInclusionProof | bytes | Proof that the in-flight exiting transaction is included in a Plasma block |

### challengeInFlightExitInputSpent

Challenges an exit from in-flight transaction input

```js
function challengeInFlightExitInputSpent(struct PaymentInFlightExitRouterArgs.ChallengeInputSpentArgs args) public nonpayable nonReentrant
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|
| args | struct PaymentInFlightExitRouterArgs.ChallengeInputSpentArgs | Argument data to challenge (see also struct 'ChallengeInputSpentArgs') |

### challengeInFlightExitOutputSpent

Challenges an exit from in-flight transaction output

```js
function challengeInFlightExitOutputSpent(struct PaymentInFlightExitRouterArgs.ChallengeOutputSpent args) public nonpayable nonReentrant
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|
| args | struct PaymentInFlightExitRouterArgs.ChallengeOutputSpent | Argument data to challenge (see also struct 'ChallengeOutputSpent') |

### deleteNonPiggybackedInFlightExit

Deletes in-flight exit if the first phase has passed and not being piggybacked

```js
function deleteNonPiggybackedInFlightExit(uint168 exitId) public nonpayable nonReentrant
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|
| exitId | uint168 | The exitId of the in-flight exit |

### processInFlightExit

Process in-flight exit

```js
function processInFlightExit(uint168 exitId, address token, address payable processExitInitiator) internal nonpayable
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|
| exitId | uint168 | The in-flight exit ID |
| token | address | The token (in erc20 address or address(0) for ETH) of the exiting output |
| processExitInitiator | address payable | The processExits() initiator |

### startIFEBondSize

Retrieves the in-flight exit bond size

```js
function startIFEBondSize() public view
returns(uint128)
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|

### updateStartIFEBondSize

Updates the in-flight exit bond size, taking two days to become effective.

```js
function updateStartIFEBondSize(uint128 newBondSize) public nonpayable onlyFrom
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|
| newBondSize | uint128 | The new bond size |

### piggybackBondSize

Retrieves the piggyback bond size

```js
function piggybackBondSize() public view
returns(uint128)
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|

### updatePiggybackBondSize

Updates the piggyback bond size, taking two days to become effective

```js
function updatePiggybackBondSize(uint128 newBondSize, uint128 newExitBountySize) public nonpayable onlyFrom
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|
| newBondSize | uint128 | The new bond size |
| newExitBountySize | uint128 |  |

### processInFlightExitBountySize

Retrieves the process IFE bounty size

```js
function processInFlightExitBountySize() public view
returns(uint128)
```

**Arguments**

| Name        | Type           | Description  |
| ------------- |------------- | -----|

## Contracts

* [Address](Address.md)
* [Bits](Bits.md)
* [BlockController](BlockController.md)
* [BlockModel](BlockModel.md)
* [BondSize](BondSize.md)
* [ECDSA](ECDSA.md)
* [Erc20DepositVerifier](Erc20DepositVerifier.md)
* [Erc20Vault](Erc20Vault.md)
* [EthDepositVerifier](EthDepositVerifier.md)
* [EthVault](EthVault.md)
* [ExitableTimestamp](ExitableTimestamp.md)
* [ExitGameController](ExitGameController.md)
* [ExitGameRegistry](ExitGameRegistry.md)
* [ExitId](ExitId.md)
* [ExitPriority](ExitPriority.md)
* [FailFastReentrancyGuard](FailFastReentrancyGuard.md)
* [FeeClaimOutputToPaymentTxCondition](FeeClaimOutputToPaymentTxCondition.md)
* [FeeExitGame](FeeExitGame.md)
* [FungibleTokenOutputModel](FungibleTokenOutputModel.md)
* [GenericTransaction](GenericTransaction.md)
* [IERC20](IERC20.md)
* [IErc20DepositVerifier](IErc20DepositVerifier.md)
* [IEthDepositVerifier](IEthDepositVerifier.md)
* [IExitProcessor](IExitProcessor.md)
* [ISpendingCondition](ISpendingCondition.md)
* [IStateTransitionVerifier](IStateTransitionVerifier.md)
* [Math](Math.md)
* [Merkle](Merkle.md)
* [Migrations](Migrations.md)
* [MoreVpFinalization](MoreVpFinalization.md)
* [OnlyFromAddress](OnlyFromAddress.md)
* [OnlyWithValue](OnlyWithValue.md)
* [OutputId](OutputId.md)
* [Ownable](Ownable.md)
* [PaymentChallengeIFEInputSpent](PaymentChallengeIFEInputSpent.md)
* [PaymentChallengeIFENotCanonical](PaymentChallengeIFENotCanonical.md)
* [PaymentChallengeIFEOutputSpent](PaymentChallengeIFEOutputSpent.md)
* [PaymentChallengeStandardExit](PaymentChallengeStandardExit.md)
* [PaymentDeleteInFlightExit](PaymentDeleteInFlightExit.md)
* [PaymentEip712Lib](PaymentEip712Lib.md)
* [PaymentExitDataModel](PaymentExitDataModel.md)
* [PaymentExitGame](PaymentExitGame.md)
* [PaymentExitGameArgs](PaymentExitGameArgs.md)
* [PaymentInFlightExitModelUtils](PaymentInFlightExitModelUtils.md)
* [PaymentInFlightExitRouter](PaymentInFlightExitRouter.md)
* [PaymentInFlightExitRouterArgs](PaymentInFlightExitRouterArgs.md)
* [PaymentOutputToPaymentTxCondition](PaymentOutputToPaymentTxCondition.md)
* [PaymentPiggybackInFlightExit](PaymentPiggybackInFlightExit.md)
* [PaymentProcessInFlightExit](PaymentProcessInFlightExit.md)
* [PaymentProcessStandardExit](PaymentProcessStandardExit.md)
* [PaymentStandardExitRouter](PaymentStandardExitRouter.md)
* [PaymentStandardExitRouterArgs](PaymentStandardExitRouterArgs.md)
* [PaymentStartInFlightExit](PaymentStartInFlightExit.md)
* [PaymentStartStandardExit](PaymentStartStandardExit.md)
* [PaymentTransactionModel](PaymentTransactionModel.md)
* [PaymentTransactionStateTransitionVerifier](PaymentTransactionStateTransitionVerifier.md)
* [PlasmaFramework](PlasmaFramework.md)
* [PosLib](PosLib.md)
* [PriorityQueue](PriorityQueue.md)
* [Protocol](Protocol.md)
* [Quarantine](Quarantine.md)
* [RLPReader](RLPReader.md)
* [SafeERC20](SafeERC20.md)
* [SafeEthTransfer](SafeEthTransfer.md)
* [SafeMath](SafeMath.md)
* [SpendingConditionRegistry](SpendingConditionRegistry.md)
* [Vault](Vault.md)
* [VaultRegistry](VaultRegistry.md)
