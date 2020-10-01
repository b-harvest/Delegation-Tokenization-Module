# **Delegation-Tokenization-Module**

Delegation tokenization module, a native Cosmos-SDK module which allows delegations to be tokenized to fungible and NFT tokens

## **1. Problem**

- Each delegation has different
    - Validator to stake on
    - Accumulated rewards
    - Risk of double signing
- How we can standardize staked atom into universal one token?

## **2. Solution**

- Module Name : **DelegationTokenization Module**
- Utility
    - Delegators can tokenize their delegation into `FToken` and `YToken`
    - Tokenized delegations either can be redeemed or auctioned to any `FToken` holder after maturity
- Basic Analogy of each token
    - `FToken`
        - It has potential rights to redeem tokenized delegation whose maturity is passed
        - The time schedule of redeem right
            - If you are holding `FToken`, someone is definitely holding `YToken` without `FToken`
            - This `YToken` related delegation will be auctionized after its maturity
            - Therefore, "EVERY" `FToken` will get its redeem right no longer than the maturity length
            - So, `FToken` can be viewed as a kind of "FORWARD CONTRACT" on delegation
    - `YToken`
        - `YToken` holders can enjoy very high yield rate with high slashing risk
        - But `YToken` holder should redeem delegation or get auctioned by `FToken` holders after maturity

## **3. Process**

### **Delegation Tokenization**

- Description : a delegation with 100 bonded Atoms splitted into two kinds of tokens
    - `FToken` : Fungible delegation token
    - `YToken` : Non-fungible delegation yield token uniquely mapped to this delegation
    - 100 Atom delegation = 90 `FToken` + the `YToken` NFT
- Delegation ownership transfered to module account

### **Transferable `YToken` NFT**

- Ytom holder can transfer the `YToken` NFT to other addresses
- Ownership of `YToken` is stored in `TokenizedDelegation` object
- YToken is not transferable when tokenization status is not in `Tokenized`

### **Delegation Redeem Process**

- **Before maturity**
    - redeem requestor sends 90 `FToken` and the `YToken` NFT to the module account
        - sent `FToken` and `YToken` are burnt
        - delegation ownership transfered to redeem requestor
            - including all accumulated and automatically withdrawed rewards
    - redeemed `TokenizedDelegation` is deleted
- **After maturity or Automated unbonding triggered**
    - auction held for the delegation ownership
        - payment token : `FToken`
        - starting price : 90 `FToken` and goes up
        - auction period : 20,000 blocks (about 33 hours)
    - hook execution after auction completion
        - final auction winning price = 90 + X `FToken`
        - X `FToken` sent to `YToken` holder
        - 90 `FToken` burnt
        - the `YToken` burnt (whoever is holding)
        - delegation ownership transfered to auction winner
            - including all accumulated and automatically withdrawed rewards
    - if auction has no winner after auction period?
        - sudden death auction started
            - anyone paying 90 or more `FToken` will immediately get delegation ownership

## **4. Price Discovery Mechanism**

### **Price Balancing from the Market**

- If `FToken` price is much cheaper than Atom price
    - Traders buy `FToken` and participate on `TokenizedDelegation` auctions
- If `FToken` price is much more expensive than Atom price
    - Traders tokenize delegations and sell created `FToken` in the market
- If `YToken` price is much cheaper than its intrinsic value
    - Traders buy `YToken` and appropriate `FToken` to redeem the delegation
- If `YToken` price is much more expensive than its intrinsic value
    - Traders tokenize delegations and sell created `YToken` in the market

### **Intrinsic Value of `FToken` and `YToken`**

- assumptions
    - `OriginalBondedTokenAmount` = 100 Atom
    - accrued rewards at maturity = R Atom
- variables
    - X(0) : `FToken` price in Atom at tokenization
    - X(m) : `FToken` price in Atom at maturity
    - Y(0) : `YToken` price in Atom at tokenization
    - Y(m) : `YToken` price in Atom at maturity
