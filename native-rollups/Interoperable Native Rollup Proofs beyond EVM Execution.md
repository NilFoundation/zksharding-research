# Interoperable Native Rollup Proofs beyond EVM Execution

**Authors**: Handan Kilinc Alper ([handan.kilinc@alumni.epfl.ch](mailto:handan.kilinc@alumni.epfl.ch))

### **TL;DR**

This document explores how zkSharding can integrate with Ethereum’s **native rollup** model, where EVM execution is verified using the canonical `EXECUTE` precompile on L1. 

The proving statement is proposed in the assumptions that

- Removing `asyncCall` define in our EVM and introducing EVM compatible `sendMessage` to handle cross-shard communication.
- Existence of a helper function $\text{LinkCST}$ to ensure CSTs are provably linked to a valid EVM execution.

The key insight: our prover **does not reprove EVM execution** but instead verifies **higher-level consistency rules**  which ensuring zkSharding’s correctness and safety are on par with Ethereum itself.

Justin Drake states the following:

> Right now, with today's ZK rollups, the situation is that each rollup implements its own EVM verifier, which can introduce bugs or inconsistencies. These rollups often need some form of governance to keep their verifiers aligned with Ethereum L1, especially when there are protocol upgrades. What we're proposing instead is to move away from that rollup-by-rollup setup, and instead introduce a precompile at the Ethereum base layer — essentially, a native EVM execution engine that validators can use directly. This shifts **the choice of verifier from being something controlled by the rollup developers, to being something selected by Ethereum validators themselves**, similar to how validators choose their EL clients today.
> 

The core idea here is that native rollups delegate EVM execution verification to Ethereum L1 using a shared execution engine (e.g., via the `EXECUTE` precompile). It removes the need for rollups to implement and maintain their own EVM verifiers.

This has two implications:

1. Rollups no longer prove EVM execution directly.
2. Ethereum validators ensure consistency by running the canonical EVM. This removes the need for rollup-specific governance or upgrades in response to protocol changes.

As discussed in previous documents, our zkSharding proving system verifies more than just EVM state transitions. We also need to prove:

- Validity and uniqueness of internal (cross-shard) transactions.
- Proper ordering and state linkage across blocks and batches.
- Correct handling of cross-shard communication (CSTs).

This means our L1 statement includes sub-statements such as the following EVM execution relation:

$$
\mathcal{R}_\text{evm} = \Bigg\{\Big((\text{root}_{k-1}^j, \text{root}_{k}^j);(st{k-1}^j, st_{k}^j, TxList, OutTx)\Big): \\ \textbf{ourEVM}(TxList, st_{k-1}^j, bcontx_{k}^j) \rightarrow st_k^j, OutTx,\\ Root(st_{k-1}^j) = \text{root}_{k-1}^j, Root(st^j_k) = \text{root}_k^j \Bigg\}
$$

However, this relation depends on our customized EVM ($\text{ourEVM}$) that includes support for asynchronous execution via `asyncCall`, which is not part of the canonical EVM. The `EXECUTE` precompile will not support `asyncCall`, and thus cannot verify this relation directly.

### Design Decision: Replace `asyncCall` with `sendMessage`

To make zkSharding compatible with native rollup infrastructure, the proposal here is to eliminate `asyncCall` and replace it with a new EVM compatible: `sendMessage`.

- `sendMessage` allows contracts to send messages (CSTs) to other shards in a structured, deterministic way.
- These CSTs are included in `OutTransactions` of the block and committed as part of the state via Merkle root in `OutTransactionsRoot`.

How `sendMessage`  should be defined is not part of this document.

This change aligns with the canonical EVM’s synchronous model and decouples cross-shard messaging from EVM execution which allows us to use the `EXECUTE` precompile for verifying intra-shard execution.

**However, we still have a challenge i.e., verifying CSTs were actually generated:** Even with `sendMessage`, we must ensure that any CST present in `OutTransactions` was genuinely generated during EVM execution. Otherwise, anyone could inject arbitrary CSTs and claim they were legitimate outputs.

