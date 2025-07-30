# Rollbacks Design

**Authors**: [Vitaliy Kuznetsov](https://t.me/vitalyylativ) ([kuznetsov.vv@phystech.edu](mailto:kuznetsov.vv@phystech.edu))  


## Problem statement

Our current global safety guarantees are probabilistic until state change proof is submitted, implying a small probability of system failure. That comes from the fact that in a sharded system, the global system's safety is bottlenecked by the safety of a single shard, whose validators are pseudo-randomly sampled from a global pool of validators. So, malicious validators can overtake a single shard and corrupt not only its state but also the state of other shards by sending invalid messages. Note that outlined malicious activity is only possible before state change proof is submitted to the main shard, i.e. malicious actors can corrupt only not yet finalized state. Therefore, the protocol must be able to correct the cluster's corrupted state.

<aside>
💡 Most of the argumentation below is based on the following idea: the probability of such failure is set to be small (by construction), so the expected/amortized impact on the system will be negligible. In other words, we can safely stick to the simplest and the most robust approach without losing much.
</aside>

Here, we describe possible approaches to fix a corrupted state. Ideologically, we have 2 approaches: full rollbacks and partial state fixes.

## The case for partial fixes

1. Idea: add updates on top, fixing the current states. The idea is proposed in TON. The main argument for it is that the fraction of corrupted accounts in the system is too small to expose the whole system to the full state rollback. It might work for them, but in our context, we have the following problems:
    1. zkSharding is highly sensitive to errors; even one invalid message can invalidate a shard. The zkVM won't be able to generate a valid state change proof, causing the entire shard to become invalid. Moreover, errors propagate quickly (see simulation results in [scripts](/scripts/error_propagation.ipynb)), requiring almost the entire system to fix its state. This delays the system's finality as proofs for state changes need to be regenerated.
        
        ![Untitled](/figures/rollup/rollbacks-design-00.png)
        
    2. Probably, for this to work, the entire validator committee would need to handle fixes, downloading the state of large parts of the system. Otherwise, not all errors might be addressed.
2. We can avoid these issues by incurring additional complexity on zkVM: if zkVM will be able to handle temporarily invalid states, then partial fixes indeed might make sense. However, this is completely unnecessary.
3. Due to the low probability of an attack and the high implementation complexity, we opt for a simpler solution: full rollbacks.

## Initiation

For completeness, here we also outline the initiation process of a rollback.

- At least one shard stalls proof generation for a state change (we consider it an attack on the system's latency). The protocol to handle such attacks is only vaguely presented right now, and it is out of the scope of this document. After this mechanism is triggered, the malicious shard's not yet finalized blocks are revalidated by a different committee. If an error is found, fraud proof is submitted to the main shard.
- Liveness attack mechanism -> particular shard's state is revalidated (by a larger committee or a new proof generator) -> fraud proof is submitted to the main shard -> rollback is triggered
    - Note: Active proof producers should be able to submit fraud proof. However, system safety cannot rely on them.

### Exemplary cases

1. VM violation: some tx was invalidly executed. Default case for rollbacks.
2. DA attack: shard doesn’t publish block content. Don’t rollback in that case.
As I understand, DA guarantees can be linked with block validity predicate: if block is not verifiably available then don’t attest it on the main shard. This should be properly addressed in further research on in-protocol DA solution.
3. Stalling proof generation: proofs for a shard’s state change are not generated on time. Don’t rollback.
In this case protocol should initiate liveness check, i.e. revalidate current blocks with a larger/different committee.

## Rollback

The goal is to safely reset the system's current state. The simplest and safest way is to roll back to the last finalized point of the system. To make this work, we have to ensure that we stay compatible with consensus and zkVM.

### Protocol design overview

- **Approach 1**: Fork from the safe point; disregard all unfinalized execution blocks and start building again from the last finalized state.
    - This approach can trigger a slashing mechanism on execution chains because it will look like validators are double-signing. So, we have to be careful not to slash honest validators. To address this, we note that adding a round on a main chain after a rollback will solve the problem: it will make previously signed execution blocks invalid on a protocol level. This main chain block will naturally contain necessary consensus and state updates. Note that we don’t need to fork the main chain.
    - Note: the disregarded chain of blocks became invalid due to two reasons:
        1. coupling between main chain and shardchains
        2. reassignment of validators (though we probably want to reassign only problematic shard’s validators)
    - Therefore, after a rollback is triggered, new main chain block will
        - reset hashes of shardchains to the latest finalized point
        - update consensus parameters (e.g. reassign validators if needed, since the committee is already enlarged)
        - slash malicious actors
- **Approach 2**: Continue to build on top of the existing chain but change the system's state roots on each shard.
    - In contrast with the previous approach, we don’t even fork shard chains, but rather manually reset state roots. Requires holding malicious state on-chain.
    - Note that holding malicious data on-chain makes generation of state transition proofs a bit more involved and not straight away compatible with zkVM. It could be made compatible with zkVM: proof generators, before producing proof, simply disregard irrelevant blocks. Irrelevant blocks = not finalized blocks that come before rollback.
    - This is quite similar to NEAR’s approach on rollbacks, where they continue to build on top of shard chains:
        
        ![Untitled](/figures/rollup/rollbacks-design-01.png)
        
    
    Both these approaches are quite similar, yet the first one (forking finalized state) seems a bit more natural and doesn't require to store invalid data in the shard's blockchain. It is proposed to continue with this proposal: *continue to build main shard, fork execution shards from the last finalized state.*
    

### What to do with committees

The committee for the malicious shard has to be changed: either resampled or increased to a neighborhood.

- We probably want to disregard malicious nodes (those who signed malicious block) from the validator set. That is, *validators → shards assignment* should support excluding particular validators from the mapping and dynamic reassignment to a particular shard
- Note on DA and synchronization:
    - This will inevitably stall progress on this particular shard due to synchronization issues. But we can take it, since amortized costs are small.
    - At the same time, we have to make sure that shards’ state is available to new committee. That is, compatibility with rollbacks and validator resampling should be addressed in design of in-protocol DA solution. In the worst case, we can rely on data, published on L1 to restore the last finalized state.

### Retrieving the last finalized state root

The proposal assumes that honest parties have an access to the last finalized state root and the corresponding validity proof. We need to store in VM storage the following descriptors of the finalized state:

1. main shard’s state root
2. hashes of blocks from execution shards

For concreteness, let’s assume that the finalized state is checkpointed via *block header* of main shard blocks, since it contains both listed fields. 

### Important notes

- We cannot fork both main and executions shards. The history becomes ambiguous and it’s not obvious how to resolve fork (old/malicious and new block could be accepted again). Honest validators could be slashed or rollback could be triggered once again. To address that issue note that if we don’t fork main shard then only one valid fork of shard chains can exist, see picture below. This property is due to *coupling* between main shard and execution shards.

![Untitled](/figures/rollup/rollbacks-design-02.png)

- If after rollback we continue to build on top of the existing main chain then, as in the approach 2, we might face some integration difficulties with zkVM (handling a temporarily invalid state). A simple workaround to this problem is to simply exclude invalid blocks at a stage of proof generation.
- Also, some cross shard transactions might be required to execute again, hence, content of these transaction has to be available.

## Protocol specification

![Untitled](/figures/rollup/rollbacks-design-03.png)

1. **Rollback initiation**
    - *Liveness check mechanism* is initiated and the concerning state is revalidated by a larger committee. The mechanism for a liveness check is out of scope for this document.
    - **As a result:** Main shard receives a rollback initiating tx with a fraud proof.
2. **Execution**
    - **What we want**:
        - Fork from the last finalized state.
        - Slash malicious validators.
        - Reassign validators on the problematic shard.
    - **Protocol processing**: Upon processing a rollback transaction to a main chain block:
        1. Check the fraud proof.
        2. Get the header of the last finalized main shard block, $\texttt{MB}$.
        3. Set the main chain state root to the last finalized state root,
        $\texttt{MB.root}$. Continue to work with this state root.
        4. Set the shard blocks hashes to the hashes of shard blocks that were finalized by $\texttt{MB}$:
        $\texttt{currentBlock.shardBlockHashes = MB.shardBlockHashes}$.
        5. Slash malicious actors — nodes who signed the fraudulent block.
        6. Update consensus parameters and reassign validators if necessary.
        7. Set extra info about the rollback in the block header:
            1. Similar to Ethereum we could introduce $\texttt{extraData}$ field and in case of a rollback set it to the following string:$\texttt{encode("rollback", rollback\_counter, MB.height, reason, malicious\_shard\_id)}.$
    - **Enforcing**: such processing requirements can be enforced by adding a special *rollback precompile* with the above steps.
    - **Client’s local processing:**
        1. Backup the current database state to safeguard against implementation bugs.
        2. Revert the database to the last finalized state.
        3. Apply new updates as usual, in a temporary context.
        4. Disregard the backed up state after rollback is finalized.
3. **Continuation of work**
    - Validators obtain the new main chain block. From it they check the updated assignment, and if they are required to validate new shards. In that case, they start synchronizing with the corresponding committee and obtain the shard's state.

## Security

For now let’s outline the security and reliability related questions/concerns, since we don’t have a full details on several important subprotocols, like liveness check and proof generation.

### Fragile points and further research

1. Obviously, the functionality handling **fraud proofs** must be designed with a great attention, since problems with it can halt the system.
2. Validators’ assignment reconfiguration can **stall progress** on some shards due to synchronization issues, especially if the shards state became quite large. We could address this issue in the *liveness check mechanism* design. On the other hand, if we decide to simply regenerate validator assignment in a rollback block, then we might face some inner (cross shard) DA issues, where state of a particular shard might not be available.