# Sharded Token Standard Proposal

**Authors**: [James Henderson](https://t.me/JamesHendo)  

**_Motivation__**: We need token standards that scale via sharding, are Ethereum aligned, and allow community innovation.

At the time of writing the below is a proposal only, and is not a confirmed part of our product.

<aside>
⚠️

Please provide comments, suggestions, questions etc.

</aside>

# 1.0 Motivation

Our current enshrined tokens have some key advantages:

- Naturally shardable because account balances are stored in accounts, unlike centralised ledgers in ERC-20 style tokens. Parallelisation of transaction processing is less problematic.
- Tokens are conserved; if a CST carrying tokens between shards fails (e.g. out of gas), the protocol automatically returns tokens so that tokens are conserved and not lost. EVM-only smart contracts can’t solve this problem (at least not easily, they may need to implement their own consensus).

However, our current enshrined tokens are also insufficient because they

- Are not Ethereum aligned, dApps are not portable and token bridging is problematic.
- Don’t support features used by many existing tokens in Ethereum and elsewhere.
- Block community innovation and constrain ecosystem building; token changes require protocol changes.

We want the advantages of our both our enshrined tokens, and SC tokens like ERC-20, preferably without their disadvantages. Below compares several options for our token design. Enshrined tokens extended with smart contracts (blue) satisfies all main requirements.

|  | **Current Enshrined Tokens** | **Enshrined Tokens with Protocol Implemented Features** | **Smart Contract Only Tokens** | **Enshrined Tokens Extended with Smart Contracts** |
| --- | --- | --- | --- | --- |
| Ethereum Aligned
- like ERC-20 + extensions
- dApps easily ported
 | No | Yes, but minor differences | Yes, but minor differences | Yes, but minor differences |
| Scalable, shardable | Yes | Yes | No, cross shard token conservation can’t be solved | Yes |
| Protocol features required e.g.
- ZKPs
- sync committee
- prover system | some | lots, each feature requires protocol changes | none: EVM based | same as current enshrined tokens (nothing new required) |
| Token similarity across L1 bridge | low | high | highest | high |
| Community innovation, upgrades, ecosystem development
 | n/a | hard: requires nil | easy: we can’t stop people using this
sharding is hard: token conservation, scale | easy |

# 2.0 Token Design Proposal

In zkSharding we have the following support for tokens in order of most base-level to derivatives that extend the base:

1. [Enshrined Token Standard]: Retain our existing enshrined tokens and their minimal feature set. Existing enshrined tokens may be sufficient for simpler use cases. This does not conflict with the SC design below.
    - No protocol changes are necessary for this proposal. However, we may wish to support NFTs more naturally by implementing token arrays or maps, rather than our current scalar tokens, see Sec.  5.
2. [Base Smart Contract Token (BSCT) Standard]: Token creators can *extend* enshrined tokens using our BSCT standard. The BSCT standard makes clear how token builders can add enforced features to our enshrined tokens by using the EVM. The BSCT standard provides only a minimal specification for enforcing control of tokens. 
3. [Fungible Token (FT) Standard and Non-Fungible  Token (NFT) Standards] The BSCT standard is further extended with token features suitable for fungible tokens and non-fungible tokens.

Ultimately the BSCT, FT and NFT extensions of enshrined tokens are just EVM code and don’t require any protocol changes. Thus, users can build upon enshrined tokens in whatever way they like, independent of whether we implement BSCT, FT and NFT or not. This ability is (hopefully) a feature, not a bug because it allows innovation and ecosystem development. Thus, we hope that other types of tokens appear as the ecosystem grows.

## 2.1 Definitions

Below are definitions, more detail is provided in the later sections.

- ***Enshrined tokens*:** our current tokens implemented at the protocol level, similar to a native token. At the protocol level any account can store enshrined tokens, and enshrined tokens can be attached to CSTs. Importantly, enshrined tokens are automatically refunded if a CST fails, so that tokens are conserved and not lost.
- ***Governance Account:*** An account that can mint/burn enshrined tokens, and controls the token’s features. In our current enshrined token implementation, the governance account should be of type NilTokenBase so that it can create the enshrined token(s).
    - *Authorised Governance Account:* An account that has been granted privileges to invoke some governance methods. This might be useful for the token bridge, so that the bridge can  control minting and burning, but allow an authorised party (L1 token creator) to manage token logic accounts.
- ***Logic Account:*** An account that contains logic implementing one or more token features. Often the token creator will deploy these themself; however, the token creator can use existing accounts that implement token features. E.g. it may be that each shard contains a standardised transfer() logic account that is used by many tokens, while other tokens use customised transfer() logic accounts implementing non-standard logic.
    - token logic is envisioned as separate, but could be implemented as methods of either the governance account or holder accounts.
- ***Holder Account:*** An account that stores tokens and contains wrapper methods that call logic accounts. Each holder account is controlled by a custodian account but only gives the custodian access to methods controlled by governance (preventing the user from doing anything they want with the tokens). Roughly there will be one holder account per user/custodian (assuming users don’t make multiple accounts).
- ***Custodian Account*:** Any account. It has permission to call all the token methods of an associated token holder account.
    - *Authorised Custodian Account:* An account that has been granted privileges by the custodian account to invoke some custodian methods. This is likely used in approve() style methods, to allow dApps to transfer tokens from the holder account.
    
    Custodian accounts can be operated by any entity that wants control of tokens, e.g.
    
    - a user that has private keys for the custodian account,
    - a centralised exchange (Binance, Coinbase etc),
    - a dApp.
    - (possibly our L2 validators, if we put our staking contract on L2)
    
    A single account can be a custodian account for many different tokens. Wallet developers can create wallets that allow users to interact with and manage many tokens.
    
    - Note: we don’t call this an o*wner* account, because the token custodian account may be acting on behalf of a token owner. The custodian account may not be the ‘legal owner’ of the tokens.
    - Note: depending on the token design, it may be that multiple accounts have varying levels of permission. E.g. a user may give ‘approve’ permission to another account/dApp, as in ERC-20.

## 2.2 Overview

Adding enforced EVM features to enshrined tokens works by ensuring that tokens can only ever be sent to holder accounts that have been created by the governance’s holder account factory. The wallet factory sets up holder accounts so that the token features enforced, as illustrated in Fig 1. In particular, governance must ensure that it is only possible for users to send tokens between holder accounts, so that tokens are always held within controlled accounts, see Fig. 2.

Users can only hold tokens by proxy through these holder accounts. Users access token features via their custodian accounts that have privileges to call methods of their associated holder accounts.

![Fig. 1. Simple BSCT system. Users controlling custodian accounts call the governance account’s factory to create a holder account that enables them to hold and use tokens (green and yellow arrows). Note that a single account can be a custodian for many different tokens (not shown). The factory creates holder accounts with wrapper methods pointing to logic accounts (potentially with different targets for different holder accounts spread across shards, not shown). If needed, the governance account can update holder accounts and logic accounts (red arrows, not all interactions are shown e.g. to holder shard 1). Custodians interact with their tokens by calling holder account methods for example to send tokens (blue arrows](/figures/rollup/sharded-token-standard-proposal-00.png)

Fig. 1. Simple BSCT system. Users controlling custodian accounts call the governance account’s factory to create a holder account that enables them to hold and use tokens (green and yellow arrows). Note that a single account can be a custodian for many different tokens (not shown). The factory creates holder accounts with wrapper methods pointing to logic accounts (potentially with different targets for different holder accounts spread across shards, not shown). If needed, the governance account can update holder accounts and logic accounts (red arrows, not all interactions are shown e.g. to holder shard 1). Custodians interact with their tokens by calling holder account methods for example to send tokens (blue arrows

![Fig. 2. Accounts with $>0$ token balance must be holder accounts. To achieve this, governance must ensure that holder accounts can only send tokens to other holder accounts (green arrows). The token’s various smart contracts must not allow tokens to be sent to other accounts by any means (red arrows). Custodian accounts do not contain tokens. Note that technically the governance account and token logic accounts can hold tokens (not shown), but these are not controlled by users and should also restrict token transfers to associated token accounts.](/figures/rollup/sharded-token-standard-proposal-01.png)

Fig. 2. Accounts with $>0$ token balance must be holder accounts. To achieve this, governance must ensure that holder accounts can only send tokens to other holder accounts (green arrows). The token’s various smart contracts must not allow tokens to be sent to other accounts by any means (red arrows). Custodian accounts do not contain tokens. Note that technically the governance account and token logic accounts can hold tokens (not shown), but these are not controlled by users and should also restrict token transfers to associated token accounts.

## 2.3 Governance Account

The purpose of the governance account is to govern a token by enforcing users to use the token’s features. For example, to enforce that a user can’t send tokens to a blacklisted account, or to enforce that transfer fees are paid to the governance account.

Enforcement is derived from controlling both:

1. Creation of holder accounts containing enforced usage of logic accounts, and 
2. The logic accounts themselves.

**Any method that involves transferring tokens must only allow token transfers between holder accounts (see Fig 2).** If enshrined tokens are allowed to be transferred to non-holder accounts then the usage of those tokens can no longer be enforced. It is the responsibility of each token’s governance to ensure that suitable, secure holder accounts and logic accounts are used. 

Both 1 and 2 are coordinated via a whitelist of holder account addresses. Creation of holder accounts *only* occurs by calling the governance account’s holderAccountFactory(custodianAddress), (for further detail on the holder account see its corresponding section below). After successfully deploying a holder account, the holder address is added to a whitelist and logic accounts are updated, in particular the holder address must be added to the transfer() logic account so that it can receive tokens.

Governance accounts have two types of methods categorised by access permission:

1. [External/Public] These methods can be called by any account
    - holderAccountFactory(custodianAddress), a factory that deploys a new holder account with custodianAddress as its custodian account. *After* the holder account is successfully deployed, the whitelist is updated. If the holder account and governance account are on different shards, then a CST is required to message successful deployment. In most cases any account should be able to create a holder account with associated custodianAddress, but a token designer might also want to restrict this permission to custodianAddress itself.
    - [optional] receive() allow the governor to receive tokens.
    
2. [Internal/Private: Governance Account] Can be called by governance account, and optionally by an authorised governance account
    - mint(), allow the governor to mint new enshrined tokens
    - transfer(), allow the governor to send enshrined tokens to holder accounts
    - [optional] burn(), allow the governor to burn enshrined tokens.
    - [optional] updateHolderMethods(), updates the methods in holder accounts, e.g. to point holder accounts to new logic accounts. Updating holders can be made gas efficient by pointing all holder accounts (per shard) to a fixed central wrapper (per shard). The central wrapper then points to the actual logic account. Thus all holder accounts can be updated by modifying a single centralised wrapper.
    - [optional] updateLogicMethods(), updates the methods in logic accounts, e.g. to update the whitelist, blacklist, fees, or maxBalance.
    - [optional] changeAuthorisedGovernance(), add/remove/change authorised governance accounts.

Note that the above methods could be implemented as distinct accounts from the governance account so that governance can update its deployWallet, for example.

## 2.4 Holder Account

Holder accounts store enshrined tokens on behalf of the custodian account. A holder account is created by calling the governance account’s method holderAccountFactory(custodianAddress). **The holder account must not expose any ‘unofficial’ way to transfer enshrined tokens to ensure that users can only transfer tokens between holder accounts.**

Holder accounts have three types of methods categorised by authorisation:

1. [External/Public] These methods can be called by any account
    - balance(), return number of enshrined tokens
    - receive(), receive enshrined tokens
2. [Custodian Account] These methods contain authorisation that only allows custodian account to use them. Optionally, these methods may be called by an authorised custodian account.
    
    These wrapper methods point to logic accounts that implement the token’s user accessible features. These wrapper methods are intended to be ‘light’ but might contain some basic logic, e.g. check for sufficient balance.
    
    - transfer(), send enshrined tokens to another token holder account
    - [optional] burn(), send enshrined tokens to governance account for burning
    - [optional] changeAuthorisedCustodians(), add/remove/change authorised custodians, and their permissions.
3. [Governance Account] These methods allow the governance account to update enforced token logic by updating the wrapper methods, for example to point a wrapper method to a new logic account.
    - [optional] governanceUpdateMethods(), allows the governance account to change the holder account wrapper method targets.
    
    The governance account can use these update methods to 
    
    - Manage the sharding of token logic by pointing holder accounts to different logic accounts spread across shards.
        - It might be necessary to change wrapper targets if a logic account is on a congested shard.
    - Upgrade logic of token features.

<aside>
⚠️

When creating CSTs with enshrined tokens attached, the holder account must specify itself as the return address for refunding enshrined tokens in case of CST failure.

</aside>

<aside>
⚠️

Ethereum contains the SELFDESTRUCT opcode that removes a smart contract and in the process forcibly sends its ETH tokens to any specified address (independent of whether the destination has a receive()). If a holder account uses SELFDESTRUCT it must ensure that usage of SELFDESTRUCT only sends tokens back to the governance account, and not any arbitrary account. [we support SELFDESTRUCT opcode]

</aside>

## 2.5 Logic Account

Logic accounts implement the logic of the token’s features and are the targets of the pointers in holder account wrapper methods. 

The governance account can update the use of token logic accounts in two ways

- Update holder account wrapper pointers to point to different logic accounts, and
- Logic accounts expose methods that allow the governance account to update them e.g. to update the whitelist, or change a fee percentage parameter.

Logic accounts have two types of methods categorised by authorisation:

1. [External/Public] These methods can be called by any account.
    - Transfer(); used to transfer tokens between holder accounts. Implements any custom logic applied to transfers such as
        - freeze/pause transfers
        - blacklist accounts
        - charge transfer fees
        - enforce max balance restrictions (this might also be part of a holder account’s receive())
        - Preferably logic only contains read-only operations so that calls to transfer() can be processed in parallel.
    - The BSCT standard does not require other methods, but FT and NFT extensions to the BSCT will require other methods.
2. [Governance Account] These methods can only be called by governance account.
    - the transfer() logic account must expose updateWhitelist() to enable new accounts to be used.
    - [optional] send(), ensures any tokens that might end up in a logic account are not stuck and can be sent to another destination (holder account, or governance account). send() should use the transfer() logic account.

The standard does not specify how the token logic should be organised, the developer/governance account can decide:

- one account that implements all token features, or
- separate accounts that each implement a subset of features

# 3.0 Potential Concerns

Below are a list of potential issues or points to note that may be raised and what to do about them. Some are UX related, some are security related. IMO nothing in here seems like a reason not to use this proposal.

### Governance/Token Developer

1. What to do if holder account deployment fails (out of gas)?
    - To ensure security of the whitelist, the whitelist should only be updated *after* holder account deployment (not before).
    - If a holder account is successfully deployed, but the confirming CST sent to the governance account fails, then the holder account will not be added to the whitelist. In this case the user will not be able to use that holder account and must deploy a new holder account. The user can use the same custodian account for the new holder account.
2. A token custodian account may control multiple holder accounts for the same token. Token implementations should not assume a 1-1 mapping.
3. Governance accounts may want to have non-standard authorisation e.g. multisig, which should be supported at the protocol level (is this implemented yet?). E.g. large stablecoin issuers would likely require this.
4. Governance actions e.g. updating holder account pointers to token logic may fail. It is up to the governance account to retry.
5. It seems possible to create accounts that implement generic methods and can therefore be used by may different tokens. 
    
    For example, a holder account factory could be implemented to create a generic holder account with a set of standard wrapper methods. When calling the factory, governance accounts for different tokens would only need to pass the custodian address and pointers to logic accounts.
    
6. Async logic account update/upgrade. It is not possible to simultaneously update all logic accounts spread across shards. E.g. if pausing a token, then shards will pause before/after each other depending on when the CSTs containing the pause update are processed at their destinations.
7. Async methods. Some non-sharded ERC-20 methods cannot be used identically in a sharded context when implemented using this BSCT standard. E.g. if balanceOf(holderAccount) is invoked cross-shard, then by the time that a balance is returned cross-shard, the balance may have subsequently changed (the holder sent or received tokens while the CST was ‘in transit’). 
8. It would be possible to organise BSCT at a ‘meta’ level, where a governance contract can mint different types of tokens and assign control to sub-governors.
    
    This might be suitable for a future version, but seems best implemented as a service (perhaps developed/run by the community), where token creators pay a supervening governor to take care of the token deployment, adds the new token’s support to existing userbase (holder accounts) etc.
    

### Wallet Developer

1. Wallet developers can create token custodian accounts that manage manage as many tokens as required. In this design it should be easy for wallet developers to update their wallets to support new token standards and new tokens. 

### General

1. SELFDESTRUCT is a tricky way to send tokens, check that there are no other tricky token sending methods that we need to be careful about that might be used to circumvent transfer().
2. Like ERC-20, the BSCT standard is not a guarantee that any particular implementation of the standard is correct and secure. **For example, it is up to developers/governance to ensure that users can only be transfer tokens between holder accounts by correctly implementing holder accounts, logic accounts and interfacing them appropriately.**
3. Industry Standard: This standard can be used in other sharded systems, provided that they implement enshrined tokens that guarantee token refunds for failed cross-shard interactions, which is the main reason for incorporating enshrined tokens.
4. We could add precompiles that implement generic parts of this standard. E.g. precompile for holderAccountFactory to create a generic holder account.

# 4.0 Fungible Token Standard

Section 2 describe the base smart contract token standard that allows enshrined tokens to be extended with EVM smart contract based features. This section defines a set of features (i.e. methods and events) for a fungible token (FT). These features are to be integrated into governance accounts, holder accounts and logic accounts where appropriate.

ERC-20 forms the basis for this standard 

- https://eips.ethereum.org/EIPS/eip-20.
- https://ethereum.org/en/developers/docs/standards/tokens/erc-20/

<aside>
⚠️

Do we want to add/remove any features from the list of vanilla ERC-20 features below?

</aside>

- function name() public view returns (string)
    
    [OPTIONAL] Returns the symbol of the token. E.g. “ABC”
    
    This is already implemented in nilTokenBase for the underlying enshrined token. However, the governance, holder and logic accounts may also implement name()
    
- function symbol() public view returns (string)
    
    [OPTIONAL] Returns the name of the token - e.g. “ABCToken”
    
    This is already implemented in nilTokenBase for the underlying enshrined token. However, the governance, holder and logic accounts may also implement symbol()
    
- function decimals() public view returns (uint8)
    
    [OPTIONAL] Returns the number of decimals the token uses - e.g. 8, means to divide the token amount by 100000000 to get its user representation.
    
    The governance, holder and logic accounts may implement decimals()
    
- function totalSupply() public view returns (uint256)
    
    Returns the total token supply. 
    
    This is already implemented in nilTokenBase for the underlying enshrined token. However, the governance account may also implement totalSupply().
    
- function balanceOf(address _owner) public view returns (uint256 balance)
    
    Returns the account balance of another account with address _owner
    
    This is already implemented in nilTokenBase for the underlying enshrined token. However, the governance and holder accounts may also implement balanceOf(). Note that if _owner is cross shard, balanceOf() requires CSTs such that _owner’s balance may have changed before the CST returning the balance is processed.
    
- function transfer(address _to, uint256 _value) public returns (bool success)
    
    Transfers _value amount of tokens to address _to, and MUST fire the Transfer event. The function SHOULD throw if the message caller’s account balance does not have enough tokens to spend.
    
    Note Transfers of 0 values MUST be treated as normal transfers and fire the Transfer event.
    
    This already exists in the BSCT standard. Within holder accounts permission is restricted to the custodian account, and optionally authorised custodian accounts. Transfer() must be implemented in both holder account method wrappers and at least one logic account. The transfer logic account(s) makes use of the enshrined token’s sendToken() method.
    
- function approve(address _spender, uint256 _value) public returns (bool success)
    
    Allows _spender to use holder account’s transfer() multiple times, up to the _value amount. If this function is called again it overwrites the current allowance with _value.
    
    Approve() must be implemented in holder accounts, optionally wrapping a logic account.
    
    NOTE: To prevent attack vectors, clients SHOULD make sure to create user interfaces in such a way that they set the allowance first to 0 before setting it to another value for the same spender. 
    
- function allowance(address _owner, address _spender) public view returns (uint256 remaining)
    
    Returns the amount which _spender is still allowed to withdraw from _owner.
    
    Implemented in holder accounts.
    

- event Transfer(address indexed _from, address indexed _to, uint256 _value)
    
    MUST trigger when tokens are transferred, including zero value transfers.
    
    A governance account which mints new tokens SHOULD trigger a Transfer event with the _from address set to 0x0 when tokens are created. +++ do we want this???
    
- event Approval(address indexed _owner, address indexed _spender, uint256 _value)
    
    MUST trigger on any successful call to approve(address _spender, uint256 _value).
    

# 5.0 Non-Fungible Token Standard

NFTs are frequently used to represent ownership of elements from a collections of artworks. For example there are 10 000 crypto punks https://cryptopunks.app/.

## 5.1 Proposed Protocol Change

Within our current enshrined token NFT collections can be implemented in one of the following ways

1. For each NFT in a collection create an enshrined token, with total supply = 1. A governance account for an NFT collection can mint as many enshrined tokens as it needs. 

It seems that our protocol can support the creation of any number of distinct enshrined tokens, so it is not a problem if in our system e.g. over time users cumulatively create 1 million NFT collections each with 10 000 elements, totalling 10 billion distinct enshrined tokens. (+++is this really true?).
2. An entire NFT collection is encoded in a single enshrined token. Individual elements in a collection are represented by a positional number system. I.e.  where owning $Y$ out of $Y_{\rm max}$ tokens for the ith NFT in a collection is represented by the number of tokens given by $Y\times Y_{\rm max}^i$ tokens, where $i$ is an exponent, not an index.

However, both these methods are awkward. **Instead we should extend our enshrined tokens to array tokens or token maps, instead of scalar tokens. In an array based token the ith NFT in a collection is naturally represented by the ith element in a token array, or token map.** Further, by allowing elements to be of type uint, the ith NFT can be fractionalised into any number of tokens for partial ownership.

## 5.2 NFT Methods

ERC-721 forms the basis of the NFT standard

- https://ethereum.org/en/developers/docs/standards/tokens/erc-721/
- https://eips.ethereum.org/EIPS/eip-721

<aside>
⚠️

Do we want to add/remove any features from the list of vanilla ERC-721 features below?

</aside>

The non fungible token (NFT) standard implements the following

- function balanceOf(address _owner) external view returns (uint256);
- function ownerOf(uint256 _tokenId) external view returns (address);
- function safeTransferFrom(address _from, address _to, uint256 _tokenId, bytes data) external payable;
- function safeTransferFrom(address _from, address _to, uint256 _tokenId) external payable;
- function transferFrom(address _from, address _to, uint256 _tokenId) external payable;
- function approve(address _approved, uint256 _tokenId) external payable;
- function setApprovalForAll(address _operator, bool _approved) external;
- function getApproved(uint256 _tokenId) external view returns (address);
- function isApprovedForAll(address _owner, address _operator) external view returns (bool);

# 6.0 BSCT Implementation

Things developers must not get wrong: (please add to this list if something is missing)

1. Any method transferring tokens out of a holder account using a CST must set the protocol refund address to the holder account, so that tokens are returned upon CST failure.
2. Only governance accounts, holder accounts, logic accounts can hold tokens. All methods of governance accounts, holder accounts, logic accounts must only send tokens amongst themselves. Tokens must not be sent to custodian accounts, or other zkSharding accounts.
    - If governance accounts, holder accounts or logic accounts implement SELFDESTRUCT. Then SELFDESTRUCT must only send tokens to governance accounts, holder accounts, or logic accounts; mostly likely sending to either another holder account via the transfer logic account, or to the governance account perhaps to be burnt. The token whitelist must be updated to remove the selfdestructed holder account. To ensure the integrity of the whitelist, the whitelist update should be confirmed before the selfdestruct opcode is executed.
    
    I think we should allow SELFDESTRUCT to send tokens because it is a ‘backup’ way of funding an account which is not payable and so it has some use cases. I.e. if you need ETH (our fee token) in a non-payable account, create another account containing ETH, then selfdestruct it and force the ETH into the non-payable account. However, selfdestruct is controversial because once an address is selfdestructed, anyone can then replace it and potentially exploit contracts that are setup to call the selfdestructed version.
3. Holder account factory should create holder accounts with a non-deterministic address, so that a malicious actor cannot ‘front run’ address usage by creating accounts at addresses the factory would use, preventing holder account creation
    - use CREATE2 with a non-deterministic input combining:
        - the custodian address (this limits attacks to specific custodians, rather than all custodians)
        - a recent block hash (this prevents ‘frontrunning’ address creation a long time in advance)
        - User provided salt. Users generate some rng and submit it when calling the factory. This should prevent front running, but may not be possible for automated holder account creation.
        - salt from a randomness oracle
4. If the receiver of a token transfer does not have a holder account there are several options
    1. revert the transfer
    2. deploy a holder account for the receiver and deposit the tokens, paid for by either
        1. the token sender
        2. governance
    3. store tokens and make them available once the receiver creates a holder account
    
    The token designer can choose which method(s) they use, but must consider this scenario.
    
5. For dApps that interact with tokens, it must be understood that reading the balance of a holder account cross shard is slightly different to reading the balance of a holder account on the same shard. When the balance of a cross shard account is received, it is possible that the actual balance has changed, because the holder account has sent or received tokens in the time between reading its balance and the balance being returned to another shard via a CST.
6. The most gas efficient/cheapest method for upgrading and sharding optimisation is to have holder account wrappers point to a central wrapper so that only the central wrapper needs updating, not each individual holder account wrapper.