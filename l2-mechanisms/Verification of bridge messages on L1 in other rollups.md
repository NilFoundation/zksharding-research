# Verification of bridge messages on L1 in other rollups

**Authors**: Handan Kilinc Alper ([handan.kilinc@alumni.epfl.ch](mailto:handan.kilinc@alumni.epfl.ch))  

**_Motivation_**: This document provides a detailed comparison of the approaches used by StarkNet, Scroll, zkSync Era, and Optimism to verify L1-to-L2 and L2-to-L1 messages on L1.

The security of the L1-to-L2 and L2-to-L1 communication channels in rollups (and other rollup designs) is fundamentally based on the assumption that **L1 verifies the execution of L1 messages on L2 and the execution of L2 messages on L1.** This document focuses on how this verification is implemented on L1 for prominent rollups such as StarkNet, Scroll, zkSync Era, and Optimism.

### Key Differences in Rollup Approaches

Among these rollups, StarkNet stands out because the input to its verification contract on L1 includes both L1 and L2 messages**.** In contrast, Scroll, zkSync Era, and Optimism use a similar approach to verify L2 messages: by maintaining a separate Merkle tree for successfully executed L2 messages. However, the way these rollups validate the Merkle tree root differs:

1. **zkSync Era and Scroll**:
    - The verification contract on L1 checks the Merkle root of the L2 message tree after batch submission using a SNARK proof. This proof ensures the root is valid and that L2 messages were processed correctly.
    - Users consume L2 messages by providing a Merkle tree proof of the specific message's inclusion in the tree.
2. **Optimism**:
    - The L2-to-L1 Merkle root becomes valid on L1 only after the challenge period**.**
    - Users similarly provide an MT proof to consume their L2 messages.

### Handling L1-to-L2 Messages: StarkNet vs. Scroll and zkSync Era

**Optimism** does not verify the execution of L1 messages on L2, relying instead on its fraud-proof mechanism. In contrast, **zkSync Era** and **Scroll** include L1 message execution as part of the batch proof. While their solutions are similar, their implementations have subtle differences:

### **Scroll**: Explicit L1 Message Queue with Bitmap Tracking

- Scroll uses a **dedicated L1 Message Queue**, which explicitly tracks processed and unprocessed L1 messages.
- A **bitmap** is used to indicate skipped L1 messages during batch processing, ensuring a clear separation between consumed and pending messages.
- **Advantages**:
    - Ensures robust tracking of L1 message states.
    - Simplifies recovery and cancellation mechanisms for skipped messages.
- **Disadvantages**:
    - Adds complexity to the L1 contract for managing the message queue and bitmap.
    - Handling skipped messages may introduce overhead in L1 verification.

### **zkSync Era**: Priority Queue Mechanism

- zkSync Era uses a **priority queue** to manage L1-to-L2 messages, ensuring all messages in the queue will eventually be processed.
- Unlike Scroll, zkSync Era does not require a bitmap for skipped messages, as it provides a way for a user to claim unsuccessful execution on L2.
- **Advantages**:
    - Simpler mechanism without the need for skipped message tracking.
    - More straightforward to maintain on-chain state.
- **Disadvantages**:
    - Keeping larger MT since MT should hold both deposit and withdrawal messages.
    - Extra work for users.
    - All L1 messages are considered as enforced L1 messages.

The details of each rollup’s verification mechanism for the bridges are below:

---

# 1. StarkNet

