# **Delegation-Tokenization-Module**

Delegation tokenization module, a native Cosmos-SDK module which allows delegations to be tokenized to fungible and NFT tokens
</br></br>
## 1. Problem

- Each delegation has different
    - Validator to stake on
    - Accumulated rewards
    - Risk of double signing
- How we can standardize staked atom into universal one token?
</br></br>
## 2. Solution

### Utility

- Delegators can tokenize their delegation into `NToken` and `YToken`
- Tokenized delegations either can be
    - redeemed by submitting both `NToken` and `YToken`
    - or auctioned to any `NToken` holder after maturity

### Basic Analogy of each token

- `NToken`
    - It has ultimate rights to redeem "any" tokenized delegation whose maturity is passed
- `YToken(V,M)`
    - It is fungible over same validator `V` and same maturity `M`
    - `YToken` holders get accrued rewards until redemption, but are exposed to slashing risk
    - `YToken` holders will
        - redeem delegation with `NToken` and `YToken` submitted to the module before maturity
        - or get auctioned by `NToken` holders after maturity
</br></br>
## 3. Parameters, States and Messages

### Parameters

- `DefaultTokenizationRatio`
    - The starting value of `TokenizationRatio`
    - This value decides how much of slashing loss can be covered by `YToken`
    - Lower `DefaultTokenizationRatio` results in less slashing risk for `NToken` holders

- `MaturityLength`
    - The length of tokenization period until maturity
    - Longer `MaturityLength` results in
        - Less auction amount for each maturity
        - Less opportunity for `NToken` holders to redeem `TokenizedDelegation`

- `TokenizationFrequency`
    - The frequency of delegation tokenization execution
    - Shorter `TokenizationFrequency` results in
        - Too many kinds of `YToken(V,M)` floating around
        - Too frequent auction held
            - But more diversified auction amount for each period

- `AuctionPeriod`
    - The length of auction before entering into `SuddenDeath` mode
    - Shorter `AuctionPeriod` results in
        - Higher risk of significantly discounted auction winning price

### **States**

- `TokenizedDelegation`
    - `DelegationIndex`
        - pointing the delegation object which is tokenized
    - `TokenizationDateTime`
        - datetime of tokenization
    - `Maturity`
        - the maturity datetime of this `TokenizedDelegation`
    - `Status`
        - `Tokenized` / `AuctionPeriod` / `SuddenDeath`

- `DelegationAuction`
    - `DelegationIndex`
        - pointing the tokenized delegation object to be auctioned
    - `AuctionMaturity`
        - block height when the auction finalized
        - when 0, the auction entered `SuddenDeath` status
    - `HighestBid`
        - the bidder and bid price with highest bid price

### **Messages**

- `MsgDelegationTokenization`
    - `TokenizationRequestor`
        - the account address of tokenization requestor
    - `DelegationIndex`
        - pointing the delegation object to be tokenized

- `MsgRedeemDelegation`
    - `RedeemRequestor`
        - the account address of redeem requestor
    - `RedeemToken`
        - sending `NToken` and `YToken` for redemption
    - `DelegationIndex`
        - pointing the delegation object to be redeemed

- `MsgDelegationAuctionBid`
    - `Bidder`
        - the account address of the bidder
    - `DelegationIndex`
        - pointing the tokenized delegation object which is under auction to be participated
    - `BidToken`
        - the bid price in `NToken`
        - the tokens are escrowed in the module until
            - higher bid is coming in
            - or auction is finalized
</br></br>
## 4. Process

### **Delegation Tokenization**

- Description : a delegation with `OriginalBondedAmount` of `StakingToken` splitted into two kinds of tokens
    - `NToken` : Fungible delegation notional token
    - `YToken(V,M)` : Fungible delegation yield token for validator `V` and maturity `M`
    - Tokenization of `OriginalBondedAmount` of delegation → (`TokenizationRatio` * `OriginalBondedAmount`) of `NToken` + `OriginalBondedAmount` of `YToken(V,M)`

- Delegation Tokenization Schedule and Maturity
    - Delegation Tokenizations are executed `TokenizationFrequency`
        - `TokenizationFrequency` is a governance parameter
        - Users can reserve delegation tokenization for next available execution datetime
    - `MaturityLength` : A governance parameter which decides the length of the maturity for every delegation tokenization

- Delegation ownership transfered to module account after tokenization

### **Delegation Redeem Process**

