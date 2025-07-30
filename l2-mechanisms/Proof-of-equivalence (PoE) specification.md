# Proof-of-equivalence (PoE) specification

**Authors**: [Ilya Marozau](https://www.linkedin.com/in/imarozau)  

**Motivation**: 

EIP-4844 introduces a new type of transaction that includes "blobs" of data. To ensure that this data is available and correctly represented, the network must be able to verify its availability without needing to download the entire data. KZG commitments allow for this by providing a cryptographic proof that the data committed to by the proposer is indeed available. Validators can sample random portions of the data and check against the KZG commitment, making it possible to verify data availability efficiently without needing to download the full data.

Along with this commitment, the equivalence of data in batches must also be proven. Equivalence refers to the question: "How can we be sure that the data in blobs contains all the necessary information?" Although simple in definition, this question requires a complex answer. This is where Proof of Equivalence (PoE) comes into play.

It must be said that PoE is not only specific to Ethereum but almost any DA layer solution.  

**Scope**:

This document covers the following questions:

- Equivalence verification for main shard batch.
- PoE schema for execution shard batches.

**Terminology**:

- Batch — processed L2 transactions composed and compressed together. We say 1 batch for 1 shard.
- BLOB — binary large object. Refer to the batch that is already saved in blob storage.
- Block — validated set of transactions.
- Bundle — grouped raw non-validated transactions.

**EIP-4844 blob verification**

Blob data is inaccessible at the execution level, complicating the verification of submissions. This is where KZG commitments can be used.

Data, such as raw transactions, sent as blob transactions is first specifically sequenced. Let's call this sequenced data D. Sequencing occurs in "slots" of 32 bytes.

First, the data must be transformed in polynomial P of degree < |D|: 
$\exist g:D \rarr P$

$P(i)=D_i,i<|D|$

In fact, g, in most cases, can be represented by Lagrange interpolation.

L1 blob storage will receive D as input and calculate the commitment based on a predefined secret s and the secp256r1 curve.

$C=P(s) \times G_1$

Thus, the only accessible data from the blob is the keccak256 hash of the commitment, which can be obtained using a special opcode that returns:

 $o = version + BE(keccak256(C))[1:31]$

The sync committee acts as both the prover and data provider.

Suppose we want to prove that specific data is included in the commitment and validate the batch commit itself to L1. It should be noted that blob data is serialized and indexed (vector v). The data is kept in the vector as below:

$\vec{v^k} =  \left[ \begin{array}{cc}
(z_0, y_0) \\ ... \\(z_{k-1}, y_{k-1})\end{array} \right], y_i=P(z_i)$

$D = \tau(z|P(z_i))$ 

Tau is the data transformation function.

That is the part of the proof input that is used by the updated contract:

```solidity
/// input[0:32] -- vector size (sz)
/// input[32: 32 * sz] -- (input[i] -- z, input[i + 1] -- y)
/// input[32*sz: 32*(sz+1)] -- proof
/// input[32*(sz+1): 32*(sz+2)] -- commitment
function updateState(bytes calldata input) {}
```

To prove all data in a single proof, the simplest method is to construct a polynomial of order k−1 that passes through all the points. This can be achieved similarly to data transformation using Lagrange interpolation:

$R(x) = \sum_{i}^{k-1}y_i \prod_{j}^{k-1}\frac{x-z_j}{z_i-z_j}$

Note that $z_0,...,z_k$ are roots of P(x)-R(x) (polynomial remainder theorem).

That means that a single quotient that is used for proof can be calculated as:

$Q(x) = \frac{P(x)-R(x)}{\prod_{i}^{k-1}(x-z_i)}$

hence for the KZG proof, we have:

$C=P(s)G_1$

$\pi = Q(s)G_1$

Prover will send the following data:
$[\vec{v^k}, \pi, C]$,

where every $z_i$ is the portion (section) of data from the blob. For example this way we can send state roots or aggregated signatures of validators from different shards. 

**Proof of equivalence problem**

**Architecture Interpretation:**

- "How can we prove to the L1 chain contract that the blob data is the same as what we have proven?"

**Mathematical Interpretation:**

- "How can we ensure the committed polynomial is interpolated through the same points as in the validity proof?"

**Main shard PoE**:

The placeholder proof system is PLONK-based with KZG commitments from public inputs. This means that for the PoE (Proof of Equivalence) of the main shard, it is sufficient to prove that two commitments point to the same data.

Let's say we have commitment C from the sync committee and H from the prover.

To prove they point to the same data, we perform an evaluation at the same random point:

To choose the random point, we can use the Fiat-Shamir heuristic and open both commitments at y = h(C, H) where h is keccak256.

The first idea (requires **verification and elaboration**). The Schwartz–Zippel Lemma states that polynomials are equal with high probability if their values are equal at a randomly selected point. Then we can

1. Get a KZG commitment $C_{\mathrm{KZG}}$ to public input $PI(X)$
2. (inside circuit) Get a Hash commitment $C_{\mathrm{Hash}}$ to the same public input $PI(X)$
3. (inside circuit) Sample evaluation point $\zeta$ pseudo-randomly
4. (inside circuit) Get the evaluation $PI(\zeta) = z$ (can we do this while maintaining soundness guarantees? - additional research is needed)
5. Set new public input for verification $\left( C_{\mathrm{Hash}}, \zeta, z \right)$
6. Get the proof that $C_{\mathrm{KZG}}$ corresponds to a **univariate** polynomial, which is equal to $z$ at point $\zeta$

For further clarification, at a minimum we need to know:

- What type of KZG is used in EIP-4844 ****and is it possible to obtain a proof for step 6?
- What size of public input will be acceptable for us, because we will have to strictly limit its size in the circuit.

If the target snark uses a different modulus or is not polynomial-based (e.g., it uses R1CS), a more complex method is needed to prove equivalence. In our case it can work like that:

1. Let P(x) represent the polynomial encoded by the blob. Create a commitment Q in the zk-snark scheme, which encodes the values $v_1,...,v_n$  where $v_i = P(\omega^i)$.
2. Choose x by hashing the commitments of P and Q.
3. Prove P(x) using the point evaluation precompile.
4. Apply the barycentric interpolation formula  $P(x) = \frac{x^{N-1}}{N} \times \sum_{i} \frac{v_i \cdot \omega^i}{x - \omega^i}$ to perform the same evaluation within the zkp. Ensure that the result matches the value proven in step (3).

Step (4) requires mismatched-field arithmetic, but PLOOKUP techniques can handle this with fewer constraints than an arithmetic-friendly hash function. Note that $\frac{x^{N-1}}{N}, \ \omega^i$ can be precomputed to simplify the process.

**Execution shards PoE**:

![Untitled](/figures/l2-mechanisms/poe-specification-00.png)

In the first version of the cluster, we can use transposition schema. Execution n

**M(V)PT proof**

For execution shards it’s important to keep states on L1 and update them with MPT proof from the prover:

- The prover sends validity proof to the verifier. The verifier evaluates proof and updates the state of the main shard on-chain;
- Prover sends MPT proof from the main shard for the execution shard it wants to proof. The size of the proof for m shards is $O(m*p*log_p(N))$. In practice, the size of the proof is
 $\theta(m+p*O(log_p(N)))$, where p is the alphabet of the node extension, and N global data size for execution shard.
- If proof is correct L1 contract updates execution state roots

The primary reason to do this is to “big proofs”. In case we store only main shard proof, the size of MPT will be $O(m*p*log_p(N) + p*log_p(M))$, where M is the size of the main shard trie.

MPT proofs for execution shards:

- Storage: 32 * k
- Initial proof size: 32 * k + 32 * 512 (logp=~2)
- Proof by request: 32 * 512 * e (requests)

Total gas for 10 shards and e=1000 (e.g. bridges): 

20000 * 10 + 267264 * 16 *+ 16384000 **  16 = **266620224 gas**

no MPT proofs for execution shards:

- Storage: 0
- Initial proof size: 0
- Proof by request: 32 * (512 + 512 (main shard state root)) * e (requests):

Total gas for 10 shards and e=1000 (e.g. bridges): 0 + 0 *+* 1024 * *32 ** 1000*16 = **524288000 gas**

Delta: 524288000/266620224 = ~1.97

Thus, it's almost twice as cost-effective to maintain such a setup. If e is smaller, the system will become more expensive. For instance, the Orbiter Finance protocol verifies the on-chain root approximately 100 times per L2 block.