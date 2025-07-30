# (Part 2) What We Need to Prove to L1 in the Mainnet 

**Author**: Handan Kilinc Alper ([handan.kilinc@alumni.epfl.ch](mailto:handan.kilinc@alumni.epfl.ch))  

In [Part 1](/l2-mechanisms/(Part%201)%20What%20We%20Need%20to%20Prove%20to%20L1%20in%20the%20Mainnet%20.md), we outlined the security guarantees that a zkSharding proof must provide to L1. Our focus was on defining *what* we need to prove such as transaction validity, state integrity, and proper linking of blocks, without prescribing *how* this proof is implemented.

In this part of the document, we shift our focus to **how we represent zkSharding’s state** in a way that both satisfies these guarantees and remains practical for implementation

Initially, our design considered representing the entire zkSharding state ****i.e., the state of all execution shards from the perspective of the main shard. While conceptually clean, this approach creates practical challenges:

- It requires the proof to process all execution shard data centrally, which can exceed prover capacity, especially as the number of shards or transaction volume grows.

As a result, we now consider an execution-shard-centric representation. That is:

- Each execution shard is responsible for proving its own state transitions and their correctness.
- The main shard remains responsible only for verifying finalized block signatures and referencing shard states.

Each execution shard block maintains a **direct (visible) link** to its predecessor via the `PrevBlockHash` field in its header. While forks can occur, this ensures that each block belongs to a linear history, defined by following parent links recursively.

However, internal transactions introduce **indirect (invisible) links** between shards. These occur when a block in shard $i$ includes an internal transaction that originated from shard $j$ i.e., an outgoing message in block $B_k^j$.

To ensure that these cross-shard links are provable and consistent, we introduce the following **inclusion constraint**:

<aside>
👉🏻

A block is provable only if all blocks it references—either through visible (previous block) or invisible (incoming internal transactions) links—are:

- present **in the same batch**, or
- present in a **previous batch that is already proven**.
</aside>

This ensures that **every referenced block** is either:

- Proven in the current circuit, or
- Already committed to L1 in a previous proof.

### Why Is This Necessary?

Recall the **internal transaction validity relation** defined in [Part 1](/l2-mechanisms/(Part%201)%20What%20We%20Need%20to%20Prove%20to%20L1%20in%20the%20Mainnet%20.md). It states that for an internal transaction $tx$ to be valid:

1. $tx$ must appear in the **outgoing transaction tree** ($OutTx$) of some execution shard block, and
2. That block must be either:
    - In the current batch, or
    - Proven in a previous batch and properly linked to the current batch.

Let’s consider an example:

> **Example:** Suppose block $B^i_5$ in shard $i$ includes an internal transaction $tx$ that originated from $B^j_3$ in shard $j$.
> 
> 
> To prove that $tx$ is valid:
> 
> - We need to show that $tx \in B^j_3.OutTx$
> - We need to prove that $B^j_3$ was part of the batch proven in $gst_ℓ-1$, or is part of the current batch $gst_ℓ$.

If $B^j_3$  is *not* included in either the current or previous batch, we cannot validate the Merkle inclusion proof for $tx$. This **breaks the soundness** of the proof system because we’d be asserting the validity of an untraceable transaction.

To ensure efficiency and scalability, it is crucial that the state commitment is defined such that the prover does not require access to all previous batches to generate the L1 proof.

To address this, we introduce two proposals, each offering a different method to represent and update the state commitment while preserving security and correctness guarantees.

## Proposal 1: Enforcing Message Order during Tx Inclusion

This proposal introduces modifications to both the **block structure** and **block validation rules** to ensure that cross-shard (internal) transactions are processed in strict sequential order. 

**Block Structure Change:** Each block in execution shard $j$ maintains an extended view of previously generated internal transactions. Instead of only including the new `OutTransactions` (internal transactions) generated in that block, the `OutTransactions` tree is incrementally extended by appending the newly created messages to those already included in the previous block. Periodically, this tree can be pruned to reduce size and improve efficiency.

