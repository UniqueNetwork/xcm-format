---
Title: Dropped Assets Claimers
Number: 0
Status: Draft
Version: 0
Authors:
 - Daniel Shiposha
Created: 2023-06-20
Impact: Low
Requires:
Replaces:
---

## Summary

The proposed change provides a way to set claimers to potentially dropped assets.
Currently, determining a claimer of dropped assets is implementation-specific, and there is no way to set a custom claimer.
The ability to set a custom claimer origin makes it easier to rescue the dropped assets, especially in the case of a reserve-based transfer.

## Motivation

Currently, determining a claimer of dropped assets is implementation-specific, and there is no way to set a custom claimer. The implementation could choose a predictable claimer origin such as the same origin as the origin of the XCM message. But also, this choice could be arbitrary if there is no origin, e.g., if `ClearOrigin` was executed before an error happened.

The latter case includes reserve-based transfers. Given that parachains mostly rely on reserve-based transfers, a way to define a custom claimer for the dropped assets makes rescuing more convenient since it provides more ways to do so compared to an arbitrary claimer setting defined by a specific implementation.

This RFC proposes adding a new instruction, `AddDropedAssetsClaimer`. The new instruction sets a custom claimer to particular dropped assets defined by an asset filter.

## Specification

The instruction would have the following specification:

```rust
AddDropedAssetsClaimer { claimer: MultiLocation, assets: MultiAssetFilter }
```

The `AddDropedAssetsClaimer` instruction allows XCVM to hint to the asset drop implementation that the specified `claimer` can claim the `assets` if they are dropped in the case of an error.

This instruction will add a new `claimer` for all assets the asset filter accepts.
It means:
 1. We can set several claimers to the same assets, and any of them can claim these assets. This case includes the situation when asset sets defined by asset filters from different `AddDropedAssetsClaimer` instructions [have a non-empty intersection](#setting-multiple-claimants).
 2. Alternatively, we could set several claimers to different assets where each claimer has a right to claim only assets intended for them.

The information about the claimers should be a part of the XCM Context so that Asset Transactors could also know to whom to provide a claim in case of an error. 

##### Examples

###### Usage together with a reserve-based transfer
```rust
WithdrawAsset(/* <assets-to-transfer> */),
InitiateReserveWithdraw {
    assets: Wild::All,
    reserve: /* <reserve-chain> */,
    xcm: vec![
        // In case of an error on the <reserve-chain>,
        // `<MY_ACCOUNT>` can claim `All` the dropped assets.
        AddDropedAssetsClaimer {
            claimer: AccountId32 {
                network: None,
                id: /* <MY_ACCOUNT> */,
            },
            assets: Wild::All,
        },
        BuyExecution { /* ...snip... */ },
        DepositReserveAsset {
            assets: Wild::All,
            dest: /* <destination-chain> */,
            xcm: vec![
                // In case of an error on the <destination-chain>,
                // `<MY_ACCOUNT>` can claim `All` the dropped assets.
                AddDropedAssetsClaimer {
                    claimer: AccountId32 {
                        network: None,
                        id: /* <MY_ACCOUNT> */,
                    },
                    assets: Wild::All,
                },
                BuyExecution { /* ...snip... */ },
                DepositAsset { /* ...snip... */ },
            ].into(),
        }

    ].into(),
}
```

###### Setting multiple claimants

```rust
AddDropedAssetsClaimer {
    claimer: /* <FIRST_CLAIMER_MULTILOCATION> */,
    assets: Wild::All,
}
AddDropedAssetsClaimer {
    claimer: /* <SECOND_CLAIMER_MULTILOCATION> */,
    assets: Wild::All,
}
```

In the example above, both `<FIRST_CLAIMER_MULTILOCATION>` and `<SECOND_CLAIMER_MULTILOCATION>` can claim the dropped assets. However, the claim can be used only once; the first to claim takes the assets.

###### More specific asset filters
```rust
AddDropedAssetsClaimer {
    claimer: /* <FIRST_CLAIMER_MULTILOCATION> */,
    assets: Wild::AllOf {
        id: /* Fungible Asset ID */,
        fun: Fungible,
    },
}
AddDropedAssetsClaimer {
    claimer: /* <SECOND_CLAIMER_MULTILOCATION> */,
    assets: Wild::AllOf {
        id: /* NFT Asset ID */,
        fun: NonFungible,
    },
}
```

In this case, the `<FIRST_CLAIMER_MULTILOCATION>` can claim dropped fungibles of a given ID, and the `<SECOND_CLAIMER_MULTILOCATION>` can claim the dropped NFTs from a given collection.

## Security considerations

There are no security risks related to the new instruction from the XCVM perspective.

## Impact

The impact of this proposal is Low. It introduces a new instruction, so XCVM implementations have to be updated.

## Alternatives

Currently, there is no way to set a custom claimer.

