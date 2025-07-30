# Token Hooks for Enshrined Tokens 

**Authors**: [Aleksandar Damjanovic](https://t.me/aleksweb3) ([aleksdamjanovicwork@gmail.com](mailto:aleksdamjanovicwork@gmail.com))  

## Abstract

This RFC outlines the proposed extension of enshrined tokens in zkSharding with token hooks. This extensions aims to provide developers with greater flexibility and control.  Token hooks enable functionality such as fee deduction, transfer pausing, blacklisting and other programmable logic. 

## Motivation

The decision to introduce token hooks is driven by the need to implement customizable transfer logic ,while maintaining enshrined tokens at the protocol level , including but not exclusively:

- Fee deductions on transfer
- Blacklisting (preventing user’s account from transferring/receiving the token)
- Pausing of all transfers on the particular shard/all shards
- Custom Logic

By integrating hooks, we enable token developers to customize transfer behavior while keeping fundamental token mechanics enshrined within the protocol.

## Design Overview

### Hooks

Each shard $S$ must maintain a map between a set of *tokenID*s and a set of *tokenHooksAddress*es located on $S$. Each *tokenHooksAddress* implements two hooks:

1. ***tokenHooksAddress.Send*(numTokens, receiveAddress) → numHookTokens**
    - Called during execution of precompiles and transactions that send tokens
    - Can modify the transferred amount (e.g., applying fees, adding incentives)
    - Can prevent transfers if conditions (such as blacklisting) are met.
    
    It accepts:
    
    - numTokens (uint256) - the number of tokens specified in the transaction or precompile call)
    - receiveAddress (address) - the account specified in the transaction or precompile call as the receiver of the tokens)
    
    It returns: 
    
    - numHookTokens (int256)  - adjusted transfer amount,  change in token balance for tokenHooksAddress

1. ***tokenHooksAddress.Receive*(numTokens, receiveAddress) numHookTokens**
    - Called during execution of opcodes and transactions that receive tokens
    - Can adjust the received amount (e.g., deductions, distributing a percentage to some other contract)
    - Can prevent receive if conditions (such as blacklisting) are met.
    
    It accepts:
    
    - numTokens (uint256) - the number of tokens specified in the transaction or precompile call)
    - receiveAddress(address) - the account specified in the transaction or precompile call as the receiver of the tokens
    
    It returns: 
    
    - numHookTokens (int256)  - adjusted transfer amount,  change in token balance for tokenHooksAddress

### Updating Hook Address

Additionally, the token owner can call *updateTokenHooks*(*shard*, *newTokenHooksAddress*) function that updates a token’s *tokenHooksAddress* in *shard*. 

<aside>
💡

**The hooks are an extension of the enshrined token standard, nothing prevents the developer from creating a token that doesn’t use hooks. All they need to do is to not specify the hookAddress so that the hooks are not invoked on token transfer.** 

</aside>

### Hook Call Sequence

The execution order of hooks in a transaction is structured as follows:

senderAccount → sendHookAddress |→| receiveHookAddress → receiverAccount

The |→| indicates a potential cross shard call, whereas →is a call to the same shard.

### Integration of Hook Calls in Existing Precompiles

The hooks are to be inserted into the following precompiles/transaction types: 

- **`SEND_TRANSACTION (0xfc)`**
- **`ASYNC_CALL (0xfd)`**
- **`SEND_TOKEN_SYNC (0xd2)`**
- **`SEND_REQUEST (0xd8)`**

## Protocol Enforced rules

- Hooks must be executed when a token transfer occurs via relevant precompiles  (if the hook accounts are deployed on source and destination shard)
- Hooks cannot be invoked recursively.
- Tokens are not transferred to the hookAccount when invoking sendHook or receieveHook. Instead, sendHook and receiveHook are passed a numTokens parameter, and return another numHookTokens parameter indicating to the protocol the number of tokens to add/remove from the hookAccount balance
- The refunds still happen if the transaction fails/reverts like in the initially designed token standard
- Hooks are per-shard - a token developer must deploy the receiveHook on the destination shard if needed.

## Transition from ERC-20

Moving from ERC-20 to the enshrined token standard in zkSharding requires adapting token logic to a hook-based approach to ensure compatibility. Since tokens are fundamental to DeFi, this transition necessitates modifications in DeFi protocols, particularly replacing the approve and transferFrom pattern.

Key differences:

- No Balance Mapping in Smart Contracts  - this design choice remains consistent with the previous enshrined token proposal.
- Transfers Use Precompiles Instead of transfer()  , approve() and transferFrom  - token transfers occur via precompiles, with hooks handling custom logic when configured. The DeFi protocols are to build their logic around these precompiles.
- Hooks replace transfer logic customization - developers can implement Send and Receive hooks to customize their token via pausing, blaklisting, fee collection etc.

The new enshrined token standard will be able to be extended via extensions like the classic ERC-20. Some of these extensions are: 

- Capped - capping the total supply
- TokenBurnable - burning of own tokens
- Votes - support for voting
- Pausable -  pausing transfers of the  token - utilizes hooks
- Freeze  - freezing users account - utilizes hooks
- Wrapper - for wrapping/creating derivative tokens of a token which gets locked on the wrapper contract
- Flashmint - for supporting flashmints

## Conclusion

This RFC proposes token hooks as a flexible extension to enshrined tokens in zkSharding, enabling custom transfer logic while maintaining protocol-level guarantees. Hooks facilitate use cases such as fee deduction, blacklisting, and transfer pausing without modifying core token mechanics.