# Economics specification

**Authors**: [Ilya Marozau](https://www.linkedin.com/in/imarozau), [Aleksandar Damjanovic](https://t.me/aleksweb3) ([aleksdamjanovicwork@gmail.com](mailto:aleksdamjanovicwork@gmail.com)), [Raluca Diugan](https://x.com/ctzn101)  

# Introduction

Zero-knowledge rollups represent a transformative layer-2 solution for scaling blockchain ecosystems, notably enhancing Ethereum’s capacity by bundling of numerous transactions off-chain while securing them on-chain. zkRollups leverage cryptographic proofs to verify the integrity of off-chain computation, allowing for efficient computation and significantly reducing gas fees.

However, to ensure the long-term sustainability and widespread adoption of zkRollups, a detailed economic framework is essential. This specification delves into the economic mechanics backed by deep competitive analysis, multiple-layer fee structures, and transaction fee model implementation details. By analyzing these aspects, we aim to establish an economically viable model that aligns with the principles of fairness, efficiency, and alignment with Ethereum’s broader ecosystem.

Key considerations in this specification include transaction fee models that adapt to fluctuating demand, cost-efficient proof generation, incentives for node operators, and mechanisms to maintain decentralized verification. Beyond these aspects, we address unique challenges associated with a shared system—a new direction in which the composability of homomorphic, interoperable chains plays a central role in the system architecture.

# Preliminaries

To provide context for our proposed fee solution for zkSharding, we begin with a conceptual overview of key mechanisms. These preliminaries cover the Transaction Fee Mechanism (TFM) definition, First-Price Auctions (FPA), EIP-1559, and EIP-4844, each of which lays the groundwork for understanding the economic and technical motivations behind our approach.

### **Transaction Fee Mechanism (TFM)**

A Transaction Fee Mechanism (TFM) is a protocol component responsible for determining which transactions are included in a block, setting the fees required from transaction creators, and deciding how these fees are distributed [1]. In essence, a TFM ensures that network resources, such as block space, are used efficiently and fairly. When designing a TFM, it is crucial for mechanism designers to consider potential strategic manipulations and ways to mitigate them (this will be discussed in detail in the L2 fee model chapter). Typically, a TFM is represented as a triple (x, p, q), where x is a feasible allocation rule, p is a payment rule, and q is a burning rule [2]:

**Allocation Rule (x)**

The allocation rule determines which pending transactions from the mempool are included in the current block, ensuring that the total size of selected transactions does not exceed the block’s maximum capacity. 

**Payment Rule (p)**

The payment rule determines the fee each transaction creator pays to the block’s miner, based on the transaction’s inclusion in the current block and the chain’s transaction history. This fee is measured in the native currency, per unit of transaction size.

**Burning Rule (q)**

The burning rule specifies the portion of the transaction fee that is permanently removed from circulation. This “burned” amount acts as a deflationary mechanism, similar to stock buybacks, benefiting holders by reducing the total currency supply. Alternatively, this burned amount can be redirected to other beneficiaries, such as a foundation or future miners. 

### **First-Price Auctions (FPA)**

In blockchain networks, first-price auctions (FPAs) are among the simplest and most widely used TFMs. In an FPA, users bid on transaction fees by specifying the maximum amount they are willing to pay to have their transaction included in the next block. Miners, or validators, then prioritize transactions based on these bids, selecting those with the highest fees to maximize their revenue. Notable examples of networks that use FPA incl ude Bitcoin and Ethereum (prior to the implementation of EIP-1559).

### EIP-1559

Our TFM is heavily inspired by EIP-1559, adopting many of its core properties and benefiting from its extensive implementation across other layer 2 solutions (as discussed further in the competitor analysis section). Consequently, we will examine this TFM in greater detail.

The original EIP-1559 base fee adjustment formula is as follows[3]:

$$
\text{baseFee} := \text{baseFee}_{\text{previous}} \times \left( 1 + \frac{1}{8} \times \frac{\text{gasUsed}_{\text{previous}} - \text{gasTarget}}{\text{gasTarget}} \right)
$$

Where:

$\text{baseFee}$  - current block’s base fee

$\text{baseFee}_{\text{previous}}$ - previous block’s base fee

$\frac{1}{8}$ - the adjustment parameter. At 100% block fulness the base fee will increase by 12.5%

$\text{gasUsed}_{\text{previous}}$ - total gas used in previous block

$\text{gasTarget}$ - target gas used

Key Ideas of this TFM are:

| Number | Description |
| --- | --- |
| 1. | Each block has a protocol-determined base fee (per unit of gas) that serves as a reserve price, and paying it is required for transaction inclusion. The base fee is based solely on previous blocks and does not depend on the transactions in the current block. |
| 2. | Base fee revenue is burned and permanently removed from circulation, providing no direct revenue to miners. |
| 3. | A target block size (in gas) is introduced to measure demand, adjusting the base fee up or down based on whether the latest block size exceeds or falls below this target. |
| 4. | Transactions include a tip and a fee cap; a transaction is only included if its fee cap meets or exceeds the block’s base fee. |
| 5. | If a transaction with tip δ, fee cap c, and gas limit g is included in a block with base fee r, the transaction creator pays *g* multiplied by the smaller of (*r* + *δ*) or *c* ETH. Of this amount, the base fee revenue (i.e., *g* multiplied by *r*) is burned, while the remainder goes to the miner. |

Game Theoretic properties of this TFM are as follows [4]:

| Property | Description | Mechanism in EIP-1559 |
| --- | --- | --- |
| Myopic Miner Incentive Compatibility (MMIC) | Ensures that miners, acting in their short-term self-interest, are incentivized to follow the protocol’s allocation rule without introducing fake transactions and strategic manipulation.  | EIP-1559 is MMIC because burning the base fee removes any incentive for miners to manipulate transaction selection to control future fees. This ensures that myopic miners maximize their utility by following the protocol’s rules, rather than introducing fake transactions. |
| Off-Chain Agreement (OCA) Proofness | Prevents miners and users from collaborating off-chain to improve their joint utility beyond what is achievable on-chain. | EIP-1559 is OCA-proof because the base fee burn is determined solely by past blocks and is unaffected by the current actions of miners or users. This mechanism ensures that the allocation rule already maximizes the combined (joint) utility of miners and users on-chain, leaving no room for any off-chain agreement to increase their joint benefit further. |
| User Incentive Compatibility (UIC)  | Ensures that users have a straightforward, optimal bidding strategy that aligns with the protocol’s rules, especially outside of periods of high demand. | EIP-1559 achieves UIC by encouraging users to set their fee cap based on their transaction’s value and include a small tip for miners. This design simplifies honest bidding, as the base fee adjusts automatically with demand. Under normal conditions, users can expect predictable costs, though during high-demand periods UIC may weaken, requiring users to adjust their bids more actively to ensure timely inclusion. |

### EIP-4844

EIP-4844 introduces blob-carrying transactions, a new transaction type optimized for Layer 2 rollups to post data efficiently on Ethereum. Unlike calldata, the data in blob-carrying transactions is stored temporarily and then pruned, significantly reducing storage costs—by as much as 98% compared to traditional methods. This temporary storage allows rollups to handle data availability needs at a far lower cost.

To regulate fees, EIP-4844 implements a blob base fee that adjusts dynamically based on network demand, similar to EIP-1559’s gas fee model. This dedicated blob fee structure keeps costs predictable and separate from standard transaction fees, enabling rollups to scale sustainably. It is worth noting, however, that these transactions still incur some regular gas fees on Layer 1. By leveraging blob-carrying transactions from the outset and data compression, zkSharding can achieve substantial cost savings and improved scalability, making transactions far more affordable for users.

# Scope, Properties & Requirements

We define this phase as "Stage 1" of the economic model with the following scope:

- **Primary Focus** — TFM and core Layer 2 economic design;
- **Decentralized TFM** — the transaction fee model should be designed with the assumption of trustless calculation and transparent estimation assumptions;
- **No Validator Incentives** — as the token utility and cluster decentralization strategy remain undetermined, validator incentives are not included in this stage;
- **No Token Utility** — token utility will be addressed in Stage 2;
- **Payments and Rewards** — all payments, rewards, and treasury allocations will be conducted in ETH.

Overall, the scope can be described as a solution for a centralized/permissioned cluster, fully utilizing a messaging protocol with ETH as the utility token. This aligns well with the goals for the initial version of the current mainnet plan.

The final economic model aims to achieve the following characteristics (properties):

- **Self-sustaining** — operates independently without requiring ongoing intervention from us or the community; must be reliable;
- **Transparent** — clear and predictable processes of calculation of all parameters and fees;
- **Decentralized by design** — TFM shall consider further decentralization and validators behavior;
- **Responsive** — capable of quick reactions to network demands both of internal and external transactions;
- **Cost-effective** — enables cheap and competitive transactions to cost in the assumption that decentralization brings additional overhead;
- **Efficient block space allocation** — optimizes the market of block space within the scope of the whole cluster.

# Comparative analysis

This section examines the transaction fee mechanisms (TFMs) of selected rollups, focusing on their design choices, typical fee levels for an average transaction, and shifts in profitability following the introduction of EIP-4844. By analyzing these elements, we aim to establish a foundation for our proposed fee solution. 

### **Overview of Transaction Fee Mechanisms in Selected Rollups**

**Scroll**

On Scroll (after Curie upgrade) the total transaction fee structure is given by [6]:

$$
\text {totalFee = l2Fee + l1Fee}
$$

Where each component is determined as follows:

**L2 fee.** [7] The L2 fee covers Scroll’s operational costs and consists of: 

- L2 Sequencer Fee: A fixed fee for transaction sequencing. Currently set at 0.001 Gwei.
- Proving Fee: A fixed fee that covers the computational cost of proof generation. Currently set at 0.0382 Gwei.
- Verification fee: A variable fee is used to cover the proof verification transaction costs on L1.  This fee scales with the **L1 Base Fee** and adjusts dynamically to account for base fees changes on L1. The verification fee is given by:

$$

\text{verificationFee} = \frac{\text{l1BaseFee} \times 17}{100000}
$$

L2 fee combines these components and becomes: 

$$

\text{l2BaseFee} = \text{l2SequencerFee} + \text{provingFee} + \text{verificationFee} \\ \text{l2Fee = gasUsed} \times \text{l2BaseFee}
$$

As with other rollups, the base fee on Scroll is not burned like on Ethereum. Additionally, $\text{l2BaseFee}$ is capped at a maximum value of 10 Gwei to prevent excessive costs:

$$

\text{l2BaseFee} = \min(\text{l2BaseFee}, \text{l2BaseFeeMax})
$$

**L1 Fee** [8]. The L1 fee calculation includes a commitment cost and a blob data cost, adjusted by the Curie fork parameters:

$$
\text{L1 Fee} = \frac{\text{commitScalar} \times \text{l1BaseFee} + \text{blobScalar} \times \text{data.length} \times \text{l1BlobBaseFee}}{\text{PRECISION}}
$$

Where:

- $\text{commitScalar}$ (454446963710) -  factor represents the transaction’s commitment cost.
- $\text{l1BaseFee}$ - is the current Layer 1 base fee.
- $\text{blobScalar}$ (378247843) -  scales with the transaction data length  $(\text{data.length})$, weighted by $\text{l1BlobBaseFee}$ (current blob base fee on L1)  to account for DA costs.
- $\text{PRECISION}$ (1e9) -  a scaling factor ensuring the final fee value has the correct units.

As we can see, Scroll employs a distinct approach to calculating transaction fees. Rather than adjusting base fee levels on a block-by-block basis in response to demand, it utilizes fixed, hardcoded costs combined with variable costs that periodically update based on the $\text{l1BaseFee}$. For the L1 portion of the fees it utilizes the same combination of fixed hardcoded scalars and variable costs. It is also important to note that Scroll currently uses a centralized sequencer. 

**Linea**

Linea uses a variation of Ethereum’s EIP-1559 (without base fee burn) with a few modifications [9]. The base fee remains constant at 7 Wei per gas and does not adjust based on network congestion. Instead, adjustments are applied to the priority fee on a block-by-block basis.

The priority fee consists of:

- $\text{fixedCost}$: A set infrastructure cost of 0.03 Gwei per transaction.
- $\text{variableCost}$ which accounts for data submitted to L1 and is calculated as follows:

$\text{variableCost} = \min \left( \max \left( \frac{(\text{avgBaseFee} + \text{avgPriorityFee}) \times \text{blobExecGas} + \text{avgBlobBaseFee} \times \text{blobGas}}{\text{bytesPerSubmission}} \times \text{profitMargin}, \text{minBound} \right), \text{maxBound} \right)$

Where:

- $\text{avgBaseFee}$ - average weighted base fee on L1
- $\text{avgPriorityFee}$ - average weighted priority fee on L1
- $\text{blobExecGas}$ - expected execution gas for blob submission on L1
- $\text{avgBlobBaseFee}$ - average weighted blob base fee on L1
- $\text{blobGas}$ - expected gas usage for blob data handling
- $\text{bytesPerSubmission}$ - conversion factor for per-byte data submission cost
- $\text{profitMargin}$ - multiplier to ensure network sustainability
- $\text{minBound}$ - minimum bound for variable cost to prevent fees from dropping too low
- $\text{minBound}$ - maximum bound for variable cost to cap fees at a reasonable level

This calculation ensures that the transaction fee cover the costs of submitting transaction’s blob data to L1. 

**Priority Fee Determination.** Linea then determines the priority fee as follows: 

$\text{minGasPrice} = \text{variableCost}_{t-1}$

$\text{baseFeePerGas} = \text{vanillaProtocolBaseFee = 7 Wei}$ 

$\text{priorityFeePerGas} = \text{minimumMargin} \times \left( \frac{\text{minGasPrice} \times \text{compressedTxSizeInBytes}}{\text{txGasUsed}} + \text{fixedCost} \right)$

Where:

- $\text{minGasPrice}$  - is the minimum gas price  derived from previous block’s variable cost
- $\text{variableCost}_{t-1}$ - previous block’s variable cost
- $\text{minimumMargin}$ - margin ensuring sustainability (1.2 for RPC method linea_estimateGas)
- $\text{priorityFeePerGas}$ - the calculated priority fee per gas for the transaction
- $\text{compressedTxSizeInBytes}$ - the size of the Layer 2 transaction after compression, measured in bytes
- $\text{txGasUsed}$ - the total gas used by the transaction on Layer 2
- $\text{fixedCost}$ - a fixed cost added to every transaction

In conclusion, Linea’s fee model differs from EIP-1559 by keeping the base fee constant at 7 Wei per gas, rather than allowing it to adjust dynamically with network demand. Instead of modifying the base fee, Linea introduces flexibility through a variable priority fee, which is adjusted on a block-by-block basis to reflect transaction size and recent network costs. This shift places the adaptive element on the priority fee rather than the base fee.

**ZKsync**

We will not cover zkSync’s transaction fee mechanism (TFM) in this section, as it employs a fundamentally different approach from the other rollups discussed and our solution as well. However it is important to mention this as there is no one TFM fits all. 

### Transaction Fee

Below, we outline the target levels of transaction fees our solution should maintain to stay competitive:

| Network | Median Fee (ETH/USD)* | ERC20 Transfer Median Fee (ETH/USD)** |
| --- | --- | --- |
| Scroll | 0.000013/0.0337259 | 0.0000210582/0.0534789853 |
| Linea | 0.000010/0.0255436
 | 0.0000331905/ 0.0844282881 |
| ZKsync | 0.000006/0.0152882 | 0.0000077257/ 0.0195353459
 |

At transaction fees below 10 cents, demand is generally price inelastic; small price fluctuations tend not to impact user behavior significantly. However, high-activity users may still be sensitive to minor fee differences, so maintaining a competitive fee structure is advantageous. The ultimate goal is to keep our transaction fees at or below those of competing networks.

For a detailed overview and comprehensive review of all Transaction Fee Mechanisms (TFMs), visit: [**Market analysis of Economic Models**](/economics/market-analysis.md). Generally, in most cases—such as Scroll, Linea, Optimism, and Arbitrum (with some changes)—the transaction fee model follows an EIP-1559 variation where the base fee is not burned. Instead, base fees are allocated to the sequencer, contributing to rollup revenue.

**Median fees include both L2 and L1 components for each rollup in October 2024. The USD equivalent is based on the daily average ETH price on each transaction day.*

*** ERC20 transfer fees encompass both simple and more complex transfer events across observed rollups.*

# High-level design

The proposed Transaction Fee Mechanism (TFM) consists of these key components:

- **Shard-Specific Base Fees**: Each shard maintains its own base fee to control gas demand and encourage load distribution across the network.
- **Adjustment mechanism**: A modified EIP-1559 mechanism with a faster adjustment response during periods of high congestion, enabling individual shards to adapt base fees dynamically.
- **Message Premium**: A “take it or leave it” premium, inspired by EIP-1559, to manage message congestion. This premium directly rewards validators, incentivizing them to prioritize message inclusion.
- **Message base fee discount**: To make message sending cost-effective for users, a discount on the message base fee is applied compared to the actual base fee.
- **Limited Transaction Gas Space**: The transaction gas space is limited and dynamically adjusted based on message gas usage. This incentivizes validators to prioritize message (CST) inclusion, as it increases their potential for tip-based rewards and grants them additional message premiums. For message gas there is only the $totalGas$ (block gas limit). This constraint is defined as:
    - A limitation on maximum $\text{txGas}$ in the block by tying it to the $\text{mGas}$ the following way:

$$

\text{txGasMax} = \frac{\text{targetGas}}{2} + \text{mGas} \\\text{mGasMax} = \text{totalGas}
$$

- **EMA-Based Smoothing:** The model uses an Exponential Moving Average (EMA) [10] to smooth out spikes in L1 blob base fees. By dynamically adjusting the EMA window based on current fee levels, it spreads the impact of rare but extreme fee spikes over time, preventing sudden cost increases in the data availability (DA) portion of fees for users and eliminating the need for DA adjustments, such as delays.
- **Transformation of L1 Fees into Gas**: L1 portion of the fee (DA, Verification, DA Commitment) is converted into gas units, following an approach inspired by Arbitrum. Since the L1 fees are transformed into gas the fees for the end users can look like:
    - For intra-shard transactions: $\text{{fee}} = \min(\text{{Max Fee}}, \text{{Base Fee}} + \text{{Priority Fee}}) \times \text{{gasUsed}}$
    - For cross-shard transactions (messages): $\text{mFee} = \min({\text{Max Fee}},\text{{mBaseFee}} + \text{{mPremium}}) \times \text{{gasUsed}}$.
    
    This aligns with Ethereum’s gas pricing mechanism and ensures compatibility with existing tools and libraries. 
    
    The L1 fees incurred by a transaction are transformed into gas the following way: 
    
    $$
    
    \text{l1GasUsed} = \frac{\text{l1CommitmentFee} + \text{l1blobFee} + \text{proofVerificationFee}}{\text{baseFee}} \times \text{offset}
    $$
    
    $\text{{gasUsed}}$ then becomes a um of $\text{l1GasUsed}$  and $\text{l2GasUsed}$ .  $\text{l1GasUsed}$   in a block is not taken into consideration in the base fee adjustment algorithm,  nor do the tips go to validators for it. This gas used is just multiplied by the base fee and ignores priority fees (tips). 
    

In the following sections, we will explain the formula behind the proposed TFM for zkSharding.  

# Local Fee Model

As one of the main goals of the base fee is to balance gas demand via it’s price adjustment algorithm the local fee model is chosen as a best fitting model for zkSharding. Before going over the details, it is important to understand what makes the fee model ideal according to Tim Roughgarden (and a classic supply demand equilibrium concept from economics):

“Every block is fully utilized by the highest-value transactions, with all transactions paying a gas price equal to the market-clearing price.” [x] 

Since the goal of our base fee adjustment algorithm is to find (aim for) the market clearing gas price for the next block based on the demand for gas of the previous one,  we can have two situations:

- Shared fee model where the system-wide base fee levels adjusts in order to find the market clearing price for the ideal gas capacity of the whole system
- Local fee model where each shard has its own level of base fee and aims to find the market clearing price for its demand levels at any point in time

Even though the shared fee model simplifies the transaction cost estimation, it can create issues such as:

- Available block space is not optimally used across the network
- Slow price response to demand spikes on individual shards as the gas price adjustment algorithm takes into account the total demand across all shards. This can make certain shards have prolonged periods of high congestion then when employing a local fee model.
- Uniform gas pricing removes the economic signals that would normally guide users to underutilized shards, reducing the system’s ability to self-balance based on supply and demand. Some systems address this through algorithmic shard management (e.g., split/merge conditions) to balance load, though this approach operates differently from a market-based system and increases the overall complexity of the system.
- Developers and users may not be incentivized to optimize for specific shard conditions, as there is no differentiation in pricing that rewards efficient shard usage.

Considering these issues **zkSharding employs a local fee model**. It brings benefits such as: 

- Each shard adjusts its gas price based on its own demand, ensuring that block space is used efficiently
- Efficient distribution:
    - Applications aiming for cost efficiency are incentivized to deploy their contracts on less congested shards
    - Users seeking cost efficiency are incentivized to interact with the same dApp on less congested shards, reducing the load on busier ones.
- By aiming for a target gas usage in each shard’s block, we maintain system-wide congestion balance. Local fee adjustments respond to both higher and lower activity, ensuring each shard stays balanced and gas usage aims for the target, promoting overall network efficiency.

However this choice comes with a few tradeoffs in order to aim for economic efficiency: 

- Coordinating cross-shard transactions is more complex due to the need to estimate and synchronize gas prices between shards, adding overhead to the process.
- Lower prices on less congested shards may attract more users, potentially leading to congestion in previously underutilized shards, shifting bottlenecks rather than resolving them. This could be mitigated by smart wallets that dynamically route transactions to the most optimal shards based on real-time congestion and gas price data, ensuring an even distribution of traffic across the system.

[Shared vs Local Transaction Fee Pricing Model](/economics/Shared%20vs%20Local%20Transaction%20Fee%20Pricing%20Model.md) 

[Blogpost: Sharding Fee Models: Local vs Global Fee Market](/blogposts/Sharding%20Fee%20Models:%20Local%20vs%20Global%20Fee%20Market.md) 

# L2 fee model

To comprehensively cover the L2 portion of the fees in this specification, we focus on two main components:

- Intra-Shard Transactions: Base Fee and Tips
- Cross-Shard Transactions (messages): mBase Fee and Premium (dynamically adjusted tip)

### Intra-Shard Transactions: Base Fee and Tips

In order to maintain the same benefits as EIP-1559 while fulfilling the goals of the model we introduce changes that yield the following model for our L2 portion of the fees: 

**Base Fee**

$$
\text{baseFee} := \text{baseFee}_{\text{previous}} \times \left(1 +\frac{1}{101.38} \times \frac{\text{sgn}(
\frac{\text{gasUsed}_{previous}}{\text{gasTarget}} - 1
)}{1 + e^{-11 \times (0.9|
\frac{\text{gasUsed}_{previous}}{\text{gasTarget}} - 1
| - 0.45)}}\right)
$$

Where:

- $\text{baseFee}$  - current shard’s block’s base fee
- $\text{baseFee}_{\text{previous}}$ - previous shard’s block’s base fee
- $\frac{1}{101.38}$ - the adjustment parameter. meaning that at 100% block fullness, the base fee increases by 0.986% per block. This value was chosen so that, after 12 consecutive seconds of full 1-second blocks, the base fee would rise by approximately 12.5%.
- $\text{gasUsed}_{\text{previous}}$* - total gas used in previous block
- $\text{gasTarget}$* - target  gas used
- 0.45 - offset value which shifts the point at which the adjustment kicks in
- $e^{-11}$ - is used to steepen the adjustment function

*For base fee adjustment algorithm, only L2Gas is taken into account 

Upon the first look the formula looks a bit different and complex, as instead of the simple $\frac{\text{gasUsed}_{\text{previous}} - \text{gasTarget}}{\text{gasTarget}}$ we include a more complex response calculation. This is in order to fulfill fast responsiveness to congestion and fairness property. Let’s see how the changed portion of the formula ($\frac{\text{sgn}(\frac{\text{gasUsed}_{previous}}{\text{gasTarget}} - 1)}{1 + e^{-11 \times (0.9|\frac{\text{gasUsed}_{previous}}{\text{gasTarget}} - 1| - 0.45)}}$) looks like at various block fullness levels:

![image.png](/figures/economics/economics-specification-00.png)

We observe that the adjustment is more tolerant around the target block fullness, but as levels deviate further from this target, the response accelerates rapidly, fulfilling our primary goal of fairness and responsiveness to congestion. This is best illustrated by comparing our adjustment to Ethereum’s $\frac{\text{gasUsed}_{\text{previous}} - \text{gasTarget}}{\text{gasTarget}}$: 

![image.png](/figures/economics/economics-specification-01.png)

The base fee adjustment algorithm responds more gradually than the classic model within the range of 25% to 75% block fullness, particularly around 50%. However, beyond these points, the response accelerates significantly. 

Our base fee adjustment algorithm also ensures resistance to strategic manipulation by **not** redistributing any portion of the base fees to validators, similar to the design of EIP-1559. The UIC, OCA and MMIC criterion are fulfilled the same as with the original formula. [11]

**Priority Fee (Tips)** 

The priority fees (tips) function as in Ethereum’s classic transaction fee mechanism. The rollup does not take a cut from the priority fees; all tips are directed to the validator as a reward for performed work. 

### Cross-Shard Transactions (messages)

**mBase Fee**

Message base fee ($\text{mBaseFee}$) is calculated by applying a fixed discount of 10% to the current base fee. Thus making:

$$
\text{mBaseFee} = 0.9 \times \text{baseFee}
$$

This discount enables the cross-shard transaction premium to find its natural market price. Keeping the message base fee $\text{mBaseFee}$ linked to the base fee $(\text{baseFee})$ is essential, as both fees utilize the same resource. Disconnecting them would compromise the intended function of the base fee structure.

**Premium**

The message premium is an integral part of the L2 model. It serves as both a take it or leave it premium for message senders and as an incentive for validators to include the messages. It is calculated based on the previous blocks message to total gas ratio $mRatio$. As with the base fees, the message premium will also have its minimum price below which it cannot drop in order to at least cover resources spent by the validator in order to process the message. 

$$

\text{mPremium} := \text{mPremium}_{\text{previous}} \cdot \left( 1 + \frac{1}{\text{101.38}} \cdot \frac{\text{sgn}(\text{mRatio} - \text{idealRatio})}{1 + e^{-S(\text{mRatio}) \cdot \left( |2 \cdot (\text{mRatio} - \text{idealRatio})| - W(\text{mRatio}) \right)}} \right)
$$

Where: 

- $\text{idealRatio}$ = 0.33.
- $\text{mRatio} := \frac{\text{mGas}_{\text{previous}}}{\text{gasUsed}_{\text{previous}}}$
 - message to total gas ratio in the previous block. If the previous block was full with messages the ratio is 1.
- $\sigma(\text{mRatio}) = \frac{1}{1 + e^{-50 \cdot (\text{mRatio} - 0.33)}}$
- $S(\text{mRatio}) = 25 \cdot (1 - \sigma(\text{mRatio})) + 10 \cdot \sigma(\text{mRatio})$   is the steepness modifier
- $W(\text{mRatio}) = 0.35 \cdot (1 - \sigma(\text{mRatio})) + 0.45 \cdot \sigma(\text{mRatio})$   width modifier, it controls the width of the range where the function transitions from being steep to flat
- $\sigma(\text{mRatio}) = \frac{1}{1 + e^{-50 \cdot (\text{mRatio} - idealRatio)}}$
- $\frac{1}{101.38}$ - the adjustment parameter. At 100% message ratio the message premium will increase by 0.986 % at 0% premium will decrease by the same percentage

Let’s see how the changed portion of the formula ($\frac{\text{sgn} (\text{mRatio} - 0.33)}{1 + e^{-S(\text{mRatio}) \cdot \left( |2 \cdot (\text{mRatio} - 0.33)| - W(\text{mRatio}) \right)}}$) looks like at various message ratio levels:

![image.png](/figures/economics/economics-specification-02.png)

We can see that the Message Premium Adjustment is also tolerant near the ideal message to total gas ratio, but as levels deviate further from this target, especially on the positive side,  the response accelerates rapidly. Additionally, the maximum adjustment remains capped which makes the fee level predictable, as it retains nearly all the properties of our Base Fee adjustment algorithm. 

# DA fee model

The Data Availability fee consists of:

- $\text{l1CommitmentFee}$ - this portion of the fee is used to cover the gas cost of submitting the data availability (DA) transaction to L1
- $\text{blobFee}$ - this fee represents the cost of storing the transaction data in blobs on L1

### L1 Commitment Fee

This portion of the DA fee model is defined as follows: 

$$

\text{l1CommitmentFee} = \frac{\text{l1BaseFee} \times \text{expectedCommitmentGas}}{\text{expectedNumberOfTxsInBlob}} \times \frac{\text{compressionRatio} \times \text{dataSizeOfTransaction (in bytes)}}{\text{averageDataSizeOfTransaction (in bytes)}}
$$

Where:

- $\text{l1BaseFee}$ - last known baseFee on Ethereum by the system.
- $\text{ExpectedCommitmentGas}$ - expected commitment gas spent by the **DA** transaction
- $\text{expectedNumberOfTxsInBlob}$ - expected number of transactions that are going to be in the batch in total when submitted via blob. For the expected number, initially we set this as a somewhat lower number as the initial demand for our solution will be lower than regular. Once we know how much actual data of transactions our solution usually stores in the blob, we will adjust.
- $\text{dataSizeOfTransaction (in\ bytes)}$ - size of the transaction (in bytes)
- $\text{averageDataSizeOfTransaction (in\ bytes)}$ - average size of transaction we can expect. Initial value will be set at 293, as we expect that the =nil; pruned average transaction will be of that size.
- $\frac{\text{CR(dataSizeOfTransaction (in bytes))}}{\text{averageDataSizeOfTransaction (in bytes)}}$- is used to estimate how much of this portion of the commit fees should the individual user pay. CR — compression plus pruning ratio. We use the ratio between the size of the transaction and average size of the transaction to determine how much of the assigned commit part should the individual transactor pay. For double the size of the average transaction the transactor would pay double the amount of his commit fee allocation. Even though the commit gas usage is not the result of the individual transactions data size, we make it so, as larger transactions fill up a blob quicker.

### L1 Blob Fee

This portion of the DA fee model is defined as follows: 

$$

\text{l1BlobFee} = \text{compressionRatio} \times \text{dataSize (in bytes)} \times \text{blobBaseFeeSmoothed}
$$

Where: 

- $\text{compressionRatio}$ - expected data compression ratio
- $\text{dataSize(in\ bytes)}$ - size of the transaction in bytes
- $\text{blobBaseFeeSmoothed}$ - smoothed blob fee as a result of our EMA mechanism.

Blob base fee volatility is solved by introducing the Exponential smoothing formula in order to offload the blob base fee spikes on Ethereum:

$$
\text{blobBaseFeeSmoothed}_t = \alpha x_t + (1 - \alpha) \cdot \text{blobBaseFeeSmoothed}_{t-1}
$$

Where:

- $\alpha$ is the smoothing factor,  more on that in the next paragraph
- $x_t$  is the current observation (current blob base fee on ethereum)
- $\text{blobBaseFeeSmoothed}_{t-1}$ is the smoothed blob base fee from the previous ethereum block

There is no single ‘correct’ procedure for choosing $\alpha$; however, we select an $\alpha$ value that sufficiently mitigates the effects of extreme growth in the blob base fee over a longer period, while still remaining responsive to shorter-term fluctuations. Although more complex models involving additional smoothing factors (such as double or triple exponential smoothing) could be employed, we opt for the simplest approach.

Smoothing factor $\alpha$ calculation:

$$

\alpha = \frac{2}{\text{W}_t + 1}
$$

Where: $W$ - current window size

Adaptive response to sudden spikes - adjusting window size:

A standard EMA with a fixed window size may struggle to effectively smooth out extreme spikes, which can result in fee estimates that either overreact or underreact to sudden changes. To mitigate this issue, we introduced a dynamic window size. By adjusting the window size in real-time, the model ensures that during periods of elevated fees, the blob base fee becomes less sensitive to sudden spikes, thus distributing the impact over a longer period. When a fee spike is detected—specifically, when the current fee exceeds the threshold of blob base fee of 10 GWEI —the model increases the window size to a maximum value $W_{max}$ (set at 50 times the regular window size $W_{reg}$). Following this, the window size is gradually reduced as conditions stabilize (less DA transaction posting as a result of the increased blob fees)  

**After** window increases to the maximum value (500) the window is then calculated as:

$$

W_t = \max\left(W_{reg}, W_{t-1} - \frac{W_{max} - W_{reg}}{\text{Decay Duration}}\right)
$$

- $Wt$: This is the window size at the current block t. It dynamically adjusts based on the current state of the blob base fee on Ethereum.
- $W_{reg}$= 10: The regular window size. This is the baseline value used during stable periods when there are no significant spikes in the blob base fee.
- $W_{max}$ = 500: The maximum window size, set to 50 times the regular window size $W_{reg}$. This is applied when a significant spike in the blob base fee is detected.
- $W_{t-1}$: The window size from the previous block t-1, used to calculate the current window size W_t.
- ${Decay Duration}$: The number of blocks over which the window size is gradually reduced. It depends on the current blob base fee:
    - 600 Ethereum blocks if the blob base fee  $\text{blob\_base\_fee}_t > 10\ GWEI$ , implying a slower reduction in window size during high fee periods. We chose 10 gwei as a threshold as this increase compared to the usual blob base fee of 1 wei is a significant spike already.
    - 300 Ethereum blocks if the blob base fee  $\text{blob\_base\_fee}_t \leq 10\ GWEI$ , implying a faster reduction in window size when fees are lower.
    
    The Decay duration is set such that if the fees quickly reduce it will return the window to the regular value quicker in order to be more responsive to fee changes. 
    

# Proof generation fee model

Similar to other approaches in the market[12], proof generation fees are managed through a parameter called $\text{minBaseFee}$, which includes the following components: 

- $\text{operationCosts}$ - minimum fee in order to cover operational cost of the rollup.  Currently set as 0.01 gwei.
- $\text{proofGenerationCosts}$ - minimum fee in order to cover proof generation.  Currently set as 0.01 gwei.

Minimum base fee $\text{minBaseFee}$ then consists of:

$$
\text{minBaseFee}= \text{operationCosts} + \text{proofGenerationCosts} 
$$

If we take minimum base fee into account base fee then becomes: 

$$

\text{baseFee} = \max(\text{minBaseFee}, \text{baseFee})
$$

This setup may be adjusted as the proving architecture is fully established and more detailed information becomes available. Future improvements include dynamically adjusting the proof generation costs dynamically based on fiat costs of provers etc. 

Charging: [Pricing Proving/Proof Generation Charging](/economics/Pricing%20Proving%20and%20Proof%20Generation%20Charging.md) 

# Proof verification fee model & Offset

### Proof verification

Proof verification costs are calculated using a formula which, based on the L2 gas allocates the amount of fees that is to be paid for proof verification. These expected values are to be set based on benchmarking/testnet data and are to  be recalculated based on historical data. The formula is as follows: 

$\text{L1VerificationFee} =  L1BaseFee \times \text{ExpectedVerificationGas} \times \frac{\text{L2GasUsed}}{\text{ExpectedTotalL2GasInProof}}$

Where: 

- $\text{L1BaseFee}$ -  the last known base fee on Ethereum by the rollup
- $\text{ExpectedVerificationGas}$ - the expected amount of gas required to submit a proof verification transaction on L1.
- $\text{L2GasUsed}$ - the L2 gas used by the transaction
- $\text{ExpectedTotalL2GasInProof}$ - total l2 gas that can be verified. Estimated based on verification capacity

### Offset

The offset is a scaling factor applied to ensure that collected fees can consistently cover costs. This factor is recalculated with each proof verification transaction (e.g., hourly) and adjusts based on the cumulative fees collected and costs incurred over the past 24 hours. The offset is defined as:

$\text{offset} =\begin{cases}1.1, & \text{if } \frac{\text{totalCollected}}{\text{totalCosts}} \geq 1 \\1.1 + 0.4 \times \left(1 - \frac{\text{totalCollected}}{\text{totalCosts}}\right), & \text{if } \frac{\text{totalCollected}}{\text{totalCosts}} < 1 \\\end{cases}$

Where: 

- $\text{totalCollected}$ - total collected L1 fees by the rollup
- $\text{totalCosts}$ - total costs incurred by the rollup during a 24 hour period

This default value of 1.1 ensures that priority fees on L1 are adequately covered. If cumulative collections fall short of costs, the offset scales up to maintain cost coverage, with a maximum value of 1.5.

# Decentralization & Implementation

A decentralized fee model must be built with transparency and resilience against manipulation. Decentralized validator networks depend on predictable and transparent fee structures to uphold network parity and balance, ensuring that no manipulation can disrupt operations.

These attributes are supported by a detailed examination of the mechanics behind TFM design, which we know includes the following key aspects. Example of breakdown of the components. Note that the actual structure is different (e.g. verification and generation is part of the base fee):

$FeeUsed = L_1BlobFee + L_2BaseFee + L_2Tips + ProofGeneration + ProofVerification$

From that, we can derive the number of constraints and observe the sustainable and transparent calculation. Below, we present a detailed formalization of the constraints:

**L1 Blob Fee**

$Fee = E_{cr}[sozeof(Prune(Tx))] \times bf(settelmentBlock.blobBaseFee)$

$E_{cr}$  — expected compression ratio 

$Tx$ — raw transaction in bytes

$bf$ — blob fee function (see blob doc for more details)

**L1 Verification Fee**

$Fee = \frac{E[Gas_{verification}]}{E[Gas_{used}]} \times vf(settelmentBlock.blockBaseFee)$

$vf$ — verification fee function (see verification doc for more details)

**L2 Base fee**

$Fee = baseFeeTransitionFunction(baseFee_{BlockNum - 1})$ 

(see L2 fee model doc for more details)

**L2 tips**

$Fee_{externalTransaction} = Tx_{userPriority}$

$Fee_{cst} = adjustmentFeeTransitionFunction(adjustmentFee_{BlockNum - 1})$

(see L2 fee model doc for more details)

**Proof Generation**

$Fee = UserTXOpcodesDistribution \times \Delta_{opcodesGasForProof} \times baseFeeTransitionFunction(baseFee_{BlockNum - 1})$

In this paper, we demonstrate how transparency and decentralized operation are achieved within the TFM. Although some areas, such as L1 chain data availability, still require further investigation, these challenges do not compromise the core mechanics.

Our approach integrates decentralization tightly with the economic model itself. The model incorporates several pioneering innovations, making its accurate implementation on the validator and RPC nodes essential to ensure secure, deterministic network operation and an optimal user experience.

# Transaction cost estimation

We again refer to the previous breakdown: 

$FeeUsed = L_1BlobFee + L_2BaseFee + L_2Tips + ProofGeneration + ProofVerification$

*We refer to ERC-20 transactions here.* 

**L2 Tips** — Not considered here due to significant variability and lack of protocol-specific consistency.

**L2 Base Fee** — We don’t accept significant deviations from other L2 protocols, except where our higher average demand warrants a response. The average base fee from other L2s is around 0.06 Gwei, with ERC-20 transfers requiring ~75k gas.

We also include SC operations here, which may add additional costs in the amount of 0.02 Gwei. The amount is roughly estimated from the idea mentioned in the Treasury document, with the assumption of 10 network peers. 

So the total base fee is 0.08 Gwei.

**L1 Blob Fee** — Calculated based on transaction size, adjusted for compression and pruning ratios, which, in the case of our protocol, can be ~3 (see DA spec). With an average transaction size of 283 bytes, the compressed size is approximately 94 bytes. The average blob fee is ~0.5 Gwei, assuming 1 byte = 1 gas, as per EIP-4844.

**Proof Verification** — We estimate an average verification execution gas ratio of ~1:200. That is a more or less competitive number from other ZK-Rollups. Based on the proposal for proof verification, the typical adjustment might add around 20%.

**Proof Generation** — Similar to Proof Verification, though with a smaller adjustment of 5% due to improved predictability. While no projects disclose exact figures, we estimate that proof generation accounts for approximately 35% of transaction costs (took lower than in [Pricing Proving/Proof Generation Charging](/economics/Pricing%20Proving%20and%20Proof%20Generation%20Charging.md) ), translating to roughly $0.027 per transaction.

*The research team has yet to provide more precise data on these estimates.*

*So the final cost would be:*

$fee = ETH_{price} \times (0.08 \times 75000 + 0.5 \times 94 + 75000 / 200 \times 10.5  \times 1.2) + 0.027$

$

That is for 3000$ per ETH, which will be  **~0.0594$** per ERC-20 transaction.

Breakdown by price:

- L2 Base Fee — 0.018$
- L1 Blob Fee — 0.000141$
- Proof Verification — 0.0144$
- Proof Generation — 0.027$

That is very competitive considering our Competitive Analysis work. 

## References

[1]https://arxiv.org/abs/2106.01340v3 Page 10. 

[2]https://arxiv.org/abs/2106.01340v3 Page 11.

[3] https://eips.ethereum.org/EIPS/eip-1559 

[4] https://arxiv.org/abs/2012.00854 page 24.

[5] https://eips.ethereum.org/EIPS/eip-4844

[6]https://docs.scroll.io/en/developers/transaction-fees-on-scroll/ accessed Nov 3 2024

[7]https://github.com/scroll-tech/go-ethereum/blob/develop/consensus/misc/eip1559.go accessed Nov 3 2024

[8] https://scrollscan.com/address/0x5300000000000000000000000000000000000002#code accessed Nov 3 2024

[9] https://docs.linea.build/developers/guides/gas/gas-fees acesseed nov 4 2024.  

[10] [https://www.bajajfinserv.in/exponential-moving-average#:~:text=Computation of the Current EMA,) x (1 - multiplier)]](https://www.bajajfinserv.in/exponential-moving-average#:~:text=Computation%20of%20the%20Current%20EMA,)%20x%20(1%20%2D%20multiplier)%5D) assesed nov 7 2024. 

[11] Validator Strategic Manipulations

[12][Pricing Proving/Proof Generation Charging](/economics/Pricing%20Proving%20and%20Proof%20Generation%20Charging.mds)