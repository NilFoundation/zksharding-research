# Sharded DEX MEV

**Authors**: [Amirhossein Khajepour](https://t.me/RadNi)  

We have previously started working on MEV impacts in a sharded environment in [Maximal Extractable Value From The Lens of Sharding](/multi-chain/Maximal%20Extractable%20Value%20From%20The%20Lens%20of%20Sharding.md). Here, we first summarize the important points mentioned there and try to build new ideas on top of them.

# TL;DR

In summary, our observation here is that one way to solve the issue of double sandwich attack is to have flash loans and flash swaps. Therefore, in this document we will design a secure flash loan suitable for a sharded network, and later create a DEX with new a mathematical technique and flash-swaps. That being said, the proposal here solves both issues at once.

# Summary Of Previous Results

- To maximize trading efficiency in a multi-DEX environment, equilibrium naturally pushes the ecosystem towards a more locally concentrated design. In this setup, similar pairs from different DEXes colocate in the same shard. This concentration can be achieved either through sophisticated automatic relocation techniques enforced by the protocol or through a social consensus effort established in advance. For more details, see [here](/multi-chain/Maximal%20Extractable%20Value%20From%20The%20Lens%20of%20Sharding.md).
- Unlike single-domain (non-sharded) systems where arbitrage is typically considered a risk-free financial exploit, in a sharded environment, it becomes a more complex game. Sharding alters the dynamics, allowing actors to choose strategies based on their risk tolerance. This shift positively impacts shard congestion and traffic (reducing PGA issues). However, fewer arbitrage opportunities may result in less optimal trades for regular users eventually.
- A new type of attack emerges in high-volume trades with significant consequences. In essence, sandwich bots can force the final step of a trade to fail, resulting in reversions and additional trades in the preceding pairs. This allows sandwich bots to exploit more trades, hence the term "double sandwich" attack. For a detailed explanation, see [here](/multi-chain/Maximal%20Extractable%20Value%20From%20The%20Lens%20of%20Sharding.md). Moreover, the local concentration on the flip side pushes different pools to separate shards, increasing the likelihood of trades involving multiple hops.

# Cross-Shard (no-longer) Flash Loans

Flash loans are generally not compatible with sharding. They're easy to implement when the transaction execution is confined to a single environment. However, issues arise when a flash loan involves actions across multiple shards:

1. We must guarantee the return of funds.
2. After the loan is taken the vault's condition changes, potentially affecting other requests before the borrowed money is returned.

To illustrate these challenges, we first design a flash-loan protocol that guarantees the funds will be returned to the vault; however, the financial efficiency is compromised (only addressing the second issue).

## Cross-Shard Guaranteed Returns

First of all, assume the lending dApp has its contracts in every shard. Then we consider two functionalities for the contract that together guarantee the return of the funds to its origin:

1. `crossShardFlashLoan(dstShardId, callback, loanAsset, amount)` : It first of all charges the sender some premium which can be less than the actual loan’s fee. Then sends a CST which calls `executeFlashLoan(callback, fee)` function of the loan contract that lives in the shard number `dstShardId` . This CST carries `amount` of the `loanAsset` . The value of the `fee` is also the flash loan fee based on the current condition. This `fee` also may consider a TVL (time-to-live) as the maximum number of CSTs the money can be sent back and forth! It may even utilize EIP-1559 type of fee adjustment to disincentivize attackers from abusing this feature.
2. `executeFlashLoan(callback, fee)` : It first calls the `callback` which must live in the same shard with all the tokens it has received. Then checks if the user could provide both the loan and the requested `fee` . Then returns the entire money back to the sender (which is the loan contract in the source shard). If the user was unable to provide enough money it reverts which automatically returns the tokens back to the origin loan contract.

![image.png](/figures/mult-chain/sharded-dex-mev-00.png)

The above diagram shows an example where the user asks for a flash loan (1) which calls `crossShardFlashLoan` on the loan contract in shard #1. It then sends out a CST to the loan contract in the other shard (2) that subsequently calls the DEX contract with the loaned value (3). This call executes an arbitrage on behalf of the user and sends back the debt plus fee to the loan contract on shard #2 within the same transaction (4). Finally, the loaned tokens are sent back to the origin loan contract through another CST (5).

Note that this pattern can be repeated multiple times with no issues, meaning that the loan contract in the shard #2 can again add some tokens to the sequence of calls in the form of flash loan, and the design guarantees that the loans will be returned safely.

As it was briefly mentioned, this design has a fundamental issue where the money which is being loaned actually left the vault and it may have huge impacts for other requests. For instance, in the same example between the steps 2 and 3 (it’s called the cooldown period) if anybody wants to touch the loan contract in the shard #1 for whatever reason, it doesn’t have access to the entire capital it owns. You can think of the loan contract as a DEX like Uniswap which provides flash swap services. Then because the money doesn’t exist, the swaps that happen during the cooldown period may subject to too much price fluctuation which can become a source of financial exploit and MEV attack.

An easy and not-too-practical solution is to use flash-mints instead of flash-loans where the amount being loaned is minted and then later burnt when the loan is returned. Nevertheless, flash-mints have trust issues where the token minter contract has to let other dApps to mint tokens and because of these problems they haven’t became widespread. In the next section another approach is explained that solves the problem!

## Promise Vaults

The main idea is to have a vault which returns actual tokens for flash loans however doesn’t change the vault equation. Later, if a user tries to use/swap the money during the cooldown period, the vault can mint a *promise* token and use those instead. Later, the user can exchange the promise tokens with underlying tokens once they are ready!

Other dApps can also create secondary markets for these promise tokens which allow users to immediately exchange them with the underlying asset. However, note that the promise tokens are inherently short-term promises that won’t take longer than few blocks (the cooldown period is so small).

# Sharded DEX With Forced Price Rebalancing and Promise Vaults

We want to design a DEX that supports both flash loans and flash swaps with two new ideas. Then we prove that our DEX can prevent double sandwich attacks.

With that the promise vault idea we can solve the issue with flash loans; however, a very important part is missing which is handling the case with flash swaps. For that we use a technique which we call it *forced price rebalancing.* In the following we explain what this technique is and how it together with promise vaults would enable flash swaps.

The DEX architecture is as follows:

- The pair for tokens $A$ and $B$ deploys two ERC-20 tokens (PromiseA and PromiseB)
- For a normal swap if the funds are in cooldown period, it issues temporary Promise tokens as IOUs until the real tokens are available.
- Once the cooldown period is over, the pair contract allows every body who holds Promise tokens to unwrap them back to the underlying token
- It supports both flash-loans and flash-swaps
    - For flash-loans it uses the underlying and real tokens $A$ and $B$. If someone tries swap them while the tokens are in the cooldown period, we mint required promise tokens and proceed with them.
    - For flash-swaps the protocol tries to move the curve down until the result of the flash-swap in the other shard is determined.
    - Once the swap is finished, the curve moves back up to where it was before. This process is explained next.
    
    ![image.png](/figures/mult-chain/sharded-dex-mev-01.png)
    
    For example, in the above picture, assume the protocol at the moment is in the point A of the green curve. Then a user requests a flash-swap which results in moving out 5 tokens from the horizontal axis (the red dashed-line). The constant product formula now puts the protocol in the point B of the purple curve. The issue is because it’s a cross-shard flash-swap, we don’t yet know whether the user will later return the money in the same token (going back to the point A) or it will return the debt in the other token (point E). If we stay in the point B, because the reserve values are changed it makes a price discrepancy with the actual tokens prices which is an attack vector. 
    
    In order to solve the issue we need to somehow persist the price to be the same. Note that the price is defined to be the derivative of the curve in the point were we are at. Before making the swap, the price was equal to the price in point A and green curve. Therefore, an option for us would be to move to the point D of the purple curve where the derivative is the same. But there is a problem here. If we want to move to D, then we need more horizontal tokens and remove some of the tokens in the vertical axis. But we don’t have more horizontal tokens (we initially removed them from the pool!). One option is to have some rebalancing pools in our DEX which is only used in these situations: after every flash-swap, the pools are used to move the curve to the same price (but different reserves) in exchange for a fee.
    
    <aside>
    ❓
    
    I’m personally not sure if it makes any sense or may change to an attack vector for those who put their money in those rebalancing or not. Can anyone render sandwich attacks by initiating fake cross-shard flash-swaps?
    
    </aside>
    
    There is another option that we focus on here, which is lock some of the vertical tokens in order to move the price back. Then once the flash-swap is finished, we release the locked tokens to the pool again. More precisely, instead of staying at point B, we temporarily remove 5 vertical tokens which changes the curve to the grey one and land in the point C.
    
    A more complicated example is shown below:
    
    ![image.png](/figures/mult-chain/sharded-dex-mev-02.png)
    

## Pseudocode

Contracts:

- PromiseA contract:
    - It’s an ERC-20 token whose owner is the pair contract
    - Storage parameters:
        - total supply, call it $t_{A^P}$
        - etc.
- PromiseB contract:
    - It’s an ERC-20 token whose owner is the pair contract
    - Storage parameters:
        - total supply, call it $t_{B^P}$
        - etc.
- Pair contract for tokens $A$ and $B$:
    - Storage parameters:
        - Token $A$ virtual balance, call it $s_{A^v}$
        - Token $B$ virtual balance call it $s_{B^v}$
        - Locked $A$ tokens, call it $s_{A^l}$
        - Locked $B$ tokens, call it $s_{B^l}$
    - All the market making formulas use $s_{A^v} - s_{A^l}$ and $s_{B^v} - s_{B^l}$ as the reserve values
    - Functions:
        - `swap((tokenOut, $x$), tokenIn, address callback)` :
            - Assume `tokenOut` is the address of $A$. Then it means the user needs $x$ of $A$ tokens. If the `callback` is not empty its a flash-swap request and the user is willing to return the money in `tokenIn` .
            - Let $r_{B}\ \text{and }r_{A}$ be the pair contract’s reserves balances for token $A$ and $B$ balances respectively, $w$ be the amount of `tokenIn` the user is going to be charged as a result of the swap while user only provides $z$ tokens initially ($z$ can be zero or much less than $w$).
            - If the `callback` is empty:
                - If $x < r_{A} - s_{A^l}$
                    - gives $x$ $A$ tokens to the user
                    - sets $s_{A^v} = s_{A^v}-x$
                - Else, if $x < s_{A^v} - s_{A^l}$
                    - Mints $x - y$ PromiseA tokens and give them to the user
                    - gives $y\ A$ tokens to the user
                    - sets $s_{A^v} = s_{A^v}-r_{A}$
                - Otherwise, it rejects the swap
            - If the `callback` is not empty:
                - *if `callback` lives in the same shard:*
                    - *if $x < y$,*
                        - *it gives $x\ A$ tokens to the user, works exactly the same way as uniswap’s flash swap works*
                - if `callback` lives in a different shard:
                    - if $x ≥ r_{A} - s_{A^l}$
                        - rejects the request
                    - otherwise
                        - if `tokenOut` != `tokenIn` : (it’s a flash-swap)
                            - we assume it’s a constant product market maker
                            - set $s_{B^l} = s_{B^l} + \mu$, where $\mu=r_{B}/r_{A}\times x$
                            - then charges the user with some premium (discussed in ‣ ), and calls `executeFlashLoan(callback, fee, $(\mu, B)$)` of the uniswap contract which lives in the destination shard with $x\ A$ tokens and $z\ B$ tokens and `fee` = $(w-z+$rest of the fee, $B)$
                        - if `tokenOut` == `tokenIn` : (it’s like a flash-loan)
                            - first of all charges the user with some premium (discussed in ‣ ), then calls `executeFlashLoan(callback, fee, (nil, 0))` of the uniswap contract which lives in the destination shard with $x\ A$ tokens, and `fee` = $(x+$rest of the fee, $A)$.
            - `executeFlashLoan(callback, fee, (token, amount))` :
                - Only another pair contract from a different shard can call this function. The `fee` represents what tokens we need to transfer back to the sender’s pair contract to finish the swap on the previous pair.
                - It first executes `callback` with all the tokens that it received from this call. After the execution provides the `fee` value, then it calls `finisheFlashSwap((token, amount))` of the source pair contract with `fee` tokens.
            - `finishFlashSwap((token, amount))` :
                - It checks that the sender is a pair contract in another shard.
                - assuming `token` is the $B$ token, sets $s_{B^l} = s_{B^l} -$ amount
            - `unwrapPromise(token, amount)` :
                - Assume `token` is the $A$ token’s address
                - If `amount` $\leq r_A$, it burns `amount` PromiseA tokens from the user and gives `amount` of $A$ tokens back to the user
        
        Other functionalities such as add or remove liquidity are the same as usual Uniswap.
        

# Double Sandwich Protection

A multi-hop trade in a naive sharded DEX design is threaten by double sandwich attacks. In summary, in order to maximize load distribution that is provided by sharding, the DEX architectures are motivated to distribute different trading pairs into different shards. It will become an issue when there is a big trade that involves multiple hopes. 

For big trades where the sandwich attack opportunities are rewarding, the attacker has the incentive to force the trade in the very last trading pair to fail which triggers a series of revert trades in opposite direction that the attacker can again exploit.

Now we explain how the above architecture cleverly prevents double sandwich attacks. With a closer look, we realize that the root of issue in double sandwich attack scenarios is the trader want an atomic execution across multiple trading pairs; however, due to the cross-shard async communication we can not easily bond all these different pairs together. Additionally, to start a trade, we need the outcome of the previous trades. In a naive approach, the trader starts executing the trades one by one (normal Uniswap) and hopes that the rest also would successfully happen. This optimistic execution causes the double sandwich issue. 

## Multi-Hop Cross-Shard Trades Via Flash-Swaps

We propose to use flash-swaps in every step of a multi-hop trade (other than the last step). For instance the picture below shows an example of using this approach:

![Multi-hop trade with flash-swaps happy path!](/figures/mult-chain/sharded-dex-mev-03.png)

Multi-hop trade with flash-swaps happy path!

As it can be seen, we instead let the user to lock the required money in all the trading pairs and finally execute the trades altogether! In this approach, the attacker can’t run the second sandwich attack, because there is no secondary series of trades if anything fails (in any case, the trades only happen in one direction!). The picture below shows as an example where the final trade fails, but because the consecutive steps are not involved with any profitable actions due to cleverly choosing the locking amounts:

- Sandwich attacks are impossible, because there is no trade to sandwich it!
- Arbitrages can also be proven to be non-profitable: According to the way we choose the amount of capitals that we lock for every flash-swap ($x$, $r_{B}/r_{A}\times x$), the marginal price would be the same before and after the collaterals are returned. It means during the cooldown period there can’t be any arbitrage opportunity to exploit.

![Multi-hop trade with flash-swaps when the last trade fails!](/figures/mult-chain/sharded-dex-mev-04.png)

Multi-hop trade with flash-swaps when the last trade fails!

# Slippage and Loan Fee

A sharded DEX already has slippage issues due to asynchronous communications - for a CST to be executed there is enough delay for searchers and sandwich bots to render a sandwich attack which increases price slippage.

Double sandwich attacks exacerbate the situation where a trade with high allowed slippage increases the incentive for such attacks. Conversely, low slippage leads to unpredictable execution results. Furthermore, sharding already introduces uncertainty in CST’s execution result. Combining this with low slippage creates a significant challenge.

With the idea proposed here, we can have more control over the slippage since the trades are no longer subject to forced cancelation. However, our price rebalancing introduces another source of economic inefficiency to the protocol that we must evaluate its impact and potentially remove it as much as we can.

When we lock some part of the liquidity, it means the locked part temporarily doesn’t contribute to the interactions with the pool. Consequently, we have less liquidity which results in less-efficient trades. Let’s assume a user requests a flash swap that wants $x$ of $A$ tokens with TVL $t$. The average trading volume of our pool in $A$ token is $v_A$ for $t$ blocks (This number can be a negative which shows the token is increased in its price! But the formulas slightly change I guess if it’s negative). For our purpose we can assume all of it happens within one transaction (to study the general case the arguments will be exactly the same, but the math becomes more complicated and involved). 

Our protocol changes the pool reserve’s volumes to:

$(r_A, r_B) → (r_A-x, r_B-r_B/r_A \times x + \rho)$ (assume $\rho$ is the fee which user is paying at the beginning when taking the flash-swap. We also assume the fee is in the token which user is eventually paying by)

If the swap would have been successful and non-flash, instead we would have had:

$(r_A, r_B) → (r_A-x, \dfrac{r_Br_A}{r_A - x})$

Therefore, trading $v_A$ in both cases would have resulted in the following number of $B$ tokens

- Flash-swap:
    
     $(r_B-r_B/r_A\times x + \rho) - \dfrac{(r_A - x)(r_B-r_B/r_A\times x + \rho)}{r_A-x+ v_A}$
    
- Non flash-swap:
    
    $\dfrac{ r_Ar_B}{r_A-x} - \dfrac{ r_Ar_B}{r_A-x + v_A}$
    

To make it fair for everyone, we can set the fees such that the user has to pay almost as much as the flash-swap is costing other users during the cooldown period. It means:

$\rho \approx\\ \dfrac{ r_Ar_B}{r_A-x} - \dfrac{ r_Ar_B}{r_A-x + v_A} - [(r_B-r_B/r_A\times x + \rho) - \dfrac{(r_A - x)(r_B-r_B/r_A\times x + \rho)}{r_A-x+ v_A}]$

(I tried to simplify but it got so complicated:D)

# What’s Next

- UniV3/V4 and concentrated liquidity ideas
- Formulate similar ideas for multi-asset lending pools
- If an attacker wants to frontrun a victim, is it cheaper for them to get a flash-swap? I think they’re equal. Am I right?