To handle this, we introduce a helper function $\text{LinkCST}$. This function verifies that a given CST was created as part of the EVM transition from state $st_\text{old}$ to $st_\text{new}$:

$$
\text{LinkCST}(st_\text{old}, st_\text{new}, cst, aux) \rightarrow \{0,1\}
$$

If this function returns 1, we consider the CST as valid and linked to the state transition verified by the `EXECUTE` precompile.

**Note:** The internal implementation of `LinkCST` can vary. A vanilla implementation would re-run the EVM on `aux` (e.g., input txs) and check if `cst` is emitted. However, this may be redundant since we already verified the EVM transition. Alternative lightweight implementations may be used that infer CST generation from observable state diffs.

## The Proven Statement

In this section, we describe what the zkSharding prover must show to Ethereum under a native rollup setting, where EVM execution is verified via Ethereum’s `EXECUTE` precompile

### Preliminaries

**Setup Assumptions:**

Each execution shard maintains its own local state, and every state transition is individually verified by L1 through the `EXECUTE` precompile. We assume that:

- The L1 verification contract stores verified state roots using: $\texttt{VerifiedStates}[i][k] = st^i_k$
    
    where:
    
    - $i$ is the execution shard ID,
    - $k$ is the block number within shard $i$,
    - $st^i_k$ is the state root verified by L1.
- A state root becomes **final** only when a batch proof is submitted to L1 that proves:
    - These state transitions respect the correct intra-shard transaction semantics,
    - Any included CSTs are valid and were executed in the right order.

**Block Structure and Batch:**

Each block $B^i$ in the batch includes:

$$
⁍
$$

$Tx^i$ includes **all transactions**, whether they are user-supplied or CSTs. The distinction between $InTx$ and $ExTx$ is dropped.

- We define an indicator function $\text{isCST}(tx)$ to identify cross-shard transactions (i.e., those that were generated by other shards).

Let the batch  $\mathbb{B}_\ell$ consist of blocks from all execution shards. Its global state $gst_\ell$ includes

$h_{n_1}, h_{n_2}, \dots, h_{n_m}, \quad \text{where } h_{n_i} = \mathsf{Hash}(i, \mathbb{B}[i][n_i].\text{bhead})$

💥 The zkSharding prover shows to the L1 verification contract that the batch $\mathbb{B}_\ell$ is correct with respect to all verified states and transaction validity. The full relation is:

$$
\mathcal{R} = \Bigg\{ \Big((gst_{\ell-1},\{stroot_n^i\}_{i = 1}^m, gst_{\ell}, \big\{\{stroot^i_k\}_{k = n+1}^{n + n_i}\big\}_{i = 1}^m;  \mathbb{B}_\ell\Big) \quad \Big| \quad\\  \forall j \in \{1,\ldots,m\},\forall k \in \{1, \ldots, n_j\}, \Big((B^j_{k-1}, B^j_k);\mathbb{B}_\ell\Big) \in \mathcal{R}_\text{block}\\ \forall tx \in B_k^j.Tx, \text{isCST}(tx) \rightarrow 1 , \Big(tx; \mathbb{B}_\ell[i][n_i]\Big) \in \mathcal{R}_\text{tx-validity} \\\text{ where } tx.\text{shard}_s = i\Big)
\Bigg\}
$$

$\mathcal{R}_\text{tx-validity}$ defined previously.

- **Public Inputs Now Include:**
    - The last finalized state roots $\{stroot_n^i\}_{i=1}^m$ verified by `EXECUTE`.
    - All newly verified intermediate states in the batch:$\big\{stroot^i_k\}_{k = n+1}^{n + n_i}\big\}_{i = 1}^m$ by `EXECUTE` .

