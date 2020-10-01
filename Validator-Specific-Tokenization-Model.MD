# **Delegation Tokenization Module**

## 1. Problem

- Each delegation has different
    - Accumulated rewards
    - Risk of double signing
- How can we create fungible tokens from delegations ?

## 2. Processes

- `FToken(V)` and `YToken(D)` for given validator `V` and delegation `D`
    - Delegators can tokenize their delegation into `FToken(V)` and `YToken(D)`
    - `FToken(V)`
        - Fungible token which has redemption rights on every `BondedTokenAmount` in `TokenizedDelegation` staking to validator `V`
        - `FToken(V)` is only fungible among delegations to the same validator `V`
        - All `FToken(V)` holders will be affected by slashing events from the validator `V`
    - `YToken(D)`
        - Non-fungible token which has redemption rights only on accrued rewards from the specific `TokenizedDelegation` of the delegation `D`

- Delegation tokenization process
    - Definitions
        - `TokenizationRatio(V)` for validator `V` : the amount of `FToken(V)` minted or burnt for every 1 `BondedTokenAmount` in `TokenizedDelegation` on validator `V`
    - The module account mints `FToken(V)` and `YToken(D)` and sends to the delegation tokenization requestor
        - Number of `FToken(V)` minted : `BondedTokenAmount(D)` * `TokenizationRatio(V)`
        - The unique NFT `YToken(D)` representing the reward rights on this delegation `D`

- Transfer ownership of `YToken(D)`
    - `YToken(D)` can be transfered to others via `MsgSendYToken`

- Delegation redemption process for any `TokenizedDelegation`
    - Two redemption processes available

        ① By sending both `FToken(V)` and `YToken(D)` into the module account

        - Redemption requestor receives both the delegation ownership and accrued rewards of `D`
        - Both `FToken(V)` and `YToken(D)` are burnt after redemption
        - Available period : this process is available any time

        ② By sending only `FToken(V)` into the module account

        - `FToken(V)` sender only receives the delegation ownership without accrued rewards
        - `YToken(D)` owner receives all accrued rewards from the redeemed delegation
        - Both `FToken(V)` and `YToken(D)` are burnt after redemption
        - Available period : this process is only available after the maturity of targeted `TokenizedDelegation`
    - Number of `FToken(V)` needed for redemption of delegation `D` : `BondedTokenAmount(D)` * `TokenizationRatio(V)`

- Slashing event hook
    - Definitions
        - `TotalBondedTokenAmount(V)` : the sum of total `BondedTokenAmount` in `TokenizedDelegation` staking to validator `V`
    - Update `TokenizationRatio(V)` upon every slashing event from validator `V`
        - `TokenizationRatio(V)` starts from 1
        - `TokenizationRatio(V)` is updated to `FToken(V)Supply` / `TotalBondedTokenAmount(V)` everytime when slashing happens from validator `V`

## 3. States and Messages

### States

- `TokenizedDelegation`
    - `DelegationIndex`
        - pointing the delegation object which is tokenized
    - `BondedTokenAmount`
        - the amount of token bonded for this `TokenizedDelegation`
    - `YTokenOwner`
        - the account address of `YToken(D)` owner
    - `Maturity`
        - the maturity of this `TokenizedDelegation`

### Messages

- `MsgDelegationTokenization`
    - `TokenizationRequestor`
        - the account address of tokenization requestor
    - `DelegationIndex`
        - pointing the delegation object to be tokenized

- `MsgSendYToken`
    - `Sender`
        - the account address of `YToken(D)` sender
    - `Receiver`
        - the account address of `YToken(D)` receiver
    - `DelegationIndex`
        - pointing the delegation object which YToken ownership is being transfered

- `MsgRedeemDelegation`
    - `RedeemRequestor`
        - the account address of redeem requestor
    - `RedeemTokens`
        - option1) sending `FToken(V)` and `YToken(D)` for redemption
        - option2) sending `FToken(V)` for redemption
    - `DelegationIndex`
        - pointing the delegation object which `YToken(D)` ownership is being transfered

## 4. Discussions

### Governance Voting Rights

- Problem
    - Because the ownership of delegation is transfered to the module account, governance voting right is unavailable

- Discussion
    - Who should get the governance voting rights?
    - How we can effectively prevent vote buying?
    - How we can allow governance voting without any inconvenience?

### Universally Fungible `FToken`

- Problem
    - `FToken(V)` is only fungible over the same validator `V`

- Suggested Solution
    - Hub AMM expansion to [Curve.fi](https://www.curve.fi/) like pool model
        - A `FToken` pool where users can swap any `FToken(V1)` to any other `FToken(V2)`
        - The `PoolToken` which represents the share ownership of the assets inside the pool
    - This `PoolToken` can become the universally fungible `FToken` across the validators

### NFT Compatibility over IBC

- Problem
    - There exists no standard NFT object which is compatible over IBC
    - `YToken` cannot be transfered to different blockchains via IBC

- Solution
    - After we have standard NFT object supported by IBC, this module can be upgraded so that `YToken` becomes IBC compatible NFT object

### Marketplace for Trading `YToken`

- Problem
    - There exists no way to trade `YToken` in fully decentralized way

- Suggested Solution
    - NFT DeX
        - Hub AMM can be expanded to NFT DeX which allows fully decentralized NFT marketplace
