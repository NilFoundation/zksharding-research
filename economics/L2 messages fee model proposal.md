# L2 messages fee model proposal

**Authors**: [Aleksandar Damjanovic](https://t.me/aleksweb3) ([aleksdamjanovicwork@gmail.com](mailto:aleksdamjanovicwork@gmail.com))  

We employ an approach similar to the EIP-1559 approach with a delta adjustment correction for messages.

Important observations: 

- First bid markets stay the same — we do not change or control it;
- The base fee is not shared;
- Messages fight for block space with transactions;
- Messages, in general, have higher processing priority;
- Messages can be out of gas, the same as transactions;
- Block space is volatile;

In the model, we declare that each shard controls demand/request with a base fee that increases when gas spent on the block increases the threshold. The function that describes it was proposed by Aleksandr:

![image.png](/figures/economics/l2-messages-fee-model-00.png)

![image.png](/figures/economics/l2-messages-fee-model-01.png)

The model has a significant advantage over the classic Ethereum approach: 

- Faster reaction but still predictable, as the baseFee can only grow/drop by 1 + adjustmentFactor at maximum/minimum;
- Activation threshold — no base fee jumps if gas spent insignificantly increases the target value;

The base fee has no influence on another shard as the basic idea declares that if the shard demand/request is controlled individually — the whole system will be controlled. So that if one shard is underloaded and some other is overloaded, we stimulate shifting the load from overloaded to underloaded. 

Despite the base fee model perfectly controlling demand parameters on shards, we still don’t understand how to operate with cross-shard messages. This is where the delta-adjustment function comes into play. 

### Cross-shard messages fee

The fee for the message is calculated as follows: 

$$
Message_{fee} = L2_{baseFee} + L2_{adjustmentFee}
$$

Now, let’s observe the following fact: the base fee is burned and does not incentivize the Validator in contrast to the priority fee. So the message fee has to incentivize the Validator to process the message (and even give priority), while the source chain can’t set priority for messages. This is where delta adjustment comes in. 

Delta adjustment finds a balance between the burned part, the base fee part that goes to the validator, and the adjustment fee.

The adjustment fee is a special additional part of the fee that totally goes to the validator and is paid based on the message demand and not the total block space demand. The portion of this fee defined per block based on the following approach: 

Gas for messages (M) and gas in the whole block (T)  in the same block space.

We specify which portion of the L2 base fee goes to the validator with the following formula 
(x = M/T): 

$$
\delta\left(x\right)\ =\frac{x^{0.15}}{1+e^{\left(-6.0x\right)}}
$$

The adjustment as:

$$
r\left(x\right)\ =\ \left|x^{3}\ +\ 0.1x^{2}-0.1x\right|
$$

Validator reward:

$$
\delta(x) + r(x)
$$

The plot of functions is below: 

![image.png](/figures/economics/l2-messages-fee-model-02.png)

Before we explain the details, we must note some important aspects. The most important is rule 10/10. At least 10% of the block should be spent on transactions and 10% on messages. That may not happen only if consensus agrees there are no more messages or transactions in the pool. But in fact, we show that with this model, the rule almost never occurs by "force".

When the demand for messages is low, we highly incentivize validators to process them, while users are incentivized to use messages instead of transactions as they don't pay more than the base fee. So users pay extra priority fees for transactions while messages stay moderately low fee, and validators don’t lose their profit -- a significant part of the base fee goes to them. 

Then, we define the relaxation period. The key idea is to keep a balance between message demand and validator incentivization stable and not let it disbalance.  We can see that the acceleration of the validator fee grows stable in this region while appearing as an extra fee for messages. This is where users start feeling it. One more important aspect of this approach is to avoid strategic manipulation from all players, especially validators. In fact, this is a region where we are most close to Cournot duopoly equilibrium. 

The end of the relaxation period signals that the demand for messages significantly goes up. This is where validators' rewards accelerate growth, and adjustment fees dramatically go up. That incentivizes validators to process as many messages as they can, while users are incentivized to reduce the load from messages.

Note that the message fee is totally bound to the base fee, and even if the adjustment achieves its pick, the base fee will continue to grow quickly, which will lower demand quickly.

The drawback of the model is the pike in priority fees. If priority goes up quickly while message amounts are quite slow, the validators will not have the motivation to include messages other than the minimum. But in fact, we must notice that the rise in priority fee signifies a high load, which also corresponds to the growth of the base fee, which soon brings demand for messages to a balanced level.

Dev note: we need to build a model of when to show these pikes and time corrections. 

### Reaction

We can notice that the model includes two fast periods of reaction while preserving the equilibrium point for stabilization between messages and transaction demand. In fact, the function reaction is done within a single block. Compared to the base fee, the validator decides about the ratio on the same block that includes the message. So the delta function will react immediately. 

### Price prediction

It may appear that price prediction is impossible. But it’s not so. The primary reason is that the adjustment fee may not be higher than the base fee.  So, if we calculate the derivative of the base fee function and observe the change speed within two blocks (message delivery time), it will not be significant. That means users who want to be almost totally sure their transaction will be processed without losses set double the fees for the gas. Even if no high demand for messages happens, the unused part of the fee will be returned. 

### Transaction failures

Note that the reaction happens within a single block, so other network peers can see the change within one block.

Despite possible high volatility (reaction), the fee predictions are way better than even priority fee predictions. Note that adjustment achieves 20% of the base fee only after the delta achieves 60%!

Taking into account predictions of adjustment and high reaction, the expected transaction failure can be mitigated by the following rule:

- Use the latest block adjustment fee value. It would be even better if the RPC node defined the size of the mempool of messages.
- Use as fee 2-2.5x of base fee (ETH value) for transactions that expect CTS.

Those two simple rules mitigate the risks of any transaction failure. That simple rules can  be used by our rpc nodes.