To enable ordering, each execution shard $i$ maintains a **sequence number** per destination shard $j$, denoted as $\text{seq}_j$. When shard $i$ generates a new internal transaction destined for shard $j$, it performs the following:

- Increments its local sequence counter $\text{seq}_j$.
- Associates the message with a unique key in its `OutTransactions` Merkle tree, where the key is computed as $\mathsf{Hash}(j || \text{seq}_j)$

This mechanism enables a total order on cross-shard messages from shard $i$ to shard $j$ based on their sequence numbers.

**Block Validation Role:** To preserve message ordering, the execution shard $j$ must enforce the following rule when processing internal transactions originating from shard $i$:

- Maintain a record of the **last successfully executed sequence number** from shard $i$, denoted as $\text{last\_seq}_i$.
- Accept a new message from shard $i$ only if its sequence number is exactly $\text{last\_seq}_i + 1$

If the message meets this condition, it is processed, and $\text{last\_seq}_i$ is updated accordingly. Otherwise, the message is deferred until the gap is resolved.

Given these structural changes, we next explain how our prover efficiently shows the statement represented [here](/l2-mechanisms/(Part%201)%20What%20We%20Need%20to%20Prove%20to%20L1%20in%20the%20Mainnet%20.md).  

For this, we need to first define $\text{Commit}(\mathbb{B}_0, \ldots,\mathbb{B}_\ell)$. We let 

$$
⁍
$$

where $f$ is a function. We will talk about how this can be instantiated in the end. Remember that $h_{n_i}$ is the latest block hash of the execution shard $i$ in batch $\mathbb{B}_\ell$.

💥  If we represent the state in this way, the commitment does not need previous batches, it just needs the latest batch $\mathbb{B}_\ell$ in order to obtain the represented state. Why it is important because the prover does not need all previous batches to show $\mathcal{R}_\text{com}$ (See [here](/l2-mechanisms/(Part%201)%20What%20We%20Need%20to%20Prove%20to%20L1%20in%20the%20Mainnet%20.md) for the details of the relation) i.e., correctness of the state commitment. Therefore, we convert $\mathcal{R}_\text{com}$ the following relation, which is equivalent to the relation defined [here](/l2-mechanisms/(Part%201)%20What%20We%20Need%20to%20Prove%20to%20L1%20in%20the%20Mainnet%20.md).

$$
\mathcal{R}_{\text{com}} = \Bigg\{ \Big(gst_{\ell}; \{\mathbb{B}[i][n_i]\}_{i =1}^m\Big) \quad \Big| \quad   gst_{\ell} = f(h_{n_1}||h_{n_2}||...||h_{n_m}) \text{ where } \\h_{n_i} = \mathsf{Hash}(i,\mathbb{B}[i][n_i].\text{bhead})
\Bigg\}
$$

Since $\mathcal{R}_\text{com}$ is updated with the equivalent form, now we check the relations which use $\mathcal{R}_\text{com}$.

💥 We next revisit the relation that links two consecutive batches:

- **Old Representation**: Required looking up full batches and using positional indexing (e.g., `last block` of batch).
- **New Representation**: Simplified by comparing headers and reusing the commitment function.

💡 In particular:

- If a shard does not produce new blocks in $\mathbb{B}_\ell$*, we assume $B^i_0 = B^i_{-1}$* (i.e., same block reused).
- We no longer need explicit checks for "last" block. This is handled via the header and hash linkage in $\mathcal{R}_\text{block}$.