The sequencer submits a state transition proof and associated data to the [StarkNet Core Contract](https://etherscan.io/address/0x47103A9b801eB6a63555897d399e4b7c1c8Eb5bC#code#F1#L1) on L1. This submission proves the state update's correctness on StarkNet L2 and ensures the validity of any processed messages (L1→L2 or L2→L1).

The associated data (called `programOutput`) includes: new state root, [**Message Segments**](https://etherscan.io/address/0x47103A9b801eB6a63555897d399e4b7c1c8Eb5bC#code#F1#L309): Information about messages processed during the block:

- **L1-to-L2 Messages**: Messages sent from L1 contracts that were consumed on L2.
- **L2-to-L1 Messages**: Messages sent from L2 contracts to be delivered to L1.

and some metadata.

## 1.1. Verification of Bridge Messages

1. StarkNet Core contact verifies the proof with the input `programOutput`.
2. It [processes message segments](https://etherscan.io/address/0x47103A9b801eB6a63555897d399e4b7c1c8Eb5bC#code#F1#L309) in the `programOutput` 

```solidity
// Process L2 -> L1 messages.
uint256 outputOffset = StarknetOutput.messageSegmentOffset(programOutput);
outputOffset += StarknetOutput.processMessages(
	 // isL2ToL1=
   true,
   programOutput[outputOffset:],
   l2ToL1Messages()
   );

// Process L1 -> L2 messages.
outputOffset += StarknetOutput.processMessages(
    // isL2ToL1=
    false,
    programOutput[outputOffset:],
    l1ToL2Messages()
    );
```

1. [`processMessages`](https://etherscan.io/address/0x47103A9b801eB6a63555897d399e4b7c1c8Eb5bC#code#F16#L117) function:
    - Handles both L2-to-L1 and L1-to-L2 messages.
    - Ensures message integrity via hashing and counter verification.
    - Updates message mappings (`l1ToL2Messages`, `l2ToL1Messages`) to track their lifecycle.
    - Emits events for transparency and off-chain monitoring.

## 1.2. Cancelation of L1→L2 Messages

StarkNet allows users to cancel unconsumed L1-to-L2 messages using a two-step process:

1. **Start Cancellation**: Users call [`startL1ToL2MessageCancellation()`](https://etherscan.io/address/0x47103A9b801eB6a63555897d399e4b7c1c8Eb5bC#code#F19#L162) on the StarkNet Core. This marks the message for cancellation and records the cancellation request timestamp.
2. **Finalize Cancellation**: *After a mandatory delay* (`messageCancellationDelay`), users call [`cancelL1ToL2Message()`](https://etherscan.io/address/0x47103A9b801eB6a63555897d399e4b7c1c8Eb5bC#code#F19#L176) to reclaim the funds.

**Drawbacks:** L2 needs to store L1 messages while in Scroll this is eliminated. Scroll does not submit L1 messages instead submit which ones are executed.

**Difference:**

# 2. Scroll

In the [**ScrollChain**](https://github.com/scroll-tech/scroll-contracts/blob/main/src/L1/rollup/ScrollChain.sol) contract, the sequencer's main role is to submit batches of L2 state transitions, and transactions, and process L1 messages to the L1 contract for validation and commitment. The input of the sequencer to the ScrollChain contract verifies the new state with the following function:

```solidity
finalizeBatchWithProof4844(
    bytes calldata _batchHeader,
    bytes32 _postStateRoot,
    bytes32 _withdrawRoot,
    bytes calldata _blobDataProof,
    bytes calldata _aggrProof
);
```

Here we focus on L1 messages and L2 withdrawal messages. The related inputs are `_batchHeader` and `_withdrawRoot`.

- `_batchHeader` contains related bridge messages
    - Chunks: Contains encoded data of L2 transactions and processed L1-to-L2 messages in the batch. Each chunk has:
        - L2 transaction hashes
        - **L1 message hashes** (non-skipped ones, fetched from `IL1MessageQueue`)
    - Skipped L1 Message Bitmap: A bitmap indicating which **L1 messages** were skipped or processed in the batch.
- `_withdrawRoot` is the Merkle tree root of the [withdrawal tree.](https://docs.scroll.io/en/technology/bridge/cross-domain-messaging/#withdraw-trie)

## 2.1. Verification of Bridge Messages

### **Verification of L1-to-L2 Messages**

L1-to-L2 messages originate on L1 and are consumed on L2 during batch processing. These messages must be properly verified on L1 to ensure they are acknowledged and executed by L2.

**The process starting from L1:**

1. L1 messages (typically deposit messages) are added to the `L1MessageQueue` contract on L1.
    - Each message is indexed using a `queueIndex` and can be accessed by the sequencer when committing batches on L2.
2. The sequencer consumes messages from the L1 queue and includes their hashes in the L2 batch. A bitmap (`_skippedL1MessageBitmap`) is used to indicate which messages were processed (0) or skipped (1) in the batch.
3. The batch is committed to the L1 Scroll contract via [`commitBatchWithBlobProof`](https://github.com/scroll-tech/scroll-contracts/blob/2d06562f3f252ddcd3dcda8d4bdec368ffbcc21d/src/L1/rollup/ScrollChain.sol#L330) or [`commitBatch`](https://github.com/scroll-tech/scroll-contracts/blob/2d06562f3f252ddcd3dcda8d4bdec368ffbcc21d/src/L1/rollup/ScrollChain.sol#L265).
4. During batch finalization (`finalizeBatchWithProof4844`), the proof submitted by the sequencer ensures:
    - The state transitions on L2 included all **indicated** L1-to-L2 messages.
    - The total count and hashes of processed messages match those in the batch header.
    - The skipped message bitmap accurately reflects processed/skipped L1 messages.
5. The `L1MessageQueue` contract is updated with the finalized count of processed messages. This ensures all processed messages are permanently marked as consumed.

### **Verification of L2-to-L1 Messages**

L2-to-L1 messages originate on L2 and are sent to L1 for further processing or interaction with L1 contracts. These messages must be validated to ensure they were included in the finalized L2 state.

**The process starting from L2:**

1. When an L2 contract wants to send a message to L1, it logs the message as part of the L2 state.
2. The emitted messages are hashed and included in the `withdrawRoot` or other Merkle tree structures representing the finalized L2 state.
3. During batch finalization (`finalizeBatchWithProof4844`), the proof ensures:
    - The `withdrawRoot` accurately reflects the L2-to-L1 messages in the finalized L2 state.
    - Each message was correctly processed on L2 and is ready for L1 delivery.
4. L2-to-L1 messages are retrieved from the `withdrawRoot` and processed by the respective L1 contract. The scroll bridge contract on L1 ensures:
    - The `withdrawRoot` is verified as part of the proof.
    - Messages derived from the root are authentic via [MT proof](https://github.com/scroll-tech/scroll-contracts/blob/2d06562f3f252ddcd3dcda8d4bdec368ffbcc21d/src/L1/L1ScrollMessenger.sol#L164) and match the expected structure.
5. The Scroll bridge contract emits events to indicate that L2-to-L1 messages have been received and processed.

## 2.2. Cancelation of L1→L2 Messages

Users can call [dropMessage](https://github.com/scroll-tech/scroll-contracts/blob/2d06562f3f252ddcd3dcda8d4bdec368ffbcc21d/src/L1/L1ScrollMessenger.sol#L269) to cancel an L1 message. For this, the user must identify the failed message. A message can fail due to:

- Skipping on L2 (e.g., sequencer capacity issues).
- Finalization without execution on L2.

# 3. zkSync Era

According to zkSync Era contract [Executor.sol](https://github.com/code-423n4/2023-10-zksync/blob/main/code/contracts/ethereum/contracts/zksync/facets/Executor.sol), the batch data includes the following:

```solidity
StoredBatchInfo(
_newBatch.batchNumber,
_newBatch.newStateRoot,
_newBatch.indexRepeatedStorageChanges,
_newBatch.numberOfLayer1Txs,
_newBatch.priorityOperationsHash,
l2LogsTreeRoot,
_newBatch.timestamp,
commitment
);
```

For verifying the integrity of L1-to-L2 and L2-to-L1 messages, zkSync uses inputs such as:

- **Priority operations**: Representing L1-to-L2 messages processed on zkSync. `priorityOperationsHash` in the batch is the hash of the processed priority operations.
- **Merkle roots**: Representing L2-to-L1 messages and withdrawal operations. `l2LogsTreeRoot` in the batch is the Merkle root of L2-to-L1 and L1-to-L2 messages that L1 process in one batch.

## 3.1. Verification of Bridge Messages

### **Verification of L1-to-L2 Messages**

L1-to-L2 messages (e.g., deposits or other transactions) originate on L1 and are consumed by zkSync's L2 during batch processing.

1. L1-to-L2 messages are created by calling the zkSync [`Mailbox`](https://github.com/code-423n4/2023-10-zksync/blob/main/code/contracts/ethereum/contracts/zksync/facets/Mailbox.sol) contract or bridge contracts such as [L1ERC20Bridge](https://github.com/code-423n4/2023-10-zksync/blob/main/code/contracts/ethereum/contracts/bridge/L1ERC20Bridge.sol) and [L1WethBridge.sol](https://github.com/code-423n4/2023-10-zksync/blob/main/code/contracts/ethereum/contracts/bridge/L1WethBridge.sol) . These messages are stored in the **priority queue** on L1 with a unique index and a canonical transaction hash (`canonicalTxHash`). 
    
    Before being added to the priority queue, the contract will perform several checks for the transaction, making sure that it is processable and provides enough fee to compensate the operator for this transaction. Then, this transaction will be [**appended**](https://github.com/code-423n4/2023-10-zksync/blob/26547a90d569559ef5c80ff8c523010134e84a71/code/contracts/ethereum/contracts/zksync/facets/Mailbox.sol#L369) to the priority queue.
    
2. The zkSync operator (sequencer) monitors the priority queue and processes messages in order. Messages are hashed and included in the zkSync batch commitment (`priorityOperationsHash`).
3. After processing on L2, the sequencer submits a zkSNARK proof to L1 that verifies:
    - The priority messages were processed by the sequencer (some might fail).
    - The state transitions on L2 incorporated the messages accurately.
    
    This is handled via the [`commitBatches`](https://github.com/code-423n4/2023-10-zksync/blob/26547a90d569559ef5c80ff8c523010134e84a71/code/contracts/ethereum/contracts/zksync/facets/Executor.sol#L177) and [`proveBatches`](https://github.com/code-423n4/2023-10-zksync/blob/26547a90d569559ef5c80ff8c523010134e84a71/code/contracts/ethereum/contracts/zksync/facets/Executor.sol#L311) functions in zkSync’s **Executor** contract.
    
4. During batch execution on L1, the following checks are performed:
    - The number of processed messages (`numberOfLayer1Txs`) matches the metadata.
    - The processing priority operations' rolling hash (`priorityOperationsHash`) is consistent with the priority queue.
    - The proof validates the correctness of state transitions.
5. L1 marks the processed priority messages as consumed by advancing the priority queue.

### **Verification of L2-to-L1 Messages**

L2-to-L1 messages (e.g., withdrawals or cross-domain calls) originate on L2 and must be verified and executed on L1.

1. L2-to-L1 messages are generated during L2 contract execution and stored in the **L2-to-L1 logs tree** with the root ****`l2LogsTreeRoot`. These messages include withdrawal requests, contract calls, or other data sent to L1.
2. L2-to-L1 messages are hashed and added to a Merkle tree. The **Merkle root** of this tree is included in the batch metadata when the zkSync operator finalizes a batch.
3. The zkSync operator submits a batch containing `l2LogsTreeRoot` and L1 verifies the batch.
4. Validated L2-to-L1 messages are passed to the appropriate L1 contract (e.g., for withdrawals). The **MailboxFacet** or **L1 bridge contracts** on L1 ensure the Merkle proof for each message is correct before execution.

## 3.2. Cancelation of L1→L2 Messages

- When submitting a transaction to the priority queue, the system sets an expiration timestamp (`PRIORITY_EXPIRATION`).
- If the transaction is not processed by the sequencer before this expiration, it is marked as expired.

Batch proof only proves that L1-to-L2 messages are processed. It does not claim they were successful. In case some messages fail, users can claim the failure by providing  the Merkle tree path of their failed transaction belonging to L2 log tree (https://github.com/code-423n4/2023-10-zksync/blob/26547a90d569559ef5c80ff8c523010134e84a71/code/contracts/ethereum/contracts/bridge/L1ERC20Bridge.sol#L255). The verification is same as verifying L2-to-L1 messages.

## 4.1. Verification of Bridge Messages

### **Verification of L1-to-L2 Messages**

The **L2CrossDomainMessenger** processes the deposit by updating the user's account balance, and making the assets available for use on L2. L1 does not verify it as in other zk-rollups mentioned here because if a false L1 message is executed in L2, it is told to the OptimisimPortal within the challenge period.

### **Verification of L2-to-L1 Messages**

L2-to-L1 messages in Optimism involve withdrawals or cross-domain calls initiated on L2 and finalized on L1.

1. Withdrawals are initiated on L2 via a call to the **L2ToL1MessagePasser** contract, which records the message's properties in its storage.
2. The **Batch Submitter** aggregates L2 transactions and submits them to L1 together with the Merkle root of the L2 messages.
3. After the challenge period, the withdrawal root is finalized on L1 via the OptimismPortal, which verifies that the fault challenge period has passed since the withdrawal message was proved.

## 4.2. Cancelation of L1→L2 Messages

It seems that they don’t have any process to cancel L1→L2 messages.