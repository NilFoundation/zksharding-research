# Handling Cross Shard Transaction Rejections

**Authors**: [Aleksandar Damjanovic](https://t.me/aleksweb3) ([aleksdamjanovicwork@gmail.com](mailto:aleksdamjanovicwork@gmail.com))  

## Motivation

A key challenge arises when a cross-shard transaction (CST)  arrives at a destination shard with a CST.maxFeePerGas set below the shard’s baseFeePerGas. These transactions are effectively invalid (they fail the basic  inclusion condition) and cannot be executed, yet their handling must be carefully designed to prevent inconsistencies, spam  and not covering costs incurred by this rejection. 

A naive approach would be to reject such transaction outright before inclusion. However, this raises problems of verifiability - how do we prove that the message was indeed invalid due to insufficient maxFeePerGas? Without inclusion in a block, there is no on-chain record of the rejection, making possible exploits happen. 

## Issue: Underfunded Cross-Shard Transactions (CSTs)

### Definition

A **Cross-Shard Transaction (CST)** is a transaction that originates on one shard and is intended for execution on another (in majority of cases, however it can be on one shard only across two or multiple blocks) . It carries a `maxFeePerGas` and `maxPriorityFeePerGas`, inherited either from the originating external transaction or from another CST that “spawned” it.

### The Issue

When a CST reaches its destination shard where `baseFee > maxFeePerGas`, it doesn’t fulfill the inclusion/execution condition and cannot be executed. Without a structured mechanism to handle such cases, several risks emerge:

- Spam risk -  Attackers can repeatedly send invalid CSTs, unnecessarily consuming network resources by forcing validation and rejection processing.
- Censorship/Verifiability **-** If CSTs are dropped before block inclusion, how do we prove that it is because of the above mentioned condition and not some censorship attack?
- Proof Costs - how do we cover the costs of proving this rejection? Additional circuit logic may need to be written to handle these cases.

## Requirements

In order to make the solution viable we must satisfy the following:

- On-chain verifiability - The rejection of an underfunded CST must be logged on-chain.
- Spam Prevention - Users should bear a minimal cost when submitting an invalid CST to deter spam
- Minimal Overhead - The solution must not introduce excessive computational or storage overhead.
- *Validator Incentives??? should the mechanism  ensure validators are compensated for processing invalid CSTs*

## Potential Solution Minimal Inclusion and Revert

In this approach, **all CSTs** (even those that dont fulfill the base fee condition ) are included in the block. The network/consensus  checks `baseFee <= maxFeePerGas` and  if this condition fails, the CST fails with a logged reason (e.g., `UnderfundedCST`).  

Steps:

1. The block builder includes the CST in the block
2. During execution the protocol checks if the `shardBaseFee <= CST.maxFeePerGas`
    1. if false, fail the CST
    2. if true execute as normal
3. If false:
    1. A minimal amount of gas is consumed for reading the CST, checking validity and producing a failure/revert log, and proving it.
    2. Any leftover Fee Credit  is returned to the user
4. The revert reeason `UnderfundedCST` is stored in the receipt. 
5. Outcome:
    1. We have a verifiable record that the CST was invalid
    2. the user pays a samall fee for attempting an underfunded transaction

Benefits:

- Straitghforward
- Verifiable
- Minimal Extra logic

Drawbacks

- These receipts still consume some space, and proving resources, allthough we could  probably omit some parts of it? It will not execute anyways.
- We need to charge some fees for this revert reason and consumption of block space.
- Do we need to think about validator incentives for this?

However, how do we ensure that the transaction can pay for its rejection receipt? You can imagine a simple case where the transactions  Fee Credit is not sufficient to pay for its rejection reason. 

### Minimum Fee Credit Enforcement

The above mentioned case can be solved by minimum overhead check during CST creation and feeCredit forwarding:  https://docs.nil.foundation/nil/smart-contracts/gas-forwarding

**Does the newly created CST have enough feeCredit allocated to at least pay for a worst-case (invalid) scenario?** 

If not, the parent transaction reverts, or doesn’t spawn the “child” CST. This way, each newly created CST is **guaranteed** to have enough leftover credit for either successful execution **or** for covering the cost of rejection.

**Why enforce this?** 

- Prevents scenarios where the user transaction/CST has insufficient Fee Credit left, yet tries to spawn a new CST that cannot pay the ‘rejection fee’ because it fails the `baseFee <= maxFeePerGas` condition.
- Simplifies the protocol: if the CST does exist, it can definitely pay for its own revert or proof-of-invalidity (I need a better name for this) .
- prevents  spam (or at least tries to) : malicious users cannot continually spawn underfunded messages with near-zero Fee Credit.

### Integrating with the L2 Fee Model

1. **User FeeCredit -**  On L2, a user bids with  a certain fee credit  from his balance to cover all transactions stemming from the initial one. 
2. **Propagation -** Each subsequent CST  or async transaction uses the same fee parameters (`maxFeePerGas`, `maxPriorityFeePerGas`), ensuring a consistent bid across shards.
3. Minimum Overhead Reservation **-**  When a CST spawns another CST, the parent must reserve (forward)  enough credit to pay for the child’s potential rejection.
4. Execution and  Rejection - If the CST is valid (i.e., `baseFee <= maxFeePerGas`), it executes. Otherwise, the CST fails and consumes the Fee Credit for failure. 

## Conclusion and next steps

This proposal introduces a structured approach to handling underfunded CSTs while ensuring on-chain verifiability, preventing spam, and maintaining system efficiency. By enforcing **minimum Fee Credit requirement**, we guarantee that every CST either executes successfully or has enough funding to pay for its own rejection. 

Next steps: 

- Choose gas consumption for this `UnderfundedCST` CST bounce  and minimum Fee Credit required to cover it. I think it will be minBaseFee (or the provingCosts portion of it)  * gas consumed for this bounce.
- Enforce this Fee Credit condition when creating CSTs (how to fail the creation, or revert the parent???)
- Assess the possible costs of attack - Should additional multipliers be introduced to scale the minimum Fee Credit and deter spam? What effect would this have on legitimate use cases?