$$
\mathcal{R}_{\text{link}} = \Bigg\{ \Big((gst_{\ell-1}, gst_{\ell}); (\{B^j_{-1}\}_{j=1}^m, \mathbb{B}_\ell)\Big) \quad \Big|\\(gst_{\ell-1}; \{B^j_{0}, \}_{j=1}^m) \in \mathcal{R}_\text{com}, (gst_{\ell}; \{B^j_{-1}\}_{j=1}^m) \in \mathcal{R}_\text{com}, \forall j \in \{1,\ldots, m\}:\\((B_0^j,j); (\mathbb{B}_{\ell}, 0)) \in \mathcal{R}_\text{isBlock}, \Big(B_0^{i} = B_{-1}^i\text{ OR } (B_{-1}^i,B_0^i) \in \mathcal{R}_\text{block}\Big)\Bigg\}
$$

The current form of $\mathcal{R}_\text{link}$ is equivalent to the form in [part 1](/l2-mechanisms/(Part%201)%20What%20We%20Need%20to%20Prove%20to%20L1%20in%20the%20Mainnet%20.md). The only different part is that 

1. $\mathbb{B}_\ell[j] =\emptyset$ : It is replaced with the condition $B_0^{i} = B_{-1}^i$ which implies that the state of execution shard $i$ in the new batch is not changed.
2. $((B^j_{-1},j); (\mathbb{B}_{k}, n_j)) \in \mathcal{R}_{\text{last}}, ((B_0^j,j); (\mathbb{B}_{\ell}, 0)) \in \mathcal{R}_{block},(B_{-1}^j, B_{0}^j) \in \mathcal{R}_\text{evm}, \big(k = \ell-1 \text{ OR } \forall t \in \{k+1, \ldots, \ell -1\}, \mathbb{B}_t[j] = \emptyset  \big)$: The generic form of relation in part 1 checks with this condition if the latest proven state of a particular execution shard is extended by the current batch. In our current representation of the state  $(B_{-1}^i,B_0^i) \in \mathcal{R}_\text{evm}$ is enough to link the latest proven batch to the first block of the new batch.

💥 Since all internal transactions generated in previous blocks and previous batches are part of the `OutTransactions` tree in the latest block of the batch, provers do not need to look at the previous batches’ internal transactions to show the validity of internal transactions as represented $\mathcal{R}_\text{tx-validity}$ in [part 1](/l2-mechanisms/(Part%201)%20What%20We%20Need%20to%20Prove%20to%20L1%20in%20the%20Mainnet%20.md). The prover simply needs to show that the transaction and the batch belongs to the following relation.

$$
\mathcal{R}_{\text{tx-validity}} = \Bigg\{ \Big(tx; \mathbb{B}_\ell[j][n_j]\Big)\quad\Big| 
tx\in B_{n_j}^j.OutTx \\\text{ where } tx.\text{shard}_s = j, \mathbb{B}_\ell[j][n_j] = B_{n_j}^j \Bigg\}
$$

<aside>
‼️

This only proves origin from the source shard, not whether the transaction was already executed (i.e., replay protection). To guarantee this, we introduce an additional condition in the EVM execution relation.

</aside>

💥 The represented $\mathcal{R}_\text{tx-validity}$ guarantees only that the transaction is generated by the execution shard $j.$ However, it does not show that whether this transaction has been executed in the previous batches. The current block production rule which follows certain order is introduced to mitigate this problem. Therefore, we need to add new conditions in the relation $\mathcal{R}_\text{evm}$ to guarantee  orderly execution.  Additionally, the source shard sequence number should be part of block header, which automatically binds them to the state commitment of the source shard and block context should include it so that EVM makes sure that the transactions are executed in order. So we assume that $bhead.\text{seq}$ accesses the sequence numbers of the latest executed internal transactions in the block i.e. $bhead^j_k.\text{seq} = \{\text{seq}_1^j, \ldots, \text{seq}_m^j\}$ where $bhead_k^j$ is the block header of $B_k^j$. We show the additional conditions in red below.