- **Before Maturity**
    - Redeem requestor sends (`TokenizationRatio` * `OriginalBondedAmount`) of `NToken` and `OriginalBondedAmount` of `YToken(V,M)` to the module account
        - Sent `NToken` and `YToken(V,M)` are burnt
        - Delegation ownership transfered to redeem requestor
            - Including all accumulated and automatically withdrawed rewards
    - Redeemed `TokenizedDelegation` is deleted

- **After maturity or Automated unbonding triggered**
    - Auction held for the delegation ownership for each `TokenizedDelegation`
        - Bid token : `NToken`
        - `AuctionPeriod` : a governance parameter for the auction period length
    - Auction completion before end of `AuctionPeriod`
        - If `AuctionWinningPrice` ≥ (`TokenizationRatio` * `OriginalBondedAmount`) : winning price is enough to cover originally minted `NToken` upon tokenization
            - Auction winner payout
                - The delegation ownership transfered to the auction winner including accrued rewards
                - (`TokenizationRatio` * `OriginalBondedAmount`) of `NToken` is burnt
            - YToken(V,M) holders payout
                - (`AuctionWinningPrice` - `TokenizationRatio` * `OriginalBondedAmount`) of `NToken` sent to `YToken(V,M)` holders
                    - `NToken` distribution is proportional to YToken(V,M) holding shares
                - `OriginalBondedAmount` of `YToken(V,M)` are burnt
                    - `YToken(V,M)` burns proportional to `YToken(V,M)` holding shares
        - If `AuctionWinningPrice` < (`TokenizationRatio` * `BondedAmount`) : winning price is not enough to cover originally minted `NToken` upon tokenization
            - Auction winner payout
                - The delegation  ownership transfered to the auction winner including accrued rewards
                - Entire `AuctionWinningPrice` of `NToken` is burnt
            - `YToken(V,M)` holders payout
                - Nothing is paid out to `YToken(V,M)` holders
                - `OriginalBondedAmount` of `YToken(V,M)` are burnt
                    - `YToken(V,M)` burns proportional to `YToken(V,M)` holding shares
            - Update `TokenizationRatio` as below
                - `TokenizationRatio` = `NTokenSupply` / SUM(`OriginalBondedAmount` in all `TokenizedDelegation`)
                    - Increased `TokenizationRatio` means the `NToken` value has been diluted
    - If auction has no winner after auction period?
        - `SuddenDeath` auction triggered
            - Anyone paying zero or more `NToken` will immediately win the auction
</br></br>
## 5. Advantages and Disadvantages

### Advantages

- Fungibility
    - `NToken` is fungible over all validators and maturities
    - `YToken` is fungible over same validator and same maturity
    - No NFT is utilized

- No Slashing Rate Assumption
    - `YToken` is firstly responsible for slashing loss
    - Residual slashing loss is distributed to all `NToken` by `TokenizationRatio` increment

- Independent Solution
    - It already provides universally fungible `NToken` independently
    - It does not need any kind of liquidity pool utility for fungibility

### Disadvantages

- Efficient auction participation is assumed
    - If the winning prices of auctions are not competent enough, `YToken` holders will get much less payout
    - All matured `TokenizedDelegation` are auctioned at the same time
        - Auction participants cannot utilize their capital into multiple auctions
        - There exists some capital utilization inefficiency for auction participants

- Roll-over inconvenience by `YToken` holders
    - `YToken` holders need to buy a lot of `NToken` from the market to roll over their tokenization
    - To buy enough `NToken`, they have to bring significant capital
    - If they don't roll over their tokenization until maturity
        - Auction will be automatically triggered
        - There exists some risk that the auction winning price is significantly lower than fair value
        - Lower auction winning price results in less payout to `YToken` holder
    - There can exist some services provided by large `NToken` holders
        - To roll-over the tokenization for `YToken` holders with appropriate fee
        - And send back new `YToken` with extended maturity to original `YToken` holder
        - These multiple messages can be atomically transacted in one transaction in a decentralized fashion
</br></br>
## 6. Discussions

### **Governance Voting Rights**

- Discussions
    - Because the ownership of delegation is transfered to a module account, it is undecided who has the governance voting right
    - `NToken` is meant to be used for multiple DeFi purposes including providing liquidity to AMM, collaterized for lending, and trading activities
        - Most `NToken` will be held by modules, smart contracts or pegged out to different blockchains
        - So, it is difficult to distribute governance power to `NToken` holders
    - If we only distribute governance power to `YToken` holders
        - YToken holders locked relatively much less capital than `NToken` holders
        - So, it is unfair to give the governance right only to `YToken` holders
