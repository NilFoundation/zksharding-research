# TEEs & zkSharding

**Authors**: [James Henderson](https://t.me/JamesHendo)  

# Trusted Execution Environments & zkSharding

Trusted execution environments (TEEs) have been proposed as potential replacements for zero-knowledge proofs (ZKPs) for some scenarios. Might TEEs replace ZKPs in zkSharding?

First, some context on the use of ZKPs in zksharding. In zkSharding for scalability reasons the transactions for each shard are only processed and checked by a subset of validators. However, a shard’s subset of validators could be malicious, so zkSharding requires ZKPs of shard blocks are produced to later verify that each shard block is not malicious, without having to directly process shard blocks.

An alternative approach to ensuring a valid shard block is to require that the shard block is produced by a TEE, so that it cannot be malicious. To do so would require a specific TEE feature called attestation. Some (but not all) TEEs have an attestation feature that provides cryptographic proof that a specific piece of software is run within a specific TEE and the software has not been tampered with. This attestation feature must be able to ensure that it is not possible for a validator to *appear* to use a TEE, but not actually use a TEE and therefore be able to create a malicious state update.

Attestation works because each TEE has its own public/private key pair. The private key is used to encrypt any data that the TEE outputs for use elsewhere. A TEE can publish its public key so that an actor that receives some data from the TEE can 

- decrypt the data for use, and
- verify that the data was produced by a TEE, because the public key can be cross-referenced to a list of TEE public keys publicised by the TEE manufacturer.

## Comparison between TEE  and ZKP Security & Trust Assumptions

A sharded blockchain could be built using attesting TEEs; however, it would involve different security and trust assumptions compared to zkSharding’s current ZKP approach. It is not clear that TEE’s centralised trust assumptions would be acceptable.

### TEE Security and Trust Assumptions

- TEE relies on hardware security, trust in the hardware manufacturer and integrity of the hardware.
- Hardware manufacturers are a central point of trust. This is against the ethos of decentralised blockchains because the TEE blockchain could be compromised by one participant (the manufacturer). Some ways hardware manufacturers could be compromised are
    1. A failure in the manufacturer’s internal systems that allows TEE private keys to be leaked. This may allow malicious actors to impersonate a TEE.
    2. A design flaw so that TEE trust properties do not hold. TEEs are complex and could contain flaws, problems have been found in existing TEEs, and likely new problems will be found.
        
        Intel has disclosed several vulnerabilities in the last year https://www.intel.com/content/www/us/en/developer/topic-technology/software-security-guidance/overview.html
        
        An exploit has been able to extract attestation keys (https://aepicleak.com/), which may allow a validator to *appear* to run a TEE when it doesn’t; a critical security flaw for zkSharding.
        
        A new Intel SGX flaw was just announced today! (27/08/24) Mark Ermolov: “Intel HW is too complex to be absolutely secure!”  https://x.com/_markel___/status/1828112469010596347 
        
    3. Governments can simply demand that manufacturers (secretly) provide them TEE private keys. If this occurs, then storing private keys (by the manufacturer, government or government contractor) increases the probability of a leak in 1.
        
        Governments may require hardware attestation services to allow them to impersonate TEE attestation. I.e. government runs some modified code in non-TEE hardware but the manufacturer is required to enable attestation that it was unmodified code executed in a TEE.
        
- The manufacturer’s service providing certificates could be compromised, so that
    - Malicious actor imitates the service to provide certificates as being associated with a TEE, when they are not.
    - The service becomes unavailable meaning that TEEs can no longer be verified and therefore state updates can no longer be trusted.
- Recommended Reading: A survey on the (in)security of trusted execution environments https://doi.org/10.1016/j.cose.2023.103180
- For a user a TEE seems a good security measure for keeping your wallet secure (signing within a TEE). However, using TEE for securing (potentially) hundreds of millions/billions TVL in a blockchain protocol is a much greater incentive for attackers to find an exploit, compared to breaking into user wallets. This huge incentive combined with a history of regular disclosed vulnerabilities makes TEEs unappealing.

### ZKP Security and Trust Assumptions

- ZKPs are purely cryptographic and do not rely on trust in special hardware.
- ZKPs are complex to design and produce, and therefore could contain flaws.

## TEE Performance

TEEs have reduced performance compared to running non-secure code. However, because TEEs vary dramatically in their design it is difficult to make broad claims about TEE performance. Further, TEEs are still emerging and therefore current limitations may not apply in the near future. 

Currently Intel’s Intel’s SGX creates application specific secure enclaves, whereas AMD’s SEC creates secure virtual machines. These are two very different approaches, with different tradeoffs.

- Intel’s SGX has a specific 128 or 256 MB section of memory for use by all application secure enclaves. If memory requirements are exceeded, then performance is reduced due to encrypting page files for RAM.
- AMD’s SEV allocation of protected memory to a VM is limited by total system memory.

VM based TEEs generally have shorter execution times particularly when handling memory and I/O intensive workloads. However, the overheads of application based TEEs are lower with CPU intensive workloads and since application based TEEs have stronger trust models, they may be preferred for CPU intensive work.

Compared to non-TEE processing, TEE processing is often 2 - 5 times slower.

Note that TEE currently offers better secure compute performance compared to homomorphic encryption and secure multi-part computation.

Recommended Reading: An Experimental Evaluation of TEE technology Evolution: Benchmarking Transparent Approaches based on SGX, SEV, and TDX https://arxiv.org/html/2408.00443v1

## Incorporating TEEs into zkSharding

Depending on the level of trust placed in TEEs zkSharding can make use of TEEs in the following ways

1. Lowest trust (TEE security failure does not cause zkSharding to fail): 
    - TEEs are used to users to secure user wallets (create and sign txs inside a TEE),
    - Constrain MEV by having block building performed inside a TEE which runs a non-exploitative tx ordering system.
2. Medium trust.
    - Validators run node software in TEEs to make it harder to create malicious shard blocks, improving the security and likelihood of finality of shard blocks. ZKPs are still used to provide final security and validation of shard blocks. This would reduce the probability of rollbacks, improving user experience and trust in the system.
3. Full trust:
    - Validators run node software in TEEs to ensure non-malicious shard blocks, and other non-malicious behaviours. No ZKPs are used.
        
        It seems unlikely that the centralised manufacturer TEE trust assumptions and single point of failure would be acceptable to users for replacing ZKPs in zkSharding.
        

To incorporate TEEs in 2 and 3 above, zkSharding would require that validators run their nodes within a TEE. This would make it harder for malicious validators to produce malicious blocks because they would need to subvert the TEE or zkSharding’s TEE authentication method. TEEs may also introduce a new attack vector in which zkSharding’s liveness can be prevented if the (centralised) TEE authentication method fails.

TEEs can change the (probability of) faults for the landscape of BFT consensus; malicious state transitions should be much less likely. However, using TEEs would not remove the need for a consensus algorithm because consensus is about forming agreement in the network on specific state transitions, not only correctness of state transitions. For example, a block proposer could be offline, or late to propose a block and validators need to agree (using a consensus algorithm) to skip that proposer/block, or not.

The below sketches a potential process for verifying that data received from a validator was produced by the validator in a correctly functioning TEE.

1. The validator’s TEE runs zkSharding code which generates a report on TEE configuration, code, data, hardware identifier (CPU type), public key, and nonce (to prevent replay attacks) that fingerprints the TEE. This report is combined with data to be sent for zkSharding protocol communication.  The TEE uses its (private) attestation key to sign the report and data.
2. The data is sent to the validator pool. Validators retrieve the TEE’s manufacturer’s certificate, retrieved from the TEE manufacturer (e.g., Intel's Attestation Service, AMD's Secure Encrypted Virtualization Certificate Authority). The TEE encrypted report and data is verified against the manufacturer’s certificate. The validator can then decide if the data came from (was signed by) a genuine TEE, or not.
3. Assuming 2 is passed, the report is checked to ensure that the correct code, environment etc is being used by the TEE. If this passes checks, then the data is used.

Validators would need to interact with manufacturer attestation services e.g. Intel runs an attestation service for SGX (https://www.intel.com/content/www/us/en/developer/tools/software-guard-extensions/attestation-services.html). Of course this introduces centralisation and a single point of failure.

## BlockChains that use TEEs

Some blockchains use TEEs for privacy (so that on-chain user activity is private, with MEV benefits for users) and/or blockchain security purposes to prevent malicious validator activity.

 Examples:

- Secret Network - https://docs.scrt.network/secret-network-documentation/introduction/secret-network-techstack/privacy-technology/intel-sgx/overview
    - Security: Uses Intel’s SGX, transactions are processed in TEE/SGX
    - Privacy: Transactions are encrypted, and only decrypted for use within the TEE, retaining privacy for users.
- Oasis Network - https://oasisprotocol.org/security-and-tees
    - Whitepaper - https://docsend.com/view/aq86q2pckrut2yvq
- Phala Network
    - Targets verifiable computation: AI and web3
    - Uses offchain workers in secure enclaves (TEE)
- zkSync
    - Indicates they are integrating TEEs for two factor authentication; briefly mentioned here https://www.youtube.com/watch?v=bVPzbNKs5cw&t=619s
- FlashBots (MEV boost) use TEEs in their design of SUAVE for privacy and integrity of block building
    - https://writings.flashbots.net/suave-tee-coprocessor
    - https://writings.flashbots.net/introducing-rollup-boost