$$
\mathcal{R}_{\text{block}} = \Bigg\{ \Big((B^j_{k-1}, B^j_k); \mathbb{B}_\ell\Big) \quad\Big| \\\Big((B^j_{k-1},j); (\mathbb{B}_{k-1},k-1)\Big), \Big((B^j_{k},j); (\mathbb{B}_{k},k)\Big) \in \mathcal{R}_\text{isBlock} \\
\text{EVM}(InTx_k^j||ExTx_k^j,st_{k-1}^j,{\color{red} bhead_{k-1}^j.\text{seq}},  bcontx_k^j)\rightarrow st_k^j, OutTx_k^j\\  Root(InTx_k^j) = bhead_k^j.\text{InTransactionsRoot} \\  Root(OutTx_k^j) = bhead_k^j.\text{OutTransactionsRoot} \\ \\  Root(st_j^j) = bhead_k^j.\text{StateRoot} \\ H(bhead_{k-1}^j, j) = h_{k-1}, bhead_{k}^j.\text{PrevBlock} = h_{k-1} \\ {\color{red} bhead_k^j.\text{seq} = \{\text{seq}_1^j, \ldots, \text{seq}_m^j\}:} \\{\color{red} \text{seq}_i^j = \max\{tx.\text{seq}: tx \in InTx_k^j, tx.\text{source} = i\}\cup\{bhead_{k-1}.\text{seq}[i]\}}\Bigg\}
$$

<aside>
‼️

This might require an additional change in the EVM logic to enforce orderly execution.

</aside>

💥 Since in this proposal, EVM enforces sequence checks based on per-source sequence numbers, our prover don’t need to prove uniqueness of internal transactions anymore [($\mathcal{R}_\text{unique}$).](/l2-mechanisms/(Part%201)%20What%20We%20Need%20to%20Prove%20to%20L1%20in%20the%20Mainnet%20.md)

💥 As we mention above, we need pruning mechanism to decrease size of `OutputTransactions`  tree. Therefore, if pruning occur during block production, our prover has to show the correctness of this operation to represent the state of an execution shard with the correct internal transactions i.e. The prover must be able to prove that pruning did not remove unexecuted transactions.

After these updates, the final proven relation is updated as follows:

$$
\mathcal{R} = \Bigg\{ \Big((gst_{\ell}, gst_{\ell-1}, l2tol1Root, l1Hash); (\mathbb{B}_\ell, \{B^j_{-1}\}_{j =1}^m)\Big) \quad \Big| \quad\\ \Big((gst_{\ell-1}, gst_{\ell}); (\{B^j_{-1}\}_{j=1}^m, \mathbb{B}_\ell)\Big)  \in \mathcal{R}_\text{link},\\ \forall j \in \{1,\ldots,m\},\forall k \in \{1, \ldots, n_j\}, \Big((B^j_{k-1}, B^j_k);\mathbb{B}_\ell\Big) \in \mathcal{R}_\text{block},\\ \forall tx \in B_k^j.InTx_k^j, \Big(tx; \mathbb{B}_\ell[i][n_i]\Big) \in \mathcal{R}_\text{tx-validity} \text{ where } tx.\text{shard}_s = i,\\\Big((l2tol1Tree, l1Hash, nonce, gst_\ell);\mathbb{B}_\ell\Big) \in \mathcal{R}_\text{bridge}, 
\Bigg\}
$$

In short the prover proves to L1 the following:

- The global state is correctly computed using the latest block hashes of execution shards.
- Each block’s state is the result of valid EVM execution.
- All internal transactions executed in the current batch were either:
    - newly created in the same batch, or
    - carried over from previous batches via the extended OutputTransactions tree.
- Transactions are executed in order using sequence numbers bound to the block headers.
- Bridge-related state (`l2tol1Root`, `l1Hash`, etc.) matches the storage of `L2BridgeMessenger` at the end of the batch.

## Proposal 2: Separate State of the Prover

This proposal avoids modifying the block production logic or the EVM. Instead, the prover is responsible for tracking the state of *unexecuted internal transactions* for each execution shard using a dedicated Merkle tree.

