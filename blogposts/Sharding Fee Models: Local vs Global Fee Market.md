# Sharding Fee Models: Local vs Global Fee Market

**Authors**: [Aleksandar Damjanovic](https://t.me/aleksweb3) ([aleksdamjanovicwork@gmail.com](mailto:aleksdamjanovicwork@gmail.com))  

**_Motivation_**: Economy topic blogpost, explaining the pros and cons and our rationale for individual gas pricing. 

## Outline (ignore)

**Goals**: We aim to inspire discussion within the community and identify any opposing viewpoints early. We intend to present our reasoning for preferring individual transaction pricing per shard (1 shard, 1 base fee), highlighting potential benefits and issues. We’ll discuss our approach to tackling congestion and strive to make the content understandable, supported by strong references.

**Target audience**: Researchers, engineers, people who are interested in economy related topics. 

1. **Introduction**
    1. Motivation
        1. Efficient Transaction Fee Mechanisms in Blockchain Systems
            1. Blockchain systems need efficient transaction fee mechanisms to ensure that block space is used optimally
        2. Challenges Amplified in Sharded Blockchains
            1. Sharding divides the network into multiple shards, increasing complexity in resource management.
            2. Uneven congestion and resource utilization across shards
        3. Designing an Optimal Fee Mechanism
            - The goal is to promote fairness, scalability, and efficient block utilization.
        4. Exploring Individual Gas Pricing Per Shard
            1. Investigating whether individual gas pricing can address uneven congestion.
            2. Improvement in user experience, load balancing, and economic viability of the system.
        
    2. **Problem statement:**
        1. Central question:
            1. Should the price of a single gas unit be individual for each shard depending on the congestion of the previous block, or shared between all shards?
        2. **I**mportance of the Question
            1. Why this question is critical for the design of a sharded blockchain’s fee mechanism.
            2. Discuss the implications of each model on network efficiency, user experience, and economic viability.
            3. Emphasize that the decision between shared and individual gas pricing has significant impacts on network performance.
            4. State the importance of finding a mechanism that optimizes resource utilization across all shards.
            
2. **State of the art**
    1. Basics of Transaction Fees
        1. Explanation of how transaction fees work in blockchain systems.
        2. Introduce concepts like gas, gas price, and gas limits
        3. Mention how they are used to incentivize validators briefly but not go too into detail 
    2. Fee models in Sharded solutions:
        1. What model is usually used in Sharded solutions: shared vs individual
        2. Shared is used  in several existing blockchain platforms (mention briefly without detailed discussion).
        
3. **Theoretical foundation - What makes a transaction fee mechanism ideal**
    1. Explain finding market clearing price, basically cover T.Roughgarden’s work
    
4. **Individual vs. Shared Transaction pricing model**
    1. Explanation of both Model ideas
    2. Pros & Cons of Individual Gas Pricing
    3. Impact on network performance
    4. Graphical Comparison and analysis
    5. Challenges 
    
5. **Conclusion and next steps**
    1. Summary of findings, recap the key arguments in favor of individual shard gas pricing
    2. Recognize the complexities and potential downsides.
    3. Encouraging researchers and engineers to share their thoughts.
    4. In our next economy blog post, we will delve deeper into the specific transaction fee mechanism we plan to use in our sharded blockchain.
    5. Currently Developing mechanisms to communicate gas prices across shards effectively. 
    
6. **References**
    1. Tim Roughgarden’s work on transaction fee mechanisms.
    2. Supply demand equlibrium etc

## Content

An exploration of the current gas pricing landscape in sharded systems, focusing on the advantages and disadvantages of Local vs. Global Fee Market

# Introduction

As blockchain networks grow and handle more and more transactions, the efficient utilization of limited block space becomes critical. Transaction fees are currently one of the main levers for achieving this efficiency. They serve not only as incentive for validators who secure the network but also as a mechanism to prioritize transactions. Additionally, the transaction fee mechanism, which dynamically adjusts the fee level based on network activity, helps manage congestion by increasing fees during periods of high demand and decreasing them during periods of low demand. 

In traditional blockchains, achieving this efficiency is a complex and ongoing challenge. Networks must balance between affordability of transaction fees for the end users, optimal block space usage and proper incentivization for validators, all without compromising security or decentralization. 

In sharded blockchains, where parallel transaction processing is enabled via independent but connected shards, these challenges are amplified. Why? Let us briefly cover what some of these challenges are:

- Designing a transaction fee mechanism to meet the above-mentioned balance (affordability, optimal block space usage, validator incentives, security, etc.) is challenging, as the system is split into multiple shards, each handling a potentially variable transaction workload.
- Managing congestion and overload of shards is complex, as certain shards may handle disproportionately high activity due to the presence of popular dApps, while others remain underutilized. This imbalance can lead to suboptimal performance on congested shards, such as slower transaction processing. Depending on the fee market (local or global), this may or may not impact the transaction fees paid by users.

In this blog post, we will explore different approaches to gas pricing in sharded blockchains, discuss the ideal outcome of the transaction fee mechanism, and examine the pros and cons of local and global fee market. We also assert that the Local Fee Market is the best fit for zkSharding, and we will explain the reasoning behind this decision in the sections to follow.

## Gas Pricing in Sharded Blockchains: Current Landscape

Currently there are three predominant approaches to gas pricing in sharded blockchains:

- **Algorithmic global gas pricing ([NEAR](https://docs.near.org/concepts/protocol/gas#gas-price)):** In this model the gas price is adjusted based on the total gas usage across all shards.  If the blockchain as a whole experiences higher resource demand than ideal, the gas price adjustment algorithm increases the gas price, and vice versa.
- **Fixed consensus global gas pricing ([TON](https://docs.ton.org/develop/smart-contracts/fees)):** In this model the price of gas units is determined by the chain configuration and can be changed only by consensus of validators. In this model there is no real fee market, the whole system maintains a “take it or leave it” gas price level.
- **Resource-specific global gas pricing([MULTIVERSX](https://docs.multiversx.com/developers/gas-and-fees/overview#processing-fee-egld)):** In this gas pricing model, different types of operations have their own gas prices. For example, the gas price per unit for value movement and data handling is different from that of contract execution.

## **Theoretical Foundation: Characteristics of an Ideal Transaction Fee Mechanism**

Before we dive deeper, it’s essential to ask: **What is the ideal outcome of the blockchains’ transaction fee mechanism**? To find that answer we will first quote Tim Roughgarden, cryptoeconomics and algorithmic game theory expert at [a16z](https://a16z.com/):

> “Every block is fully utilized by the highest-value transactions, with all transactions paying a gas price equal to the market-clearing price.”[1]
> 

The concept of a market-clearing price is one of the fundamental concepts of classical economic theory. What does it exactly mean? 

![**Figure 1:** Market Clearing Price Demonstration, shows the intersection of the supply (blue) and demand (orange) lines, illustrating the market-clearing price (MCP) at the point where supply meets demand. The dashed lines highlight the MCP, with  $P_{MCP}$  representing the price and  $Q_{MCP}$  representing the quantity at equilibrium.](/figures/blogposts/sharding-fee-models-00.png)

**Figure 1:** Market Clearing Price Demonstration, shows the intersection of the supply (blue) and demand (orange) lines, illustrating the market-clearing price (MCP) at the point where supply meets demand. The dashed lines highlight the MCP, with  $P_{MCP}$  representing the price and  $Q_{MCP}$  representing the quantity at equilibrium.

The **market-clearing price** (MCP) is the price at which the quantity of goods supplied matches the quantity demanded in a market, ensuring that the market is in equilibrium. At this price, there is no excess supply (surplus) or excess demand (shortage), meaning that all sellers who want to sell at this price can find buyers, and all buyers who are willing to buy at this price can find sellers.[2]

In the context of blockchains, MCP is the price at which the total amount of gas demanded equals the available supply. This price serves to prioritize transactions based on their value, ensuring that the limited space in a block is used in the most efficient and economically viable way possible. 

Transaction fee mechanisms like EIP-1559 aim to achieve a MCP for the **target block size** by dynamically adjusting the base fee per gas. To respond to fluctuations in demand, it uses variable-sized blocks, allowing block sizes to expand up to twice the target capacity when necessary. The base fee is adjusted automatically after each block, based on the gas usage in the previous block, ensuring a balance between supply and demand for block space. [3]

## **Local vs. Global Fee Market**

In a sharded blockchain which utilizes variable size blocks, uniform target block size and a base fee adjustment algorithm there can be two market-clearing prices:

- Individual shard MCP: The price at which the individual shard’s gas demand equals the target block size.
- Global MCP: The price at which the total system’s gas demand equals the target sharded system’s gas supply.

The second option might not seem intuitive right away, so let’s start by visually explaining these two MCPs using a simplified model of gas demand across different shards:

- We have 5 shards with different demand for gas as a function of current gas price.
- The demand function is as follows: $q_i(p) = block\_capacity -\beta_i p$ , where $q$ represents the gas demand for shard  $i$,  $p$ is individual shards’ gas price, block_capacity is shards’ total block gas capacity (30,000,000 in this example), while $\beta_i$ is the demand price sensitivity of the said shard (for example: $\beta_i$ = 30,000,000).
- The demand function is a decreasing function as the demand for a good or a service lowers when the price rises.
- Ideal block fullness is 50%.

**Individual shard MCP**

![**Figure 2**: Local Fee Market: Individual Shard’s MCPs, showing each shard’s market-clearing price (colored dots) and the corresponding demand lines. The red dashed line represents the ideal block fullness at 50% of gas limit.](/figures/blogposts/sharding-fee-models-01.png)

**Figure 2**: Local Fee Market: Individual Shard’s MCPs, showing each shard’s market-clearing price (colored dots) and the corresponding demand lines. The red dashed line represents the ideal block fullness at 50% of gas limit.

As shown in Figure 2, we observe that each shard has its i**ndividual market-clearing price**, where the ideal block space for that shard is fully utilized. These prices vary because each shard experiences different levels of demand, which we represent in our model through shard-specific price sensitivity. The graph highlights the market-clearing prices for individual shards, which the fee adjustment mechanism should target in order to optimize efficiency and economic viability. An important outcome of this approach is that by balancing gas usage at the individual shard level, we also achieve system-wide balance, as all shards work toward the ideal block fullness. 

**Global MCP**

![**Figure 3**: Global Fee Market: Shared MCP and Demand Changes Across Shards. The graph shows demand lines for individual shards, all having a Shard 3’s market-clearing price (green dots) under the global fee market. The red dashed line represents the ideal block fullness at 50% of gas limit utilization.](/figures/blogposts/sharding-fee-models-02.png)

**Figure 3**: Global Fee Market: Shared MCP and Demand Changes Across Shards. The graph shows demand lines for individual shards, all having a Shard 3’s market-clearing price (green dots) under the global fee market. The red dashed line represents the ideal block fullness at 50% of gas limit utilization.

In contrast, the Figure 3  illustrates the outcome of a Global Fee Market, where the system’s base fee adjustment algorithm targets a **global market-clearing price**. In our example, this price happens to be the best fitting for Shard 3**.** As a result, all shards, regardless of whether they have high or low demand, are subject to the same gas price. This may lead to imbalances, with some shards becoming overloaded while others are underutilized. In extreme cases, depending on the gas price adjustment mechanism, certain shards may experience congestion while others remain nearly empty. These extreme cases are usually handled through split and merge conditions. Some systems address these imbalances with mechanisms such as shard split/merge conditions.

## Pros and Cons of Local and Global Fee Market

Now that we’ve explored the foundational aspects and current landscape of gas pricing in sharded blockchains, let’s dive into the pros and cons of the Local and Global Fee Market. There’s no one-size-fits-all solution; the best model depends on the goals of the system. Below, we’ll briefly present the key advantages and disadvantages of each approach to help highlight their trade-offs.

**Local Fee Market**

Pros: 

- Each shard adjusts its gas price based on its own demand, ensuring that block space is used efficiently
- Efficient distribution:
    - Applications aiming for cost efficiency are incentivized to deploy their contracts on less congested shards
    - Users seeking cost efficiency are incentivized to interact with the same dApp on less congested shards, reducing the load on busier ones.
- By aiming for a target gas usage in each shard’s block, we maintain system-wide congestion balance. Local fee adjustments respond to both higher and lower activity, ensuring each shard stays balanced and gas usage aims for the target, promoting overall network efficiency.

Cons:

- Coordinating cross-shard transactions is more complex due to the need to estimate and synchronize gas prices between shards, adding overhead to the process.
- Lower prices on less congested shards may attract more users, potentially leading to congestion in previously underutilized shards, shifting bottlenecks rather than resolving them. This could be mitigated by smart wallets that dynamically route transactions to the most optimal shards based on real-time congestion and gas price data, ensuring an even distribution of traffic across the system.

**Global Fee Market**

Pros: 

- Users face a consistent fee across the entire network simplifying fee estimation, especially for cross-shard transactions.
- Easier technical implementation.
- Uniformity as users are charged the same fee per gas unit regardless of which shard they use. This has been the case in sharded solutions on the market.

Cons:

- Available block space is not optimally used across the network
- Slow price response to demand spikes on individual shards as the gas price adjustment algorithm takes into account the total demand across all shards
- Uniform gas pricing removes the economic signals that would normally guide users to underutilized shards, reducing the system’s ability to self-balance based on supply and demand. Some systems address this through algorithmic shard management (e.g., split/merge conditions) to balance load, though this approach operates differently from a market-based system and increases the overall complexity of the system.
- Developers and users may not be incentivized to optimize for specific shard conditions, as there is no differentiation in pricing that rewards efficient shard usage.

## Summary & Next Steps

In this blog post, we explored various aspects of transaction fees in sharded blockchains, including the **Local Fee Market** and **Global Fee Market** models. We also discussed the importance of achieving a market-clearing gas price and reviewed different gas pricing approaches. Among the models, we identified the **Local Fee Market** as the most fitting for zkSharding. This approach will help us manage uneven congestion by allowing each shard to adjust its gas price based on local demand. It promotes better block space utilization, improves network scalability, and provides users with more flexibility to transact on lower-cost shards if they wish.

While this model adds complexity—especially in cross-shard transactions—it leads to greater efficiency and economic viability by aligning transaction fees with shard-specific demand. The benefits in terms of resource optimization make it the most suitable choice for our solution.

In addition, we are researching the potential of **programmable transfers**, which would allow users to move their accounts/contracts between shards. This adds a new layer of flexibility, as users can optimize their transactions and participate in load balancing based on market dynamics rather than relying on predefined algorithms for state distribution. 

[CTA FOR DEVNET]

## References

[1] https://arxiv.org/abs/2012.00854 

[2] https://economictimes.indiatimes.com/definition/clearing-price

[3]  https://ethereum.github.io/abm1559/notebooks/eip1559.html 

# Twitter Thread

1/ Haven’t written a thread in a while, so let’s break down fee models in sharded blockchains.  👇

2/ In sharded blockchains, the choice between global and local fee market directly impacts congestion management, user experience and validator incentives. So, what are the key differences?

3/ A global fee market applies the same gas price across all shards. This simplifies things for users but can lead to imbalanced resource usage—some shards may be overloaded, others underutilized.

4/ To counteract these imbalances, the global fee market systems often use algorithms like split/merge conditions to balance shard load. In contrast, the local fee market adjusts gas prices individually for each shard, responding dynamically to the demand within that specific shard. 

5/  Local fee market on the other hand,  adjusts gas prices per shard based on demand. This makes resource use more efficient, encouraging users to interact with less congested shards with lower gas fees.

6/  The choice depends on network goals:

- Global fee market offers simplicity.
- Local fee market enhances scalability and resource efficiency but requires more complex cross-shard coordination.

7/ In our new article, we explore:

- The current gas pricing landscape in sharded systems
- What is the ideal outcome of transaction fee mechanism
- Local vs. Global fee market and their pros and cons

Learn more 👇 

[LINK]