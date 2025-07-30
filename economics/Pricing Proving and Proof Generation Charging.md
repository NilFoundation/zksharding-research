# Pricing Proving/Proof Generation Charging

**Authors**: [Raluca Diugan](https://x.com/ctzn101), [Aleksandar Damjanovic](https://t.me/aleksweb3) ([aleksdamjanovicwork@gmail.com](mailto:aleksdamjanovicwork@gmail.com))  

Part of the focus of the work in this document was looking into existing pricing distributions for resources involved in the generation of validity proofs. We looked at several projects spanning from zkRollups to L1 proving layers to understand how they derive proving fees. In doing so, we also concluded that defining a prover fee model requires establishing a prover allocation model which influences the mechanism design for provers. The second part of this work proposes a strawman prover fee model to be later refined based on our prover network design goals, as well as given further research into proof generation costs. 

# State of the art

We looked at how different projects determine or estimate the fees to be paid to provers completing proof generation tasks. We highlight the most relevant findings below.

- Projects such as Aztec (zkRollup), Gevulot (L1 proving layer), or Taiko* (based rollup) all seem to converge towards a similar design, namely a prover network where actors are allocated to proofs based on an allocation mechanism (off-chain auctions, prover competitiveness based on a set of parameters), and where, should the initial assigned prover fail to deliver the proof before a timeout period, the proving task can be permissionlessly undertaken by any other prover in the network. *For Taiko we are still looking at the latest design; these results are primarily based on the design up until alpha-4 testnet.
- In many designs, provers stake to join the network and can be slashed for e.g. not delivering proofs on time.
- Other designs continuously test the accuracy of a prover’s claimed capacity and availability by issuing proving tasks.
- With few exceptions, we could not find useful information about concrete pricing per resource. Exceptions:
    - In two separate sources, proving prices are derived based on the costs of running the VM, rather than individual subprocesses. For example, running a Polygon Type 1 prover in a GCP instance over a given Ethereum block incurs a cost of $0.0022 per transaction (~45min, $0.235 per block). [1]
    - Aligned (not to be confused with Aligned Layer), has estimated the costs of running proving infrastructure with a 20% profit at $0.22 per proof, to decrease to $0.12 by 2030. These costs are based on a $23,640 upfront investment in three medium performance hardware units and a utilization rate of 50% (more than 500k proofs per year). [2]
    - Projects seem to favor native token rewards for provers (Aztec, Gevulot, Taiko, MINA).
    - Some projects provide more details about their fee models, incentive mechanisms, or hardware requirements which help further reason about the proving fee model. Specific details about several projects are available on dedicated pages.
    - Linea, Taiko, and Scroll are implementing/looking to support a multi-prover approach.

We further note that most projects lack concrete details, perhaps because the decentralization of the prover is still early in many projects, or because of ongoing implementation. For example, we expect that with the launch of Gevulot in 2024/2025, more concrete details will be available.

### How zkRollups charge users for proof generation

**Scroll**

![Pie Chart 1: Users’ gas fees cover three primary costs. Source: https://scroll.io/blog/proof-recursion-scrolls-darwin-upgrade](/figures/economics/pricing-prover-00.png)

Pie Chart 1: Users’ gas fees cover three primary costs. Source: https://scroll.io/blog/proof-recursion-scrolls-darwin-upgrade

**Proving Fee**: A fixed fee of 0.0382 Gwei is added for proving purposes (our assumption is that this is for proof generation)  A sequencer fee of 0.01 gwei is also fixed and added on top of each transaction. [6]

$\text{VerificationFee} = \frac{17 \times \text{BaseFee}_{\text{L1}}}{100000}$

Simple transfer proof generation costs would then be = 21000 * 0.0382 GWEI = $0.00207879701

We can see that the values for Proving are hardcoded, how. they decided on those values remains unknown, nor we got the response from the Scroll team. 

**Zksync**

In zkSync, proof generation and verification are covered by the “batch” overhead and as of current understanding are not separated explicitly. The batch is sealed based on specific conditions: time, number of transactions, bootloader memory, transaction encoding memory usage, and pubdata bytes. A user’s fee for proof generation depends on their transaction’s resource consumption, such as computational gas, memory, and pubdata. Each resource has a predefined gas cost, and the total fee is proportional to the transaction’s contribution to the overall batch overhead. The more resources a transaction consumes, the higher its share of the proof generation cost. Users are charged upfront for worst-case resource usage, with any unused gas refunded after processing. We can conclude that the ZkSync fee model approach is not comparable to our fee model approach.  [7]

No detailed information was found in Zksync’s Github repo code. [8]

**Linea**

In Linea, the base fee is fixed at 7 Wei, while the priority fee consists of: 

- A fixed cost of 0.03 Gwei
- A variable cost based on transaction size and complexity, including blob data submission and L1 verification costs, with a profit margin applied to ensure profitability.

Based on the Discord response from the Linea team member we can conclude that the proof generation is covered currently from the fixed cost portion of the priority fee. We could not get additional information. 

No information on the proof generation cost distribution can be found at the moment in the Linea monorepo (source:https://github.com/Consensys/linea-monorepo), which is not unexpected as Linea is currently in Phase 1 of its decentralization roadmap. This phase focuses on open-sourcing the stack and expanding EVM coverage. In the future we can expect more detailed information on proof generation costs to be made public (source:  https://docs.linea.build/architecture/overview/decentralization-roadmap) 

**Starknet**

The user’s fee for proof generation on Starknet is based on the transaction’s resource usage, such as Cairo steps and applications of builtins like Pedersen hashes. Each resource has a predefined gas cost, and the fee is proportional to the transaction’s contribution to the total proof cost. The more resources a transaction consumes, the higher its share of the overall proof generation cost. [8]

**Aztec.** Aztec uses a protocol Sidecar for allocating proofs to provers whereby sequencers auction off proving rights to provers (or undertake the proving task themselves). This auction happens off-chain. Provers register by depositing a collateral for the task. In case no proof is provided in the allocated time, any prover can attempt to do so. Provers are rewarded in native tokens. Aztec community has ongoing discussions about proving rewards [10].

**Gevulot.** Gevulot is an L1 network of ZKP provers and verifiers where users submit proving requests, requests are allocated to provers by the protocol, and provers accept or reject these assignments (up to a maximum allowed number of times). Provers receive a user fee (minus rebate) and a reward in native tokens. The faster the proof generation takes place, the higher amount of the user fee the prover receives. 

**Mina.** Mina block producers must pruchase SNARKs from the snarketplace and share part of the block reward with the SNARK workers. Average SNARK fee according to information on the Minascan block explorer is 0.001071182MINA. Example of [MINA Snark Fee](https://minascan.io/mainnet/block/3NLY9PVaihbS1xQjUya8Ch1rALPaULpKgd2Vrmxd8oAggyXjGZ7x/txs). The snarkers prices are available here: https://minascan.io/mainnet/snarkers.

## Resource Pricing Considerations

There are several types of resources that are utilized in generating a proof and that could theoretically be priced. A non-exhaustive list is available below:

- **hardware acquisition** (varies based on hardware requirements, realistically starting from $1000s)
- **infrastructure setup** (hardware and network configurations)
- **effective proof generation costs**:
    - CPU cycles (execution (modifying registries, memory operations), syscalls, context switching). Note: in case we further derive proving costs per opcodes, these are the processes we consider.
    - Strongly correlated with the hardware requirements? Estimation can be done given the proof generation time in a VM and the rental costs of the VM. [1], [2]
- **network costs** (negligible even in a federated prover setting)
- **electricity costs**
    - performance per Watt (FPGAs similar to GPUs [3])
- **storage costs** (negligible)
- **L1 proof commit costs** (non-negligible, but can be undertaken by L2 actors as part of the L2 state update)

We list several concrete examples of hardware and infrastructure requirements encountered during our research: 

- 16—128 CPUs/vCPUs ([1], Succint, Mina)
- 32—1024 GB RAM ([1], Succint, Mina)
- 1-3 GPUs [1, 2]
- Mina 1 Mbps (accessible across the world [5])
- 64GB (Mina)

Aztec not included as their estimates are for full nodes which are not required for provers.

**Note:** For increased resilience, we should aim to incentivize running non-AWS/GCP servers to avoid hardware provider centralization. This can be achieved through optimization such that running the proving software can be done on average machines (e.g. 32GB RAM, 16 CPUs), or through a reward/grants program to offset initial acquisition costs.

The pricing may additionally be a function of demand — such that priority fees are included in the prover fee based on network congestion.

### Charging considerations

Currently the the projects are using native tokens to reward provers (Aztec, Gevulot, Taiko), however the prover’s cost structure mainly consists of the costs that are denoted in fiat (hardware, electricity, prover amortization by use). This means that the return of investment (ROI) is heavily influenced by the reward token price. 

1) As we covered in the previous section the prover’s incur certain setup costs, ongoing costs of operating. We define them in the following simplified way:

$\text{setupCosts}$ 

$\text{effectiveCosts}$

2) Generating a single proof then burdens the following costs: 

$\text{cProof} = \frac{\text{setupCosts}}{n} + \text{effectiveCosts}$

Where  n  is the expected total number of proofs generated during the prover’s hardware lifespan.

3) That means that the prover would prefer to earn at least some margin ($\mu$) over the $\text{cProof}$. We also must take into account the total price and total capacity for proving: 

$\text{generationFeePerGas} = \frac{\text cProof (1 + \mu)}{{gTotal} \times \text{tokenPrice}}$

In this example this generationFeePerGas is set by the protocol on daily basis. Margin % is set based on benchmarking, and can be adjusted in the future. 

4) The user is then charged: 