For each execution shard $i$, the prover maintains:

- A Merkle tree $\text{utree}^i$ whose leaves are unexecuted internal transactions,
- And its root $\text{uroot}^i$, which is *not* part of the block data but becomes part of the global state committed to L1.

<aside>
💡

We note that the same proposal works if the prover keeps a single tree instead of separate tree for each execution shard. 

</aside>

💥 Hence, the **global state** on L1 is now represented as a combination of:

- The latest block hash of each execution shard, and
- The root of its unexecuted transaction tree:

$$
⁍
$$

Therefore, our prover has to additionally show the correctness of these roots, which are part of the global state, because they are not part of the block data. 

💥 We need to show that the $\text{uroot}_\ell^i$ is correct with respect to the previous $\text{uroot}_{\ell-1}^i$

- If transaction is deleted in the unexecuted transactions tree with root $\text{uroot}_{\ell-1}^i$, then it should be a part of $InTx$ of the destination execution shard.
- If a new transaction is added to the unexecuted transactions tree with root $\text{uroot}_{\ell-1}^i$, then it should be a part of $OutTx$ of block of the source execution shard.

Let’s define a function $\mathsf{Diff}(\text{utree}_\ell^i,\text{utree}_{\ell-1}^i)$ which outputs the leaves of $\text{utree}_\ell^i$ that do not belong to $\text{utree}_{\ell-1}^i$.  

$$
\mathcal{R}_\text{uroot} = \Bigg\{\Big((\text{uroot}_\ell^i,\text{uroot}_{\ell-1}^i);(\text{utree}^i_\ell, \text{utree}^i_{\ell-1}, \mathbb{B}_\ell)\Big) \quad\Big|\\ \mathsf{MTroot}(\text{utree}_{\ell-1}^i,\text{uroot}_{\ell-1}^i), \mathsf{MTroot}(\text{utree}_{\ell}^i, \text{uroot}_{\ell}^i),\\ (\forall tx \in \mathsf{Diff}(\text{uroot}_\ell^i,\text{uroot}_{\ell-1}^i), \exist B^{i}_j \text{ such that } tx \in B^i_j.OutTx, \\tx \notin \cup_{t}\mathbb{B}_\ell[tx.\text{shard}_d][t].InTx), \\ (\forall tx \in \mathsf{Diff}(\text{uroot}_{\ell-1}^i,\text{uroot}_{\ell}^i), \exist B^{k}_j \text{ such that } tx \in B^k_j.InTx)) \Bigg\}
$$

- $(\forall tx \in \mathsf{Diff}(\text{uroot}_\ell^i,\text{uroot}_{\ell-1}^i), \exist B^{i}_j \text{ such that } tx \in B^i_j.OutTx, tx \notin \cup_{t}\mathbb{B}_\ell[tx.\text{shard}_d][t].InTx)$ shows that the newly added transactions to the unexecuted transaction tree of the execution shard $i$ are part of $OutTx$ of a block of $i$ in the batch $\mathbb{B}_\ell$. Additionally, the destination shard has not executed this newly added transaction.
- $(\forall tx \in \mathsf{Diff}(\text{uroot}_{\ell-1}^i,\text{uroot}_{\ell}^i), \exist B^{k}_j \text{ such that } tx \in B^k_j.InTx))$ shows that the deleted transactions are executed by the destination shards.

💥 Even though that our global state commitment is different than proposal 1, we define $\mathcal{R}_\text{com}$ similar to the proposal 1 which shows the knowledge of hashes that constructs the global commitment. We integrate the unexecuted root commitment to $\mathcal{R}_\text{link}$.

