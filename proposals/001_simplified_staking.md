1. **Introduction to Mining Variance**


The expected time for Bitcoin miners to find a single block is 10 minutes. This is just an average: sometimes the network will find several blocks in quick succession, and sometimes it will take a much longer time before miners stumble onto a single valid block. The algorithm that lets us calculate what percentage of Bitcoin blocks will be found after an arbitrary time_in_minutes is:

    exp( ( -1 * time_in_minutes ) / blocktime_in_minutes )

This lets us calculate that roughly 13 percent of Bitcoin blocks are produced more than 20 minutes *after* the previous solution is found, while 0.24 percent take an hour to be found. But regardless of how much time and energy has already been spent searching for the next block, in Bitcoin our expected time to find a solution remains 10 minutes at every single point in time. If the network has already spent 50 minutes in fruitless hashing, we still expect to find the next block somewhere around the 60 minute mark.

The important thing to note here is that the difficulty of hashing in Bitcoin provides an expected average time for block production which remains constant no matter how much hashing has been done in the past. It does not provide any non-probabilistic guarantee that any particular block will be produced within any timeframe.

Saito’s golden ticket mechanism has similar properties because it also uses a hashing process. We can set a target difficulty that guarantees an average of N golden tickets per M blocks, but we cannot guarantee that any arbitrary chain of M blocks will contain N golden tickets. It is not a problem for Saito if we produce too many golden tickets for any block as only one may be included in the blockchain. But the natural variance at which golden tickets are produced means that some blocks will always go unsolved. Saito’s staking mechanism is intended to address the problems this creates.


2. **Golden Ticket Variance and Staking**

This variance in golden ticket production means that we cannot guarantee that any set of M blocks contains N golden tickets. The larger M gets and the smaller N / M gets the fewer “unsolved” blocks we will have due to the law of large numbers, but it is statistically inevitable that a chain of M blocks will be produced at some point which does not have enough golden tickets.

Hitting this outer-boundary is not a problem for block production. When Saito hits the outer-limit of what is acceptable it forces nodes to produce blocks that contain golden tickets and the network assumes the properties of POW for the very next block. But what happens if we have a block that is not paid-out by a golden ticket because it has fallen too far back in the chain?

We can see that if M = 1 and N = 1 (“classic Saito”) the majority of blocks will not have a golden ticket. We could prevent this by lowering mining difficulty to make it easier to produce tickets, but this simultaneously lowers cost-of-attack by making it cheaper for attackers to produce golden tickets as well. So it is better to keep hashing difficulty targeting our desired average and solve for variance.

This is the purpose of the staking payout. As long as Saito does not have staking, the fees that are contained in unsolved blocks must either be burned or redistributed to the network through some form of block reward. Both approaches have problems. Burning so many tokens creates a terrible deflationary spiral. But adding a block reward undercuts cost-of-attack by allowing attackers to subsidize their attacks on the network: burning your own money to attack the chain is much less of a problem if the chain gifts you with free tokens to make up for your loss.

The staking payout addresses this problem by giving us a secure way to distribute the revenue from uncaptured/unsolved blocks to participants in the network without the risk of the block producer re-capturing them. Because of the way that the lottery is designed, the mechanism imposes significant costs on attackers even if they command a significant chunk of the staking payouts.


3. **How does Saito Implement Staking?**

The protocol tells us that the fees in blocks which do not payout to miners should be distributed to stakers. It does not tell us exactly what data structures we should use to track who is eligible for payment and how these payments are processed.

Our first implementation of the staking mechanism was in the “saito-lite” javascript client - we build it to find out if the mechanism might work. Our second implementation was added in our first Rust client and tried to simplify the data structures used and reduce the complexity of the implementation. The implementation was still somewhat complex.

Specifically, our current implementation gives our blockchain a “staking” object which contains three main UTXO-storing data structures:

- deposits - new deposits
- stakers - awaiting payout
- pending - paid and awaiting new epoch

When users wish to stake funds, they create special transactions which deposit UTXO into the deposits table. Staking happens in epochs (which cycle independently of the genesis period) and once all stakers are paid-out, these deposits will be able to join the cycle of staker payouts.

The onChainReorganization() function then handles the rest. When blocks are wound or unwound from the chain, stakers are plucked from the staking table and new UTXO are created that get moved from “stakers” to “pending”. Once we run out of stakers and everyone is in pending we reset the staking table, add any fresh deposits and re-calculate the payouts due to each participant. The additional computational work calculating payouts happens here due to the complex reasons involving information availability (or lack of availability) when unwinding the chain.

