# Multi-shard Batching

**Authors**: [Raluca Diugan](https://x.com/ctzn101)  

## Highlights/Summary/TL;DR

- Per-shard batching suffers from inefficient CST proving *when prover capacity is relatively large (i.e. not 1-2 blocks)* because batches have to be constrained according to CST source/destination ordering. This means that sometimes we only batch 1 block per shard.
- Multi-shard batching effectively removes this limitation. Blocks of multiple shards are still processed in order, but they do not need to be separated by shard (more efficient verification).
- In both per-shard and multi-shard batching, batches NEED NOT wait for the previous batch to be settled on L1. I.e. we can have *continuous batch generation*. The *pending* batches will be stored on the SC until a (the) proof provider is available.
- In both per-shard and multi-shard batching, batches can be committed for DA in aggregate form via mechanisms such as blob sharing.
- The two-step ordering mechanism ensures that shards are processed fairly and that for any (source, destination) pair of blocks, the source will always be included in a previous or in the same batch as the destination.
- Blocks in the candidate (for batching) list are categorized as either “provable” (i.e. “can be batched”) or “dependent” in case not all its dependent blocks are yet in the candidate list. This ensures that the correct order for each source-destination is established by waiting for all dependencies to be processed.

## Definitions

- `batching` - aggregating shard blocks into a *batch*
    - `per-shard batching` - the batch can only contain blocks from a single shard
    - `multi-shard batching` - the batch may contain blocks from more than one shard
- `proving` - generating a validity proof for a batch. In case the batch is *multi-shard*, a single proof applies to all proven state transitions (one per shard).
- `state`
    - `shard state` - the state of a shard at a given point in time. In case of *multi-shard* batches, for each shard in the batch, the shard state is updated on L1.
    - `global state` - the state of all shards at a given point in time
- `DA` - publishing batch data to L1
- `source` (block/shard) - refers to the block issuing a CST towards another block in a different shard. It also refers to the shard containing the source block.
- `destination` (block/shard) - refers to the block receiving a CST from a block in a different shard. It also refers to the shard containing the destination block.

### Inefficiencies of per-shard batching

One approach to batching is *per-shard batching* where the Sync Committee generates one batch for each shard, and requests from the Prover Network one proof for each batch. 

<aside>
Remember that to prove a CST, we need to prove it exists. In the case of per-shard batching, we cannot prove a CST’s *source* and *destination* blocks at once. This means that we must prove the source block before or at the same time with proving the CST’s destination block.
</aside>

<aside>
**Executing vs Proving CSTs**

**Execution.** When we talk about CST execution, we distinguish between an *optimistic* and a *pessimistic* approach. In the *optimistic* approach, CSTs are executed on the destination chain **before the source chain is settled on L1**. In the *pessimistic* approach, the destination chain must **wait until the source chain is settled on L1 before executing the CST**.

**Proving.** Proving CSTs is decoupled from execution since at the time of proving the CST already exists (i.e. has been executed). As such, we do not need to wait for the settlement of the proof of the batch containing the CST source block in order to generate a batch containing the CST destination block. Additionally, we can batch the two blocks in the same batch.

*However, we may wish to only prove one batch once the previous one is settled. Although this is not necessary: if we settle proofs in the order of the batches, any invalid proof will cause all subsequent batch proofs to be rejected (except in the case of disjoint batch proving which can be independently settled).* 
</aside>

**Inefficient proving with per-shard batching.** In the ideal case, we would max out the prover capacity by fitting as many blocks in a batch as possible, in order to minimize proof verification costs. This is because verification costs are constant. However, the existence of CSTs impedes this goal, as seen in the three figures below:

![Figure 1. Blocks yellow 0 (Y0) and red 0 (R0) are proven. There is a CST from Y1 to R1, which R1 references. In order to prove R1, Y1 needs to be settled. In this case, Y1 through Y4 can be batched together, and the batch proven at once. Once the batch is settled on L1, we can batch R1 through R3 and prove the new state of the red shard.](/figures/l2-mechanisms/multi-shard-batching-00.png)

Figure 1. Blocks yellow 0 (Y0) and red 0 (R0) are proven. There is a CST from Y1 to R1, which R1 references. In order to prove R1, Y1 needs to be settled. In this case, Y1 through Y4 can be batched together, and the batch proven at once. Once the batch is settled on L1, we can batch R1 through R3 and prove the new state of the red shard.

![Figure 2. In this second case, we have three CSTs:
        1. From Y1 to R1
        2. From R1 to Y2, and
        3. From Y2 to R2
In this case, we cannot prove Y1 together with Y2 (and thus with Y3 and Y4) because Y2 depends on R1, which, in turn, depends on Y1 to be proven. As such, we must limit the next yellow batch to Y1, running the prover with a single block.
Similarly, to prove Y2, we must first prove R1 (assuming the state transition caused by Y1 has been proven and settled on L1). Because R2 depends on Y2, we again cannot batch R1 and R2 together. In the end, to prove blocks Y1-Y4 and R1-R3, we must generate four batches to be proven sequentially: Y1, R1, Y2-Y4, R2-R3. Assuming the prover had a capacity for all seven blocks, the multi-shard batching approach is arguably more efficient (at least from the point of view of DA, proof verification, and hard finality on L1).](attachment:c47b4f64-6619-45c1-aa1d-9fd9dbd6f3b0:image.png)

Figure 2. In this second case, we have three CSTs:
        1. From Y1 to R1
        2. From R1 to Y2, and
        3. From Y2 to R2
In this case, we cannot prove Y1 together with Y2 (and thus with Y3 and Y4) because Y2 depends on R1, which, in turn, depends on Y1 to be proven. As such, we must limit the next yellow batch to Y1, running the prover with a single block.
Similarly, to prove Y2, we must first prove R1 (assuming the state transition caused by Y1 has been proven and settled on L1). Because R2 depends on Y2, we again cannot batch R1 and R2 together. In the end, to prove blocks Y1-Y4 and R1-R3, we must generate four batches to be proven sequentially: Y1, R1, Y2-Y4, R2-R3. Assuming the prover had a capacity for all seven blocks, the multi-shard batching approach is arguably more efficient (at least from the point of view of DA, proof verification, and hard finality on L1).

![Figure 3. In practice, we can have much more complex situations like the one above, where there are CSTs between shards every — or almost every — block. By batching blocks per-shard, we can end up with the vast majority of batches consisting of only one block. In this example, the addition of the blue shards and the two additional CSTs causes the creation of *six* batches (instead of one, assuming sufficient prover capacity).](/figures/l2-mechanisms/multi-shard-batching-01.png)

Figure 3. In practice, we can have much more complex situations like the one above, where there are CSTs between shards every — or almost every — block. By batching blocks per-shard, we can end up with the vast majority of batches consisting of only one block. In this example, the addition of the blue shards and the two additional CSTs causes the creation of *six* batches (instead of one, assuming sufficient prover capacity).

**Proving two batches at the same time.** We could potentially prove two batches in parallel and commit them together, lowering DA costs. For example, in Figure 2 above, hypothetical batches Y1-Y4 and R1-R3 could be proven in parallel. Note that in this case, there is a circular relation between the two batches, where each of them represents, in turn, the source and the destination batch of the other. As highlighted above, proving any CST on destination requires that the source block is already proven. In this case, it might be possible to optimistically assume the CSTs exist and thus prove the two batches independently, but require both proofs to be valid in order to accept the state transitions of the individual shards.

**DA and per-shard batching.** If we do not wait for one batch to be settled before generating the next one, we can aggregate per-shard batches and publish their aggregate data in a single DA commit. If we must wait for one batch to be settled before a new one is generated, then we submit DA for each batch, which, in case of one or few blocks per batch is inefficient from a DA perspective (assuming DA capacity is on par or lower with the prover capacity). To aggregate these batches in a per-shard approach, we can utilize blob sharing/aggregation mechanisms. 

## Multi-shard batching

To avoid such inefficiencies and complexity, we propose proving blocks from multiple shards together. For each shard, we prove its respective blocks with respect to the shard state. However, we generate a single proof for the entire batch which attests to the latest state of each shard.

**Properties**

- Let there be *n* shards within the system
- Each batch *b* consists of blocks from at least one shard. Each batch *b* consists of at least one block
- Within each batch *b,* there may be one or more blocks from any shard
- A batch is formed when either condition is met:
    1. the prover capacity is reached
    2. a batching timeout ocurrs
    3. DA limit is reached
- Let $\mathcal{B}$ represent the set of *ordered candidate blocks* for the next batch
    
    <aside>  
    **Candidate Block Ordering**
    
    To determine an absolute order for batching blocks which guarantees that every *destination* block will be in the same batch or in a future batch relative to its pair *source* block **from where a CST was issued, we apply the following two-step ordering approach:
    
    1. **Fairness Ordering.** First, all candidate blocks are ordered by block height per shard, by taking the next block from each shard, in the alphanumeric order of shards until no new blocks are available from any shards. This round-robin-like ordering ensures some level of fairness in processing (batching) blocks from all shards. Otherwise, some shards may have to wait too long for their blocks to be processed.
        - example: $\\ \text{shard 0: }b^5_0, b^6_0 \\ \text{shard 1: }b^3_1 \\ \text{shard 2: }b^7_2, b^8_2 \\ \text{order: }b^5_0, b^3_1, b^7_2, b^6_0, b^8_2$
            - where shards may be at different block heights due to network conditions
            - and the number of blocks per shard does not need to be equal (e.g. shard 1 only has one candidate block)
    2. **Dependency Ordering.** Second, for each candidate block in the ordered list above, we determine the list of its dependencies, i.e. of blocks that must be proven in a previous or in the same batch. As such, for each CST in a block, we collect its source block, compiling a list of all dependencies for each block. 
        - Note that each block will depend by default on its previous block in the same shard. example:
            
            $b^5_0 : b^4_0 \\
            b^3_1 : b^2_1, b^6_0 \\
            b^7_2 : b^6_2 \\
            b^6_0 : b^5_0, b^7_2 \\
            b^8_2 : b^7_2$
            
            ![Figure 4. Gray blocks are already proven in a previous batch (TBD whether settled or just batched). Blue blocks are candidate blocks for the next batch which must first be ordered.](/figures/l2-mechanisms/multi-shard-batching-02.png)
            
            Figure 4. Gray blocks are already proven in a previous batch (TBD whether settled or just batched). Blue blocks are candidate blocks for the next batch which must first be ordered.
            
            5, 3, 7, 6, 8
            
            5, 7, 6, 3, 8
            
        
        Once we determine this mapping, we can update the ordering based on the dependencies:
        
        - since $b^3_1$ depends on $b^6_0$, $b^6_0$ must be proven before or at the same time with $b^3_1$. Similarly, since $b^6_0$ depends on $b^7_2$, $b^7_2$ must be proven before or at the same time with $b^6_0$
        - resulting order: $b^5_0, b^7_2, b^6_0, b^3_1, b^8_2$
    
    **Note**: This order is about batching, and may not correspond to an objective order of blocks’ execution timeline. For example, blocks $b^6_0$ and $b^8_2$ might have been generated in their respective shards around the same time.
    
    With this resulting order, the batching can begin. 
    
    🔴 **IMPORTANT:** The candidates order should be constantly updated with new relevant information (blocks, batch status, etc.). Once a block is included in a batch, it is removed from the ordered candidate list (or its status marked as “batched,” later “proven”, etc.).
    
    - 🔴 **IMPORTANT:** Each block can be simultaneously *source* and *destination*. Within the ordered list of candidates we distinguish between the following three statues of each block:
        - *neutral* - a block is neutral if all of its source/destination blocks are already ordered in the list of candidates
        - *source* - a block is source if there is at least one CST originating in this block for which the destination block has not yet been added to the list of candidates
        - *source and/or destination* - a block is *source/destination* or *destination* if:
            - *destination* - there is at least one CST received in this block for which the source block has not yet been added to the list of candidates
            - *source/destination* - if the block satisfies both the *source* and *destination* conditions under this categorization
        
        When a block is *source* only, it can be immediately batched in its respective order. When a block is both *source* and *destination*, or only *destination*, the block cannot be batched until it reaches *neutral* or *source* status. Any block that depends on a *destination* block, cannot be batched yet.
        
        Since *source* status does not matter for batching, we can simplify this categorization in implementation to only track *destination* blocks until they reach *neutral* status. In this case, *source* blocks have a *neutral* status.
        
        See a simplified categorization below.
        
    </aside>
    
    ### [MISSING, SKIP AHEAD] Proof that this ordering ensures that Destination block will always be proven at the same time or after its corresponding Source block
    
    TODO
    
    ### Categorization of blocks in `ordered_candidate_list`
    
    - Blocks in the candidate list are either:
        - *provable* - all (if any) dependencies of a block are available (and ordered) in the candidate list; or
        - *dependent* - the block itself, or at least one of its dependencies, is not yet added (and ordered) in the candidate list
    
    ![Figure 5. Block-by-block ordering example for blocks in Figure 4. Assume there are some delays in the archival node reading from different shards, s.t. block 7 is read later than 6 by the Sync Committee. When a block is read by the SC, we check all its destination dependencies (and the *provable* status of each of its dependencies). For example, block 5 is neither a CST source, nor a destination. Since its parent block has already been batched, it is added to the `ordered_candidate_list` with the `provable` status, i.e. the block can be immediately added to a batch. In contrast, the next received block — block 6 — has block 7 as a dependency (and 5 from its respective shard which is already batched). Because block 7 is not already in the list, block 6 cannot be cleared (marked) for batching, since, if it was batched without block 7 being batched before or in the same batch, the CST from block 7 to block 6 cannot be proven. Next, when block 3 is received, although its dependency (i.e. CST source block 6) is in the list, since block 6 is not cleared for batching, neither can block 3. If block 3 had no incoming CST from block 6, then **it would have been marked as provable (yellow in the figure). Finally, when block 7 is received, we include it before block 6 since block 7 is the source of the CST in block 6 and thus must be batched first. Block 8 has no dependencies. If block 7 had not been part of the list already, then block 8 would not be provable.](/figures/l2-mechanisms/multi-shard-batching-03.png)
    
    Figure 5. Block-by-block ordering example for blocks in Figure 4. Assume there are some delays in the archival node reading from different shards, s.t. block 7 is read later than 6 by the Sync Committee. When a block is read by the SC, we check all its destination dependencies (and the *provable* status of each of its dependencies). For example, block 5 is neither a CST source, nor a destination. Since its parent block has already been batched, it is added to the `ordered_candidate_list` with the `provable` status, i.e. the block can be immediately added to a batch. In contrast, the next received block — block 6 — has block 7 as a dependency (and 5 from its respective shard which is already batched). Because block 7 is not already in the list, block 6 cannot be cleared (marked) for batching, since, if it was batched without block 7 being batched before or in the same batch, the CST from block 7 to block 6 cannot be proven. Next, when block 3 is received, although its dependency (i.e. CST source block 6) is in the list, since block 6 is not cleared for batching, neither can block 3. If block 3 had no incoming CST from block 6, then **it would have been marked as provable (yellow in the figure). Finally, when block 7 is received, we include it before block 6 since block 7 is the source of the CST in block 6 and thus must be batched first. Block 8 has no dependencies. If block 7 had not been part of the list already, then block 8 would not be provable.
    

### Multi-shard batching Algorithm

- **Proposed Batching Algorithm**
    
    ```python
    # Multi-shard batching algorithm
    
    PROVER_CAPACITY = 
    	{
    		"rw": RW_LIMIT,
    		"keccak": KECCAK_LIMIT,
    		"mpt": MPT_LIMIT,
    		...
    	}
    	
    ## PART 1: Ordering
    # Ordered list of all blocks that may be added to the batch
    # This list is continuously updates as new blocks are generated
    # or included in batches
    
    [ ] continuously retrieve blocks
    [ ] for each new block, determine its dependencies and order
    [ ] insert the new block into the ordered list in the determined order
    [ ] for each block, determine whether it is "provable" or "dependent"
    	[ ] when adding a new block, update the status of dependent blocks 
    			if applicable (e.g. for each dependent block, check if all 
    			dependencies are now in the list; this approach can be eventually 
    			optimized)
    
    ordered_candidate_blocks = [b_5_0, b_7_2, b_6_0, b_3_1, b_8_2]
    
    ## PART 2: Batching
    # The batchID will be set once the batch is sealed
    batch = new Batch()
    
    # Add candidate blocks to batch as long as sealing 
    # conditions are not met
    
    while !batch.prover_capacity_check() && 
    			!batch.DA_limit_check() && 
    			!batch.timeout_check():
    	
    	# Look for the next neutral block to add to batch
    	for block in ordered_candidate_blocks:
    		if block.status != provable:
    			continue
    		# Once a block is found, add it to batch
    		# Break to check the batch capacity
    		else:
    			batch.add(block)
    			ordered_candidate_blocks.remove(block)
    			break
    	
    	
    batch.seal() # also generates batchID
    ```
    

### Walked Through Example with state from Figure 3

In this example, we assume all blocks are already available on the archival node and retrieved one by one in round-robin fashion. We exemplify the batching assuming a prover capacity of 3 and 7, respectively, blocks. In reality, this prover capacity is not expressed in the number of blocks, but here we assume that blocks are similar in terms of their proving demands.

![Figure 6. Unexpectedly, they ended up in the same order after applying the dependency rule.](/figures/l2-mechanisms/multi-shard-batching-04.png)

Figure 6. Unexpectedly, they ended up in the same order after applying the dependency rule.

**Prover Capacity: 3 & Prover Capacity: 7**

![image.png](/figures/l2-mechanisms/multi-shard-batching-05.png)

# Batch Sealing

Once any of the batch sealing criteria is reached (e.g. prover capacity reached or timeout), the batch is sealed. A `batchID` is generated as follows:

<aside>

**Generate batchID** 

First, note that the new state of each shard is given by the latest block from the respective shard. In the example above with a prover capacity of 7 blocks, the first batch gives us the following states:

- yellow shard: Y3
- red shard: R2
- blue shard: B2

These are the latest states of the given shard.

The aggregate/global state is given by (Y3, R2, B2).

Local and global states are submitted as calldata to L1 and stored in mappings.

The batchID could be set as $H(y3, r2, b2)$, where $H$ is a hash function. This provides the batch with a virtually unique ID (extremely low probability of collision), and a way to identify checkpoints of the entire zkSharding state by batch.

y2, r1, b2
y3, r2, b2

y2, r1, b3
STATE = y3, r2, b2
batchID = y3, r2
batchID = H(H(y3, r2), b2)

</aside>

### *State representation when a batch does not contain blocks from all shards

When a batch does not contain blocks from all shards, the state of the shard with no blocks will be set to the latest state of the shard known before the given batch. See the example below: 

![Figure 7. Assume the prover capacity is 5, and the red shard encounters slow block production s.t. at the time of sealing the first batch, no red blocks are included. The global state is given by (Y3, B2, R0), with R0 being the latest known state before the batch. The global state corresponding to the second batch will be (Y4, R3, B3).](/figures/l2-mechanisms/multi-shard-batching-06.png)

Figure 7. Assume the prover capacity is 5, and the red shard encounters slow block production s.t. at the time of sealing the first batch, no red blocks are included. The global state is given by (Y3, B2, R0), with R0 being the latest known state before the batch. The global state corresponding to the second batch will be (Y4, R3, B3).

Once a batch is sealed and a batchID is set, the batch is dispatched to the Sync Committee Task Scheduler who will prepare a proving task for the Prover Network. A new batch will be initialized immediately.

# Proving

Each shard will be proven relative to its latest known state. For example, in the setup from Figure 7, the prover will prove that, based on the old state of the yellow shard Y0, the new state of the shard is Y3. On L1, only Y3 will be recorded as the shard state, and not intermediary states Y1, Y2.

Assuming there is a CST from B2 to Y2, in order to prove the new state of the blue shard Y2, the existence of this CST will also be proven as part of the same proof.

### Proving strategies:

- **Monolithic.** All statements about all blocks are proven at once. In this case, tracing happens for all blocks at once.
- **Per-shard.** For each updated shard state, a proof is generated. All proofs are aggregated into a single proof sent to L1. In this case, tracing can still happen for all blocks at once, and trace files per shard are then used to generate per-shard proofs.

### Horizontally Parallel Proofs

We define a pair of horizontally parallel proofs as two proofs that update disjoint sets of shard states such that the order of proof settlement does not matter. 

<aside>
**Parallel Proofs**

Let A, B, C, D be four shards within zkSharding. At time *t*, their proven states on L1 are: A5, B4, C9, D6.

SC can produce two batches that update disjoing sets of shard states, e.g.:

- proof *a* updates (A, B, D), and proof *b* updates (C)
- proof *a* updates (A, C), and proof *b* updates (B, D)
- or any such combination.

This also applies to three or more proofs.

Let’s take the following two batches:

- (A7, C12) and (B5, D9), each updating the state of individual shards accordingly (e.g. A5→A7)
- The global state proposed by the first batch is (A7, B4, C12, D6)
    - The old state provided is (A5, B4, C9, D6)
- The global state proposed by the second batch is (A5, B5, C9, D9)
    - The old state provided is (A5, B4, C9, D6)

As such, when applying the first batch, we obtain a new state which does not change the states of shards B and D. Similarly for the second batch, states A and C are not updated since they match the provided old state. Instead, when applying them sequentially *in any order*, L1 joins the two provided new global states:

$(a7, b4, c12, d6) \circ (a5, b5, c9, d9) = (a7, b5, c12, d9)$

</aside>

Currently, enabling parallel proof settlement might be unnecessary, and more of an optimization for the future. Some considerations of parallel proving are:

- Advantage: proofs on L1 don’t need to be settled sequentially. As such, we can prove many disjoint batches at once and settle them in arbitrary order, decreasing idle times caused by waiting for proofs.
- Disadvantage: this approach may partially conflict with the fair ordering, unless disjoint batches emerge as effects of network conditions.