$$
\mathcal{R}_{\text{com}} = \Bigg\{ \Big(gst_{\ell}; \{\mathbb{B}[i][n_i]\}_{i =1}^m\Big) \quad \Big| \quad   gst_{\ell} = f((h_{n_1},\text{uroot}^1)||...||(h_{n_m},\text{uroot}^m))\\\text{ where } h_{n_i} = \mathsf{Hash}(i,\mathbb{B}[i][n_i].\text{bhead})
\Bigg\}
$$

💥 As a result of this, the relation $\mathcal{R}_\text{link}$ is defined as follows:

$$
\mathcal{R}_{\text{link}} = \Bigg\{ \Big((gst_{\ell-1}, gst_{\ell}); (\mathbb{B}_{\ell}, \text{utree}_\ell^i, \text{utree}_{\ell-1}^i\}_{i = 1}^m\Big) \quad \Big|\\(gst_{\ell}; \{B^j_{0}, \}_{j=1}^m) \in \mathcal{R}_\text{com}, (gst_{\ell-1}; \{B^j_{-1}\}_{j=1}^m) \in \mathcal{R}_\text{com},\\\forall i \in \{1,\ldots,m\}, \Big((\text{uroot}_\ell^i,\text{uroot}_{\ell-1}^i);(\text{utree}^i_\ell, \text{utree}^i_{\ell-1}, \mathbb{B}_\ell)\Big) \in \mathcal{R}_\text{uroot}, \\\forall j \in \{1,\ldots, m\}:B_0^{i} = B_{-1}^i\text{ OR } (B_{-1}^i,B_0^i) \in \mathcal{R}_\text{evm}\Bigg\}
$$

💥 Given that $\text{uroot}_{\ell-1}^i$ is proven that correct in the previous batch, the transaction validity is guaranteed simply by showing an inclusion proof.

Transaction validity is shown with the following relation:

$$
\mathcal{R}_{\text{tx-validity}} = \Bigg\{ \Big(tx; (\mathbb{B}, \text{utree}_{\ell -1}^i)\Big)\quad\Big| i = tx.\text{shard}_s, \\\exist k: 
tx \in \mathbb{B}[i][k].OutTx \quad\text{OR} \quad tx \in \text{utree}_{\ell - 1}^i \Bigg\}
 
$$

After these updates, the final proven relation is updated as follows:

$$
\mathcal{R} = \Bigg\{ \Big((gst_{\ell}, gst_{\ell-1}, l2tol1Root, l1Hash); (\mathbb{B}_\ell, \{B^j_{-1}\}_{j =1}^m,  \text{utree}_\ell^i, \text{utree}_{\ell-1}^i\}_{i = 1}^m)\Big) \quad \Big| \quad\\ \Big((gst_{\ell-1}, gst_{\ell}); (\mathbb{B}_{\ell}, \text{utree}_\ell^i, \text{utree}_{\ell-1}^i\}_{i = 1}^m\Big)  \in \mathcal{R}_\text{link},\mathbb{B}_\ell \in \mathcal{R}_{\text{unique}},\\ \forall j \in \{1,\ldots,m\},\forall k \in \{1, \ldots, n_j\}, \Big((B^j_{k-1}, B^j_k);\mathbb{B}_\ell\Big) \in \mathcal{R}_\text{block},\\ \forall tx \in B_k^j.InTx_k^j, \Big(tx; (\mathbb{B}, \text{utree}_{\ell -1}^i)\Big) \in \mathcal{R}_\text{tx-validity},\\ \Big((l2tol1Tree, l1Hash, nonce, gst_\ell);\mathbb{B}_\ell\Big) \in \mathcal{R}_\text{bridge}, 
\Bigg\}
$$

In short the prover proves to L1 the following:

- The global state includes both the latest block hashes and the Merkle roots of unexecuted internal transactions ($\text{uroot}$) per shard.
- The prover correctly updates each $\text{uroot}$ across batches by:
    - adding new transactions from $\text{OutTx}$ of current batch blocks,
    - removing transactions proven to be executed.
