# Delegation Tokenization on Cosmos-SDK

This post explains about delegation tokenization on Cosmos-SDK. 
The contents are composed of 1) Context, 2) General Effects and Risks, 3) Possible Design Options, and 4) Technical Specs.

</br></br></br>

## 1. Context

</br>

### Delegated Proof of Stake : the Standard Concensus Mechanism for Next Generation Blockchains

- Most newly born public blockchains adopt proof of stake, especially delegated proof of stake
- dPoS has demonstrated the most stable and secure concensus mechanism for past several years

</br>

### Liquid Staking : Unlock the Staked Assets for Great Potential Economic Value

- Most staked assets in dPoS blockchains are locked in the blockchain protocols
- Unlocking the staked assets have great economic potential to thrive growth of investment and use-cases in dPoS networks

</br>

### Delegation Tokenization : Protocol Level Base Layer Support for Liquid Staking

- There should exist very secure support from the core blockchain modules for liquid staking
- Simple delegation tokenization feature on dPoS is necessary to
    - allow many staking derivatives applications to bring innovated financial utilities
    - with securely built base layer protocol as a building block
- The base layer delegation tokenization should have simple and general design

</br>

### Fungibility : Extracting the Fungible Part from Delegation Assets

- Delegation assets have below multiple risk/return sub-profiles
    - Staking Return
        - Inflationary(or non-inflationary) staking token rewards
        - Gas fee distribution
    - Staking Fee
        - Validator commission fee
    - Risk
        - Slashing risk : double signing, down time
- Non-fungibility of delegation assets across validators
    - Each validator has different staking return, staking fee, and slashing risk
- Non-fungibility over same validator
    - Different beginning/ending block height for staking
    - Different reward withdrawal activities from delegators
- Fungibility of delegation assets over same validator
    - For given identical staking period, all risk/return profiles are fungible over same validator
    - Standardization of staking periods will allow the modules to make delegation fungible over same validator

</br></br>

## 2. General Philosophy

</br>

### Providing Minimal Building Block from Cosmos-SDK Modules for Staking Derivatives

- Simplest design possible
    - The base layer Cosmos-SDK feature should have the most minimal and simplest design
    - The model should allow most other blockchains, which utilize Cosmos-SDK, to adopt the feature without problems
    - The model should not demand strong assumptions on staking environment for maximal generalization
- Transferable fungible tokens from delegation assets tokenization
    - The delegation tokens should be transferable not only inside the Cosmos Hub, but also to different networks via IBC
    - The delegation tokens should be fungible tokens for better trading standard and liquidity

</br>

### Validator-Based Fungibility

- Staking returns and risks are fungible for delegations upon same validator
- The base layer should provide validator-based fungible delegation tokens for a minimal building block
- Further synthetic process from staking derivatives can create more fungible delegation tokens over multiple validators

</br>

### Standardized Reward Distribution Period

- Even though staking returns and risks are fungible over same validator, they are not fungible over different staking periods
- Therefore, we need to have a standardized staking period so that the returns are fungible inside the selected staking period

</br></br>

## 3. Possible Risk Vectors

</br>

### Functional Errors on Base Layer

- Attack vectors on minting/burning delegation tokens
- Malfunctioning upon slashing events

</br>

### Unexpected Economic Consequences

- Validator set centralization
    - There will exist some staking derivatives products which amplify the difference of staking returns from different validators
    - It might lead to accelerate validator set centralization towards low or free commission fee
- Governance vote buying
    - Some staking derivatives products can allow users to buy/sell governance voting rights

</br>

### Hacking or Manipulation on Staking Derivatives

- Staking derivatives products
    - After we have base layer delegation tokenization, lots of staking derivatives can be provided from different blockchains
- Migration of delegation tokens to different networks
    - Some of the delegation tokens will be transferred to different networks to use such new staking derivatives products
- Hacking risk on staking derivatives
    - New staking derivatives in other blockchains can be hacked or manipulated so that significant amount of delegation tokens can be stolen
- Threat to the Cosmos Hub security
    - Stolen delegation tokens can threat the security of dPoS on the Cosmos Hub

</br></br>

## 4. Suggested Model : Delegation Tokenization Module

</br>

### `DelegationToken`

- Every delegation share becomes transferable fungible token called `DelegationToken`
    - Fungibility is over same validator

</br>

### Periodic Tokenization

- `TokenizedDelegation` and `RedeemDelegation` are only happening at `DelegationTokenizationBlock`
- `TokenizationPeriod`
    - This is a governance parameter representing the frequency of `DelegationTokenizationBlock`
    - The default value can be 100,000 blocks(about a week)

</br></br>

### Tokenization and Redemption of Delegations

- `TokenizeDelegation` : `DelegationToken` minting
    - `DelegationToken` minting happens at next `DelegationTokenizationBlock`
    - `MsgDelegationQue` : queue tokenization at next `DelegationTokenizationBlock`
- `RedeemDelegation` : `DelegationToken` burning
    - Redemption of delegation is calculated from the proportion at last `DelegationTokenizationBlock`
    - `MsgUnDelegationQue` : queue redemption at next `DelegationTokenizationBlock`

</br>

### Reward Pool Management

- Bonding-token rewards management
    - Periodically auto-rebonded at `DelegationTokenizationBlock`
- Non-bonding-token rewards management
    - The rights to pull periodically accumulated non-bonding-token rewards are minted as `RewardPoolToken` and sent to `DelegationToken` owner
    - `RewardPoolToken` is fungible over same validator and same period
- Withdraw non-bonding-token rewards from `RewardPoolToken`
    - `RewardPoolToken` holders can request immediate withdraw of non-bonding-token rewards

</br>

### Maximum Delegation Tokenization Ratio

- `MaxDelegationTokenizationRatio`
    - `DelegationTokenizationRatio` = total tokenized delegation amount / total delegation amount
    - If `DelegationTokenizationRatio` ≥ `MaxDelegationTokenizationRatio`
        - Any `TokenizeDelegation` is failed

</br></br>

## 5. Changes in Existing Modules

</br>

### Staking Module

- New function needed : `ChangeDelegatorAddr()`
    - Upon tokenization and redemption, the ownership of delegation is transferred between users and module account
    - So, there should exist such function so that the new module can handle the ownership of delegation
