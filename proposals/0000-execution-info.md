---
Title: Execution Info
Number: 0
Status: Draft
Version: 0
Authors:
 - Daniel Shiposha
Created: YYYY-MM-DD
Impact: Low
Requires:
Replaces:
---

## Summary

This RFC proposes several enhancements to XCM:
* a sender chain will be able to compute the correct weight of an XCM program to be executed on a target chain.
* a sender chain will be able to convert the computed weight to a currency suitable to pay fees on the target chain.
* a sender chain will be able to check if the XCM program will exceed the max weight on the target chain, preventing its sending in this case.

## Motivation

When sending an XCM program, the sender chain can't know beforehand the correct weight of the XCM program. And also, it can't know the currencies in which it can pay the fees and can't know the right amount of a suitable currency to execute the program.
This leads to guessing: either the sender chain itself sets an arbitrary weight limit and fees in the `BuyExecution` command, or the user initiating the sending of an XCM program is forced to guess the correct values.

This is generally okay for transferring fungible assets. However, it can become an issue in the following scenarios:
* Transferring multiple currencies - one must choose one currency to pay execution fees. However, the amount of the selected currency could be small (i.e., the guess could be wrong), leading to trapping all the assets.
* Transferring NFTs - the same situation as with multiple currencies. One must choose a currency to pay fees since it is impossible to pay fees by an NFT.
* General interoperability - since the sender chain can't know the exact weight of an XCM program and can't set the correct execution fees, creating a **reliable and convenient-to-use** cross-chain logic such as NFT metadata synchronization is impossible. E.g., one chain could provide access control logic for an NFT's attributes, and the other stores the NFT. Or there can be even more complex cross-chain logic not related to NFTs. The main obstacle here is guessing the correct weight/fee values.

This RFC proposes a possible way to solve this via new instructions and a new notion of an instruction pattern.

## Specification

The specification uses the following terms:
1. The **subscriber chain** is the chain that subscribed for **execution info** updates from a **response chain**.
2. The **response chain** is the chain that sends **execution info** updates.
3. The **execution info** is information about XCM execution on the **response chain**.
    It comprises three parts:
    * **payment methods info**
    * max weight of an XCM program
    * **instruction pattern reports**
4. The **payment methods info** is information about assets suitable to pay execution fees on the **response chain**.
    Each payment method contains:
    * an `AssetId` identifies a currency in which it is possible to pay fees
    * a formula to convert a program weight to fee in the corresponding currency
    ```rust
    #[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
    struct PaymentMethod {
        asset_id: AssetId,
        fee_polynomial: FeePolynomial<u128>,
    }
    ```