<aside>
👉🏻
We note that during the verification of batch proof, the L1 verification contract retrieves the verified intermediate states from stored verified state roots i.e., get $\big\{\mathtt{VerifiedStates}[i][k]\}_{k = n+1}^{n + n_i}\big\}_{i = 1}^m$.
</aside>

- **No Need for $\mathcal{R}_\text{link}$:**
    - Since we use L1’s `EXECUTE` as the canonical verifier of each state transition, explicit linkage across batches is unnecessary. Instead, validity is enforced via the global state commitment and verified state entries.
- **$\mathcal{R}_\text{block}$ update:** We replace the previous $\mathcal{R}_{\text{evm}}$ relation with a check that confirms CSTs were generated as a result of the EVM execution (already verified via `EXECUTE`). This is handled by a helper func

$$
\mathcal{R}_{\text{block}} = \Bigg\{ \Big((B^j_{k-1}, B^j_k); \mathbb{B}_\ell\Big) \quad\Big| \\\Big((B^j_{k-1},j); (\mathbb{B}_{k-1},k-1)\Big), \Big((B^j_{k},j); (\mathbb{B}_{k},k)\Big) \in \mathcal{R}_\text{isBlock} \\ \forall cst \in B_k^j.OutTx, \textbf{LinkCST}(st_{k-1}, st_k, cst, Tx) \rightarrow 1,  \\Root(InTx_k^j||ExTx_k^j) = bhead_k^j.\text{InTransactionsRoot} \\  Root(OutTx_k^j) = bhead_k^j.\text{OutTransactionsRoot},\\ Root(st) = bhead_k^j.StRoot =stroot\\ H(bhead_{k-1}^j, j) = h_{k-1}, bhead_{k}^j.\text{PrevBlock} = h_{k-1}\Bigg\}
$$

## Why Our Prover Setup Is Safe at Ethereum Level

To ensure Ethereum accepts only **correct** and **consistent** state transitions from zkSharding, we rely on two foundational principles:

### 1. **Soundness of the proof system**

This means:

- A proof can only be generated if the witness (i.e., the internal data such as CSTs, transaction execution, state roots) satisfies the mathematical relation we defined.
- If a prover tries to submit a forged CST, reuse a transaction, or inject an invalid state — they won’t be able to construct a valid proof (unless they can solve a computationally hard problem)

Soundness is a key property of our safety guarantees because  the relation $\mathcal{R}$ we define encodes all the necessary validity rules such as correct CST origin, proper transaction inclusion, ordered execution, etc. If the prover can produce a valid proof under $\mathcal{R}$, then Ethereum can trust that all of these conditions were satisfied.

### 2. **Canonical Verification Logic at L1**

This addresses a *deeper trust issue* — who verifies the execution?

Here’s the distinction:

- In most zk-rollups today, the rollup team implements their own zk-EVM, and Ethereum just trusts their verifier contract.
- But Ethereum doesn’t audit or maintain that verifier  so if there’s a bug or inconsistency, Ethereum could be fooled.

With native rollups:

- Ethereum itself verifies the execution using the `EXECUTE` precompile.
- `EXECUTE` takes a transaction list and a pre-state and checks that the resulting post-state is valid *according to Ethereum’s own EVM rules*.
- This removes the need to trust custom verifiers since Ethereum directly checks the core logic.

This verification logic is another key property of our safety guarantees at the Ethereum level because our batch proof never tries to “reprove” EVM execution. Instead, we show that all state roots are already **verified via `EXECUTE`**, and we prove only the higher-level consistency logic: transaction validity, CST linkage, block ordering, etc.

<aside>
‼️

This makes zkSharding as safe as Ethereum’s own execution, as long as:

- Our relation $\mathcal{R}$ encodes the right semantics, and
- The CST mechanism (e.g., `sendMessage`, `LinkCST`) is designed so that it only produces verifiable, unambiguous messages linked to EVM state. This is important while designing these functions. We will cover the exact requirements and properties these functions must satisfy (e.g., determinism, uniqueness) in a dedicated document to ensure Ethereum level safety.
</aside>