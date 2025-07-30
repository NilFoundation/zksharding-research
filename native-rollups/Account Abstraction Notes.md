# Account Abstraction Notes

**Authors**: [James Henderson](https://t.me/JamesHendo)  

**_Motivation_**: Our sharded zkSharding design has account abstraction built in However, pivoting to a native rollup cluster means we likely need to use Ethereum’s steps toward account abstraction. 

# 1.0 Summary

Our sharded zkSharding design has account abstraction built in - all accounts are smart contracts, EOAs don’t exist. However, pivoting to a native rollup cluster means we likely need to use Ethereum’s steps toward account abstraction. 

There are a lot of EIPs related to account abstraction, ~20ish proposed/scheduled for the next 3 Ethereum hardforks. Ethereum seems like it will eventually have account abstraction, and native rollups probably will too (assuming they aren’t replaced) though it will take years.

However, two of the most immediate EIPs related to account abstraction are EIPs 4337 and 7702, these are discussed in this doc.

- EIP 4337 is only infrastructure, it isn’t a protocol change. Our community could implement it at any time, or we can do it for them now, or later.
    - Solves most account abstraction use cases, though in a awkward, complex way (it requires a bunch of infrastructure). It isn’t adequate for the long term and further work/upgrades are required.
    - Not new, used by serious projects.
- EIP 7702 will be implemented in Pectra-2025, and presumably will be supported in the future by the native rollup EXECUTE. We won’t have to do anything specific at the protocol level other than use an up to date EVM - probably up-to-date EVM will be a requirement of native rollups anyway.
    - Solves the execution part of AA, EOAs ‘contain code’ e.g. allow users to do multiple operations atomically, custom gas payment.
    - Doesn’t solve security; there is still an EOA key that can control everything, even if some custom auth exists ‘under’ the EOA key.

# 2.0 EIP 4337- Account Abstraction Using Alt Mempool

## 2.1 Useful Resources

https://eip.fun/eips/eip-4337

https://eips.ethereum.org/EIPS/eip-4337

https://www.erc4337.io/docs

https://www.youtube.com/watch?v=PZ8svp68NXM

https://www.youtube.com/watch?v=mmzkPz71QJs

## 2.2 Summary

- no protocol changes, only uses smart contracts and off-chain infrastructure
- user has a smart contract wallet that uses custom authorisation, features etc
    - it isn’t an EOA, the user doesn’t need an EOA private key
        - authorisation is any custom authorisation method in the wallet.
    - must have correct interface to interact with the entry point smart contract (see below)
    - must include gas payment, either
        - user pays
        - paymaster - someone else (e.g. dApp) pays
    - user’s smart contract wallet can be created using a dApp wallet factory
- Users create `UserOperation` which are a bit like a tx and submit them to a special `UserOperation` mempool
    - describes the user’s intent (calls)
    - has user wallet authorisation data attached (e.g. biometric data)
        - this allows a 3rd party to use your smart contract wallet’s features.
- A bundler (like a sequencer or block builder) bundles a set of`UserOperation` from the `UserOperation` mempool (from various users) into a tx which is signed by the bundler’s EOA. The tx calls the special entry point smart contract with the bundle of `UserOperation` s attached as call data.
- The entry point smart contract processes the bundle of `UserOperations`
    - e.g. https://etherscan.io/address/0x5ff137d4b0fdcd49dca30c7cf57e578a026d2789#code
    - bundler pays the gas directly, but the interface requires that gas is reimbursed
        - user doesn’t need ETH to submit a tx if another account pays for gas as a paymaster.
    - Calls the user wallet account authorisation
    - If auth succeeds, makes the user’s requested calls/deployment
    

## **2.3 Features of using EIP 4337**

- increases gas cost compared to standard EOA tx, due to extra wallet call operations and dApp interactions.
- users can bundle multiple actions together
- gas can be paid in different tokens, or paid by dApp paymaster
- custom wallet security, access and recovery.

![Fig 1. A schematic of the EIP 4337 bundling infrastructure.](/figures/native-rollups/account-abstraction-notes-00.png)
Fig 1. A schematic of the EIP 4337 bundling infrastructure.

## 2.4 EIP-4337 Infrastructure

There are various tools/infrastructure that implement EIP 4337

- Reference design: https://github.com/eth-infinitism/account-abstraction

Various smart wallets (incomplete list)

- https://docs.stackup.sh/docs/erc-4337-overview
- https://www.alchemy.com/smart-wallets
- https://docs.safe.global/home/what-is-safe

# 3.0 EIP7702 - Set EOA account code

## 3.1 Useful Resources

https://eips.ethereum.org/EIPS/eip-7702

https://eip.fun/eips/eip-7702

https://eip7702.io/

https://www.youtube.com/watch?v=_k5fKlKBWV4

https://www.youtube.com/watch?v=WG_0EiHtKlc

## 3.2 Summary

- Will be a part of Pectra 2025 https://ethereum.org/en/roadmap/pectra/
- Currently EOAs don’t contain code; as part of AA, we want EOAs to ‘contain code’.
- EIP7702 introduces a new transaction type (`TX_TYPE=0x04`, `SET_CODE_TX_TYPE`) that updates an EOA so that it contains specific ‘delegation designator’ code - a pointer to a smart contracts
    - SetCode tx provides an authorisation list containing a set of addresses (pointers) containing functionality you want an EOA to have
        - These addresses are used like delegate call; the bytecode at the endpoint of the address is ‘cut and pasted’ into the EOA.
            - Intention is for the pointer endpoint to be a smart contract wallet
        - Other accounts (or the EOA itself) can now call the EOA and use the pointers as if the EOA were a smart contract with bytecode
        - the delegation can be updated with a new setCode tx.

- EIP7702 solves the execution part of account abstraction - EOAs can effectively contain code.
- EIP 7702 does NOT solve the security part of account abstraction - the EOA is still controlled by its private key, not custom authorisation. Even though the EOA may point to a smart contract wallet with custom authorisation, the private key can be used to change the EOAs delegation/pointers.

## 3.3 Implementations

EIP-7702 is live on the Sepolia, Holesky, Odyssey Ethereum TestNets, and will be included in the Ethereum Mainnet Pectra 2025 hard fork.

EIP7702 can be tested in tools like (incomplete list)

- https://www.paradigm.xyz/2025/02/announcing-foundry-v1-0
- https://docs.zerodev.app/sdk/getting-started/quickstart-7702

# 4.0 Scroll Account Abstraction

Scroll accounts are the same as Ethereum https://docs.scroll.io/en/technology/chain/accounts/

- EOAs
- Contract accounts

https://scroll.io/blog/towards-the-wallet-endgame-with-keystore

Scroll uses a minimal rollup called keystore whose purpose is to manage account authorisation

- based on a proposal by Vitalik https://notes.ethereum.org/@vbuterin/minimal_keystore_rollup
- Uses a minimal L2 that is only used for user security
    - minimal operations to update and store keys,
    - use ZKP for privacy
- Separates verification logic and asset holdings; assets can be dispersed in multiple places, but code to manage them is in one place.
    - Could be used to authenticate accounts across L2s
- Works on top of EIP 4337, and 7702, not in place of
    - e.g. keystore rollup provides authorisation data for smart contract wallet in 4337 or 7702