5. An **instruction pattern** is a pattern of an XCM instruction describing a set of the instruction's parameters' values.
    Each **instruction pattern** is unique and can be identified by its hash. Also, **instruction patterns** are ordered from more-specific to less-specific.
    See [the corresponding section](#instruction-pattern) for details.
6. **Instruction pattern report** results from weighing a particular **instruction pattern** on the **response chain**. The pattern is identified by its hash. The pattern could be either valid or invalid. If it is valid, the result will carry the pattern's weight. If it is invalid, the result will carry an XCM error. 
    ```rust
    #[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
    struct PatternReport {
        hash: XcmHash,
        result: Result<Weight, XcmError>,
    }
    ```

### Instruction pattern

#### Introduction

Each chain supporting XCM has a Weigher for XCM programs. The Weigher takes a program as a parameter and computes the program's weight. It does that by calculating the weight of each program instruction and then adding those weights together.

However, the Weigher usually considers only a subset of the instruction parameters. For instance, [Asset Hub's XCM Weigher](https://github.com/paritytech/cumulus/blob/26d725762c08b2b31fe40ae0ca0c16969f91ee2b/parachains/runtimes/assets/asset-hub-polkadot/src/weights/xcm/mod.rs#L53-L63) only accounts the number of assets in the `WithdrawAsset` instruction, but it doesn't care what assets are in the list. The Weigher can even ignore all instruction parameters and return the default weight as [Asset Hub's weighing implementation of the `ReportHolding` instruction](https://github.com/paritytech/cumulus/blob/26d725762c08b2b31fe40ae0ca0c16969f91ee2b/parachains/runtimes/assets/asset-hub-polkadot/src/weights/xcm/mod.rs#L119-L121). Nonetheless, the Weigher could consider all the instruction parameters or a different subset of the parameters. Also, the Weigher could utilize branching to compute the weight of some special cases depending on one or several parameters' values.

Given that the Weigher (in general) considers a subset of the instruction's parameters, we can conclude that the Weigher computes the weight not for a particular instruction with definite parameters but for a predefined set of instructions. For instance, for the `WithdrawAsset` instruction, the Asset Hub's Weigher computes the weight for the set of instructions defined by the following pattern: `WithdrawAsset(vec![<any-multiasset>, ...])`.

Except for Weighers, the patterns arise in inter-chain communication. Chains (usually) don't send unique-structured XCM programs. Instead, they send programs that can be grouped into several structure categories, e.g., reserve-based transfer programs. Also, chains don't use every XCM instruction. They use only those that appear in the said program categories. Since sender chains know what instruction patterns they will send, if they knew how to weigh the patterns according to the Weigher of the target chain, they could compute the correct weight of an entire XCM program before sending it.

Since we can use patterns on both sides to know the correct weight of an XCM program, we could arrange weight info communication between chains if we could express these patterns. As such, this RFC proposes a way to express XCM instruction patterns and proposes new instructions to work with them.

#### Instruction pattern specification

```rust
// Note: to disambiguate patterns, all lists (such as a vector inside the `MultiAssetsPattern`) must be sorted.

#[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub enum Specificity<T> {
    Definite(T), // The order matters. The most specific variant must be before the less specific.
    Any,
}

#[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
enum InstructionPattern<Call> {
    WithdrawAsset(Specificity<MultiAssetsPattern>),
    ReserveAssetDeposited(Specificity<MultiAssetsPattern>),
    ReceiveTeleportedAsset(Specificity<MultiAssetsPattern>),
    QueryResponse {
        // query_id shouldn't affect the weight
        response: Specificity<ResponsePattern>,
        // max_weight: shouldn't affect the weight
        // querier: shouldn't affect the weight
    },
    TransferAsset {
        assets: Specificity<MultiAssetsPattern>,
        beneficiary: Specificity<MultiLocationPattern>
    },
    TransferReserveAsset {
        assets: Specificity<MultiAssetsPattern>,
        dest: Specificity<MultiLocationPattern>,
        xcm: Specificity<XcmPattern<()>>
    },
    Transact {
        origin_kind: Specificity<OriginKind>,
        require_weight_at_most: Specificity<Weight>,
        call: DoubleEncoded<Call>
    },

    // `HrmpNewChannelOpenRequest` arguments shouldn't affect its weight
    HrmpNewChannelOpenRequest,

    // `HrmpChannelAccepted` arguments shouldn't affect its weight
    HrmpChannelAccepted,

    // `HrmpChannelClosing` arguments shouldn't affect its weight
    HrmpChannelClosing,

    ClearOrigin,

    // `DescendOrigin` arguments shouldn't affect its weight
    DescendOrigin,

    // `query_id` shouldn't affect the weight
    // `max_weight` shouldn't affect the weight
    ReportError(Specificity<MultiLocationPattern>),

    DepositAsset {
        assets: MultiAssetFilterPattern,
        beneficiary: Specificity<MultiLocationPattern>
    },
    DepositReserveAsset {
        assets: MultiAssetFilterPattern,
        dest: Specificity<MultiLocationPattern>,
        xcm: Specificity<XcmPattern<()>>
    },
    ExchangeAsset {
        give: MultiAssetFilterPattern,
        want: Specificity<MultiAssetsPattern>,
        maximal: Specificity<bool>
    },
    InitiateReserveWithdraw {
        assets: MultiAssetFilterPattern,
        reserve: Specificity<MultiLocationPattern>,
        xcm: Specificity<XcmPattern<()>>
    },
    InitiateTeleport {
        assets: MultiAssetFilterPattern,
        dest: Specificity<MultiLocationPattern>,
        xcm: Specificity<XcmPattern<()>>
    },

    // `query_id` shouldn't affect the weight
    // `max_weight` shouldn't affect the weight
    ReportHolding {
        response_dest: Specificity<MultiLocationPattern>,
        assets: MultiAssetFilterPattern
    },
    
    BuyExecution {
        fees: Specificity<MultiAssetPattern>
        // `weight_limit` shouldn't affect the weight of `BuyExecution`
    },
    RefundSurplus,
    SetErrorHandler(Specificity<XcmPattern<Call>>),
    SetAppendix(Specificity<XcmPattern<Call>>),
    ClearError,
    ClaimAsset {
        assets: Specificity<MultiAssetsPattern>,
        ticket: Specificity<MultiLocationPattern>
    },
    
    // `Trap` arguments shouldn't affect its weight
    Trap,
    
    // `SubscribeVersion` arguments shouldn't affect its weight
    SubscribeVersion,
    
    UnsubscribeVersion,
    BurnAsset(Specificity<MultiAssetsPattern>),
    ExpectAsset(Specificity<MultiAssetsPattern>),

    // `ExpectOrigin` arguments shouldn't affect its weight
    ExpectOrigin,

    // `ExpectError` arguments shouldn't affect its weight
    ExpectError,

    // `ExpectTransactStatus` arguments shouldn't affect its weight
    ExpectTransactStatus,

    QueryPallet {
        // Since the `MaxPalletNameLen` is defined, we can always know
        // the weight upper bound for this instruction regardless the `module_name`

        // `query_id` shouldn't affect the weight
        // `max_weight` shouldn't affect the weight
        response_dest: Specificity<MultiLocationPattern>
    },

    // Since the `MaxPalletNameLen` is defined, we can always know
    // the weight upper bound for this instruction regardless of its arguments
    ExpectPallet,

    // `query_id` shouldn't affect the weight
    // `max_weight` shouldn't affect the weight
    ReportTransactStatus(Specificity<MultiLocationPattern>),

    ClearTransactStatus,

    // `UniversalOrigin` arguments shouldn't affect its weight
    UniversalOrigin,

    ExportMessage {
        network: Specificity<NetworkId>,
        destination: Specificity<JunctionsPattern>,
        xcm: Specificity<XcmPattern<()>>
    },
    LockAsset {
        asset: Specificity<MultiAssetPattern>,
        unlocker: Specificity<MultiLocationPattern>
    },
    UnlockAsset {
        asset: Specificity<MultiAssetPattern>,
        target: Specificity<MultiLocationPattern>
    },
    NoteUnlockable {
        asset: Specificity<MultiAssetPattern>,
        owner: Specificity<MultiLocationPattern>
    },
    RequestUnlock {
        asset: Specificity<MultiAssetPattern>,
        locker: Specificity<MultiLocationPattern>
    },
 
    // `SetFeesMode` arguments shouldn't affect its weight
    SetFeesMode,

    SetTopic([u8; 32]),

    ClearTopic,

    // `AliasOrigin` arguments shouldn't affect its weight
    AliasOrigin,

    // `UnpaidExecution` arguments shouldn't affect its weight
    UnpaidExecution,

    // New instructions
    SubscribeExecutionInfo {
        // `query_id` shouldn't affect the weight
        // `max_response_weight` shouldn't affect the weight
        patterns: Specificity<Vec<InstructionPattern<()>>>,
    },
    UnsubscribePatterns(Vec<XcmHash>),
    UnsubscribeExecutionInfo,
}

#[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub struct XcmPattern<Call>(pub Vec<InstructionPattern<Call>>);

pub type JunctionPattern = Specificity<Junction>;

#[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub enum JunctionsPattern {
        Here,
        X1(JunctionPattern),
        X2(JunctionPattern, JunctionPattern),
        X3(JunctionPattern, JunctionPattern, JunctionPattern),
        X4(JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern),
        X5(JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern),
        X6(JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern),
        X7(JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern),
        X8(JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern, JunctionPattern),
}

#[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub enum MultiLocationPattern {
    // The order matters. The most specific variant must be before the less specific.
    Full {
        parents: u8,
        interior: JunctionsPattern,
    },
    Parents(u8),
    Junctions(JunctionsPattern),
}

#[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub enum AssetIdPattern {
    Concrete(MultiLocationPattern),
    Abstract([u8; 32]),
}

#[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub enum FungibilityPattern {
    Fungible(Specificity<u128>),
    NonFungible(Specificity<AssetInstance>),
}

#[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub enum MultiAssetPattern {
    // The order matters. The most specific variant must be before the less specific.
    Full {
        id: AssetIdPattern,
        fun: FungibilityPattern,
    },
    Id(AssetIdPattern),
    Fun(FungibilityPattern),
}

#[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub struct MultiAssetsPattern(pub Vec<Specificity<MultiAssetPattern>>);

#[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub enum WildMultiAssetPattern {
    // The order matters. The most specific variant must be before the less specific.
    AllOfCounted {
        id: AssetIdPattern,
        fun: Specificity<WildFungibility>,
        count: u32,
    },
    AllOf {
        id: AssetIdPattern,
        fun: Specificity<WildFungibility>,
    },
    AllCounted(u32),
    All,
}

#[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub enum MultiAssetFilterPattern {
    // The order matters. The most specific variant must be before the less specific.
    Collection(MultiAssetsPattern),
    Wild(WildMultiAssetPattern)
}

#[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub enum ResponsePattern {
    Null,
    Assets(MultiAssetsPattern),

    // `ExecutionResult` argument shouldn't affect the weight
    ExecutionResult,

    // `Version` argument shouldn't affect the weight
    Version,

    // Since the `MaxPalletNameLen` is defined, we can always know
    // the weight upper bound for this response regardless of actual pallet info.
    // We only need the number of received `PalletInfo` objects.
    PalletsInfo(u32),

    // `DispatchResult` argument shouldn't affect the weight
    DispatchResult,

    // New variant
    ExecutionInfoUpdate {
        payment_methods: Option<Vec<Specificity<PaymentMethod>>>,
        // `max_weight` shouldn't affect the weight
        pattern_reports: Option<Vec<Specificity<PatternReport>>>,
    },
}
```

##### InstructionId

The instruction ID enum is needed to group several patterns of one instruction kind.
It helps the subscriber chain Weigher to find the correct instruction patterns for a given XCM instruction. It identifies the XCM instruction kind omitting its parameters.

```rust
pub enum InstructionId {
    WithdrawAsset,
    ReserveAssetDeposited,
    // ... snip ...
    AliasOrigin,
}

impl<C> Instruction<C> {
    pub fn id(&self) -> InstructionId {
        match self {
            Self::WithdrawAsset(_) => InstructionId::WithdrawAsset,
            Self::ReserveAssetDeposited(_) => InstructionId::ReserveAssetDeposited,
            // ... snip ...
            Self::AliasOrigin(_) => InstructionId::AliasOrigin,
        }
    }
}
```

### New instruction: `SubscribeExecutionInfo`

```rust
SubscribeExecutionInfo {
    query_id: QueryId,
    max_response_weight: Weight,
    patterns: Vec<InstructionPattern<()>>,
}
```

During the execution of the `SubscribeExecutionInfo` instruction, the response chain will compute the weights of each instruction pattern specified in the `patterns` field, and then it will send the `QueryResponse` with the `ExecutionInfoUpdate` response containing all the pattern reports, max XCM weight and payment methods.

After sending initial execution info, the response chain can later send updates on any part of the execution info.

### New `Response` variant: `ExecutionInfoUpdate`

```rust
enum Response {
    // ... snip ...
    ExecutionInfoUpdate {
        payment_methods: Option<Vec<PaymentMethod>>,
        max_weight: Option<Weight>,
        pattern_reports: Option<Vec<PatternReport>>,
    },
}
```

* The `payment_methods` field can contain an update of payment methods.
A payment method identifies a currency in which one can pay fees for execution on the response chain. Also, it contains a polynomial formula to convert XCM weight into the corresponding currency.
    ```rust
    #[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
    struct PaymentMethod {
        asset_id: AssetId,
        fee_polynomial: FeePolynomial<u128>,
    }
    ```
    The `payment_methods` is a complete list of all suitable currencies. It means the response chain shall send a new list containing all the new methods to add or remove a payment method.

* The `max_weight` field can contain an update of the max weight of an XCM program on the response chain. This information can help the subscriber chain to check if an XCM program will exceed the max weight if sent (and hence the program will definitely fail), preventing this situation.

* The `pattern_reports` field can contain an update of some pattern reports. Unlike payment methods, this list includes updates only for specified items (i.e., the reports not specified in this list will not be updated/removed).<br/>
A pattern report identifies the corresponding instruction pattern by its hash (the `hash` field) and contains the result of weighing the instruction pattern in the `result` field.
    ```rust
    #[derive(Encode, Decode, Clone, PartialEq, Eq, PartialOrd, Ord)]
    struct PatternReport {
        hash: XcmHash,
        result: Result<Weight, XcmError>,
    }
    ```

### New instruction: `UnsubscribePatterns`

To unsubscribe from specific pattern reports, the subscriber chain can use the new `UnsubscribePatterns` instruction:
```rust
UnsubscribePatterns(Vec<XcmHash>)
```

The parameter of this instruction specifies a list of instruction patterns' hashes in which the subscriber chain is not interested anymore. 

### New instruction: `UnsubscribeExecutionInfo`

To unsubscribe from all execution info updates, the subscriber chain can use the new `UnsubscribeExecutionInfo` instruction:
```rust
UnsubscribeExecutionInfo
```

### Example: Using patterns to weigh a program on both sides

#### Implementing a Weigher on the response chain

To weigh both instruction patterns and real XCM instructions, the response chain can implement weighing the instruction patterns only and convert real instructions to instruction patterns and then weigh these patterns.

The conversion function should convert an instruction to a pattern with the same specificity as the pattern used by the Weigher. For instance, if the Weigher doesn't care what kind of assets are specified in the `WithdrawAsset` pattern, the conversion function should convert the `WithdrawAsset` instruction to the corresponding pattern but with `Sepcificity::Any` in place of each asset as shown in the following section.

##### Converting XCM instructions to definite patterns

```rust
fn inst_to_pattern<Call>(inst: &Instruction<Call>) -> InstructionPattern<Call> {
    match inst {
        Instruction::WithdrawAsset(assets) => {
            InstructionPattern::WithdrawAsset(
                Specificity::Definite(MultiAssetsPattern(vec![Specificity::Any; assets.0.len()]))
            )
        },
        Instruction::ReserveAssetDeposited(assets) => {
            InstructionPattern::ReserveAssetDeposited(
                Specificity::Definite(MultiAssetsPattern(vec![Specificity::Any; assets.0.len()]))
            )
        },
        Instruction::ReceiveTeleportedAsset(assets) => {
            InstructionPattern::ReceiveTeleportedAsset(Specificity::Any)
        },
        // ... snip ...
        Instruction::ClearOrigin => InstructionPattern::ClearOrigin,
        // ... snip ...
        Instruction::DepositAsset {
            assets: MultiAssetFilter::Wild(WildMultiAsset::AllCounted(count)),
            ..
        } => {
            InstructionPattern::DepositAsset {
                assets: MultiAssetFilterPattern::Wild(WildMultiAssetPattern::AllCounted(*count)),
                beneficiary: Specificity::Any
            }
        },
        // ... snip ...
        Instruction::BuyExecution { .. } => InstructionPattern::BuyExecution { fees: Specificity::Any },
        // ... snip ...
    }
}
```

##### Weighing instruction patterns on the response chain

```rust
fn weigh_inst_pattern<Call>(pattern: InstructionPattern<Call>) -> Result<Weight, XcmError> {
    match pattern {
        InstructionPattern::WithdrawAsset(assets) => match assets {
            Specificity::Definite(assets) => {
                Ok(WITHDRAW_ASSET_WEIGHT.saturating_mul(assets.0.len() as u64))
            },
            Specificity::Any => Ok(Weight::MAX) /* Could be some reasonable upper bound */,
        },
        InstructionPattern::ReserveAssetDeposited(assets) => match assets {
            Specificity::Definite(assets) => {
                Ok(RESERVE_ASSET_DEPOSITED_WEIGHT.saturating_mul(assets.0.len() as u64))
            },
            Specificity::Any => Ok(Weight::MAX) /* Could be some reasonable upper bound */,
        },
        InstructionPattern::ReceiveTeleportedAsset(assets) => Err(XcmError::Unimplemented) /* the response chain doesn't trust anyone */,
        // ... snip ...
        InstructionPattern::ClearOrigin => Ok(CLEAR_ORIGIN_WEIGHT),
        // ... snip ...
        InstructionPattern::DepositAsset {
            assets: MultiAssetFilterPattern::Wild(WildMultiAssetPattern::AllCounted(count)),
            ..
        } => Ok(DEPOSIT_ASSET_WEIGHT.saturating_mul(count as u64)),
        // ... snip ...
        InstructionPattern::BuyExecution { .. } => Ok(BUY_EXECUTION_WEIGHT),
        // ...  snip ...
    }
}

fn weigh_program<Call>(xcm: &Xcm<Call>) -> Result<Weight, XcmError> {
    xcm.0.iter()
        .map(inst_to_pattern)
        .try_fold(Weight::zero(), |acc, p| {
            let pattern_weight = weigh_inst_pattern(p)?;
            Ok(acc.saturating_add(pattern_weight))
        })
}
```

Now, the response chain can weigh an instruction pattern so it can send the weight info to any subscriber chain about any requested patterns.
The response chain can also weigh any XCM program by converting its instructions to corresponding patterns and utilizing the same weight function as with patterns to ensure the pattern weights and program weights are always the same.

#### Implementing a Weigher on the subscriber chain 

Unlike the Weigher on the response chain, the Weigher on the subscriber chain doesn't know how to weigh patterns since it doesn't (and can't) share the same code.
Instead, the subscriber chain's Weigher has to rely on the information the response chain provides. Also, it weighs only instructions that the subscriber chain will actually use, ignoring everything else.

Let's assume that the subscriber chain will send to the response chain only those XCM programs that will perform a reserve-based transfer to another chain. I.e., the response chain will act as a reserve chain. It means the subscriber chain will only use the following instructions on the response chain:
* `WithdrawAsset`
* `ClearOrigin`
* `BuyExecution`
* `DepositReserveAsset`

However, the weight of the `WithdrawAsset` and the `DepositReserveAsset` depends on their parameters (the count of assets). The subscriber chain can subscribe to the respective patterns for these instructions to gather the correct weight.

Here is how the subscriber chain could store the weight info and use it to weigh a specific XCM instruction:
```rust
#[pallet::storage]
pub type XcmMaxWeight<T: Config> = StorageMap<
    Hasher = /* <hasher> */,
    Key = MultiLocation, // MultiLocation of a response chain
    Value = Weight,
    QueryKind = ValueQuery
>;

#[pallet::storage]
pub type XcmPatternWeight<T: Config> = StorageDoubleMap<
    Hasher1 = /* <hasher> */,
    Key1 = MultiLocation, // MultiLocation of a response chain
    Hasher2 = Identity,
    Key2 = XcmHash,
    Value = Weight,
    QueryKind = OptionQuery
>;

#[pallet::storage]
pub type XcmInstructionPatterns<T: Config> = StorageMap<
    Hasher = /* <hasher> */,
    Key = InstructionId, // See the definition in the "InstructionId" section
    Value = Vec<(InstructionPattern<T::RuntimeCall>, XcmHash)>,
    QueryKind = OptionQuery 
>;
```

In the `XcmInstructionPatterns` storage, the subscriber chain stores all the instruction patterns that it will ever use. For instance, for `WithdrawAsset` instruction it will store the following:
```
KEY   = InstructionId::WithdrawAsset
VALUE = vec![
    (
        WithdrawAsset(Specificity::Definite(
            MultiAssetsPattern(vec![Specificity::Any; 1])
        )),
        {pattern-hash}
    ),
    (
        WithdrawAsset(Specificity::Definite(
            MultiAssetsPattern(vec![Specificity::Any; 2])
        )),
        {pattern-hash}
    ),
    (
        WithdrawAsset(Specificity::Definite(
            MultiAssetsPattern(vec![Specificity::Any; 3])
        )),
        {pattern-hash}
    ),
]
```
It is crucial to store patterns for any given instruction ID in the correct order. The patterns under the instruction ID must be ordered from most specific to less specific. The ordering is intrinsic to the instruction pattern definition itself. See the `InstructionPattern` definition - it derives the `Ord` trait. Ordering the patterns ensures the correct pattern matching (and hence the correct weighing).

Here is a possible implementation of the subscriber chain's Weigher:
```rust
fn weigh_foreign_inst<T: Config>(target_chain: &MultiLocation, inst: &Instruction<()>) -> Result<Weight, XcmError> {
    let inst_id = inst.id(); // See the definition in the "InstructionId" section
    let inst_patterns = <XcmInstructionPatterns<T>>::get(&inst_id)
        .ok_or(XcmError::Unimplemented)?;

    for (pattern, pattern_hash) in inst_patterns.iter() {
        // The patterns are ordered from most specific to less specific.
        // The most specific pattern matches first.
        if inst.matches(pattern) {
            return <XcmPatternWeight<T>>::get(target_chain, pattern_hash)
                .ok_or(XcmError::WeightNotComputable);
        }
    }

    Err(XcmError::WeightNotComputable)
}

fn weigh_foreign_program<T: Config>(target_chain: &MultiLocation, xcm: &Xcm<()>) -> Result<Weight, XcmError> {
    let program_weight = xcm.0.iter().try_fold(Weight::zero(), |acc, inst| {
        let pattern_weight = weigh_foreign_inst::<T>(target_chain, inst)?;
        Ok(acc.saturating_add(pattern_weight))
    })?;

    let max_weight = <XcmMaxWeight<T>>::get(target_chain);
    if program_weight.all_lt(max_weight) {
        Ok(program_weight)
    } else {
        Err(XcmError::WeightLimitReached(max_weight))
    }
}
```

#### Example of subscribing and weighing an XCM message before sending

##### Subscription for execution info updates

The subscriber chain sends the following XCM program to subscribe to the execution info updates from the response chain (assuming it won't send more than 3 assets at once):
```rust
SubscribeExecutionInfo {
    query_id: <subscription-query-id>,
    max_response_weight: <max-weight>,
    patterns: vec![
        WithdrawAsset(Specificity::Definite(
            MultiAssetsPattern(vec![Specificity::Any; 1])
        )),
        WithdrawAsset(Specificity::Definite(
            MultiAssetsPattern(vec![Specificity::Any; 2])
        )),
        WithdrawAsset(Specificity::Definite(
            MultiAssetsPattern(vec![Specificity::Any; 3])
        )),

        ClearOrigin,

        BuyExecution { fees: Specificity::Any },

        DepositReserveAsset {
            assets: MultiAssetFilterPattern::Wild(WildMultiAssetPattern::AllCounted(1)),
            dest: Specificity::Any,
            xcm: Specificity::Any,
        },
        DepositReserveAsset {
            assets: MultiAssetFilterPattern::Wild(WildMultiAssetPattern::AllCounted(2)),
            dest: Specificity::Any,
            xcm: Specificity::Any,
        },
        DepositReserveAsset {
            assets: MultiAssetFilterPattern::Wild(WildMultiAssetPattern::AllCounted(3)),
            dest: Specificity::Any,
            xcm: Specificity::Any,
        },
    ],
}
```

##### ExecutionInfoUpdate response

The response chain will send the query response with the execution info update:
```rust
QueryResponse {
    response: Response::ExecutionInfoUpdate {
        // Example payment methods:
        // the subscriber chain can pay execution fees in DOT or USDT or in something else
        payment_methods: Some(vec![
            PaymentMethodInfo {
                asset_id: Parent.into(),
                fee_polynomial: <DOT fee polynomial>,
            },
            PaymentMethodInfo {
                asset_id: <USDT MultiLocation>,
                fee_polynomial: <USDT fee polynomial>,
            },
            // ... etc ...
        ]),
        max_weight: Some(Weight::from_parts(<ref-time>, <pov-size>)),
        pattern_reports: Some(vec![
            // NOTE: The pattern reports may be in any order.
            // NOTE: the `hash!` macro is used for illustrative purpose only.
            PatternReport {
                hash: hash!(ClearOrigin),
                weight: Ok(Weight::from_parts(<ref-time>, <pov-size>)),
            },
            PatternReport {
                hash: hash!(BuyExecution { fees: Specificity::Any }),
                weight: Ok(Weight::from_parts(<ref-time>, <pov-size>)),
            },
            PatternReport {
                hash: hash! {
                    WithdrawAsset(Specificity::Definite(
                        MultiAssetsPattern(vec![Specificity::Any; 1])
                    ))
                },
                weight: Ok(Weight::from_parts(<ref-time>, <pov-size>)),
            },
            PatternReport {
                hash: hash! {
                    WithdrawAsset(Specificity::Definite(
                        MultiAssetsPattern(vec![Specificity::Any; 2])
                    ))
                },
                weight: Ok(Weight::from_parts(<ref-time>, <pov-size>)),
            },
            PatternReport {
                hash: hash! {
                    WithdrawAsset(Specificity::Definite(
                        MultiAssetsPattern(vec![Specificity::Any; 3])
                    ))
                },
                weight: Ok(Weight::from_parts(<ref-time>, <pov-size>)),
            },
            PatternReport {
                hash: hash! {
                    DepositReserveAsset {
                        assets: MultiAssetFilterPattern::Wild(WildMultiAssetPattern::AllCounted(1)),
                        dest: Specificity::Any,
                        xcm: Specificity::Any,
                    }
                },
                weight: Ok(Weight::from_parts(<ref-time>, <pov-size>)),
            },
            PatternReport {
                hash: hash! {
                    DepositReserveAsset {
                        assets: MultiAssetFilterPattern::Wild(WildMultiAssetPattern::AllCounted(2)),
                        dest: Specificity::Any,
                        xcm: Specificity::Any,
                    }
                },
                weight: Ok(Weight::from_parts(<ref-time>, <pov-size>)),
            },
            PatternReport {
                hash: hash! {
                    DepositReserveAsset {
                        assets: MultiAssetFilterPattern::Wild(WildMultiAssetPattern::AllCounted(3)),
                        dest: Specificity::Any,
                        xcm: Specificity::Any,
                    }
                },
                weight: Ok(Weight::from_parts(<ref-time>, <pov-size>)),
            },
        ])
    },
    query_id: <subscribe-query-id>,
    max_weight: <response-handle-weight>,
    querier: Some(SUBSCRIBER_CHAIN_MULTILOCATION),
}
```

Upon receiving the execution info update, the subscriber chain will update the corresponding storages:
```rust
let response_chain = <the response chain ML>;
let update = <execution-info-update>;

if let Some(new_payment_methods) = update.payment_methods {
    // Store them somewhere for future use.
    // A payment method defines a conversion between XCM weight and the corresponding currency.
}

if let Some(new_max_weight) = update.max_weight {
    <XcmMaxWeight<T>>::insert(response_chain, new_max_weight);
}

if let Some(pattern_reports) = update.pattern_reports {
    for report in pattern_reports {
        if let Ok(weight) = report.result {
            <XcmPatternWeight<T>>::insert(response_chain, report.hash, weight);
        }
    }
}
```

After receiving the execution update, the subscriber chain can weigh any XCM program to be sent to the response chain beforehand using the Weigher for foreign XCM programs (the `weigh_foreign_program`).
After computing the weight, the subscriber chain can convert it to a currency suitable for paying execution fees on the response chain. Knowing the exact amount of execution fees, the subscriber chain can verify if the program is correctly constructed, i.e., it will possess the correct amount of assets and pay enough fees.
Or the subscriber chain itself could modify a program in such a way as to guarantee the correct amount is withdrawn from the Holding Register, and the correct fees are set in the `BuyExecution` instruction.

### Example: possible pattern matching implementation

<details>
    <summary>The code</summary>

```rust
pub trait PatternMatch<P> {
    fn matches(&self, pattern: &P) -> bool;
}

impl<P, T: PatternMatch<P>> PatternMatch<Specificity<P>> for T {
    fn matches(&self, pattern: &Specificity<P>) -> bool {
        match pattern {
            Specificity::Definite(p) => self.matches(p),
            Specificity::Any => true,
        }
    }
}

impl<P, T: PatternMatch<P>> PatternMatch<Vec<P>> for Vec<T> {
    fn matches(&self, pattern: &Vec<P>) -> bool {
        if self.len() == pattern.len() {
            self.iter().zip(pattern.iter()).all(|(item, pattern)| item.matches(pattern))
        } else {
            false
        }
    }
}

impl<P, T: PatternMatch<P>> PatternMatch<Option<P>> for Option<T> {
    fn matches(&self, pattern: &Option<P>) -> bool {
        match (self, pattern) {
            (Some(v), Some(p)) => v.matches(p),
            (None, None) => true,
            _ => false,
        }
    }
}

macro_rules! impl_eq_pattern {
    ($($ty:ty),* $(,)?) => {
        $(
            impl PatternMatch<$ty> for $ty {
                fn matches(&self, pattern: &$ty) -> bool {
                    self == pattern
                }
            }
        )*
    };
}

impl_eq_pattern! {
    Junction,
    WildFungibility,
    NetworkId,
    Weight,
    PaymentMethod,
    PatternReport,
    u128,
    AssetInstance,
    OriginKind,
    bool,
    Option<(u32, Error)>,
    MaybeErrorCode,
}

impl PatternMatch<JunctionsPattern> for Junctions {
    fn matches(&self, pattern: &JunctionsPattern) -> bool {
        match (self, pattern) {
            (Junctions::Here, JunctionsPattern::Here) => true,
            (Junctions::X1(j1), JunctionsPattern::X1(p1)) => j1.matches(p1),
            (Junctions::X2(j1, j2), JunctionsPattern::X2(p1, p2)) => {
                j1.matches(p1) && j2.matches(p2)
            },
            (Junctions::X3(j1, j2, j3), JunctionsPattern::X3(p1, p2, p3)) => {
                j1.matches(p1) && j2.matches(p2) && j3.matches(p3)
            },
            (Junctions::X4(j1, j2, j3, j4), JunctionsPattern::X4(p1, p2, p3, p4)) => {
                j1.matches(p1)
                && j2.matches(p2)
                && j3.matches(p3)
                && j4.matches(p4)
            },
            (Junctions::X5(j1, j2, j3, j4, j5), JunctionsPattern::X5(p1, p2, p3, p4, p5)) => {
                j1.matches(p1)
                && j2.matches(p2)
                && j3.matches(p3)
                && j4.matches(p4)
                && j5.matches(p5)
            },
            (Junctions::X6(j1, j2, j3, j4, j5, j6), JunctionsPattern::X6(p1, p2, p3, p4, p5, p6)) => {
                j1.matches(p1)
                && j2.matches(p2)
                && j3.matches(p3)
                && j4.matches(p4)
                && j5.matches(p5)
                && j6.matches(p6)
            },
            (Junctions::X7(j1, j2, j3, j4, j5, j6, j7), JunctionsPattern::X7(p1, p2, p3, p4, p5, p6, p7)) => {
                j1.matches(p1)
                && j2.matches(p2)
                && j3.matches(p3)
                && j4.matches(p4)
                && j5.matches(p5)
                && j6.matches(p6)
                && j7.matches(p7)
            },
            (Junctions::X8(j1, j2, j3, j4, j5, j6, j7, j8), JunctionsPattern::X8(p1, p2, p3, p4, p5, p6, p7, p8)) => {
                j1.matches(p1)
                && j2.matches(p2)
                && j3.matches(p3)
                && j4.matches(p4)
                && j5.matches(p5)
                && j6.matches(p6)
                && j7.matches(p7)
                && j8.matches(p8)
            },
            _ => false,
        }
    }
}

impl PatternMatch<MultiLocationPattern> for MultiLocation {
    fn matches(&self, pattern: &MultiLocationPattern) -> bool {
        match pattern {
            MultiLocationPattern::Full {
                parents,
                interior
            } => self.parents == *parents && self.interior.matches(interior),
            MultiLocationPattern::Parents(parents) => self.parents == *parents,
            MultiLocationPattern::Junctions(interior) => self.interior.matches(interior),
        }
    }
}

impl PatternMatch<AssetIdPattern> for AssetId {
    fn matches(&self, pattern: &AssetIdPattern) -> bool {
        match (self, pattern) {
            (Self::Concrete(location), AssetIdPattern::Concrete(pattern)) => location.matches(pattern),
            (Self::Abstract(self_id), AssetIdPattern::Abstract(id)) => self_id == id,
            _ => false,
        }
    }
}

impl PatternMatch<FungibilityPattern> for Fungibility {
    fn matches(&self, pattern: &FungibilityPattern) -> bool {
        match (self, pattern) {
            (Self::Fungible(amount), FungibilityPattern::Fungible(pattern)) => amount.matches(pattern),
            (Self::NonFungible(item), FungibilityPattern::NonFungible(pattern)) => item.matches(pattern),
            _ => false,
        }
    }
}

impl PatternMatch<MultiAssetPattern> for MultiAsset {
    fn matches(&self, pattern: &MultiAssetPattern) -> bool {
        match pattern {
            MultiAssetPattern::Full {
                id: id_pat,
                fun: fun_pat,
            } => self.id.matches(id_pat) && self.fun.matches(fun_pat),
            MultiAssetPattern::Id(pattern) => self.id.matches(pattern),
            MultiAssetPattern::Fun(pattern) => self.fun.matches(pattern),
        }
    }
}

impl PatternMatch<MultiAssetsPattern> for MultiAssets {
    fn matches(&self, pattern: &MultiAssetsPattern) -> bool {
        if self.0.len() == pattern.0.len() {
            self.0.iter().zip(pattern.0.iter()).all(|(asset, pattern)| asset.matches(pattern))
        } else {
            false
        }
    }
}

impl PatternMatch<WildMultiAssetPattern> for WildMultiAsset {
    fn matches(&self, pattern: &WildMultiAssetPattern) -> bool {
        match (self, pattern) {
            (
                Self::AllOfCounted {
                    id,
                    fun,
                    count,
                },
                WildMultiAssetPattern::AllOfCounted {
                    id: id_pat,
                    fun: fun_pat,
                    count: count_pat
                }
            ) => id.matches(id_pat) && fun.matches(fun_pat) && count == count_pat,
            (
                Self::AllOf { id, fun },
                WildMultiAssetPattern::AllOf { id: id_pat, fun: fun_pat }
            ) => id.matches(id_pat) && fun.matches(fun_pat),
            (Self::AllCounted(count), WildMultiAssetPattern::AllCounted(count_pat)) => count == count_pat,
            (Self::All, WildMultiAssetPattern::All) => true,
            _ => false,
        }
    }
}

impl PatternMatch<MultiAssetFilterPattern> for MultiAssetFilter {
    fn matches(&self, pattern: &MultiAssetFilterPattern) -> bool {
        match (self, pattern) {
            (Self::Definite(assets), MultiAssetFilterPattern::Collection(pattern)) => assets.matches(pattern),
            (Self::Wild(wild), MultiAssetFilterPattern::Wild(pattern)) => wild.matches(pattern),
            _ => false,
        }
    }
}

impl PatternMatch<ResponsePattern> for Response {
    fn matches(&self, pattern: &ResponsePattern) -> bool {
        match (self, pattern) {
            (Self::Null, ResponsePattern::Null) => true,
            (Self::ExecutionResult(_), ResponsePattern::ExecutionResult) => true,
            (Self::Version(_), ResponsePattern::Version) => true,
            (Self::PalletsInfo(infos), ResponsePattern::PalletsInfo(num)) => infos.len() == *num as usize,
            (Self::DispatchResult(_), ResponsePattern::DispatchResult) => true,
            (
                Self::ExecutionInfoUpdate {
                    payment_methods,
                    pattern_reports,
                    ..
                },
                ResponsePattern::ExecutionInfoUpdate {
                    payment_methods: payment_methods_pat,
                    pattern_reports: pattern_reports_pat,
                },
            ) => payment_methods.matches(payment_methods_pat)
                && pattern_reports.matches(pattern_reports_pat),
            _ => false,
        }
    }
}

impl<Call> PatternMatch<XcmPattern<Call>> for Xcm<Call> {
    fn matches(&self, pattern: &XcmPattern<Call>) -> bool {
        self.0.matches(&pattern.0)
    }
}

impl<C> PatternMatch<InstructionPattern<C>> for Instruction<C> {
    fn matches(&self, pattern: &InstructionPattern<C>) -> bool {
        match (self, pattern) {
            (
                Self::WithdrawAsset(assets),
                InstructionPattern::WithdrawAsset(pattern),
            ) => assets.matches(pattern),
            (
                Self::ReserveAssetDeposited(assets),
                InstructionPattern::ReserveAssetDeposited(pattern),
            ) => assets.matches(pattern),
            (
                Self::ReceiveTeleportedAsset(assets),
                InstructionPattern::ReceiveTeleportedAsset(pattern),
            ) => assets.matches(pattern),
            (
                Self::QueryResponse { response, .. },
                InstructionPattern::QueryResponse { response: respone_pat },
            ) => response.matches(respone_pat),
            (
                Self::TransferAsset {
                    assets,
                    beneficiary,
                },
                InstructionPattern::TransferAsset {
                    assets: assets_pat,
                    beneficiary: beneficiary_pat,
                },
            ) => assets.matches(assets_pat) && beneficiary.matches(beneficiary_pat),
            (
                Self::TransferReserveAsset {
                    assets,
                    dest,
                    xcm,
                },
                InstructionPattern::TransferReserveAsset {
                    assets: assets_pat,
                    dest: dest_pat,
                    xcm: xcm_pat
                },
            ) => assets.matches(assets_pat) && dest.matches(dest_pat) && xcm.matches(xcm_pat),
            (
                Self::Transact {
                    origin_kind,
                    require_weight_at_most,
                    call,
                },
                InstructionPattern::Transact {
                    origin_kind: origin_kind_pat,
                    require_weight_at_most: require_weight_at_most_pat,
                    call: call_pat,
                },
            ) => origin_kind.matches(origin_kind_pat)
                && require_weight_at_most.matches(require_weight_at_most_pat)
                && call == call_pat,
            (Self::HrmpNewChannelOpenRequest { .. }, InstructionPattern::HrmpNewChannelOpenRequest) => true,
            (Self::HrmpChannelAccepted { .. }, InstructionPattern::HrmpChannelAccepted) => true,
            (Self::HrmpChannelClosing { .. }, InstructionPattern::HrmpChannelClosing) => true,
            (Self::ClearOrigin, InstructionPattern::ClearOrigin) => true,
            (Self::DescendOrigin(_), InstructionPattern::DescendOrigin) => true,
            (
                Self::ReportError(response_info),
                InstructionPattern::ReportError(pattern),
            ) => response_info.destination.matches(pattern),
            (
                Self::DepositAsset {
                    assets,
                    beneficiary,
                },
                InstructionPattern::DepositAsset {
                    assets: assets_pat,
                    beneficiary: beneficiary_pat,
                },
            ) => assets.matches(assets_pat) && beneficiary.matches(beneficiary_pat),
            (
                Self::DepositReserveAsset {
                    assets,
                    dest,
                    xcm
                },
                InstructionPattern::DepositReserveAsset {
                    assets: assets_pat,
                    dest: dest_pat,
                    xcm: xcm_pat,
                },
            ) => assets.matches(assets_pat) && dest.matches(dest_pat) && xcm.matches(xcm_pat),
            (
                Self::ExchangeAsset {
                    give,
                    want,
                    maximal,
                },
                InstructionPattern::ExchangeAsset {
                    give: give_pat,
                    want: want_pat,
                    maximal: maximal_pat,
                },
            ) => give.matches(give_pat) && want.matches(want_pat) && maximal.matches(maximal_pat),
            (
                Self::InitiateReserveWithdraw {
                    assets,
                    reserve,
                    xcm,
                },
                InstructionPattern::InitiateReserveWithdraw {
                    assets: assets_pat,
                    reserve: reserve_pat,
                    xcm: xcm_pat,
                },
            ) => assets.matches(assets_pat) && reserve.matches(reserve_pat) && xcm.matches(xcm_pat),
            (
                Self::InitiateTeleport {
                    assets,
                    dest,
                    xcm,
                },
                InstructionPattern::InitiateTeleport {
                    assets: assets_pat,
                    dest: dest_pat,
                    xcm: xcm_pat,
                }
            ) => assets.matches(assets_pat) && dest.matches(dest_pat) && xcm.matches(xcm_pat),
            (
                Self::ReportHolding {
                    response_info,
                    assets,
                },
                InstructionPattern::ReportHolding {
                    response_dest: response_dest_pat,
                    assets: assets_pat,
                }
            ) => response_info.destination.matches(response_dest_pat) && assets.matches(assets_pat),
            (
                Self::BuyExecution {
                    fees,
                    ..
                },
                InstructionPattern::BuyExecution {
                    fees: fees_pat,
                }
            ) => fees.matches(fees_pat),
            (Self::RefundSurplus, InstructionPattern::RefundSurplus) => true,
            (Self::SetErrorHandler(xcm), InstructionPattern::SetErrorHandler(pattern)) => xcm.matches(pattern),
            (Self::SetAppendix(xcm), InstructionPattern::SetAppendix(pattern)) => xcm.matches(pattern),
            (Self::ClearError, InstructionPattern::ClearError) => true,
            (
                Self::ClaimAsset {
                    assets,
                    ticket,
                },
                InstructionPattern::ClaimAsset{
                    assets: assets_pat,
                    ticket: ticket_pat,
                }
            ) => assets.matches(assets_pat) && ticket.matches(ticket_pat),
            (Self::Trap(_), InstructionPattern::Trap) => true,
            (Self::SubscribeVersion { .. }, InstructionPattern::SubscribeVersion) => true,
            (Self::UnsubscribeVersion, InstructionPattern::UnsubscribeVersion) => true,
            (Self::BurnAsset(assets), InstructionPattern::BurnAsset(pattern)) => assets.matches(pattern),
            (Self::ExpectAsset(assets), InstructionPattern::ExpectAsset(pattern)) => assets.matches(pattern),
            (Self::ExpectOrigin(_), InstructionPattern::ExpectOrigin) => true,
            (Self::ExpectError(_), InstructionPattern::ExpectError) => true,
            (Self::ExpectTransactStatus(_), InstructionPattern::ExpectTransactStatus) => true,
            (
                Self::QueryPallet { response_info, .. },
                InstructionPattern::QueryPallet {
                    response_dest: response_dest_pat
                },
            ) => response_info.destination.matches(response_dest_pat),
            (
                Self::ExpectPallet { .. },
                InstructionPattern::ExpectPallet,
            ) => true,
            (
                Self::ReportTransactStatus(response_info),
                InstructionPattern::ReportTransactStatus(response_dest_pat),
            ) => response_info.destination.matches(response_dest_pat),
            (Self::ClearTransactStatus, InstructionPattern::ClearTransactStatus) => true,
            (Self::UniversalOrigin(_), InstructionPattern::UniversalOrigin) => true,
            (
                Self::ExportMessage {
                    network,
                    destination,
                    xcm,
                },
                InstructionPattern::ExportMessage {
                    network: network_pat,
                    destination: dest_pat,
                    xcm: xcm_pat,
                },
            ) => network.matches(network_pat) && destination.matches(dest_pat) && xcm.matches(xcm_pat),
            (
                Self::LockAsset {
                    asset,
                    unlocker,
                },
                InstructionPattern::LockAsset {
                    asset: asset_pat,
                    unlocker: unlocker_pat,
                },
            ) => asset.matches(asset_pat) && unlocker.matches(unlocker_pat),
            (
                Self::UnlockAsset {
                    asset,
                    target,
                },
                InstructionPattern::UnlockAsset {
                    asset: asset_pat,
                    target: target_pat,
                },
            ) => asset.matches(asset_pat) && target.matches(target_pat),
            (
                Self::RequestUnlock {
                    asset,
                    locker,
                },
                InstructionPattern::RequestUnlock {
                    asset: asset_pat,
                    locker: locker_pat,
                },
            ) => asset.matches(asset_pat) && locker.matches(locker_pat),
            (Self::SetFeesMode { .. }, InstructionPattern::SetFeesMode) => true,
            (Self::SetTopic(topic), InstructionPattern::SetTopic(expected)) => topic == expected,
            (Self::ClearOrigin, InstructionPattern::ClearOrigin) => true,
            (Self::AliasOrigin(_), InstructionPattern::AliasOrigin) => true,
            _ => false,
        }
    }
}
```
</details>

## Security considerations

There are no security risks related to the new instructions from the XCVM perspective.

## Impact

The impact of this proposal is Low. It introduces new instructions, so XCVM implementations have to be updated.

## Alternatives

### Constant weight

The response chain could advertise the max XCM weight and payment methods only, and the subscriber chain could always overestimate the weight of all the programs and force the user to always pay the maximum fees on the response chain, ensuring the XCM program execution.
The obvious drawback of this solution is that the user must almost always pay more than needed.
All the assets left after the `BuyExecution` could be deposited back to the user. Yet, the user must possess the max amount of fees to initiate the execution, even if it turns out to be cheap.

### Querying the weight info for each program

The subscriber chain could query the weight of a program from the response chain before sending the program to actual execution. However, weighing and sending the answer back must also be covered by fees. Yet these fees could be predefined, and the subscriber chain could charge the sender user to pay these extra fees.
Nonetheless, this induces an additional round-trip for each XCM message, slowing the inter-chain communication down and wasting the corresponding XCMP/HRMP channel capacity.

### Advertising the XCM Weigher code

The response chain could advertise the Weigher code so the subscriber chain could weigh any XCM program correctly. The code could be a simple formula or a complex branching program.
However, this implies we have to agree on the Weigher code encoding, which seems to be a somewhat complex task. Especially considering that XCM is meant to be used not only by the Dotsama ecosystem.
Also, those Weigher programs must execute in some sandbox since those programs are inherently untrusted because it is a code received from an outside world. This could induce extra complexity to implement such a mechanism.
