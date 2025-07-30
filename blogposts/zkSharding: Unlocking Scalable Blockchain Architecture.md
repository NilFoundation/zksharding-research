# zkSharding: Unlocking Scalable Blockchain Architecture

**Authors**: [Ilia Shirobokov](https://www.linkedin.com/in/ilia-shirobokov-963466162)

*This is a blog post for [Stanford Blockchain Review](https://review.stanfordblockchain.xyz). We want to bring new audience to our community using their publication channel. Their audience is mostly Stanford students and professors, but also some of researchers from professional community.* 

*In the blog post we want to cover zkSharding in general. We do not suppose that the reader knows anything about sharded blockchains. They had article from Near but it was focused on chain abstraction. In the best case, we don’t want to have just “zkSharding intro” but some twist on it.* 

### Potential contents:

Redefining Multi-Chain Systems As Blockchain Sharding 

1. Setup: Discuss that multi-chain is the new black. Put it like evolution of the old idea. However, they’re not structured so it’s hard to compare them.
2. 
3. Blockchain sharding definition as a network partitioning with parallel computations and communication between chains
4. Taxonomy: define main differentiations between the multi-chain systems using zkSharding as an example. 

# Actual Blog Post [WIP]

## **Introduction**

=nil; is a sharded zkRollup that aims to scale applications by running them in parallel on different shards using zkSharding architecture. In the course of our research, we wanted to find and define parallels between zkSharding and similar systems. This article addresses this question by explaining the key features of zkSharding through a taxonomy of sharded blockchain architectures.

While zkSharding is zkRollup, its rollup-specific features are out of the scope of this article.

### **Sharded Blockchain Architectures**

In this article, we’re generalizing sharding to include blockchain architectures that enable scaling through partitioning of state and execution.

Within this broad scope, we can examine multi-chain architectures like Polkadot or Cosmos; unified sharded systems like zkSharding or NEAR; and even Ethereum’s rollup-centric roadmap.

While these systems vary in architecture and in particular designations around sovereignty and interoperability, they share the same goal of enabling parallel processing to improve the overall scalability of the network, similar to traditional sharding architectures.

### Why We Need a Standardized Evaluation Framework

Although they share a similar goal and there are parallels in their approaches, comparing these architectures is far from straightforward. Each system comes with its own set of trade-offs, whether in terms of decentralization, security, or consensus mechanisms. Without a structured framework for evaluation, achieving a uniform assessment is nearly impossible.

On one hand, a robust evaluation framework would help developers come to the best choice when deciding on a blockchain for their applications.

On the other, such a framework would help protocols developers identify and apply the best market practices and understand which features their customers actually need. 

To address this, we propose a clear taxonomy to help categorize and compare sharded blockchain architectures. In this article, we aim to define such a taxonomy, using zkSharding as a case study.

**Disclaimer:** *While multi-chain systems emphasize creating interconnected ecosystems, it’s important to recognize the role of interoperability—the mechanism that enables data transfer between chains. In this article, we do not consider interoperability solutions separately from the whole system that they create.*

## **Defining Blockchain Sharding**

Sharding in blockchain still lacks a universally agreed-upon definition. However, for the purposes of this article, we define *blockchain sharding* as a mechanism that partitions computational power and state to allow parallel execution of transactions within a blockchain architecture. It splits the original system into shards, where each shard is responsible for processing only a portion of the transactions.

As mentioned above, this broad definition allows us to consider multi-chain systems systems that implement blockchain sharding. For the sake of consistency, we will refer to them as *sharded systems* throughout this article, with their individual chains functioning as *shards*.

In some cases [[one](https://docs.multiversx.com/learn/sharding/), [two](https://shardeum.org/blog/sharding-types/)], blockchain sharding is classified in the following way:

1. **Network sharding** – dividing the network’s nodes into smaller groups.
2. **Transaction sharding** – splitting the transaction load across different shards.
3. **State sharding** – partitioning the blockchain’s state (account balances, smart contract data) into separate shards.

Additionally, sharding can be either *dynamic* or *static*. Dynamic sharding adjusts the number and configuration of shards based on network demands, while static sharding maintains a fixed number of shards.

We do not consider this classification as sufficient for comparing different architectures. For example, to the best of our knowledge, all existing network-sharded projects are also state-sharded. Moreover, state sharding does not explain anything about the underlying mechanisms: zkSharding, Polkadot, and zkSync Elastic Chains can all be classified as state-sharded, yet these projects are very different.

To properly evaluate and compare these systems, we need to extend beyond the traditional categories and consider other factors such as security models, consensus mechanisms, and inter-shard communication protocols.

## Properties of Sharded Systems

The evaluation framework suggested below is based on key properties of sharded systems that define their behavior and efficiency. 

### Shard Creation

When it comes to shard creation, we categorize it into two main types:

- **Permissioned**: In this model, a new shard can only be created with explicit permission, either from the protocol or through social consensus. NEAR is an example of such a system.
- **Permissionless**: Permissionless systems allow anyone to create a new shard at any time without needing approval. For example, zkSync allows anyone to create their own rollup.

Shard creation has significant implications for the overall system, most significantly cross-shard messaging. **Permissioned systems** can adjust shard creation dynamically, scaling based on the network's current load. This ensures efficient resource allocation and allows for stricter control over security and interoperability.

In contrast, **permissionless systems** often result in new chains with their own technical stacks, enabling the creation of specialized shards optimized for specific tasks. However, this flexibility can introduce complexities, such as inconsistent UX and unpredictable security guarantees. For instance, shards with different execution environments, like EVM or WASM VMs, may enhance specialization but reduce interoperability, complicating cross-shard development. Even more, any difference in trust assumptions between shards influences cross-shard messaging capabilities.

### Fee Model

In the context of sharded systems, the fee model defines how transaction fees are calculated across different chains or shards. We distinguish between two primary types:

- **Local Fee Market**: In these systems, each chain or shard calculates its fees separately, either fully independently or with some base fee applied across the entire network (eg. Cosmos).
- **Global Fee Market**: In this model, the protocol defines a unified fee structure for all chains or shards, where fees are the same across the entire system (eg. NEAR or TON).

![image_2024-09-17_124116138.png](/figures/blogposts/zksharding-00.png)

The **global fee market** model makes cross-shard transactions more predictable for users, as they don’t have to worry about varying fee structures across different chains. However, this simplicity comes at the cost of leaving all load-balancing to the dynamic sharding mechanisms, which increases the overall complexity of the system.

### Inter-Chain Structure

 The **Chain Structure** defines the rules for how blocks are connected between different shards. In essence, all sharded systems can be represented in some form of a Directed Acyclic Graph (DAG). Though not always immediately obvious, whenever a transaction from one shard is included in another shard, the destination shard incorporates a fingerprint of the originating shard at a specific time, forming a DAG.

However, the strictness of this DAG structure can vary:

- **DAG with a Single Source**: The most strict structure, where all shards not only maintain links between each other but also share a common ancestor, such as the genesis block. This creates a tightly coupled system with a unified starting point. Sharded systems like NEAR or TON follow this model.
- **DAG with Enforced Updates**: In this model, shards are independent chains but must periodically include information about other shards. For instance, this could involve embedding the block hashes of other shards into each new block. As we discuss in the next sections, zkSharding falls into this class.
- **DAG without Enforced Updates**: In this looser structure, shards only include information from other shards on-demand, typically when a cross-shard transaction occurs.

![image_2024-09-17_124938455.png](/figures/blogposts/zksharding-01.png)

The inter-chain structure, alongside cross-shard messaging, introduces an additional layer to the architecture: the **topology of shards**. Just as networks can have topologies with dedicated nodes to simplify communication, shareded systems may have specific shards designated for coordination. In most cases, these **coordination shards** exist to optimize communication between other shards, facilitating smoother cross-shard interactions. 

Closely related to the **inter-chain structure** property is the concept of **address space partitioning**, which plays a crucial role in how accounts are distributed across shards. There are two main approaches to address space partitioning:

- **Non-Intersected Address Space**: In this model, each address belongs to only one shard at any given time. This means that a particular address A can only exist on a specific shard, ensuring that accounts are uniquely assigned to individual shards.
- **Full Address Space**: In this model, all shards share the same full range of address values, meaning the same address can exist on multiple shards simultaneously. Interestingly, permissionless shard creation usually implies such an address space model (e.g. any interoperable rollup stack).

### Cross-shard messaging

Cross-shard messaging define how cross-shard transactions are executed within a sharded system. They govern several key processes, including:

- How the source shard creates a cross-shard transaction
- How the destination shard validates incoming transaction validity
- Whether cross-shard transactions have the same expressivity as intra-shard transactions or if they are limited in certain aspects
- Whether cross-shard transactions are atomic or non-atomic
- And many other considerations…

Due to the complexity and variety of these mechanisms, it is challenging to establish a strict classification. However, we can outline some of the most important distinctions.

**By Atomic Property:**

- **Atomic**: A cross-shard transaction is either successful on both the source and destination shards, or it is reverted on both. This ensures that either all parts of the transaction succeed or none do, maintaining a consistent state across shards.
- **Non-Atomic**: In this model, a cross-shard transaction can succeed on one shard while failing or being reverted on another. This introduces potential inconsistency but can offer faster transaction processing. The inconsistency is usually solved via resolution process specific for each system.

![image_2024-09-17_155149319.png](/figures/blogposts/zksharding-02.png)

It is important to note that atomic mechanisms can be built on top of non-atomic transactions <add link to our atomic swap>

**By Validation Mechanism:**

- **Optimistic**: In this approach, the destination shard applies cross-shard transactions optimistically, assuming they are valid if they are signed by the validators of the source shard. If an issue arises later, the transaction can be reverted. This mechanism provides faster execution but carries some risk of invalid transactions being processed initially.
- **Full Verification**: The destination shard applies cross-shard transactions only after fully validating their correctness from the source shard.

While **full verification mechanisms** offer strong guarantees of correctness, they often result in reduced computational distribution (i.e., less effective sharding) or introduce delays in cross-shard communication, particularly when using advanced cryptographic tools like ZKPs. 

**By Permission:**

- **Permissioned**: Even when the chain is technically compatible with others, sending cross-shard transactions may still require explicit approval, either from the protocol or the social layer. This is more common when **chain creation** is permissionless, as cross-shard messaging are often permissioned to maintain security and control over cross-chain interactions.
- **Permissionless**: Once a chain or shard is created, it can freely send transactions to other shards without needing any further approval.

## Examining zkSharding Through the Proposed Evaluation Framework

<summary table here with all properties>

To categorize zkSharding within the framework we've established, we must first introduce its core concepts. zkSharding partitions its state into the m**ain shard** and several **execution shards**. The main shard’s role is to synchronize and consolidate data from the execution shards. All validators validate the main shard.

**Execution shards** act as workers, executing user transactions with an extended EVM. They communicate through a cross-shard messaging protocol, enabling the creation of cross-shard applications. Each execution shard is supervised by a committee of validators, and validator sets are periodically rotated across shards.

In addition, **state transition proofs** are generated for each shard periodically. These proofs are then aggregated and sent to the Ethereum network, which serves as the settlement layer. Each shard is finalized upon proof generation, ensuring secure state transitions.

![flows-8a267f0593b35e7ebdf0fb397da29784.png](/figures/blogposts/zksharding-03.png)

### Chain Creation

From the **chain creation** perspective, zkSharding operates as a **permissioned-by-protocol** system: new shards are only created when existing execution shards become overloaded. Each new shard starts from an empty genesis block with no prior user data.

=nil;,'s first version of zkSharding will be **permissioned-by-governance**, where new shards are manually added via system updates. Over time, zkSharding is expected to transition from **static** to **dynamic sharding** while remaining permissioned in terms of chain creation.

### Fee Model

zkSharding employs a **local fee market model**. While a shared base fee applies to all transactions, additional fees are determined via a first-bid model. The shared base fee covers:

- **L1 Proof Verification**: The KZG commitment scheme ensures a fixed cost for proof verification, independent of shard load.
- **Main Shard Maintenance**: Though the main shard does not process user transactions, it plays a vital role in validator distribution and PoS mechanism control.

### Inter-Chain Structure

zkSharding follows a **DAG with Enforced Updates** structure, where execution shards and the main shard are connected through **ShardDAG** rules:

- Each block links to the previous block in its chain
- Each block references a previous block in the main shard
- Each block links to a set of blocks from other shards

This structure ensures periodic updates across shards and maintains coordination. More details on ShardDAG can be found [here](https://ethresear.ch/t/sharddag-ordering-and-exploitation-in-sharded-blockchains/20203).

Additionally, zkSharding follows non-intersected address space architecture. 

### Cross-shard messaging

*At the moment, we’re experimenting with two cross-shard communication protocols. They are very similar and differ only in how they handle errors. For clarity, we present the simpler mechanism below.*

In zkSharding, cross-shard messaging between shards is achieved through cross-shard messaging protocols. Contracts can call precompiled contracts for cross-shard transactions, following these steps:

1. A new transaction (*tx*) is created in the source shard’s block and added to an outgoing message queue.
2. Validators of the destination shard process the incoming message from the source shard’s header.
3. The transaction is added to the destination shard’s block and processed as usual.
4. If *tx* is reverted, a bounce message is sent back to the source shard, requiring the original contract to handle cross-shard errors.

From the developer's perspective, this mechanism is implemented as a classic asynchronous programming paradigm, similar to communicating with smart contracts outside of the dApp chain. familiar from any communication with smart contracts outside of the chain.

From the **cross-shard messaging** perspective, zkSharding operates as:

- **Non-atomic**: Transactions on the destination shard may be reverted, and smart contracts must handle these errors.
- **Optimistic**: Validators in the destination shard process transactions without waiting for zk-proof generation. In case of malicious validators on the source shard, rollbacks can occur. More details can be found in our [security blog post](https://nil.foundation/blog/post/security_zkSharding).
- **Permissionless**: Any shard can send transactions to other shards without additional setup, ensuring free-flowing cross-shard communication.

### Combinations of the Properties

Let’s explore how specific combinations of zkSharding's properties enable a set of interesting features:

**Permissioned Chain Creation + Optimistic Non-Atomic cross-shard messaging** **= Application Scaling**

With **Permissioned Chain Creation**, zkSharding provides control over the execution environment of all shards, ensuring a **unified execution environment** across the network. This consistency allows developers to split their applications across shards and process transactions in parallel.

For example, a decentralized exchange (DEX) can distribute its liquidity pools across different shards. In this setup, high demand on one trading pair will not affect the performance of other pairs, even in the short term. To make this scalable architecture practical, **optimistic, non-atomic cross-shard messaging** ensures that cross-shard transactions are processed rapidly. While non-atomicity allows for potential reversion, it offers the speed needed for real-time application scaling. In this case, transactions can be processed as fast as two block times without delay for additional verification. 

![image (3).png](/figures/blogposts/zksharding-04.png)
At the moment, we’re actively improving this design to make development experience of such applications even better. For more on this concept, see our [post on sharding as parallelism](https://nil.foundation/blog/post/sharding_as_parallelism).

An additional challenge here is creating developer-friendly programming tools for handling async communication. Follow along with the development of zkSharding’s async developer experience [here](https://docs.nil.foundation/nil/getting-started/essentials/handling-async-execution).

**DAG with Enforced Updates + Local Fee Model = Market-Driven Load Balancing**

zkSharding’s **chain structure** allows users to manage their accounts across shards. In a non-intersecting account space model, users can “move” smart contracts from one shard to another. In a full address space, this could be achieved through mirroring the contract across multiple shards.

When each shard operates with its own gas pricing under a **local fee model**, a natural, market-driven load-balancing mechanism emerges. Application owners can choose to migrate their contracts to less congested shards based on the current gas prices. In theory, this migration can even be automated using on-chain triggers, dynamically optimizing for shard performance and gas costs.

This is part of our ongoing research and future work in zkSharding design.

 

**Optimistic Non-Atomic Cross-Shard Messaging + DAG with Enforced Updates + ZKPs = Trustless Parallel Computation**

We’ve already discussed how **optimistic, non-atomic cross-shard messaging** facilitates efficient cross-shard applications. However, we must also address the computational power limitations of individual shards: for example, today we cannot run complex mathematical modeling on any EVM chain.

In zkSharding, we can add more shards without necessarily requiring more validators. If a shard is sufficiently decentralized, each validator can simply operate an additional machine for a new shard.

*Note: There is a misleading belief that adding new shards means adding new validators. However, the whole point of sharding is to utilize parallel computational resources. This means that adding a new shard requires existing validators to run additional servers to maintain it. Adding new validators does not improve the security or scalability of the system.*

The next step involves applying **ZKPs** (Zero-Knowledge Proofs). As ZKPs become faster, the economic incentive to attack an optimistic shard decreases, allowing for a smaller validator set and shorter optimistic windows. As ZKP technology advances, each shard becomes more computationally efficient. Note: a validator set still has to be decentralized to ensure censorship resistance.

This leads to a vision of a future with **thousands of shards**, each with a small validator set, processing data in parallel. This would function as a **decentralized, trustless cloud computer**, scaling blockchain computing to an entirely new level. 

While this vision may still be a distant future, it is the future we are working toward with zkSharding.

## Conclusion

With the proposed framework, we're enabling builders to pick and choose properties they need in their sharded systems. For example, a fast high-throughput DEX that needs to interact with dApps on other chains might need a sharded system with non-atomic, optimistic and permissionless cross-shard messaging. Another application might need something else--the point of the framework is to decrease the time builders have to spend dealing with architecture complexity and enable faster development and ecosystem growth.