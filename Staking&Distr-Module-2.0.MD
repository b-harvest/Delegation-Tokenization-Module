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

**`DelegationToken`**

- Every delegation share becomes transferable fungible token called `DelegationToken`
    - Fungibility is over same validator

**Periodic Tokenization**

- All activities around `DelegationToken` are only happening at `DelegationTokenizationBlock`
- `TokenizationPeriod`
    - This is a governance parameter representing the frequency of `DelegationTokenizationBlock`
    - The default value can be 100,000 blocks(about a week)

**List of acitivities handled at `DelegationTokenizationBlock`**

- Delegation : `DelegationToken` minting
    - Delegation can happen anytime
    - `DelegationToken` minting happens at next `DelegationTokenizationBlock`
        - No rewards are accumulated until next `DelegationTokenizationBlock`
        - It is optimal to delegate at or right before `DelegationTokenizationBlock`
        - `MsgDelegationQue` : execute delegation at next `DelegationTokenizationBlock`
- Undelegation : `DelegationToken` burning
    - Undelegation can happen anytime
    - Redemption of delegation is calculated from the proportion at last `DelegationTokenizationBlock`
        - No rewards can be redeemed after last `DelegationTokenizationBlock`
        - It is optimal to undelgate at or right after `DelegationTokenizationBlock`
        - `MsgUnDelegationQue` : execute undelegation at next `DelegationTokenizationBlock`
- Redelegation : `DelegationToken` burning and minting
    - Redelegation can happen anytime
    - Redelegation : burning the holding `DelegationToken`, and minting the new `DelegationToken`
    - Burning and Minting
        - `DelegationToken` minting happens at next `DelegationTokenizationBlock`
            - No rewards are accumulated until next `DelegationTokenizationBlock`
        - Redemption of delegation is calculated from the proportion at last `DelegationTokenizationBlock`
            - No rewards can be redeemed after last `DelegationTokenizationBlock`
        - `MsgRedelegationQue` : execute redelegation at next `DelegationTokenizationBlock`

**Reward Pool Management**

- Bonding-token rewards management
    - Periodically auto-rebonded at `DelegationTokenizationBlock`
- Non-bonding-token rewards management
    - The rights to pull periodically accumulated non-bonding-token rewards are minted as `RewardPoolToken` and sent to `DelegationToken` owner
    - `RewardPoolToken` is fungible over same validator and same period
- Withdraw non-bonding-token rewards from `RewardPoolToken`
    - `RewardPoolToken` holders can request immediate withdraw of non-bonding-token rewards
