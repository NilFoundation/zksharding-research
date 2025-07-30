# zkSharding Sync Committee and L1 DA [DRAFT]

**Authors**: [Raluca Diugan](https://x.com/ctzn101)  

# 1. Synchronization Layer

In zkSharding, a distributed ledger is maintained at the level of each shard. The main shard blocks contain, among others, references to *m* execution shard blocks. To settle the new L2 state on L1, block execution needs to be proven and data made available to Ethereum. For efficiency, shards do not prove their new state nor submit their data individually to L1. Instead, a synchronization layer consisting of the *Sync Committee Nodes* and a *Prover Network* handles these processes for all shards at once. These infrastructural components aggregate data from all shards, prove the correct execution of it, and relay the data, the new state root, and the state validity proof to L1.

## 1.1. Architecture

### 1.1.1. Sync Committee

The Sync Committee (SC) represents the core infrastructural component at the intersection of shards, the prover network, and L1. It consists of several sub-components, each handling designated workflows and interacting with each other to produce the outputs required to update the zkSharding state on L1. 

<aside>

**Mainnet and Decentralization**

At time of mainnet launch, the Sync Committee will consist of a single *trusted* node operated by =nil;. Gradually, the Sync Committee will be decentralized (as the name suggests) into a network of SC nodes. These nodes will either handle operations in a leader-based fashion or distribute and coordinate tasks among each other. 

</aside>

**1.1.1.1. Observer**

The Observer is a SC actor tasked with tracking the growth of all shards in order to estimate when a batch can be sealed. It communicates to the Aggregator the main shard blocks interval $\mathcal{I} := \{m_k, \dots, m_l\}$ to be placed in the next batch.

**1.1.1.2. Aggregator**

The Aggregator is a SC actor tasked with downloading data from the cluster, encoding it for the DA layer, and communicating to the Proof Requester the set of blocks part of the new batch to be proven. It collects the block data from all main shard blocks within $\mathcal{I}$, the block data from all corresponding execution shard blocks, prepares the next DA batch $\mathcal{B}$, compresses it, and compiles blobs holding the compressed data. For each blob, it additionally derives the *proof-of-equivalence* inputs. 

**1.1.1.3. Proof Requester** 

The Proof Requester issues DFRI proof requests to the Prover Network. Upon receiving proofs back from a Proof-provider, it verifies their validity. To this end, the Proof Requester issues a new task of type `verify` to the Prover Network. Any prover can pick up the task and produce a result. *Please note that this design is temporary and assumes that the Prover Network is trusted, which will be the case for mainnet launch. In the future, verification should be handled by the Proposer, prior to submitting a state update transaction to L1.* 

<aside>

**Mainnet and Decentralization**

Initially, =nil; will operate all (possibly only one) Proof-providers. As such, we do not need to reward proof generation at this stage. In future upgrades, proof requests will need to be compensated and a robust proof allocation and reward mechanism implemented.

</aside>

**1.1.1.4. Proposer**

The Proposer is a SC actor tasked with assemblying the two types of transactions to L1: the data availability transaction and the state update transaction. It receives blob data from the Aggregator and proofs from the Proof Requester. It retrieves the latest L1 gas prices to determine how much to bid for each transaction. It compiles the two transactions and submits them to L1.

This process ends the zkSharding synchronization protocol.

### 1.1.2. Prover Network

The Prover Network handles `DFRI` proof requests from the Sync Committee’s Verifier. The network consists of Proof-providers. 

<aside>

**Mainnet and Decentralization**

At launch, the prover network will be entirely operated by =nil;. In the future, we will focus on designing the communication protocols between the Sync Committee and Proof-providers, as well as coordination mechanisms between Proof-providers in the case of distributed proof generation.

</aside>

Each Proof-provider consists of multiple workers (provers) which can handle one, several, or all types of proving tasks (one or more DFRI steps, PLONK proof generation). The Proof-provider receives tasks for one batch (defined by the interval of main shard blocks $\mathcal{I}$) from the Sync Committee and generates DFRI tasks according to the dependency map. Provers listen to and retrieve these tasks, and execute them by running the proof-producer. Once a proof is complete, it is sent to the Verifier for verification. 

**Notes on hardware requirements.** The Sync Committee is supposed to be a relatively light node (when compared to the Proof-provider or its prover components). Its storage and computation requirements should be maintained low in order to facilitate decentralization later on. One of the most computationally intensive SC task foreseen for the future is proof verification prior to proposing a new state to L1 when proofs are generated by untrusted Proof-providers. Similarly, storage requirements should be kept low, with most operations being stateless and with data archival services being offered through separate services (even if operated by the same entity).

**Alternatives to Sync Committee for Data Aggregation.** One of the main roles of SC is to aggregate data from across all shards and publish it to L1 in a cost-efficient manner for the entire L2 network. Without considering the proving benefits of aggregating work from across shards, several competing solutions to SC have been proposed. In the future, we hope to compare these in more detail to provide a clear picture for the DA-related benefits of using a Sync Committee in sharded or decentralized L2 settings. Such proposals include:

- **Blob Aggregation.** Proposals such as https://ethresear.ch/t/blob-aggregation-step-towards-more-efficient-blobs/21624 aim to provide rollups underutilizing the space of a single blob with a way to aggregate their individual data with that of other rollups for more efficient posting. This has the double effect of making DA more cost-efficient for the rollup using the shared blobspace and freeing up otherwise idle blobspace for the rest of the ecosystem.
- **Flexible Blob Sizes.** Another approach is for Ethereum to support blobs of varying sizes such that rollups posting less (or more) data than the current 128KB limit to pack and pay proportionally to the amount of bytes used.

## 1.2. Processes Description

### 1.2.1. Data Aggregation

**1.2.1.1. [UNDER DISCUSSION] Batch Sealing Criteria** 

The Observer monitors the evolution of shards to determine when a batch should be formed. The `batch sealing algorithm` ran by the Observer takes into account the following criteria, in order of priority:

1. **Prover Capacity.**
    
    The prover capacity is defined as an aggregate of all individual circuit limits (see the table below), such as the number of read-write operations. The batch must meet all these limits at once. 
    
    - *expressed as percentage by the Observer? (reasoning in case data limit kept)*
    
    | **TYPE** | **MAX_VALUE** | **MIN_VALUE** |
    | --- | --- | --- |
    | max_copy | 30,000 |  |
    | max_rw_size | 60,000 |  |
    | max_keccak_blocks | 500 |  |
    | max_bytecode_size | 20,000 |  |
    | max_mpt_size | 30 |  |
    | max_zkevm_rows | 25,000 |  |
    | max_exp_rows | 25,000 |  |
    | max_rows | 500,000 |  |
2. **DA Capacity.** For zkSharding, we post data to L1 as blobs. Each Ethereum block is limited to processing a small number of blobs (maximum 9 by Pectra). In practice, we can send multiple blob-carrying transactions, ignoring this limit, or choose to restrict each batch by the maximum blobspace per block. The Observer estimates the blobspace used based on the data available in the in-progress batch. It creates a temporary structure holding transaction and block data for DA, compresses it, and estimates the blob usage. As long as the limit is not reached, more blocks can be added. 
    - **[UNDER REVIEW] DA commits and Blob scheduling: Limit and Distribution**
        
        For design simplicity, we can limit our batch to fit over a single DA transaction, i.e. a maximum of 9 blobs’ worth of bytes, and increase this number as the blob count increases with future Ethereum forks or market dynamics if blob targets are adjusted by stakers. This approach helps with L1 transaction robustness (See 1.3. Security Policies) since we only have one DA transaction to retry in case of failure, compared to a more complex retry scenario with multiple DA transactions.
        
        Alternatively, we can introduce more flexibility in the number of DA transactions, *essentially removing the DA limit for one batch*. If one batch can be posted over as many DA transactions as demanded, we gain flexibility across at least three dimensions:
        
        - `BATCH_DATA_SIZE`**:** the size of the batch after compression can be as large as needed.
        - **Full potential of prover capacity:** this approach prevents scenarios where the prover capacity goes underutilized because of data restrictions.
        - **Flexible L1 posting configuration/schedule**: because we are not limited to fit all blobs into a single L1 transaction, we can distribute blobs across multiple L1 blocks, potentially benefitting from faster inclusion. For example, if a block already includes 5 blobs and we only have 4 more to add (after including, e.g. 3 in a previous transaction), then the current L1 block producer could include our transaction.
        - **Robustness:** a flexible posting policy enables us to adapt to network conditions and can help include our blobs earlier than if submitted all in a single DA transaction.
        
        The following are downsides of this approach:
        
        - **Higher Costs.** Since we would be potentially sending multiple DA transactions, the `L1CommitmentFee` portion of the `L1DAFee` (which is part of the broader `L1Fee`) will be paid for each transaction, although each time less gas would be consumed than if all blobs are sent at once. Assuming the blob gas prices do not change, the `L1BlobFee` (part of the `L1DAFee`) is not affected by this change. The commitment fee covers the gas for several sanity checks and storage updates.
        - **Increased complexity to recover from transaction failure.** To ensure that blobs are processed in order on L1, we need to add additional logic or calldata: either each DA transaction can only be sent to L1 if all the previous ones are confirmed, or, if blobs are sent out-of-order in multiple txs, we need to be able to reconstruct the order because blobs are not self-contained.
        - **Implementation.** This approach requires implementing additional logic for distributing the blobs over multiple transactions in a locally optimal way.
        
        If we take a look at [L1 blocks](https://blobscan.com/blocks), many of them include blob-carrying transactions in a *tetris*-like fashion, i.e. from multiple rollups. We can take advantage of the fact that blob submission is irregular across the ecosystem, and let L1 block producers optimize the inclusion of zkSharding blobs. The way we split our set of blobs can be dynamic based on network conditions, while maintaining a low transaction count. One approach is posting $\texttt{floor(max\_blobs/2)}$ per DA transaction; this way, the two transactions could still be fit together if no blobspace is demanded by higher-bidding rollups, or in short time after each other, depending on the number of blobs from other rollups. This approach can help us keep DA bids constant since we just “try to fit” in one partial block, instead of reserving an entire one, and potentially bidding higher.
        
        The exact **distribution of blobs** over commit transactions can be determined by the Proposer based on network conditions.
        
        **Alternative: Flexible batching limit.** The Observer may also only relax the DA limit *if* the estimated prover capacity falls under a certain threshold (e.g. < 80%). This means, that if the prover could accommodate more shard blocks, in order to run the prover efficiently, we can bypass the usual DA limit despite the in-progress batch reaching it.
        
        **This design should be revisited with new insights from the blob market and our actual blobspace usage over time.**
        
        [TODO] Add blob-to-tx distribution logic to Proposer if proposal accepted.
        
    
    <aside>
    
    If we choose to not restrict the number of DA transactions per batch, the Observer doesn’t need to do any data estimations.
    
    </aside>
    
3. **Timeout.** To maintain fast (soft) finality, i.e. transaction data being made available on L1, as well as to utilize the Prover Network efficiently by not keeping it idle, we set a timeout for sealing a batch. If L2 network traffic is too low such that neither prover or DA constraints have been reached by the in-progress batch, if the `MAX_BATCH_SEAL_DELAY` has been reached, the batch is considered complete and its coordinates sent to the Aggregator for further processing. This timeout can be initially set to the time it takes to prove one batch. 

**Batch Sealing Algorithm.** The algorithm takes as input the batch constraints and attempts to form a batch before reaching these constraints. It starts by downloading the first unprocessed main shard block, downloading all referenced execution shard blocks, then all unreferenced execution shard blocks by tracking down an individual shard until the last included (in a previous batch) block. While adding blocks to the batch, it re-executes transactions and updates their aggregate consumption of the prover capacity. After processing each main shard block and all referenced and unreferenced execution shard blocks pertaining to it, the algorithm checks that the consumption is still within *all* constraints. If none of the limits has been reached, it proceeds to download the next main shard block and process it analogously. Optionally, it can check the consumption percentage per constraint and, if they are above a given threshold, not proceed with testing the inclusion of the next main shard block. If the capacity has been surpassed on any limit, the algorithm rolls back one main shard block (to be included in the next batch), saves the interval of main shard blocks, and terminates.

Once a batch is formed, the interval of main shard block IDs is sent to the Aggregator and the Observer starts observing data for a new batch. 

```go
// Observer Algorithm

// Pseudocode for checking data limits
// Needed in scenarios where there are many data-heavy txs, but with little computation s.t. one subgraph might exceed the maximum DA. This is not likely to happen, especially as MAX_DA_BATCH should increase over time with new blobspaces on Ethereum. However, we need this catch since in case it does happen, once in a million batches, we either need to find a fitting subgraph or somewhat inform the Proposer how many DA txs to prepare (the Proposer could simply figure it out from the number of blobs provided, but we still need to check DA limits from the second subgraph we want to add)

MAX_DA_BATCH = MAX_BLOBS_PER_BLOCK * BYTES_PER_BLOB

checkDALimit(inProgressBatch, newSubgraph) {
		
	if inProgressBatch == empty // i.e. we are at first subgraph
			
			s = evaluateBatchSize(0, newSubgraph)
			if s > MAX_DA_BATCH
				DA_tx_count = s / MAX_DA_BATCH
			
			else
				DA_tx_count = 1
	else // we try to add new subgraph
		
			s += evaluateBatchSize(inProgressBatch, newSubgraph)
			if s > MAX_DA_BATCH
				return inProgressBatch // defer the new subgraph to the next batch
			else
				inProgressBatch = addSubgraphToBatch(inProgressBatch, newSubgraph)
		
}
```

**1.2.1.2. Main Shard Block Download**

Once the Aggregator has received the interval of main shard blocks to be batched, it starts downloading the respective main shard blocks via the cluster RPC. For each block, it retrieves the list of referenced and unreferenced execution shard blocks to be included in the same batch.

**1.2.1.3. Execution Shard Blocks Processing**

Based on the referenced execution shard block hashes, the Aggregator retrieves all non-referenced shard blocks that pertain to the same main shard block by traveling down the shard chains until previously batched blocks are reached (see figure below). For each non-referenced shard block, the Aggregator retrieves its hash and adds it to the `Batch` to be sent to the Proof-provider.

![View of L2 blocks for batch formation. Green blocks are part of the previous batch (sent for proving/proven). The grey blocks will not be included in the in-progress batch (because of capacity). Orange and yellow blocks are all part of the in-progress batch. Orange shard blocks are referenced in the main shard blocks, while yellow ones need to be retrieved by traveling down the individual shard chains until already proven blocks (green) are reached. 
*Note: not displayed: main shard block references in shard blocks (arrows from shard blocks to main shard blocks).*](/figures/l2-mechanisms/zksharding-sync-committee-00.png)

View of L2 blocks for batch formation. Green blocks are part of the previous batch (sent for proving/proven). The grey blocks will not be included in the in-progress batch (because of capacity). Orange and yellow blocks are all part of the in-progress batch. Orange shard blocks are referenced in the main shard blocks, while yellow ones need to be retrieved by traveling down the individual shard chains until already proven blocks (green) are reached. 
*Note: not displayed: main shard block references in shard blocks (arrows from shard blocks to main shard blocks).*

### 1.2.2. Batching

**1.2.2.1. Batch Structure for Data Availability**

The Aggregator creates two new batch-related objects:

- `Batch` - which contains cordinates for the batch definition, such as the interval of main shard blocks to be proven, the old and new state roots the data will be proven against, and other required information for the prover;
- `CommitBatch` - which contains the shard block and transaction data that will be made available on L1. This data preserves the core of each transaction and relevant block information, such that the L2 state could be restored given all the blob history or that, with the proper tools for decoding blob data, users can verify the inclusion of their L2 transaction directly from L1. The batch consists of two parts: the `BatchHeader` which contains metadata about the batch, including its identifier, and the `BatchData` which contains the compressed stream of transaction and block data.
- **zkSharding Transaction and Block objects**
    
    Below are the latest structures for the zkSharding transaction and block.
    
    ```go
    // zkSharding Transaction format
    type Transaction struct {
    		TransactionDigest {
    				Flags                  TransactionFlags          []byte
    				FeeCredit              Value                     32 bytes
    				MaxPriorityFeePerGas   Value                     32 bytes
    				MaxFeePerGas           Value                     32 bytes
    				To                     Address                   20 bytes
    				ChainId                ChainId                    8 bytes
    				Seqno                  Seqno                      8 bytes
    				Data                   Code                      []byte
    		}
    		From           Address                   20 bytes
    		RefundTo       Address                   20 bytes
    		BounceTo       Address                   20 bytes
    		Value          Value                     32 bytes
    		Token          []TokenBalance            
    				{
    					Token    TokenId (type Address)    20 bytes
    					Balance  Value                     32 bytes
    				}
    		RequestId      uint64                     8 bytes
    		RequestChain   []*AsyncRequestInfo       
    				{
    					Id     uint64                       8 bytes
    					Caller Address                     20 bytes
    				}
    		Signature      Signature                 []byte
    
    }
    
    // RPCInTransaction (what currently gets retrieved by the Sync Committee) has the following extra/missing fields:
    + Success       bool                   1 byte
    + BlockHash     Hash                  32 bytes
    + BlockNumber   BlockNumber            8 bytes
    + GasUsed       Gas                    8 bytes
    + Hash          Hash                  32 bytes
    + Index         uint64                 8 bytes
    - RequestChain  []*AsyncRequestInfo   28 bytes
    
    // zkSharding Block format
    type Block struct {
    		Id                       BlockNumber           8 bytes
    		PrevBlock                Hash                 32 bytes
    		SmartContractsRoot       Hash                 32 bytes
    		InTransactionsRoot       Hash                 32 bytes
    		OutTransactionsRoot      Hash                 32 bytes
    		OutTransactionsNum       TransactionIndex      8 bytes
    		ReceiptsRoot             Hash                 32 bytes
    		ChildBlocksRootHash      Hash                 32 bytes
    		ConfigRoot               Hash                 32 bytes
    		LogsBloom                Bloom                []byte
    		Timestamp                uint64                8 bytes
    		BaseFee                  Value                32 bytes
    		GasUsed                  Gas                   8 bytes	
    }
    
    // RPCBlock (what currently gets retrieved by the Sync Committee) has the following extra/missing fields:
    + Hash                 Hash              32 bytes
    + ShardId              ShardId            4 bytes
    + Transactions         []*RPCInTransaction
    + TransactionHashes    []Hash           []32 bytes
    + ChildBlocks          []Hash           []32 bytes 
    + MainChainHash        Hash              32 bytes
    - OutTransactionsRoot  Hash              32 bytes
    - OutTransactionsNum   MessageIndex       8 bytes
    - ConfigRoot           Hash              32 bytes
    ```
    
    **Question:**
    
    - is DbTimestamp in RPCBlock the same as Timestamp in Block?
    - what are Currency and RequestChain
- **What transaction and block data do other rollups publish to Ethereum?**
    
    Treatment of transaction data-rollups only, no state diffs rollups.
    
    - **Scroll**
        
        <aside>
        
        **Summary**
        
        TODO @Raluca Diugan January 21, 2025 
        
        </aside>
        
        - **Detailed Notes**
            - Scroll groups blocks into chunks, chunks into batches, then bundles batches to generate a single validity proof for a single batch.
                - the chunk is defined by a start and an end block (identified by its number/hash)
                - Batch  ‣
                - DABatch ‣
    - **Linea**
        
        <aside>
        
        **Summary**
        
        Linea batches multiple non-empty blocks into a batch, and compresses the batch into a blob. It publishes multiple blobs at the same time. Each blob comes with its own KZGProof (`dataProof`). See the code snippets below for what data is preserved from txs/blocks.
        
        </aside>
        
        - **Detailed Notes**
            - blob submission (5 consecutive txs) by Linea https://etherscan.io/tx/0x21a3b07a42fd10be5a02475114d3634fc2cd1583c80b405053e88ef2a3a98bcb
                - 26 blobs over 5 txs (4 * 6 + 2)
                - 1 KZGProof and evaluation point per blob
                - blobs have zero padding at the end → indication that Linea packs data into blobs based on block/transaction
                - `blobPadded` contains the compressed data (`compressedStream`), padded to fit the blob format; the commitment is then generated on `blobPadded` (and then versionedHash, etc.)
            - encoding of
                - block for compression ‣
                    - `EncodeBlockForCompression` takes a `types.Block` (the block must have at least one tx) as input and encodes it
                - transaction for compression ‣
                    - `EncodeTxForCompression` takes a `types.Transaction` as input and encodes it (via the local method `EncodeTxForSigning` ‣ )
                    
                    ```go
                    // NOTE: types.Block and types.Transaction are structs from the geth library
                    
                    // EIP-1559 tx fields preserved by Linea for compression/DA
                    
                    // Linea        // zkSharding equivalent
                    ChainId         ChainId
                    Nonce           Seqno
                    GasTipCap       
                    GasFeeCap       
                    Gas             
                    To              To
                    Value           Value
                    Data            Data
                    AccessList      --
                    
                    // Block fields preserved by Linea for compression/DA
                    
                    // Linea             // zkSharding equivalent
                    Time                 Timestamp
                    len(Transactions)    len(Messages)
                    Hash                 Hash
                    Transactions         Messages
                    ```
                    
            - Linea’s `blobMaker` will attempt to add as many blocks as possible to an in-progress batch, and revert when the blob/decompression circuit capacity has been exceeded (see implementation here ‣)
    - **Arbitrum**
        
        <aside>
        
        **Summary**
        
        TODO @Raluca Diugan January 21, 2025 
        
        </aside>
        
        - **Detailed Notes**
            - `l2MessageData` is encoded to blobs via `EncodeBlobs` method https://github.com/OffchainLabs/nitro/blob/b6577a12f599df970005502096df3b7242cb5ced/util/blobs/blobs.go#L61C6-L61C17 (formatting blobs according to blob structure (4096 field elements)
            - the `maybePostSequencerBatch` method prepares the blob data by calling `encodeAddBatch` on the `sequencerMsg` (i.e. the `l2MessageData`)
            - `sequencerMsg` is derived from batch segments (their `compressedBuffer`s)
                - ‣
                - 
- **Motivation for the `CommitBatch` structure**
    
    We send data to L1 in the form of blobs which are only temporarily made available on Ethereum. Nonetheless, this temporary availability makes it possible for users and third-party services to retrieve the data, independently verify it, extract required information, or archive it for posterity. In this sense, we perceive the L1 as [a bulletin board for downloading essential L2 data](https://notes.ethereum.org/@vbuterin/proto_danksharding_faq#If-data-is-deleted-after-30-days-how-would-users-access-older-blobs).
    
    As such, since the L2 state can be reconstructed from blobs (wherever they are stored long-term), we include in the `CommitBatch` the L2 block and transaction data necessary to facilitate this reconstruction in full. 
    
    At the same time, we include sufficient information to enable users (or third-party services) to individually generate merkle proofs for withdrawals and failed deposits, or in the case of a full L2 exit. Similarly to how dYdX users could use data (made available as calldata) to exit the L2 when the sequencer stopped operating, users on our network can request data from archival or exit services to exit the L2. For completing regular withdrawals or for recovering locked funds from failed deposit transactions, the blob availability window (~18 days) should be enough to retrieve merkle proof data directly from L1.
    
    
- **ShardDAG considerations for DA and proving**
    
    [TODO]
    
- **[IN PROGRESS IN SEPARATE DOCUMENT] Data Objects**
    
    [TODO] add shardDAG fields
    
    **Pruned fields in** `PrunedTransaction`**:**
    
    - `Seqno` - can be retrieved from previous state
    - `To`/`From` - can be shortened to ~4 bytes if encoding them in an MPT
    - `Value` - can be shortened to ~4.5 bytes
    - `Signature` - `r`, `s`, `v` fields are not necessary for computing the state update
    - `FeeCredit` - could be computed for the entire batch.  [TODO]
    - `MaxPriorityFeePerGas` and `MaxFeePerGas` - can be computed for the entire batch
    
    ```go
    // Data object to be sent to the Proof-provider and stored on SC nodes
    // This can be thought of/referred to as a batch "definition" 
    type Batch struct {
    		BatchId                   string
    		NewStateRoot              common.Hash
    		OldStateRoot              common.Hash
    		MainShardBlocksInterval   []BlockNumber	// the interval of main shard blocks part of the batch
    		ExecutionShardBlocks      []common.Hash // block hashes of all execution shards referrenced and non-referenced pertaining to the batch
    		WithdrawalRoot            common.Hash // Note: requires implementation of bridge processes
    		l2tol1root                common.Hash
    }
    
    // Data object to be (1) compressed, (2) PoE'ed, (3) blobified, and (4) posted to L1
    type CommitBatch struct {
    		BatchHeader                BatchHeader
    		BatchData                  BatchData
    }
    
    // Holds information about the batch of data
    type BatchHeader struct {
    		BatchId                    string
    		Timestamp                  uint64 // of the last block in batch
    		NumBlocks                  uint32 // number of blocks in batch
    		NumTxs                     uint32 // number of transactions in batch
    }
    @Raluca add fee-related fields to batch/block (globally calculated per batch or block)
    // Holds the compressed data stream of blocks + transactions
    type BatchData struct {
    		PrunedBlocks              []PrunedBlock
    		PrunedTransactions        []PrunedTransaction
    }
    
    type PrunedBlock struct {
    		ShardId                   ShardId
    		Id                        BlockNumber
    		PrevBlock                 Hash
    		Timestamp                 uint64
    }
    
    type PrunedTransaction struct {
    		ChainId                   ChainId 
    		ShardId                   ShardId // retrieved when processing one block
    		To                        Address // pruned to 4B
    		From                      Address // pruned to 4B
    		BounceTo                  Address // ONLY WHEN != From
    		RefundTo                  Address // ONLY WHEN != From
    		Value                     Value // pruned
    		FeeCredit                 Value // TBD
    		Data                      Code
    }
    
    // From zkEVM Engineering team
    // NEW: data format of Batches packed into commitBatch blobs
    
    // Uncompressed part //
    type BatchHeader struct {
    	Magic   uint16 = 0xDEFA
    	Version uint16 = 0x0001
    }
    
    // Compressed part == binary marshaled proto.Batch object //
    // Proto definition below // 
    
    syntax = "proto3";
    
    package sync_committee_types;
    option go_package = "/proto";
    
    // Uint256 represents a 256-bit unsigned integer as a sequence of uint64 parts
    message Uint256 {
        repeated uint64 word_parts = 1;  // 4 uint64 parts composing the 256-bit number
    }
    
    // Address represents an Ethereum address
    message Address {
        bytes address_bytes = 1;  // 20-byte address
    }
    
    // Nil transaction binary representation which is going to be stored on the L1 in blob format
    message BlobTransaction {
        uint32 flags = 1;
        uint64 seq_no = 2;
        Address addr_from = 3;
        Address addr_to = 4;
        optional Address addr_bounce_to = 5;
        optional Address addr_refund_to = 6;
        Uint256 value = 7;
        bytes Data = 8;
    }
    
    message BlobBlock {
        uint32 shard_id = 1;
        uint64 block_number = 2;
        bytes prev_block_hash = 3;
        uint64 timestamp = 4;
        repeated BlobTransaction transactions = 5;
    }
    
    message Batch {
        string batch_id = 1;
        uint64 last_block_timestamp = 2;
        uint64 total_tx_count = 3;
        repeated BlobBlock blocks = 4;
    }
    ```
    

**1.2.2.2. Compression Algorithm**

To minimize our blobspace requirements, we compress the `CommitBatch` data via `zstd`. This approach is taken by other zkRollups that make available either transaction data (Linea, Scroll) or state diffs (Starknet, zkSync Era). For a comprehensive study on compression in rollups, see [*SoK: Compression in Rollups*](https://ieeexplore.ieee.org/abstract/document/10634469), by Palakkal et al.

<aside>

**Future Upgrades**

Until we better understand how realistic data looks like for us and until we fix the batch structure to L1 and perform more analysis including with compression heuristics, we can utilize one of the top performing compression algorithms according to this study: https://github.com/quantstamp/l2-compression/blob/main/docs/compression-preprint.pdf

The top performing lossless compression algorithms (for Optimism 2022-2023 data) for the batched approach (as opposed to one transaction at a time) are:

The average compression ratio is computed as: $c_r = \frac{compressedDataSize}{originalDataSize}$, i.e. the lower the better. 

- Brotli (used by Arbitrum) - 37% average compression ratio, average time to compress 7212s
- zstd (used by Scroll) - 37%, 2173s (see implementation https://github.com/scroll-tech/da-codec/blob/main/encoding/zstd/zstd.go)
- lzma (the lzss variant is used by Linea) - 37%, 3616s
- zlib - 39%, 853s

`zstd` additionally performs the best among the four in the per-transaction approach. Based on the overall performance results from this study, unless the implementation requirements are too high by comparison, we should opt for `zstd`.

**`zstd` is developed by Facebook and operates under a dual BSD or GPLv2 license.**
https://github.com/facebook/zstd

</aside>

For now, we use `zstd` *as is*. In the future, we can look into optimizations such as training a dictionary on our network’s data.

<aside>

**Research Question:** Compression Verification via System Contracts

</aside>

**1.2.2.3. Blob Composition**

Once the DA batch data has been compressed, it can be fit over blobs. Since the Dencun upgrade, rollups can choose to make data available on Ethereum as Binary Large Objects (*blobs*), thus reducing DA costs. These objects will be sent alongside a *blob-carrying* transaction to L1. The full specifications of EIP-4844 Shard Blob Transactions are available [here](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4844.md).

**Batch-to-Blobs.** Depending on the total amount of bytes resulting from compression, a batch will fit over several blobs. The blobs are *not self-contained* meaning that they cannot be decoded to the original data without having the entire set of blobs encoding the compressed stream of data. A `batchToBlobs` mapping is stored by the Sync Committee Aggregator to help identify blobs (by their versionedHash) encoding each batch. Later on, the L1 main rollup contract will preserve a list of `versionedHashes` of blobs for each batch. These two sets must be equal.

**Blob format.** A blob consists of 4096 field elements of 32 bytes each. Our compressed stream of bytes $\mathcal{C}$ is divided over several blobs (formatted accordingly), with the last one padded with 0s, if needed. For each blob, the Aggregator additionally derives a KZG commitment $c$, its versioned hash $v$, a random point $z$, the point’s evaluation $y$, the KZG proof $\pi_{kzg}$ attesting to this evaluation result, and a blob proof $\pi_{blob}$. It then sends all blobs and their corresponding $(c, v, \pi_{blob})$ to the Proposer, and the blobs and their corresponding $(c, z, y,\pi_{kzg})$ to the Verifier. 

**Blob limits.** Due to the Pectra update coming to the Ethereum testnet in February 2025 and to mainnet in March 2025, we can plan for the new blob limits, as follows.

```go
// see geth params https://github.com/ethereum/go-ethereum/blob/master/params/protocol_params.go
// @dev Decide on reading from Geth vs Config (to align with future L1 upgrades)

TARGET_BLOBS_PER_BLOCK_ELECTRA = 6
MAX_BLOBS_PER_BLOCK_ELECTRA = 9 // the limit to use for testnet v2
```

This gives a limit of 1179648 bytes per `commitBatch` transaction. For testnet v2, we can limit the Sync Committee to **one DA transaction**, i.e. impose a blob count limit of 9 per compressed batch. 

<aside>

**Future upgrades**

For testnet v3, mainnet, or future upgrades we can extend DA to multiple L1 transactions. Particularly for mainnet (where gas costs us real ETH), to prevent having to set high priority fees to have our 9-blob transaction go through in due time, we could split them across several transactions, coordinating with other rollups. For example, if one rollup intends to publish 3 blobs, we can bid for 3 of ours in the same block.

**Future work:** look into coordination mechanisms for blobspace, e.g. blob aggregation.

</aside>

**Note:** For testnet v2, there is no Verifier. Thus the tuple above is sent directly to the Proof-provider, and $\pi_{kzg}$ is also sent to the Proposer who will assemble the `dataProof`.

- **Ecosystem-wide challenges with blobs and how this impacts zkSharding**
    
    <aside>
    
    **Reassess blob market by May 2025 to understand the effects of Pectra, Fusaka estimates, new rollups being launched, and existing rollups’ strategies.**
    
    </aside>
    
    EIP-4844 was introduced as an alternative (and temporary) storage for rollups to combat the high costs of Ethereum calldata. Blobspace started trading at very cheap (1 wei per unit of blob gas, i.e. per byte stored), which made blobs accessible and preferred by many rollups. This situation lead to increased demand for blobspace (from both new and old rollups), such that since around November 2024 blob gas prices have been consistently high, occassionally surpassing the costs of calldata such as in this [June 20th 2024](https://medium.com/@samuelisaac1995/uncovering-blob-transaction-patterns-and-gas-fee-spikes-on-arbitrum-june-20th-and-may-19th-167a2642c3ad) incident. Ethereum researchers and developers are working on bringing more blobspace to Ethereum (and Vitalik himself considers this to be a top priority for at least the next upgrade), with the Pectra upgrade doubling the blob target to 6 blobs (and increasing the limit to 9). For perspective, the end-game of the Danksharding roadmap is 64 blobs per Ethereum block. In order to reach this, however, Ethereum needs to implement other improvements such as PeerDAS, to ensure their own validators are not affected by increased bandwidth demands. PeerDAS is also expected to go into effect in 2025 or perhaps the first half of 2026 (in the Fusaka upgrade). With our launch scheduled for Q4 2025 (and operating only testnets until then), we might be minimally affected by the current trends in the blob market. We should reassess this situation by May 2025 (when the ecosystem is expected to run into blobspace limits again) to give us enough time to pivot for mainnet launch (e.g. to using CALLDATA) or find other mitigation strategies than the ones we already plan for (e.g. smart scheduling of commit transactions). With future upgrades, we can additionally implement changes to our DA policy.
    
    - **Possible mitigations for us to implement**
        - (already planned) **data compression:** compress data in order to utilize less blobspace;
        - calldata fallback mechanism/**calldata spillover**: if the blobfees surge to the point of surpassing calldata costs, temporarily commit data as calldata;
        - (already planned) smoothed EMA-based charging for DA: distribute excess fees caused by a surge in DA costs across a longer window of L2 transactions. While not directly increasing blobspace, it softens the fee impact on L2 users;
        - **smart blob scheduling** and coordinated posting schedules: collect L1 network information to help determine the best time to post blobs (e.g. prevent splitting blobs across multiple transactions, set fees accordingly to ensure commit transactions go through); using such protocols for coordinating blob posting with other L2s;
        - **blob-sharing standardization:** share partially empty blobs with other L2s.
    - **Sources and Further Resources**
        
        Some resources discussing this problem at length or for exploring the current state of the blob market:
        
        - https://x.com/gauthamzzz/status/1880342051721736267
        - https://x.com/0xBreadguy/status/1880396149238165723
        - https://blobscan.com/
        - https://medium.com/@samuelisaac1995/uncovering-blob-transaction-patterns-and-gas-fee-spikes-on-arbitrum-june-20th-and-may-19th-167a2642c3ad
        - https://dune.com/glxyresearch_team/eip-4844-blobs
        - https://ethereum-blob-simulator.netlify.app/
        - https://ethresear.ch/t/on-blob-markets-base-fee-adjustments-and-optimizations/21024
        - https://ethresear.ch/t/blob-aggregation-step-towards-more-efficient-blobs/21624/1
        - https://paragraph.xyz/@spire/shared-blob-compression
        - https://ethresear.ch/t/potential-impact-of-blob-sharing-for-rollups/20619
- **Blob generation implementation sketch**
    
    <aside>
    
    **Open question: Implementation of blob formatting, required for PoE (for using the point evaluation precompile)**
    
    Based on this explanation from [Linea](https://github.com/Consensys/linea-monorepo/blob/main/prover/lib/compressor/blob/v1/blob_spec.md#final-blob-byte-alignment), the blob needs to be formatted into field elements for verification purposes, such that not all 4096*32 bytes within a blob can store batch data, i.e. for each 32-byte field (256 bits) only 254 bits can be used for data. 
    
    According to [EIP-4844 specs](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4844.md):
    
    > The precompile MUST reject non-canonical field elements (i.e. provided field elements MUST be strictly less than `BLS_MODULUS`).
    > 
    
    Scroll’s approach is to introduce one 0 byte before every 31 bytes of data (see `makeBlobCanonical` method ‣). See Linea’s [implementation](https://github.com/Consensys/linea-monorepo/blob/main/prover/lib/compressor/blob/encode/encode.go). We do not yet have a set approach for how to deal with the formatting question and whether simply replacing the one byte with 0 is a sound approach.
    
    </aside>
    
    ```go
    // Parameters from geth client
    // https://github.com/ethereum/go-ethereum/blob/master/params/protocol_params.go
    // e.g. from Arbitrum https://github.com/OffchainLabs/nitro/blob/master/util/blobs/blobs.go
    BLOB_LENGTH = params.BlobTxBytesPerFieldElement * params.BlobTxFieldElementsPerBlob
    
    // For each batchId (string), maintain an ordered list of blob hashes
    batchToBlobs := make(map[string][]common.Hash) // geth type
    
    // TODO
    // This method fits the stream of data over multiple blobs in canonical form
    
    func CompressedDataToBlobs(compressedStream []bytes) {
    	return blobs
    }
    
    // TODO
    // This method returns a random evaluation point based on blob data
    func ComputeBlobEvaluationPoint() {
    	return z
    }
    
    // Generate blobs
    // CommitBatch is the batch to be processed to blobs
    compressedStream := Compress(CommitBatch)
    
    blobs := CompressedDataToBlobs(compressedStream)
    
    // Generate blob-related data
    
    var blobHashes []common.Hash
    var sidecar types.BlobTxSidecar
    var dataProofs []bytes
    
    for i := range blobs {
    
    	// ensure each blob is precisely 128KB by 0-padding at the end
    	blob := padBlob(blob)
    
    	KZGCommitment := kzg4844.BlobToCommitment(blobs[i])
    	versionedHash := kzg4844.CalcBlobHashV1(SHA256, KZGCommitment)
    	
    	// Add versionedHash to map tracking which blobs contain which batch
    	// Note: the list of hashes must be ordered
    	batchToBlobs[batchId].append(batchToBlobs[batchId], versionedHash)
    	
    	z := computeBlobEvaluationPoint()
    	y, KZGProof := kzg4844.ComputeProof(blobs[i], z)
    	blobProof := kzg4844.ComputeBlobProof(blobs[i], KZGCommitment)
    	
    	// prepare dataProof according to the following mapping
    	// z        | y        | KZGCommitment | KZGProof
    	// 32 bytes | 32 bytes | 48 bytes      | 48 bytes      
    	// add the dataProof in the dataProofs variable above
    
    	blobHashes[i] = versionedHash
    	
    	sidecar.Blobs[i] = blobs[i]
    	sidecar.Commitments = KZGCommitment
    	sidecar.Proofs = blobProof
    
    }
    ```
    

**1.2.2.4. Batch Storage**

For now, the Sync Committee stores historical batches. It indexes them according to the `batchID` and stores the compressed data stream and the interval $\mathcal{I}$ of main shard blocks (which defines the batch) for each of them. Archival nodes can take over this task, with the Sync Committee only storing the most recent batches. 

The Aggregator stores batch data and proof requests locally. It stores the `Batch` (not to be confounded with the `CommitBatch`) used to generate proving requests, as well as all issued requests. In case the prover fails, it simply reissues a task for the same batch, fetching the batch coordinates from storage. As the prover is stateless w.r.t. shard data, transaction data is downloaded from the cluster with each request. The Proof-provider stores issued tasks, their status, and result. In the future, old batch and task data could be pruned on a rolling basis, as it reaches an expiry threshold.

The Proposer stores blob data temporarily, until all DA transactions are submitted and confirmed on L1. Blobs will be available on Ethereum afterwards, for any actor wishing to download it within the availability window. Blobs are guaranteed to be stored on Ethereum for ~18 days, after which they can be pruned. Blobs may also be retrievable via third-party storage providers such as Ethereum Swarm (up to 1 year) and Google Cloud Storage ([source](https://docs.blobscan.com/docs/features)). The Proposer also keeps the state root history.

<aside>
<img src="/icons/science_green.svg" alt="/icons/science_green.svg" width="40px" />

**Mainnet and Decentralization**

In the future, data archival can be handled by a dedicated committee. 

Also see utilities such as Base’s https://github.com/base-org/blob-archiver.

Initially, the Sync Committee stores both `Batch` and `CommitBatch` for each `batchID`. Later on, it will only store the batch metadata (i.e. `Batch`), with the block/transaction data being stored with dedicated and third party archive services.

</aside>

### 1.2.3. Requesting Proofs and Proving

**1.2.3.1. Placeholder**

For mainnet, Placeholder — our proof system — consists of two components:

- a distributed FRI (dFRI) interactive oracle proof of proximity (IOPP), and
- Plonkish arithmetization.

The DFRI algorithm is a version of batched FRI which distributes workload across multiple prover workers, eventually producing a (large) proof. The resulting proof (along with other arguments for L1) should ideally fit the L1 execution client limits (e.g. 128KB for Geth and Nethermind).

- **1.2.3.2. [Not for Mainnet] KZG + PLONK**
    
    The zkSharding proof system consists of two broad components. The first one, a `DFRI + PLONK` sub-system, takes the L2 batch data and generates a large proof of the state transition. The second one, a `KZG + PLONK` sub-system, takes the large proof as input (among other public and private inputs), and generates a succint proof that is eventually sent to L1 for verification alongside public inputs. We refer to them as the *inner*-proof and *outer*-proof, respectively. 
    
    Once the inner-proof is generated, the outer-proof can be generated. For this, the prover takes as input a set of public and private inputs, including the inner-proof. It proves its correctness as well as other statements such as data equivalence. 
    
    - **Public inputs**. The KZG prover takes the random evaluation point $z$, the evaluation result $y$, the KZG commitment to blob data $c_1$, the KZG proof of evaluation, among other required arguments as public input.
    - **Private inputs.** The KZG prover takes the DFRI proof, the compressed data (prior to blob encoding), as well as other required arguments as private input.
    
    The prover computes a KZG commitment $c_2$ over the compressed data.
    

**1.2.3.3. [Pending PoE for Mainnet Decision] Input Generation for Data Equivalence**

- **Proof of Equivalence of Blob Data**
    - **PoE Schema**
        
        [TODO]
        
        depending on aggregation or not, either passed with agg or with single proof request
        
        **Cost considerations and PoE Aggregation.** 
        
        [TODO]
        
        https://eprint.iacr.org/2020/081.pdf
        
    
    <aside>
    <img src="/icons/book_green.svg" alt="/icons/book_green.svg" width="40px" />
    
    **PoE by Starknet**
    
    Beautiful, clear explanation of PoE by Starknet https://community.starknet.io/t/data-availability-with-eip4844/113065#using-blobs-the-commitment-equivalence-trick-4
    
    </aside>
    

**1.2.3.4. Proof Verifier**

To ensure that the proof we send to L1 is correct and prevent resubmitting a new one, we perform an additional verification at the Sync Committee level. This verification can be done by one of the Sync Committee modules (Task Scheduler or Proposer) or by the Proof-provider themselves, prior to returning the proof. If done by the Proof-provider, it can be either part of the same task request or a separate task issued after the proof is available to the Sync Committee. This approach is compatible to a decentralized prover network where independent verifiers can take proving tasks, and others verification tasks and be rewarded accordingly.  

**Mainnet Verifier Placement.** Until we have a clearer understanding of the hardware requirements for verifying the proof on the Sync Committee, the Verifier is implemented as part of the Proof-provider. A new `verification` task is issued by the Task Scheduler upon receiving a proof. A dedicated prover will be responsible for verifying the proof by running the `DFRIVerifier`. The result of this request is recorded on the Proof-provider and Sync Committee. If the verification is successful, the Aggregator forwards the proof to the Proposer. A C++ `DFRIVerifier` is currently under development.

<aside>
<img src="/icons/info-alternate_green.svg" alt="/icons/info-alternate_green.svg" width="40px" />

**Post-mainnet Upgrades/Verifier on SC.** Once we determined the costs of off-chain verification, we can decide whether the Verifier should be run by Sync Committee or remain on the Proof-provider. To prepare for the decentralized prover network setting, where trustless nodes provide the Sync Committee with state transition proofs, we can task one of the Sync Committee modules with the verification. To ensure the shortest communication path to recovery in case of an invalid proof, verification should be undertaken by the Task Scheduler. If the proof is invalid, the Task Scheduler can reissue the proving task (to the same or a different Proof-provider). If no valid proof is generated after several attempts, the cluster should be notified.

</aside>

### 1.2.4. Transaction Submission

**1.2.4.1. CALLDATA and Transaction compilation**

- **Decoupling DA commit and state update transactions.**
    
    The Sync Committee Proposer submits two types of transactions to L1, via the `commitBatch` and `updateState` methods. This approach achieves:
    
    - **Higher accuracy of estimated L1 DA fees.** By minimizing the delay between submitting L2 transactions and publishing them (within batches) on L1, the estimated L1 DA fees charged from users are expected to be more accurate, compared to publishing data only when the validity proof is available, e.g. one hour in the future.
    - **Individual optimization for L1 fees.** Ideally, both data commitment and state update would happen as soon as the payload is available. (data and proof, respectively). In case of impracticable L1 fees caused by a sudden surge in L1 activity, L2 could delay the two actions until fees return to acceptable levels. By separating the two workflows, both can be optimized for lower L1 fees, with the condition that data must be published prior to submitting the proof.
    - **Sync Committee Robustness.** In case of Proof-provider failures, publishing data can continue unhindered at a constant pace, preventing overloading the DA bandwidth at a later time.
    
    This approach is similar to Scroll or Linea. By contrast, Starknet performs the two actions in a single transaction, at the cost of high DA latency (e.g., block [#988336](https://starkscan.co/block/988336) was confirmed on L2 ~18 hours prior to commitment to L1). 
    
- **Sending `dataProof` with `updateState` transaction instead of `commitBatch`**
    
    [TODO] add motivation
    

We send two types of transactions to Ethereum:

- **Type 2** (or `0x02` or `DynamicFeeTx`)
- **Type 3** (or `0x03` or `BlobTx` or EIP-4844). This transaction is accompanied by a `sidecar`.

Below are the transaction types from `geth`.

- **DynamicFeeTx**
    
    ```go
    // source: https://github.com/ethereum/go-ethereum/blob/master/core/types/tx_dynamic_fee.go
    
    type DynamicFeeTx struct {
    	ChainID    *big.Int
    	Nonce      uint64
    	GasTipCap  *big.Int // a.k.a. maxPriorityFeePerGas
    	GasFeeCap  *big.Int // a.k.a. maxFeePerGas
    	Gas        uint64
    	To         *common.Address `rlp:"nil"` // nil means contract creation
    	Value      *big.Int
    	Data       []byte
    	AccessList AccessList
    
    	// Signature values
    	V *big.Int
    	R *big.Int
    	S *big.Int
    }
    ```
    
- **BlobTx + BlobTxSidecar**
    
    ```go
    // source: https://github.com/ethereum/go-ethereum/blob/master/core/types/tx_blob.go
    
    type BlobTx struct {
    	ChainID    *uint256.Int
    	Nonce      uint64
    	GasTipCap  *uint256.Int // a.k.a. maxPriorityFeePerGas
    	GasFeeCap  *uint256.Int // a.k.a. maxFeePerGas
    	Gas        uint64
    	To         common.Address
    	Value      *uint256.Int
    	Data       []byte
    	AccessList AccessList
    	BlobFeeCap *uint256.Int // a.k.a. maxFeePerBlobGas
    	BlobHashes []common.Hash
    
    	// A blob transaction can optionally contain blobs. This field must be set when BlobTx
    	// is used to create a transaction for signing.
    	Sidecar *BlobTxSidecar `rlp:"-"`
    
    	// Signature values
    	V *uint256.Int
    	R *uint256.Int
    	S *uint256.Int
    }
    
    type BlobTxSidecar struct {
    	Blobs       []kzg4844.Blob       // Blobs needed by the blob pool
    	Commitments []kzg4844.Commitment // Commitments needed by the blob pool
    	Proofs      []kzg4844.Proof      // Proofs needed by the blob pool
    }
    
    ```
    
- **Setting transaction parameters**
    
    ```go
    // Currently, the calldata of the commitBatch transaction consists of
    type CommitBatchCalldata struct {
    		batchIndex        string
    		blobCount         uint256
    }
    
    // Currently, the calldata of the updateState transaction consists of
    type UpdateStateCalldata struct {
    		batchIndex        string
    		l1tol2Root        common.Hash
    		withdrawalRoot    common.Hash
    		L1MessageHash     common.Hash
    		newStateRoot      common.Hash
    		oldStateRoot      common.Hash
    	  validityProof     []bytes
    	  dataProofs        []KZGProof
    }
    ```
    
    ```go
    // commitBatchTx
    commitBatchCalldata := CommitBatchCalldata{
      // set variables according to struct
    }
    
    commitBatchTx := types.BlobTx{
    	ChainId:    ETHEREUM_MAINNET_ID,
    	Nonce:      [TODO], // utility that reads latest nonce and increments it
    	To:         ZKSHARDING_ROLLUP_CONTRACT_ADDRESS,
    	Value:      bytes32(0),
    	Data:       commitBatchMethodID + commitBatchCalldata, // to hex
    	AccessList: [TODO],
    	BlobHashes: blobHashes, // from Aggregator
    	Sidecar:    sidecar // from Aggregator
    	... // see below for fees
    }
    
    // Add transaction signature
    
    // updateStateTx
            
    updateStateCalldata := UpdateStateCalldata{
    	// set variables according to struct
    }
    
    updateStateTx := types.DynamicFeeTx{
    	ChainId:    ETHEREUM_MAINNET_ID,
    	Nonce:      [TODO], // utility that stores latest nonce and increments it
    	To:         ZKSHARDING_ROLLUP_CONTRACT_ADDRESS,
    	Value:      bytes32(0),
    	Data:       updateStateMethodID + updateStateCalldata, // to hex
    	AccessList: [TODO],
    	... // see below for fees
    }
    
    // Add transaction signature
    ```
    

**1.2.4.2. Setting Transaction Fees**

To submit the two types of transactions we must specify the fees to be paid to L1. We must first retrieve the quotas from L1 via the `L1FeeMonitor` mechanism, then set appropriate values for our transactions. Because these values are fluctuating with L1 network traffic, it is not wise to fix them, otherwise we risk overpaying or delaying the processing of our transactions.

The fees we collect are the following:

- `baseFee` - the minimum amount we need to pay per gas unit used. When running a local node, this value can be retrieved by reading `BlobFee` from the `Header` of an Ethereum block.
- `blobBaseFee` - the minimum amount we need to pay per blob gas unit used. When running a local node, this value can be calculated as shown below, after retrieving `ExcessBlobGas` from the `Header` of an Ethereum block. For testnet v2, we use the latest `blobBaseFee` provided by the RPC, and set the `maxFeePerBlobGas` to twice this value. In the future, we can adjust and update it according to market averages over a period of time.
- `maxPriorityFeePerGas` - the highest amount we are willing to pay per gas unit.

**Economics.** Depending on the RPC provider, some fields can be automatically filled if left empty. For example, we might skip setting the `maxPriorityFeePerGas` and only set the `maxFeePerGas`, we risk delays in when the transaction is processed, i.e. until the base fee drops low enough ([source](https://docs.alchemy.com/docs/maxpriorityfeepergas-vs-maxfeepergas#when-to-use-maxpriorityfeepergas-vs-maxfeepergas-a-hrefwhen-to-use-max-priority-fee-per-gas-vs-max-fee-per-gas-idwhen-to-use-max-priority-fee-per-gas-vs-max-fee-per-gasa)). The `baseFee` can also be automatically filled by the RPC provider since this is a parameter whose value is set by the network anyways. For an inclusion with minimal delays, we should set the value of `maxPriorityFeePerGas`.

**Note:** there is a small delay (maybe a few Ethereum blocks) between the time we query these values and the time the transaction will be processed on L1. As such, we should probably set them slightly higher than the estimated total to avoid the transaction being reverted because the price went up in the meantime.

- **Collecting fee parameters and filling in the respective transaction fields**
    
    ```go
    // Retrieve fees
    // API endpoint: eth_gasPrice - returns gas price in wei
    baseFee := rpc.eth_gasPrice()
    
    maxPriorityFeePerGas := rpc.eth_maxPriorityFeePerGas()
    
    maxFeePerGas := baseFee + maxPriorityFeePerGas
    
    // Blob-only related prices
    blobBaseFee := rpc.eth_blobBaseFee() 
    
    // 
    maxFeePerBlobGas := blobBaseFee * 2
    
    // Estimate how much gas each transaction would consume
    estimatedGasCommitBatch := ethNode.eth_estimateGas(commitBatchTxObject) + error_factor
    estimatedGasStateUpdate := ethNode.eth_estimateGas(updateStateTxObject) + error_factor
    
    // Prepare transactions
    commitBatchTx := types.BlobTx{
    	...
    	...
    	GasTipCap: maxPriorityFeePerGas,
    	GasFeeCap: nil,
    	Gas: estimatedGasCommitBatch,
    	...
    	...
    	BlobFeeCap: maxFeePerBlobGas,
    	...
    }
    
    updateStateTx := types.DynamicFeeTx{
    	...
    	...
    	GasTipCap: maxPriorityFeePerGas,
    	GasFeeCap: nil,
    	Gas: estimatedGasStateUpdate,
    	...
    	...
    }
    ```
    

**1.2.4.3. Submission and Error handling**

<aside>
<img src="/icons/report_red.svg" alt="/icons/report_red.svg" width="40px" />

To prevent the `updateState` transaction from reverting because data is not available, we should make sure that the DA transactions have been successfully processed on L1.

**Note:** this might only be technically relevant when using PoE, but logically data should always come before the proof.

</aside>

**BlobTx.** For testnet v2, we publish a batch as soon as the corresponding blobs are ready and the `BlobTx` compiled. The exponential moving average (EMA) technique used for charging L2 users for L1 DA costs helps mitigate surges in blob fees, such that delaying submitting `commitBatch` transactions is not necessary. If we permit any delays, the MAX_DELAY should be set to the expected proving time, to ensure that the data is available before the proof.

**DynamicFeeTx.** Submitting the `stateUpdate` transaction as soon as possible provides the fastest path to hard finality. However, we do not have a mechanism to mitigate the effects of high L1 gas fees. One approach would be to set a MAX_GAS_PRICE (derived based on, e.g. average historical values, e.g. 50 gwei) that the Proposer is allowed to bid, and delay the submission until the price falls under this limit. 

**1.2.4.4. Separation Between Committer and Proposer**

As we send two transactions to L1 for the same batch (one DA, one settlement), one natural question is to determine whether they should be sent by the same entity, and, in that case, via the same account. 

<aside>
<img src="/icons/info-alternate_green.svg" alt="/icons/info-alternate_green.svg" width="40px" />

Until SC decentralization (and thus for mainnet), there will be a single SC node ran by =nil;. As such, the same entity will propose both transactions. Upon decentralization, both tasks could be done by the same SC node or by different ones, depending on the protocol design.

</aside>

For mainnet, we have the option of setting up a single EoA for the sole SC node, or one for each type of interaction with L1: a Committer and a Proposer. A quick look at a few other prominent zkRollups:

- Scroll: their transaction sender [seems to be the same](https://github.com/scroll-tech/scroll/blob/develop/rollup/internal/controller/sender/sender.go) for both types of transactions. However, they [seem to be sent via different addresses](https://etherscan.io/address/0xa13BAF47339d63B743e7Da8741db5456DAc1E556). According to [L2Beat](https://l2beat.com/scaling/projects/scroll#permissions), Scroll has 2 different EoAs for each committing data and proposing new states, although only one for each is used.
- Similarly, Linea uses different EoAs to send the two transactions: [Committer](https://etherscan.io/address/0x46d2F319fd42165D4318F099E143dEA8124E9E3e) and [Proposer](https://etherscan.io/address/0x52FF08F313A00A54e3Beffb5C4a7F7446eFb6754).
- These two roles also use different EoAs in the case of [zkSync Era](https://etherscan.io/address/0x5d8ba173dc6c3c90c8f7c04c9288bef5fdbad06e).

Some reasons for setting up separate roles and granting them to different addresses are:

- **preparing for higher control granularity:** in the future, if we want to grant different roles do different SC nodes (either or both), we already have the Access Control policies implemented accordingly; For example, we can increase the security requirements for the `updateState` workflow to a multisignature, while keeping them lower (one EoA) for `commitBatch`.
- **increased resilience in case of private key loss:** only one flow is affected and only funds from one address are lost.

### 1.2.5. L1 Contracts

As a rollup, we utilize Ethereum mainnet as our data availability and settlement layer. Transaction data, once encoded accordingly, is sent to Ethereum’s blob storage, and the proof of the corresponding state transition is sent to and verified on L1. Additionally, the rollup provides additional services such as bridging (and later forced transaction inclusion) via L1 through its native bridge. In the following, we describe the high-level overview of the rollup’s L1 architecture and showcase how the two types of transactions are processed on L1, resulting in a complete zkSharding state update.

**1.2.5.1. Access Control for the main rollup contract `NilRollup.sol`**

The main rollup contract’s access control policy is illustrated below:

![Access Control for the main rollup contract on Ethereum. Green: EoA, Orange: EoA or Multisig. ](/figures/l2-mechanisms/zksharding-sync-committee-01.png)

Access Control for the main rollup contract on Ethereum. Green: EoA, Orange: EoA or Multisig. 

The main contract distinguishes between fives roles:

- `DEPLOYER` - the EoA deploying the contract;
- `OWNER` - the owner of the contract;
- `DEFAULT_ADMIN` - the sole admin of the contract;
- `ROLE_ADMIN` - manager of different roles (`COMMITTER`, `PROPOSER`, `VERIFIER`)
- `ROLE` - EoA who can perform its granted role, e.g. `COMMITTER` can call `commitBatch`.

`OWNER` and `DEFAULT_ADMIN` can be transfered. `ROLE_ADMIN` and `ROLE` can be *granted*, *revoked*, or *renounced.* In case no `ROLE` can perform its designated action, the `ROLE_ADMIN` is tasked with it. If no `ROLE` or `ROLE_ADMIN` is available to perform an action, the `DEFAULT_ADMIN` will be responsible for it (e.g. propose state update).

**1.2.5.2. L1 Contract Architecture**

In classic rollup fashion, the on-chain part of zkSharding consists of several contracts, interacting with each other during some of the workflows. The main contracts are as follows:

| Contract Name | Description | Sepolia Address |
| --- | --- | --- |
| Proxy for `NilRollup.sol` | Contract storage and entry point for the main rollup contract | 0x796baf7E572948CD0cbC374f345963bA433b47a2 |
| `NilRollup.sol` | Main rollup contract for submitting blobs to DA and state transition proofs for settlement. It interacts with the Verifier contract and with the Bridge Messenger | 0xB48430965Bd383125d58823D28EFc72642E8615A |
| `Verifier.sol` | Verifier contract which verifies the state transition proof against some public input |  |
|  | Contract for depositing a specific token (ETH, wETH, ERC20, etc.) |  |
|  | Bridge Messenger which processes deposit messages and sends them to the Trusted Relayer |  |

The Contract Architecture is roughly pictured below:

![zkSharding L1 Contract Architecture. Example workflow: The Sync Committee Proposer submits a `updateState` transaction by interacting with the proxy contract. The proxy contract calls the corresponding method on the main contract and corresponding logic is executed. As a subroutine, the Verifier contract is called to verify the state transition proof against given public inputs. ](/figures/l2-mechanisms/zksharding-sync-committee-02.png)

zkSharding L1 Contract Architecture. Example workflow: The Sync Committee Proposer submits a `updateState` transaction by interacting with the proxy contract. The proxy contract calls the corresponding method on the main contract and corresponding logic is executed. As a subroutine, the Verifier contract is called to verify the state transition proof against given public inputs. 

**1.2.5.3. DA Transaction processing**

Once the `Proposer` has submitted a `commitBatch` transaction to the main rollup contract, the blob sidecar is decoupled and replicated on the Consensus Layer (CL). On the Execution Layer (EL), the blob hashes are processed and stored for further reference.

**1.2.5.4. State Update Transaction processing**

The `stateUpdate` execution logic occurs in several substeps. First, various consensus and bridge-related conditions are verified. Equivalence of data is verified for the previously stored data commitments against the provided KZG proofs. Then, the public input is compiled. Lastly, the verifier is called to verify the state transition proof based on the public input. If the proof is valid, the rollup state root is updated.

**1.2.5.5. Bridging**

The Sync Committee does **not** act as a trusted relayer for deposit transactions. Its role is limited to collecting executed deposits and withdrawal information from the cluster, in similar fashion to processing any other L2 transaction, and requesting a proof of correct execution. 

When preparing the `updateState` transaction, the Proposer includes the `L1MessageHash`, the `l2tol1Root`, and the `withdrawalRoot` the transaction’s calldata to be processed on the L1 contract.