## Purpose of this documentation

- To add very simple liquid staking feature on existing staking/distribution modules
- To add auto-bonding feature on existing staking/distribution modules

## Reasoning

- Current staking/distribution modules do not anticipate the liquid staking feature
- Liquid staking module without the evolution of existing staking/distribution modules is very complicated and needs many assumptions
- It is natural for existing staking/distribution modules to be upgraded to possess better features which are demanded from the market
- Native staking/distribution modules upgrades will easily allow other Cosmos-SDK based blockchains to adopt such new features without complicated logics

## Delegation Share Tokenization

### Concept

`**DelegationToken**`

- Every delegation share becomes transferable fungible token called `DelegationToken`
    - Fungibility is over same validator

**Periodic Tokenization**

- All activities around `DelegationToken` are only happening at `DelegationTokenizationBlock`
- `TokenizationPeriod`
    - This is a governance parameter representing the frequency of `DelegationTokenizationBlock`
    - The default value can be 100,000 blocks(about a week)

**List of acitivities handled on `DelegationTokenizationBlock`**

- Delegation : `DelegationToken` minting
    - Delegation can happen anytime
    - `DelegationToken` minting happens at next `DelegationTokenizationBlock`
        - No rewards are accumulated until next `DelegationTokenizationBlock`
        - It is optimal to delegate right before `DelegationTokenizationBlock`
- Undelegation : `DelegationToken` burning
    - Undelegation can happen anytime
    - Redemption of delegation is calculated from the proportion at last `DelegationTokenizationBlock`
        - No rewards can be redeemed after last `DelegationTokenizationBlock`
        - It is optimal to undelgate right after `DelegationTokenizationBlock`
- Redelegation : `DelegationToken` burning and minting
    - Redelegation can happen anytime
    - Redelegation : burning the holding `DelegationToken`, and minting the new `DelegationToken`
    - Burning and Minting
        - `DelegationToken` minting happens at next `DelegationTokenizationBlock`
            - No rewards are accumulated until next `DelegationTokenizationBlock`
        - Redemption of delegation is calculated from the proportion at last `DelegationTokenizationBlock`
            - No rewards can be redeemed after last `DelegationTokenizationBlock`
- Withdrawal
    - It is now impossible to request withdrawal of rewards
    - Delegators can sell portion of their `DelegationToken` holdings in the market to mimic rewards withdrawal activity

**Reward Pool Management**

- Reward distribution auto-bonding mechanism
    - Bonding-token rewards
        - Periodically auto-rebonded at `DelegationTokenizationBlock`
    - Non-bonding-token rewards
        - These rewards are periodically sold out via `RewardPoolBlockDeal` mechanism until `DelegationTokenizationBlock`
        - Incoming bonding token from `RewardPoolBlockDeal` is auto-rebonded just like bonding-token rewards
- `RewardPoolToken`
    - `RewardPoolToken` is minted at every `DelegationTokenizationBlock`
    - Number of `RewardPoolToken` minted = Total supply of `DelegationToken` (for each validator)
    - This token will be sold to any potential buyer who pays better price
    - At `DelegationTokenizationBlock`
        - All `RewardPoolToken` holders receive proportional amount of accrued non-bonding-token rewards
        - `RewardPoolToken` are burnt after receiving their portion of accrued non-bonding-token rewards
- `RewardPoolBlockDeal`
    - Every new `RewardPoolBlockDeal` is triggered right after minting of `RewardPoolToken`
    - Anyone can submit bid bonding-token to buy `RewardPoolToken`
        - Each bid has
            - `BidPrice` : in bonding-token
            - `BidQuantity` : amount of bid as the quantity of `DelegationToken`
            - `BidHeight` : the block height where the bid is committed
        - `BidPrice` * `BidQuantity` amount of bonding-token is escrowed in the module until finalization of `RewardPoolBlockDeal`
    - Match algorithm : just like orderbook based block deal
        - Every bid is sorted by

            1) higher `BidPrice` 

            2) larger `BidQuantity`

            3) earlier `BidHeight`

        - Bids are filled accordingly until total supply of `RewardPoolToken` is matched
        - The last bids(possibly multiple) can be partially filled
        - If non-bonding-token rewards are not completely sold, residuals will be sold together for next `RewardPoolBlockDeal`
