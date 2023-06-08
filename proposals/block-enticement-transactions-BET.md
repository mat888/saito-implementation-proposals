# Saito Implementation Protocol -- 003 Block Enticement Transactions (BETs)

## Abstract

A *block enticement transaction* (BET) is a special transaction to encourage network nodes to commit themselves to specific blocks at depth *N*, and to impose costs on block publishers who contribute extra blocks at depth *N*. The risk of a BET losing money increases the more likely it is that publishing a block at depth *N* will be redundant; the rational size of a BET at that depth will then decrease, and network nodes will have less incentive to switch to those redundant blocks, contributing to single-block-finality.

| Field   | Value                   |
| ------- | ----------------------- |
| Author  | Matthew Wilson          |
| Status  | Proposed                |
| Type    | Implementation Proposal |
| Created | June 3, 2023           |

## Logic

A BET on a block contains a reference to that block, and functions as follows:

* A BET is only valid at a depth greater than the block it references.
* The BET fee contributes normally to routing work rewards when the BET is in the block immediately following the block at depth *N* that it references, that is, block *N+1*.
* If the BET is validly included but does not reference the previous block, the fee payout is halved.

### Half Refund

The reason a BET is half-refunded to the sender when the referenced block is not adopted is so that the BET is still costly when unsuccesful, and also so network nodes maximize their reward by adopting the block which maintains the full fee of the BET which pays them the most.

The reduction in reward to network nodes could be distributed to miners and stakers rather than being refunded at all, but ultimately the block producer could recoup those fees by quickly spending the same inputs into a normal transaction. When the BET's routing reward is reduced by paying shifting reward towards miners and stakers, the publisher of the BET has motivation to overwite it. Routers can be incentivized to take the overwriting transaction for half the fee of this type of failed BET.

Because the transaction uses the same inputs, only one can be included; as long as the plain transaction pays more than half the fees of the BET, routing nodes will prefer it and the block producer can avoid paying out the failed BET. For this reason it is simpler just to refund that 50% of fee payment to the block producer automatically in the case of a failed BET.

## Network Incentives

Without an enticement, nodes receiving multiple blocks at once will be largely ambivalent in deciding which to move forward with which can fracture consensus and detract from security and usability. When proposed blocks include a BET with a routing signature, receiving nodes can maximize their reward by committing to blocks with the largest BET.

Since a BET only pays its full routing reward when it is included in the block directly after the block it references, nodes eligible for that reward will prefer that referenced block. If a node receives multiple BETs of varying reward for many distinct blocks at depth *N* and propagates all of them, they are *decreasing* the chance that the BET which rewards them the greatest is eventually accepted. They may still earn routing share from all BETs, but only one will grant the *full routing share*; so nodes are incentivized to only propagate the block with the largest BET in their favor.

Even if a node receives multiple potential blocks at depth *N* with equalling rewarding BETs, they will always be better off committing to the propagation of a single block rather than all of them. When Node A fractures the perspective of their neighboring nodes, they increase the chance Node B, which sent to its nieghbors *only one block* will outweigh the split consensus Node A created.

There exist other benefits even if BET size across blocks at depth *N* is similar:

1. The amount of blocks at depth *N* being deliberated on will likely be less when BETs are included.
2. Extremely disruptive stalemates can be settled by block producers including further BETs on the same block.

## Block Publisher Incentives

Block publishers compete for the implicit reward of their block being accepted when they BET on it, likely in the form of routing work they have invested in that block. The rational maximum BET a block producer will include is that which keeps them in profit.

Block producers punish themselves by sending BETs for their block when they are aware that a block at depth *N* already exists; their BET will remain valid, but can and will likely be included on a fork detached from that tardy producer's block. Network nodes who are sent blocks with weak BETs can easily filter them. Block producers will want to compete to entice the rest of the network to accept their block, but the longer they wait to publish the less of a BET they can justify risking, further solidifying blocks published earlier and encouraging them to work on the next block instead.

As time passes and block producers are less willing to risk their money publishing at depth *N*, single block finality concretizes. Publishers sensing any remote risk of placing a BET on a late block are better off being first in line for the following block - with one exception: a publisher who stands to gain an outsized reward at depth *N* may be able to justify placing a large enough BET to sway the network to revert at that depth, but the positive of this is that it *prioritizes* high paying transactions. Just like with normal routing work, BETs used to attack the network or revert consensus will always suffer a security fee via the paysplit which makes sustained malicious activity infinitely expensive.

Still, there may be concern that the use of BETs could backfire against the goal of reducing the network fork rate if an attacker decides to include a large BET on an already produced block in an attempt to halt or revert consensus. This attack is perfectly possible without the BET implementation - an attacker uses funds spent at depth *N* to create a new block at depth *N* with a high a enough fee total to encourage other nodes to adopt that fork. Whether this type of network bribe takes place in a plain transaction or a BET, the cost is equivalent.

## Malicious Fracturing

A BET imposes cost on nodes using routing work to maliciously split consensus at depth *N*. For every block at some depth a malicious node must pay a competitive BET rate (one which convinces the network to accept their block over honest block publishers) multiplied by the number of blocks they produce at that depth. Doing this continuously already requires spending money on routing work and Golden Tickets - the mitigation BETs uniquely help with is the delay in consensus nodes with enough routing work may impose for free by intermittently producing two or more blocks at once; this attack is no longer free.

## Conclusion

In theory, BETs can offer powerful incentives to discourage the network from producing multiple blocks at depth *N* and can give single blocks some finality as block producers avoid placing BETs necessary to sway the network when it is likely a block at depth *N* already exists.

BETs have the added benefit in that they do not introduce closure; nodes can still earn Saito purely from routing work and later use it to BET on their own blocks without special privilege. Like Bitcoin mining, the cost is entirely upfront and reward comes after. Depending on how the market prices the competitive BET rate for new blocks, it may also be an effective deterrent against malicious fracturing.