$\text{generationFeeCharged} = \text{gUsed} \cdot \text{generationFeePerGas}$

5) Prover profit after processing user’s transaction then becomes: 

$\text{fiatProfit} = \text{generationFeeCharged} \cdot \text{tokenPrice} - cProof \cdot \frac{gUsed}{gTotal}$

Where: 

| Parameter/Variable | Description |
| --- | --- |
| setupCosts | Initial hardware and infrastructure setup costs for operating the prover |
| effectiveCosts | Ongoing costs for proving one batch (e.g., electricity, maintenance) |
| cProof | Total cost to generate one proof |
| setupCosts/ n  | amortized hardware cost per proof where n is the number of proofs generated over lifespan |
| gTotal |  total gas capacity of proven batch , or total gas in proven batch |
| gUsed | user transaction gas used (this is multiplied by the verification gas price)  |
| generationFeePerGas  |  how much the user will be charged per gas unit for proof generation |
| mu | profit margin |
| tokenPrice | price of reward token in fiat |
| generationFeeCharged | actual fee charged from the user |
| fiatProfit | prover’s fiat profit for a specific user transaction |
| cProof * gUsed/gTotal | actual cost incurred for the user’s transaction  |

There are two approaches to ensuring prover participation (proving)  in the system:

- At the moment of proving $fiatProfit > 0$
- At the moment of proving $fiatProfit \leq 0$  if the prover expects the reward token price to appreciate