What we essentially have is three separate data tables that must be kept in sync across all nodes. This requires the order of all slips in all three tables to be maintained, which requires data structures with fast insert and read operations in the middle. In order to ensure that these data structures can be maintained in sync across ATR rebroadcasts, we also require new UTXO and transaction types so that nodes can differentiate between the contents of these structures and all three staking tables can be properly reconstructed after a single genesis period, regardless of where the staking payouts are in their own cycle and when observers start monitoring the blockchain.

The code that handles winding and unwinding the blockchain is also more complicated as it needs to keep these external data structures in sync. Unwinding is a particular problem as it is difficult to do things like roll backwards across multiple staking epochs and be able to properly calculate the payouts that the network must make to stakers when we reverse course and roll forward. The existence of multiple circular loops (the genesis period and the staking cycle) which can overlap in almost any sort of combination also makes reasoning about chain operations difficult.

There are some advantages to this approach:

it is similar to how other networks handle staking
onChainReorganization() can blackbox the staking tables

scaling the approach involves “classic” computer science optimization problems

But the implementation has many downsides:

- the approach is complicated

- we have new forms of data redundancy, as changes to the staking tables must now update the main UTXO hashmap in order to keep slips stored in the staking table spendable.


- our initial data structures are very basic and not intended for scale. we will have challenges optimizing large data structures with efficient find-at-middle and insert-at-middle properties, and we practically do not know how large these tables will get in practice, what percentage of UTXO will want to stake?
 
- there is significant one-off computational work that needs to be done every time the staking table is “reset”. The added burden of handling this work will slow the speed at which some blocks can be added to the chain and may open timing-related attacks against the blockchain.


- the new UTXO and transaction types introduce complexity.


- validating the spendability of UTXO slips is more complicated as we cannot simply check our default UTXO hashmap. In some cases we need to verify position in the staking table as well. Efficiently dealing with this problem simply throws it to user wallets - requiring them to know if they are spending (i.e. withdrawing) UTXO that are in the staking table or normal spendable UTXO.

- The existence of the staking object affixed to the blockchain class imposes a perhaps unnecessary computational burden on the network. It also adds redundancy as the UTXO must be simultaneously available in the UTXO hashmap.

- we have unknown questions that we must answer before we can be sure the implementation is well-designed. If we permit all of the tokens on the network to stake we are permitting a massive staking table. If we do not permit all of the tokens to stake we are potentially introducing closure, which may provide attackers with a vector for increasing their share of the staking table and share of staking payouts.


- the mechanism is not elegant


4. **Suggested Implementation: ATR Staking**

We remove the staking table completely. We remove the additional “transaction types” associated with ATR rebroadcasts.

When a block is produced the network continues to collect revenue for its staking treasury from unsolved blocks as at present. But the fee transaction does not contain any payments to stakers. Only routers and miners receive payment via the golden ticket mechanism.

Staking payouts are instead issued to ATR transactions on rebroadcast. ATR transactions continue to be charged a rebroadcast fee as previously. But ATR transactions are now also credited with a percentage of the staking treasury. The amount credited should reflect the staker’s share of the total UTXO value being rebroadcast through the system. High-value UTXO should earn more than low-value UTXO.

We expect that 25% of all transaction fees will eventually be distributed back to existing transactions as a staking payout. We do not know how many transactions will end up being rebroadcast, but this suggests that users should be able to cover costs if the UTXO they lock-up with these transactions is 400 percent larger than the average ATR transaction.

Users focused on store of value should create extremely compact transactions. Users focused on permanent storage should overpay considerably so that the staking payout can cover the costs of forever. No-one can calculate what the actual costs are in perpetuity, but users who model their costs appropriately.


Advantages:


- ATR staking is simple and elegant


- ATR staking re-uses the hashmap (no new data structures)


“muh transient storage” issues disappear

Disadvantages:

- Slips that are profitable may accrue a larger and larger payout even if they are unspendable (i.e. Dead Man’s Wallet). Richard has suggested addressing this problem by slightly increasing their size each loop such that they will eventually flip into a cost-drain.

- It is hard for me to see a problem with this approach, and so I would value suggestions and feedback. The most powerful thing for me about this is that it radically simplifies the number of data structures we need to support. It also eliminates the need for the blockchain to manage a “staking cycle” as well as its “genesis period” – we re-use our existing epoch.

- This may also provide a simple way to handle the “when staking” problem. Instead of monitoring the staking tables separately we would need to monitor the chain for unspent large-value UTXO and re-insert them into the staking tables on reset.
