# Maximal Extractable Value From The Lens of Sharding

**Authors**: [Amirhossein Khajepour](https://t.me/RadNi)  

Summary of the acronyms used in this document:

| *MEV* | Maximal extractable value  |
| --- | --- |
| *CST* | Cross-shard transaction (also referred as *internal transactions*) |
| *PGA* | Priority gas auction |
| *DEX* | Decentralized exchange |

---

In this document we only focus on three widely used MEV strategies:

- *sandwich attack*
- *frontrunning*
- *backrunning*
    - *arbitrage*
    - *collateral liquidation*

Long-tail MEV strategies are not in the scope of what is covered here.

Sharding adds two new properties to the MEV game which didn’t exist as serious before

- split of state
- asynchronous execution (delayed execution)

Below we will be discussing how these two features impact MEV exploitation for the mentioned strategies. In every part, the root of the issue is identified, then the impacted strategy and finally the phenomenon is explained.  

## Liquidity Distribution

Cause: s*tate split*

Impacted strategies:

- *frontrunning*
- *sandwich attack*
- *arbitrage*

---

### Background

One of the driving factors of MEV opportunities is liquidity. Most of the strategies, other than single domain arbitrages require high volume of capital to enable the attacker for exploiting (arbitrages are rather an exception as the attacker can leverage flash loans and have zero initial capital). In general, the attacker’s strategy highly depends on the DEX design and how the liquidity is distributed in our system. 

In order to use the full potential of sharding’s scalability, a DEX in =nil; will have its trading pairs spread across multiple shards. It definitely adds inevitable non-atomicity issues because of the asynchronous cross-shard communication. However, good news is that with looking at uniswap we know that most of the popular trading pairs have their own highly liquid pair contract. We expect the same happens in nil. 

### Multi DEX Regime

With having multiple active DEX in =nil; one impactful question is will they decide to place the liquidity pool of the same trading pairs in a single shard? e.g. will the liquidity pool contract of USDC <> USDT for both uniswap and sushiswap live in a single shard (concentrated Fig. 1), or two different shards (fragmented, Fig. 2)?

![Fig. 1: An example of a concentrated liquidity model in a multi-DEX regime. The USDC <> USDT trading pair for all of the uniswap, sushiswap, and pacakeswap are located in the shard #1. The same is true for WBTC <> WETH trading pairs in shard #2.](/figures/mult-chain/maximal-extractable-value-from-the-lens-of-sharding-00.png)

Fig. 1: An example of a concentrated liquidity model in a multi-DEX regime. The USDC <> USDT trading pair for all of the uniswap, sushiswap, and pacakeswap are located in the shard #1. The same is true for WBTC <> WETH trading pairs in shard #2.

![Fig. 2: An example of a fragmented liquidity model in a multi-DEX regime. Different trading pairs exist in a shard and liquidity of a single trading pair may scattered in different shards.](/figures/mult-chain/maximal-extractable-value-from-the-lens-of-sharding-01.png)

Fig. 2: An example of a fragmented liquidity model in a multi-DEX regime. Different trading pairs exist in a shard and liquidity of a single trading pair may scattered in different shards.

If all the DEXes agree on concentrating their liquidity in same shards, it will definitely helps with increasing the efficiency of end user trades. However, this regime also unnecessitates the MEV bots to fragment their liquidity into multiple shards; hence they can exploit more MEV opportunities. Additionally, having concentrated liquidity means using most of the capacity of shards of highly popular trading pairs; hence those shards can’t really include liquidity pool contract of other popular pairs (I’m assuming). 

We, therefore, expect to see DEX front-ends to provide multiple options for users: 

1. trade with optimal routing; however, non-atomic, risky, more transaction fee (requires multiple CSTs), and slightly more delayed (because of CSTs)
2. quicker trade with less optimal routing but no-risk and atomic → similar to non-sharded chains

### Risky Arbitrage

Liquidity distribution becomes also an issue for arbitrage bots as they no longer can leverage flash loans and must fragment their liquidity in multiple shards which results in less MEV exploits for them.

For arbitrage bots, cross-shard arbitrages are inevitable. They, therefore, may adopt different strategies depending on the level of risk they will take. A risky bot may immediately try to exploit a cross-shard arbitrage opportunity and another bot may wait for a longer time to accumulate multiple exploits and execute them altogether via a single CST. 

***This will eventually result in less competition among bots and less PGA; therefore, lower transaction fees for users and at the same time non-optimal trades for end users.***

### Double Sandwich: A New Attack!

An interesting observation is that the users who chose to go through the optimal routing path with multiple hop swaps that belong to more than one shard, they may be subject to multiple sandwiching attacks. There may be situations where some of the user’s CSTs fail which results in bounce messages. 

***These bounce messages will trigger another swap events in the source shard which are again potential MEV opportunities.***

For example, assume a user wants to perform the following trade: USDC → USDT → WBTC → WETH

Let any of the trading pairs involved in this trade live in a different shard. Then, user sends a swap request which trades USDC for USDT in the first shard. Then, upon success of this transaction, a CST will be sent to another shard to trade USDT for WBTC. Consequently, it results in another CST for a trade of WBTC to WETH in the third shard. If this CST fails for any reason, the bounce message must exchange the WBTCs back to USDC via another two trades to give it back to the user. These two new trades are another source for sandwich exploit. 

For big trades (like over million dollar trades) it may turn to a serious issue. The trader in this case will naturally choose an optimal trading path (otherwise they may pay several hundred thousands dollar extra). Therefore, the trade may involve multiple hops. In these situations MEV bots may even deliberately force user’s CST to fail with slightly changing the pool’s liquidity ratio to take advantage of multiple sandwiches.

![Fig. 3: An example of a double sandwich attack. User is trading USDC for WETH through 3 hops: USDC → USDT → WBTC → WETH. Red boxes are frontrun transactions and blue boxes are back run transactions. Green boxes are the user related transactions. Attacker sandwiches all the three user’s trades and somehow manage the last trade to fail (it can be via a non-profitable sandwich attack to move the price to hit the user’s slippage tolerance). Consequently, it has the opportunity to sandwich user’s two bounce trades.](/figures/mult-chain/maximal-extractable-value-from-the-lens-of-sharding-02.png)

Fig. 3: An example of a double sandwich attack. User is trading USDC for WETH through 3 hops: USDC → USDT → WBTC → WETH. Red boxes are frontrun transactions and blue boxes are back run transactions. Green boxes are the user related transactions. Attacker sandwiches all the three user’s trades and somehow manage the last trade to fail (it can be via a non-profitable sandwich attack to move the price to hit the user’s slippage tolerance). Consequently, it has the opportunity to sandwich user’s two bounce trades.

## Multi Region Exploits

Cause: *asynchronous execution*

Impacted strategies:

- *frontrunning*
- *backrunning*
- *sandwich attacks*

---

There are three regions (shards) involved in an attack scenario, victim’s region, attacker’s region, and exploit region.

For many reasons, these regions may be different shards, such as liquidity fragmentation. This will create an environment with below cases:

1. all the attacker, victim, and exploit regions are the same
2. attacker lives in the exploit region, but victim lives elsewhere
3. attacker lives in the victim’s shard but not exploit region
4. attacker lives in different shard than both of the victim and exploit regions

Number 1 is exactly similar to Ethereum. In the case number 2, victim’s exploitable transaction is a CST. Because of shardDAG’s ordering enforcement, it becomes partially predictable when the victim’s transaction will be processed. It makes it easier for the attacker to know where they should place their transaction (for both frontrunning and backrunning). Also, they have more time to identify and prepare for the exploit. 

The priority between cases 2 and 3 in the view of attacker depends on shardDAG enforcement rules (later on this in the block space separation section).

In the cases number 3, exploiting victim’s transaction is rather risky. It’s because the exploit region’s validators may backrun, frontrun, or even sandwich attacker’s transaction.

In the last case, attacker must take a serious risk to exploit victim’s transaction and we expect this type of attack to happen much less frequently than the others.

Some of the multi region attack examples are discussed in [Rolling in the Shadows: Analyzing the Extraction of MEV Across Layer-2 Rollups].

### Oracle Delay

The case number two may have some useful applications which are not necessarily poisonous MEV. For example, an oracle update which is coming within a CST, because of the cross-shard message passing delay, may enable services like loan insurances.

For example, imagine both a lending and a staking application lives in the same shard. A user then may want to take a loan with half of its initial budget as a collateral, and stake the other half in the staking platform. Then the user may subscribe to a loan insurance service which is allowed to remove user’s funds from staking platform and use them as the collateral top up in the events of liquidation. 

It’s only possible because there is a delay between when the oracle update is being made, and when it actually takes place in the destination shard. The loan insurance service can take advantage of an off-chain solution to prove to the staking contract that an oracle update CST is coming latter in future which liquidates this particular account, and use this proof as a permission to move user funds to secure it’s lending position.

This services are not possible in Ethereum as the oracle update instantly will be executed in every dApp who needs it.

## Block Space Separation

Cause: *asynchronous execution*

Impacted strategies:

- *frontrunning*
- *backrunning*
- *sandwich attacks*

---

MEV can create a game for the validators to choose how much of the block space they wish to use for processing CSTs and how much they use on executing external transactions.

If protocol doesn’t consider any minimum for CSTs (through shardDAG block validity conditions or any other direct enforcement rules), in some events there may be CSTs with absolutely zero MEV profit in the head of the incoming messages for a shard. Then the block producers would instead fill their blocks with the external messages which halts processing CSTs from the message queue. It may result in shard blocks having finality issues as they don’t include a sufficiently big subgraph in shardDAG. 

The simplest solution here is to have two completely separate spaces for CSTs and external transactions of blocks. And extend the execution rule such that first CSTs must be processed, then external transactions. Per each block, CSTs will be processed until the point that either the incoming messages queue is empty, or there is no more space left to execute more. 

This design will change dynamics of the MEV game in our system. One of the direct consequences is that it diminishes the power of exploit region attackers when the victim’s transaction is a CST (second case in the above classification). Subsequently, for highly profitable MEV opportunities, block builders (or MEV searchers) of multiple shards must collude to render a successful attack.

Nevertheless, separating the block spaces contradicts with current shardDAG design. In order to prevent halting of a shard’s local transaction’s execution in the overloaded periods, the protocol allows block proposer to order external transactions and CSTs based on priority fee. It introduces an asymmetry in power between accounts local to a shard, and other accounts. On the other hand, this feature also enables MEV exploiters of the exploit region to create fake overloaded periods by generating dummy and cheap transactions to sandwich and backrun a profitable transaction.

# Summary/Next Steps

The impact of sharding on MEV (Maximal Extractable Value) exploits was discussed in relation to two key characteristics of a sharded system: state split and delayed execution.

Sharding complicates the MEV game for everyone involved. We examined the most commonly adopted MEV strategies in this new model and discussed where issues might worsen for end users and where they might help reduce the impact of harmful MEV.

In summary:

- *Liquidity distribution* limits bots' exploit capacity but also makes sub-optimal and riskier trades for end users.
- Asynchronous execution introduces *multi-region attack* models where different adversaries may have different capabilities. It may also introduce new useful services such as loan insurance.
- ShardDAG enforcement rules will necessarily change the MEV game through, for example, *block space separation*.

To rate the severity of the mentioned impacting vectors, attack vectors like double sandwich can become serious issue as it may remove some of the usecases from the system unless having atomic cross-shard communications. For the rest, in some cases the impact and cost on the end user may increase as there are more attack surfaces (multiple regions are involved in an exploit).

The results here are just some initial explorations into this topic. Unfortunately, all existing papers and analyses only consider single-sharded environments. Even those focused on multi-chain settings can't be completely applied to a sharded system (most of the above arguments are examples inherent to a sharded system).

Therefore, we still need an in-depth understanding of both attack and defense strategies. Here are some potential future directions:

- Finding optimal DEX design regarding liquidity distribution
- Developing routing and aggregator algorithms for end-user trades according to DEX design
- Finding mitigation approaches specifically for serious threats such as double sandwiching attacks
- Studying other instances of MEV