However as we seen in Aztec, Gevulot, Taiko the provers are content with getting rewards in the reward token without taking into account the price of said tokens at the time of proving. This begets the question: **Does our solution need to continuosly adjust this proof generation fee based on fiat price?** 

## Proving Model

At the moment, there is no set design for the zkSharding proving model. Components of these prover model we will have to define are:

- enshrined vs. outsourced network:
    - Do we aim to have a dedicated `zkSharding Prover Set` staked within our system? Or do we want to leverage external proof networks, e.g. Gevulot custom provers running zkSharding proving software? This choice will influence the design space for our prover allocation mechanism.
- prover allocation mechanism:
    - How are proving tasks allocated to provers? Leader-based (protocol-based: rotation, via VRFs, weighted; market-based: auction), racing, etc. This question will inform the proof generation fee model. For example, in a racing situation, if only the fastest prover gets rewarded, less performant nodes may completely drop out leading to increased centralization. If the protocol decides the next prover randomly, then all provers have the same chance of being selected, incentivizing them to remain in the network.
    - resilience mechanism: What happens if a prover fails to deliver their task? Is the task reallocated to another prover, does it become an open race for the entire network?
- proof price setting:
    - Who sets the price of a proving task? It can be either set by the user (in this case we mean the protocol/sequencer), by the prover, or both can meet in the middle through a double-sided auction or orderbook mechanism. If the prover sets the price, the protocol will need to ensure the price (for a fast enough proof) can be covered through user fees, or at least subsidized, without increasing user fees too much. The protocol can set the price by estimating operational costs.
- security against malicious provers:
    - How do we disincentivize malicious prover behavior (withholding attacks, incorrect proof submission)? E.g., provers may be required to provide a deposit to join the network which can be slashed in case of misbehavior.

In the early stages of the network, we can have one or more provers operated and/or trusted by =nil;. However, long-term the prover network should be gradually decentralized to minimize our operational costs, increase the system’s resilience, and leverage more cost-effective, specialized providers.

## Proof Generation Fee Model

We propose a strawman proof generation fee model to be iterated upon as we advance the present research. This model assumes the following system model:

