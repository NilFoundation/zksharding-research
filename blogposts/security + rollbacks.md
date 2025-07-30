# Security + rollbacks

**Authors**: [Vitaliy Kuznetsov](https://t.me/vitalyylativ) ([kuznetsov.vv@phystech.edu](mailto:kuznetsov.vv@phystech.edu))  

# Blogpost plan

**Goals:** We seek to gather constructive feedback from the community and initiate a public discussion on zkSharding, so in the posts like this we strive to clearly outline the problems we are facing and solutions we currently have. Explain the security concerns of sharded systems before full zk finalization, making it clear why these issues matter. Define the problem of a corrupted state and propose a protocol to solve it.

**Target audience:** People whose feedback is interesting to us, mainly researchers and engineers.

1. Introduction
    1. Motivation:
        1. probabilistic security of zkSharding before zk finalization:
            1. For each epoch, a pseudorandom seed is generated based on the users’ activity; this seed is used to generate a mapping of validators to shards. We can assume this seed to be truly random and unbiased. Hence, the mapping produced is also a random mapping.
            2. Validators assigned to a shard run a consensus protocol to commit proposed blocks. This protocol has particular safety guarantees: an invalid block could not be committed if number of malicious validators is less than some safety threshold (third or half).
            3. Since, in our case, the assignment is random, there is a chance that malicious validators may always overtake a single shard and commit an invalid block.
        2. that's why protocol needs a way to detect such behavior and fix the corrupted state.
    2. Problem statement:
        1. introduce requirements for a protocol for rolling back errors once they are detected
        2. highlight that by nature of a sharded system (validators check only validity of their shard) cross-shard messages propagate errors in this system
        3. note that this is a general problem for all sharded systems, not just zkSharding
2. Approaches to fixing the corrupted state
    1. Ignore the problem:
        1. discuss the approach of setting parameters so that the probability of an attack is negligibly low, refer to [these calculations](/scripts/time_to_fail.ipynb):
        
        ![Untitled](/figures/blogposts/security-rollbacks-00.png)
        
        1. note that the size of a shard's committee should be quite high (>hundred) to achieve the desired probability
    2. Partial fixes:
        1. quickly discuss TON's initial proposal
        2. highlight high implementation costs involved in this approach
    3. Full rollbacks:
        1. show [simulation results](/scripts/error_propagation.ipynb), that errors propagate quite fast
        
        ![Untitled](/figures/blogposts/security-rollbacks-01.png)
        
        1. note that after a block processes a malicious message, it becomes subject to future changes (fixing updates or simply a rollback), which will require regenerating validity proofs
        2. summarize: effectively the system will have to redo the majority of unfinalized computations and regenerate corresponding validity proofs
        3. hence, state the idea that we have to stick with a simple and robust approach of a full rollback
3. Proposal outline
    1. Description of the proposal:
        1. Picture of what's going on + protocol steps
        
        ![Untitled](/figures/blogposts/security-rollbacks-02.png)
        
    2. pros and cons
        1. existence of in-protocol rollback mechanism allows manageable committee size
        2. simple and robust proposal (probably besides part with rollback initiation and fraud proving)
    3. Address key questions:
        1. Why don't fork the main shard?
        2. Can there be an alternative design, where we fork every chain?
        3. What will happen with messages, deposits and withdrawals?
        4. etc
4. Possible issues and what is left out of scope (do we want to include it? maybe under "further directions" at the end)
    1. Initiation of the rollback: liveness check mechanism + fraud proving part
    2. Synchronization issues after resampling validators
5. Final thought and conclusion
    1. Summary: with low probability malicious activity can happen, which will be automatically prevented by the protocol
    2. Key idea: hence the actual frequency of such attacks might be much lower since validators won't be motivated to act maliciously
    3. Closing thoughts: since shards operate under a common protocol we can design it so that we don't have to rely heavily on optimistic assumption

## Introduction

zkSharding offers a promising solution for scalability issues in L2s, but it comes with its own set of security challenges, especially before settlement on L1.

![Security and rollbacks article_blog.png](/figures/blogposts/security-rollbacks-03.png)

One such challenge is the *probabilistic security* of the system:
• **Randomness origins**: For each epoch, a randomness beacon generates a pseudorandom seed, for example, based on users' activity. This seed is used to assign validators to shards. Assuming the seed is truly random and unbiased, as a result, the mapping is also random.
• **Сonsensus safety guarantees**: Validators assigned to a shard run a consensus protocol to commit proposed blocks, which provides certain safety guarantees — specifically, an invalid block could not be committed if the number of malicious validators is below a certain threshold ratio $f$, typically a third or half.
• **1% attack**: Hence, due to the randomness of the assignment, there is a non-zero probability that malicious validators could occupy a higher ratio than the safety threshold $f$, dominating a shard, leading to the commitment of an invalid block. This type of attack is commonly referred to as a 1% attack.

Given this risk, the protocol must include mechanisms or incentives to **detect such behavior** and **correct any resulting corrupted state**. While detection is critical and expected to happen promptly (since no valid proof could be generated for an invalid state transition), our focus is on the **correction** aspect.

## Approaches to correcting the corrupted state

### Ignoring the problem (or rather "ensuring this is not a problem")

The most straightforward approach is to configure parameters so that the probability of an attack is negligibly low. Let’s break this down:

Assume that the system has $N$ validators, $n$ is a shard size, $t = \lfloor\frac{N}{3} \rfloor$ is the maximal number of malicious nodes, $f$ is a threshold security parameter of a shard, representing the maximal fraction of malicious nodes on a shard. Then, we can obtain the odds of malicious validators overtaking *a particular shard*, noting that a number of malicious validators occupying a shard, denoted below by $X$, follows [hypergeometric distribution](https://en.wikipedia.org/wiki/Hypergeometric_distribution):

$$

⁍
$$

The next step is to get an upper bound of the probability of malicious validators overtaking *any of $\lfloor \frac{N}{n} \rfloor$  shards*:

$$
p_{\texttt{fail}} < \lfloor \frac{N}{n} \rfloor p_{\texttt{local\_fail}}
$$

Let's assume for concreteness that the epoch duration is one hour; that is, validators are rotated once per hour. Meaning each hour, we flip a coin and with probability $p_\texttt{fail}$ we lose, i.e., the state could be corrupted. Then, the number of epochs before the state could be corrupted follows the [geometric distribution](https://en.wikipedia.org/wiki/Geometric_distribution). We can roughly estimate the number of epochs before a possible state corruption as the mean of this distribution $\frac{1}{p_{\texttt{fail}}}.$ Only roughly, since this distribution is right-skewed with a heavy tail, meaning that typical samples from this distribution might be quite far away from the mean.

![Security and rollbacks article_1.png](/figures/blogposts/security-rollbacks-04.png)

Let’s assume that we set the time until failure to a millennium, that is $p_{\texttt{fail}} = \frac{1}{1000\cdot365\cdot24} \approx 10^{-7}$, and set shard’s security parameter $f=\frac{1}{2}$. From the graph above we can see that achieving the desired probability requires a shard committee size of 200. That amounts only to 5 shards in total. Also, as noted above, from the properties of this distribution we can achieve a fail within 10 years with about 1% probability.

Although this approach partially mitigates risks, it significantly compromises scalability by requiring large committee sizes. As a result, it falls short as a standalone solution.

### Partial Fixes

Another approach involves partial fixes, such as those proposed in [TON’s initial design](https://ton.org/whitepaper.pdf). This method involves tracing back from accounts that initiated a malicious state transition and identifying all accounts affected by this transition. However, this method has several drawbacks:

1. **Proof regeneration and increased finalization latency:**
Since accounts that receive cross-shard transactions from a malicious account are also considered compromised; any shard containing at least one such compromised account becomes subject to state changes. The latter requires the regeneration of state transition validity proofs for the affected shards, leading to increased finalization latency. Our simulations, show that errors can propagate rapidly across shards, i.e. ratio of malicious shards to all shards increases quite fast, so the time to corrupt a half of shards is quite small:

![Security and rollbacks article_2.png](/figures/blogposts/security-rollbacks-05.png)

1. **Impact on User Experience:**
More importantly, while targeting only corrupted accounts might initially appear to be an effective strategy for enhancing user experience, we argue that the most active users are likely to still encounter the impacts of partial state updates. Much like any real-world network, the “network of accounts” has a pattern where a small subset of accounts generates the majority of activity. These high-traffic accounts (DEXes, marketplaces, gaming apps) will almost surely be subject to state corrections. Consequently, this will affect the UX of majority of users who were active at that time.
2. **Fragility:**
Moreover, high implementation costs of partial fixes, coupled with the low probability of their trigger, $p_\texttt{fail}$, make this approach less appealing for a practical adoption.

Given these challenges, we advocate for a *robust and straightforward* approach: implementing a full rollback mechanism described below.

## Proposal outline

Our proposed solution involves an in-protocol rollback mechanism, designed to address corrupted states effectively. Below is a visual representation of the process. Notation: "MB N" stands for the $N$-th main shard block, "SB i" — a block of $i$-th execution shard.

![Security and rollbacks article_3.png](/figures/blogposts/security-rollbacks-06.png)

### Steps:

- **What we want to achieve**:
    - Fork from the last finalized state.
    - Slash malicious validators.
    - Reassign validators on the problematic shard.
- **Protocol processing**: Upon processing a rollback transaction in a main shard block:
    1. *Initiation step:* Check the fraud proof of a malicious behavior
        1. Footnote: The fraud proof submitted in the initiation step is used to demonstrate that malicious behavior occurred in the network. This proof can take the form of a zk proof or a combination of transaction data and a state witness, similar to the approach used in optimistic rollups. Because the malicious state roots are stored on-chain, each node has access to these roots and can independently revalidate the fraud proof against the known malicious state roots. This decentralized validation ensures that the proof is correct and that the rollback is justified.
        2. 
    2. Get the header of the last finalized main shard block, $\texttt{MB}$.
    3. Set the main chain state root to the last finalized state root, $\texttt{MB.root}.$  Continue to work with this state root.
    4. Set the shard blocks hashes to the hashes of shard blocks that were finalized by $\texttt{MB}$: $\texttt{currentBlock.shardBlockHashes = MB.shardBlockHashes}$.
    5. Slash malicious actors — nodes who signed the fraudulent block.
    6. Update consensus parameters and reassign validators if necessary.
    7. Set extra info about the rollback in the block header:
        1. Similar to Ethereum we could introduce $\texttt{extraData}$ field and in case of a rollback set it to the following string:$\texttt{encode("rollback", rollback\_counter, MB.height, reason, malicious\_shard\_id)}.$

### Benefits

- The existence of an in-protocol rollback mechanism allows for a smaller committee size, making the system more efficient and scalable. By embedding a rollback mechanism within the protocol, we disincentivize malicious parties to attempt attacks. Knowing that any corruption they cause will be automatically corrected, attackers are less likely to invest resources in such efforts. The impact of any successful attack is limited to the time scale before zk finalization, which is currently on the order of tens of minutes. As research in proof systems advances, we expect this time scale to decrease, further reducing the potential impact of malicious activities.
- The proposed solution is both simple and robust, providing a straightforward method to address corrupted states. This simplicity helps in maintaining the integrity of the system without introducing unnecessary complexity.

## Final thoughts and further directions

While the proposed mechanism outlined above meets current requirements, there are several areas identified for enhancement and further exploration within the context of zkSharding, including:

- Integration of the Sync Committee, in particular managing correct L1↔L2 communication
- Compatibility with the recently proposed cross-shard communication and transaction ordering protocol, [ShardDAG](https://bit.ly/ShardDAGproposal)

We will continue to publish research on these topics in the near future, you can follow our latest research here

In summary, the probability of malicious activity within our system is set to a reasonably low value, and the protocol is designed to automatically detect and fix such issues. The design of our protocol, which ensures that all shards operate under a unified framework, allows us to minimize reliance on optimistic assumptions about validator behavior.