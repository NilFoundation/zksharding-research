# L1 Data Availability specification

**Authors**: [Ilya Marozau](https://www.linkedin.com/in/imarozau)  

“Data availability”(DA) refers to ensuring that the data behind a newly proposed block—which is necessary to verify the block’s correctness—is accessible to other participants on the blockchain network. To understand the significance of data availability, this doc reveals the idea of different DA levels for shared rollup.

### DA in sharded rollup

Rollups that utilize the power of sharding should guarantee the following layers of Data Availability (DA):

- **Local Shard DA**: Ensures data accessibility for local network peers. Light clients and validators must be able to access the correct data to verify the state of the shard.
- **Global DA**: Pertains to the latest updates for the main shard. All validators and the sync committee should have sufficient data to verify its new state.
- **Cross-shard DA**: Data must be properly available and pruned during a validator rotation event.
- **L1 DA**: Provides guarantees that anyone outside of the rollup can reproduce and verify the correctness of the entire rollup history. In most cases, when only a centralized sequencer exists, this is the only DA on such rollups.

In this document, we observe the initial core design of L1 DA for the zkSharding technology.

### Settlement and Data Availability Layers

Before we dive into the details of design, architecture, and research, we need to specify some of the zkSharding layers.

**The settlement layer** specifies where and how rollups are secured and reach finality. As a “rollup,” it operates on top of a base network. There are rollups known as “sovereign rollups” that do not have a settlement layer, but nil network is not a sovereign rollup. It utilizes zk proofs to verify state transitions on the base chain (currently Ethereum) to reach finality.

**The DA layer** addresses where data is posted. Some rollups, called “validiums,” do not have such a layer. zkSharding is not a validium technology; it utilizes a DA network to store off-chain data for liveness and verification purposes.

The modularity of modern rollups is crucial, allowing each layer to operate on different solutions, such as, for example, keeping a DA layer on Celestia and a settlement layer on Ethereum. For **zkSharding, there is no specific preference for which layers to use**, and this specification does not mandate any particular configuration. However, **some design decisions are based on the idea of utilizing the same layer** for both settlement and DA.

### Scope of L1 DA on zkSharding

Despite the initial definitions provided in this specification, each rollup details its approach and objectives individually.

In this section, we explicitly specify and elaborate on the specific goals zkSharding aims to achieve with L1 commitments, as well as what it does not intend to achieve and why, with strict alignment.

**What zkSharding L1 DA supports:**

**Liveness:**

One of the most important aspects is the need for additional liveness guarantees, despite zkSharding utilizing a decentralized network of validators and clients. The specifics of the consensus protocol necessitate these guarantees because there is a probabilistically low chance that an entire shard could be controlled by malicious validators. That's enough to set N/k malicious peers (N - total validators, k - shards). While the consensus can manage malicious behavior (e.g. wrong block, transactions), it cannot address issues if these validators decide to remove the whole shard's blockchain. This is where Layer 1 data availability (L1 DA) provides crucial support, completely removing the risk of such occurrences.

The risk of complete loss of shard’s data is one of the most terrible for any blockchain. From the game theory perspective, all bad actors may have higher gain from the reputational and business side, which proves the possibility of the action. 

**State Recovery & DA validity via Accessibility (data retrievability)**

It refers to he mechanisms for state correction or rebuilding. Unlike liveness, recovery provides an opportunity to ensure the correctness of rollbacks, reproduce parts of the state, or verify the correctness of cross-shard or local data availability.

**Data Transparency and Accessibility**

Many infrastructure applications, such as oracles and bridges, rely on data submitted to Layer 1 that has passed proof of equivalence and validity. It is crucial for these applications to depend on info included in the data availability layer. For instance, some applications use Decentralized Verifier Networks (DVNs) to verify the inclusion of cross-chain transactions by relying on data submitted to L1 DA from rollups. This aspect is critically important for zkSharding, as independent execution shards are settled on the main shard, which alone is insufficient for such applications.

**Off-chain Validation**

To ensure the correctness of state transitions, individual users, chains, and infrastructure layers must be able to independently verify these transitions. The first reason is that zkProofs take some time to produce, while data availability (DA) commitments can be several times faster. Advanced or tightly integrated parties can significantly improve their "change response" properties without relying solely on validity commitments. Although they may still face risks such as "committee over control," the potential for state transition failures is mitigated. The second major reason is trust—trust in the verifier. This is a well-known issue, particularly significant to the Ethereum community and its infrastructure. Some robust protocols, such as LayerZero and the new version of [Li.Fi](http://li.fi/), may rely on both zk and off-chain validation at the same time.

**What zkSharding L1 DA doesn’t support:**

**Dispute Resolution**

Validity proofs eliminate the need for dispute resolution from the data availability layer, as they are enough for security. 

**Cross-chain Interoperability**

The current design and approach do not focus on cross-chain interoperability with other Layer 2 solutions. This is due to the lack of composability, EVM-equivalence, and protocol-level interoperability with other solutions, such as for example zkLink and zkSync have, or in Arbitrum, and Optimism.

**Composability**

zkSharding does not integrate composability with the Layer 1 network, nor does it provide any composability within the DA layer.

### Protocol properties

zkSharding's L1 data availability specifies the following key properties necessary for the shared and decentralized nature of the technology:

- **Permissionless:** Anyone can join or leave the committee at will. There is no centralized authority to grant or restrict access to the protocol or censor participation.
- **Fast Response:** The system must react to faults or collisions with minimal delay to avoid blocking the overall operation and finalization of the L2 network.
- **Liveness:** The chance of the protocol stopping operations must be negligible. There should be a rapid way to restore operations even if the committee stops functioning.
- **Non-blocking Operation:** The DA protocol consists of multiple functional blocks, none of which should be able to block or halt the others. It must support asynchronous operation to ensure a fast process.
- **Decentralization:** The protocol must not rely on any single incentivized party for any aspect of the system.
- **Market Efficiency:** The protocol's efficiency should be evident, considering factors such as economics, security, trust, and operation speed, to remain competitive.

### High-level design

 

To offload consensus proposers intentionally while maintaining decentralized, robust, and strict adherence to the claimed properties, we introduce the Synchronization Committee. This protocol component is responsible for ensuring delivery guarantees to the settlement and availability layers. In this model, the proposer's role ceases after block propagation.

For a high-level overview, refer to Figure 1. The Synchronization Committee uses the state of shards and their respective chains as input. An observer component tracks the main shard chain to accurately estimate the potential proof and data availability size. When a certain threshold is reached, the observer requests the aggregator to process a snapshot of size n (blocks). For simplicity, the observer can be thought of as a light node that estimates data growth and the execution cycle of the main shard, while the aggregator functions as an archive node, operating on data from time T to T+n.

The aggregator operates in two dimensions: it processes data for the verifier and composes DA batches along with KZG and Proof-of-Equivalence (PoE) commitments. It prepares k+1-batches (k-shards + main), as each commit must be processed individually. For the verifier, it provides public input, including blocks and state differences from T to T+n.

The verifier acts as the "proof requester". It creates an "proof order" transaction with designated payment for proof generation and then enqueues it with public input in the local request queue. Upon receiving proof from the prover network, the verifier sends the validity proof metadata (e.g. signature of the prover and verifier) to committee storage and proof itself to the proposer.

The proposer composes a bundle of transaction requests from the aggregator and verifier and submits them to L1 appropriately. For example, in the case of Ethereum, it sends batches to blob storage and validity/batch proofs to the chain contract.

The finsherman has a specialized role in verifying whether everything was correctly submitted to L1. If the proposer saves submission statements for a specific block, and the fisherman finds them incorrect or if the proposer commits DA for the next snapshot before including the previous one, the fisherman prepares a proof of non-inclusion for the synchronization committee. The proposer will be slashed, and the fisherman rewarded. This also applies if the proposer stops operating, which is determined if the proposer does not present the hash of submitted transactions within a certain time period.

![Untitled](/figures/l2-mechanisms/l1-da-00.png)

Figure 1. L1 DA design

The on-chain component primarily consists of three parts: the L1 chain, the zkEVM verifier, and the batch (DA) verifier. For example, on Ethereum, the DA verifier is implemented as a precompiled contract. The L1 chain's tasks include updating the state of the main shard, paying for the proof generation if the validity proof is correct (i.e., verification has passed for the latest state from T to T+n), and verifying data availability (DA) equivalence for the main and execution shards.

### Workflow description

In this section, we in details observe the end-to-end workflow: 

1. The observer monitors the execution on the main shard to limit the potential proof size and also tracks individual shards to ensure that the DA batch for any particular shard does not exceed a specified limit.
2. When one of the above takes place, the observer registers a snapshot. This snapshot marks the time (block) on the main shard from T (the latest proven state on L1) to T + n, indicating the point at which the observer considers that it can start to record the next batch.
3. It sends a snapshot to the aggregator that starts processing data from T to T+n. 
    1. For L1 DA, aggregator starts composing k+1 batches. The type of commitment is not specified here, but:
        1. if it’s state diffs — it composes a merkle trie diff for each shard for the snapshot time.
        2. if transactions — aggregates them and prunes not important data (e.g., signatures).
        
        Also, aggregator prepares kzg and poe proofs.
        
        - For kzg, we marshal L1 DA and interpolate it in the polynomial. Then prepare proof, evaluation point and commitment (more details read in EIP-4844 spec).
        - For PoE, there must be a prepared evaluation at the common random point for kzg and fri, and commitment should be embedded into public input for verifier (read PoE spec for details).
    2. For the validity proof, it composes blocks and states. 
4. Then the aggregator send two requests:
    1. Validity proof generation. As mentioned in the design section verifier, creates a special payment “transaction” with a designated price for the proof generation.  It will be signed by the prover who generated the proof. The verifier also keeps the request queue in order and submits proofs in sequential order. Prover network details is not a scope of the spec. 
    2. When validity proof is received from the prover, it passes sanity check and submitted to the proposer. The details are outside of the spec for now.
    3. Send batches to the proposer with corresponding proofs. 
5. Proposer receives multiple requests and ensures strict submission ordering to L1. In general, it works the following way: 
main shard batch → main shard kzg + validity proof → execution shards batches → execution shards kzg + PoE. 
6. The proposer saves the transaction hash and block hash (from L1) in the synchronization committee's global storage. Global storage will play the role of time-limited nodes-shared data. The details of it will be specified in the design of the decentralized committee. This data is sufficient to prove almost any cheating by the proposer with the assistance of the fisherman.
7. **Note: The fisherman role will be reconsidered and redesigned with new suggestions regarding the use of light clients for that.* 
The fisherman role is to track all submissions and, in the event of malicious actions by the proposer, provides proof to the committee. Some examples of such behavior include:
    1. Incorrect submission order: Present the submitted transactions and Merkle Patricia Tree (MPT) proof to show that the submission was incorrect.
    2. Ceasing operations: Demonstrate a lack of commitments in the committee storage; if there are any, show that there are no corresponding submissions on L1.
    3. Posting incorrect transactions: Provide proof that the transactions on L1 or in storage do not match the requests
8. On the on-chain side, L1 blob storage receives batches from both the main and execution shards, with each batch corresponding to a specific shard.
9. The chain contract then receives validity, KZG evaluation point, and proof. First, it opens commitment at the point on submitted data (hash(Commitment)) to verify that the point belongs to the polynomial (original data). Then the point can be used as part of public input of the validity proof. Specific point and details are described in PoE doc.
10. As soon as the block containing the validity proof reaches finality on L1, L2 also achieves hard finality. As soon as finality is reached, SC members are entitled to the reward from the staking contract.