**Prover Network.** The prover network is heterogeneous in terms of proving capacity, lower bounded by the minimum hardware requirements. It is competitive in the sense of each prover attempting to obtain the highest possible profit for its own tasks as well as for any “abandoned” tasks, i.e. when faulty provers interrupt proof generation and the proof either gets reassigned or becomes available to all provers. Provers are part of a dedicated network for zkSharding, although operating within the wider blockchain ecosystem such that, given similarities in hardware requirements, they must remain continuously incentivized to operate provers for zkSharding. Alternative models can explore an open proof market. 

**Trust Model.** Provers can be either rational or malicious. Malicious provers are deterred via slashing, e.g. for lying about their capacity or withholding proofs. In the case of rational provers, operational faults may happen such that a recovery mechanism is necessary. 

**Economic Model.** The safety of the protocol relies on provers’ expending their computational resources to fullfill expensive tasks. Provers wish to obtain a profit. First, provers’ costs (per task as well as amortized setup costs) must be offset. For example, if a prover spends *c* per task, and invested *M* to cover the generation of 500k proofs, then the offset is *c + M/500k*. Then, for an expected profit *P* over *n* proofs, the prover needs an additional revenue of *P/n* average per proof. The prover revenue is thus composed of the *offset fee* and the *expected profit*. These two components need to be covered by the user fee and the network reward. 

## Proof Generation Fee Model  - Formula

We will use the TFM implementation details as a reference in how we want to account for “proving gas”. And for inspiration we will take Linea’s approach of fixed and variable cost, except that this variable cost in our case will be the proof generation cost set by the system.  

If we look at TFM implementation details doc we can see that there is a minimum base fee below which the transaction will not be included in the block: 

$\text{minBaseFee} = 0.01 \text{Gwei}$ 

We also can see that we have $\text gasProving$ which denotes the portion of the gas that is for proving. The proof generation fee model’s goal is to charge the users enough so the prover is profitable for generating the proof. In order to cover these costs we introduce a variable part to the $\text{minBaseFee}$ 

$\text{minBaseFee} = \text{minProtocolCost} + \text{variableGenerationCost}$

For simplicity of charging and taking into consideration the TFM implementation gas proving calculation (gasProving * baseFee) we use the following to determine the VariableGenerationCost.  

$\text{variableGenerationCost} = \text{generationFee} - \text{minProtocolCost}$   

We subtract the minProtocolCost from the benchmarked GenerationFee  as it will be included in the  baseFee anyways to avoid double counting.

**However, If we maintain the current setup where the gasProving is multiplied by baseFee we face an issue where in the periods of baseFee spikes the prover will be rewarded more than needed.** If we do not want this to happen then we must not charge the user for gasProving the full baseFee amount but just the minBaseFee, the rest returning to the user/putting to the treasury or rewarding to prover (dynamic percentage) if he proves in time/fast. Another issue is in approximating the break-even  $\text{generationFee}$ if gas is seperated to protocolGas and provingGas and prover has a limit in how much totalGas (protocolGas+ provingGas) it can prove. If using historical data we can assume the provingGas limitations then this problem could be solved. 

## Next Steps

The initial proof generation fee model should be refined following further research, review, and possibly simulation. With the launch of Aztec’s Sidecar, Gevulot, and possibly a revision of MINA’s tokenomics, we can also better understand what other projects are doing. Further steps include:

- Defining the zkSharding proving model (e.g. prover allocation mechanism, price setters)
- Attempt to obtain proving costs per opcode by running zkSharding provers
- Expand the fee model and simulate it
- Perform benchmarking to gather hardware requirements to help estimate prover setup costs

## References

[1] https://docs.polygon.technology/cdk/architecture/type-1-prover/testing-and-proving-costs/

[2] https://www.aligned.co/post/10-billion-revenue-market-size-by-2030

[3] https://hackmd.io/@Cysic/BJQcpVbXn#What-is-the-best-hardware-to-use-now-and-the-future

[4] https://docs.succinct.xyz/getting-started/hardware-requirements.html

[5] https://worldpopulationreview.com/country-rankings/internet-speeds-by-country

[6]  *https://github.com/scroll-tech/go-ethereum/blob/develop/consensus/misc/eip1559.go*

[7] https://docs.zksync.io/zk-stack/concepts/fee-mechanism
[8] https://github.com/matter-labs/zksync

[9]  https://docs.starknet.io/architecture-and-concepts/network-architecture/fee-mechanism/

[10] https://forum.aztec.network/t/request-for-comments-aztecs-block-production-system/6155