- equations

    ① at tokenization : 90*X(0) + Y(0) = 100

    ② redeem just before maturity : 90*X(m) + Y(m) = 100+R

    ③ expected auction winning price = (100+R) Atom = (100+R)/X(m) `FToken`

    ④ amount receivable by `YToken` holder after auction = Y(m) = (100+R)/X(m) - 90 `FToken`

    ⑤ combining ② and ④ gives

    - 90*X(m)^2 - (190+R)*X(m) + (100+R) = 0
    - solving above quadratic equation yields : X(m) is **always equal to 1** for any R
    - X(0) = X(m) because they are fungible
    - Y(0) = 10 from ①
    - Y(m) = 10+R → only `YToken` gets all the rewards

## **5. Draft Technical Spec**

### **States**

- `TokenizedDelegation`
    - `DelegationIndex`
        - pointing the delegation object which is tokenized
    - `OriginalBondedTokenAmount`
        - the amount of token bonded at the time of tokenization
    - `NFTOwner`
        - the account address of `YToken` owner
    - `Maturity`
        - the maturity of this `TokenizedDelegation`
    - `Status`
        - `Tokenized` / `AuctionPeriod` / `SuddenDeath` / `Redeemed`
- `DelegationAuction`
    - `DelegationIndex`
        - pointing the tokenized delegation object to be auctioned
    - `Maturity`
        - block height when the auction finalized
        - when 0, the auction entered `SuddenDeath` status
    - `MinBidPrice`
        - the minimum bid price = `OriginalBondedTokenAmount` * 0.9
    - `HighestBidPrice`
        - the latest highest bid price

### **Messages**

- `MsgDelegationTokenization`
    - `TokenizationRequestor`
        - the account address of tokenization requestor
    - `DelegationIndex`
        - pointing the delegation object to be tokenized
- `MsgSendYToken`
    - `Sender`
        - the account address of `YToken` sender
    - `Receiver`
        - the account address of `YToken` receiver
    - `DelegationIndex`
        - pointing the delegation object which YToken ownership is being transfered
- `MsgRedeemDelegation`
    - `RedeemRequestor`
        - the account address of redeem requestor
    - `RedeemFToken`
        - sending `FToken` for redemption
    - `DelegationIndex`
        - pointing the delegation object which `YToken` ownership is being transfered
- `MsgDelegationAuctionBid`
    - `Bidder`
        - the account address of the bidder
    - `DelegationIndex`
        - pointing the tokenized delegation object which is under auction to be participated
    - `BidPrice`
        - the bid price in `FToken`
- `MsgGovVoteFromYTokenOwner`
    - `DelegationIndex`
        - pointing the tokenized delegation object which represents the governance voting power of the `YToken` holder
    - `ProposalID`
        - ID of target proposal to be voted for
    - `Vote`
        - vote decision option chosen by `YToken` owner

## **6. Issues**

### **Governance Voting Rights**

- Problem
    - Because the ownership of delegation is transfered to a module account, it is undecided who has the governance voting right
- Proposed Solution
    - As unstake Atom does not have any governance right, it is natural that `FToken` holders also do not possess any governance voting rights
    - Instead, because the `YToken` holders have full benefit of rewards and full responsibility of slashing, `YToken` owner should have the governance voting right
    - If the ownership of `YToken` is changed during voting period, the new owner can overwrite the governance voting
- Process
    - `MsgGovVoteFromYTokenOwner` is sent to the DelTokenization module
    - The module account votes on the given governance proposal on the behalf of `YToken` owner

### **Validator Power Centralization**

- Problem
    - `YToken` APR is 10 times amplified, resulting in more sensitive market response for different commission rates on each validator
    - This can cause a strong tendency of
        - over-pricing of `YToken` with lower commission rate validator
        - under-pricing of `YToken` with higher commission rate validator
    - These tendency can lead to the validator power centralization, because of pricing imbalance of each `YToken`
- Discussion
    - The possible risk of validator power centralization is a result of the nature of selfish incentive motivation of any human being
    - Therefore, it is not caused by tokenization of delegation, but by the human nature
    - Can we prevent validator power centralization somehow?