- Each transaction executed in the batch is proven to exist in the $\text{uroot}$ of its source shard.
- All EVM executions and blcoks are valid and respect transaction effects.
- Bridge-related state (`l2tol1Root`, `l1Hash`, etc.) matches the storage of `L2BridgeMessenger` at the end of the batch.

## Comparison

Both relations $\mathcal{R}$ defined in proposal 1 and 2 define the conditions under which a batch commitment $gst_\ell$ is valid for verification on L1. The key difference is how internal transaction validity is enforced.

- Proposal 1: via extended $OutTx$ in the block.
- Proposal 2: via a separate $\text{utree}$ representing unexecuted internal txs.

### ⚙️ Structural Comparison

**Proposal 1** 

- $OutTx$ tree is cumulative. Each new block extends the previous one.
- The destination shard can prove inclusion of an internal tx in the source shard’s latest $OutTx$.
- Since the $OutTx$ tree grows cumulatively, no unexecuted tree state tracking is needed as in the Proposal 2.
- No need for $\text{utree}_{\ell}/\text{utree}_{\ell-1}$  but time to time pruning prood needed.
- The relation assumes that EVM execution enforces transaction order via sequence numbers (e.g., tx with `seq = 5` cannot be executed before `seq = 4` is seen).

**Proposal 2**

- Transaction validity proven by checking inclusion in $\text{utree}_{\ell-1}^i$, i.e. the destination shard checks if the tx was unexecuted previously.
- $\text{utree}_{\ell-1}^i$ maintains an evolving Merkle set of unexecuted txs.
- The prover must prove **correct update** of $\text{utree}_{\ell-1}^i$:
    - Added txs come from $OutTx$ of source shard
    - Removed txs are in $InTx$ of the current batch
- The linking relation $\mathcal{R}_\text{link}$ ensures each shard continues from a valid state and $\text{utree}_{\ell}^i$ transition is valid.

### 🟰 Why do They Show the Same Statement

Both relations enforce the same security and correctness conditions for transaction validity:

| **Security Property** | **$\mathcal{R}$ in Proposal 2** | **$\mathcal{R}$ in Proposal 1** |
| --- | --- | --- |
| Prove $tx$ is generated | Show $tx \in OutTx$ in prev. batch via $\text{utree}_{\ell-1}$ | Show $tx \in OutTx$ in source shard’s cumulative Merkle tree |
| Prove tx not executed | Show $tx \in \text{utree}_{\ell-1}$ and not removed and also show that internal transaction in the batch executed only once.  | Enforced via execution order and uniqueness constraint |

In short:

- Proposal 1 shifts tracking responsibility to **the block producer** and enforces ordering via EVM sequence numbers.
- Proposal 2 shifts that responsibility to the **prover**, who must maintain the set of unexecuted txs in a separate tree ($\text{utree}$).

### ⚖️ Advantages/Disadvantages

**Proposal 1 Advantages:**

- No additional prover-side data structures or state required.
- All validity checks rely on existing block data.
- Simplifies tx validity proof by encoding ordering directly in EVM logic.

**Proposal 1 Drawbacks:**

- Requires changes to block structure and EVM logic.
- Cumulative trees may grow large—requires pruning logic and its verification.
- A bit larger block headers due to sequence metadata.

---

**Proposal 2 Advantages:**

- No need to modify block production logic or EVM.
- Clear separation between block data and prover-side logic.
- More flexible and modular which is ideal for retrofitting to existing systems.

**Proposal 2 Drawbacks:**

- Adds complexity to prover’s responsibilities.
- Requires additional Merkle diff proof to show correctness of $\text{uroot}$ updates.
- L1 verification involves more external state (e.g., $\text{uroot}$) beyond block hashes.

| **Feature** | **Proposal 1** | **Proposal 2** |
| --- | --- | --- |
| Block structure changes |  Yes |  No |
| EVM logic changes |  Yes (enforces message ordering) | No |
| Additional prover state | No |  Yes (uroot trees) |