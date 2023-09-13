
## Reducing Routing Signature Cruft

Saito counters Sybils and fairly pays contributing network nodes who relay transactions predicated on attaching routing signatures for each hop in the path a transaction takes towards block production. While this approach is sustainable and self-funding for deeper reasons, it can add undesirable baggage as transactions accumulate several routing signatures, multiplying the size of small transactions.

Below is an example of the routing data necessary for a two-hop transaction. The signatures verify that the address in their same field is entitled to claim work and sign for the next router in the path.

```
[
"0" : {
  "from" : "03ef28eba9f770682ee3805937ee73cc8d25052213afe775977d02e859954c98ad",
  "sig" : "a93e1c8d825edd61188d9af7e2c694b4d606aea09787f0443de1c081fb2778b631a3669e72944ad9b33ae4a7adb8817abba8893e1e34a217d21a01032753f456",
  "to" : "02f85c4312b2385f96e1aa55ec86c99fd1596cec0914266a86e2e2df70bc368809"
} ,
"1" : {
  "from" : "02f85c4312b2385f96e1aa55ec86c99fd1596cec0914266a86e2e2df70bc368809",
  "sig" : "0b988c8260890d4dcc041f576f88c0ff6682a83e029b4dc7f3dc28ae60d3e7d300b9385f1c57953e810eaf390d976a3558d33e6e436d3f012902d8f047919975",
  "to" : "2AnXk35mKoJPHRWkViMWkFkZCR7JBWMwSfAShnZBK9jRv"
}
]
```

While there is some redundant data in the example, such that the "from" address, which already specified from the previous "to" address, one can see how the size of a transaction grows as it is propagated more and more.

## Improving Economic Finality for Individual Transactions

Transactional security in Saito is a function of the cost to orphan all work after said transaction is included in a block. Either due to normal, short-range re-orgs or from a determined attacker, the cost of reverting a transaction is, in effect, the level of security and finality that transaction has. When the size of the transaction is large enough to motivate an attempted double spend, the receiver should of course wait until the cost of reverting said transaction is greater than the amount on the transaction itself to accept it.

Since the work required to produce blocks in Saito comes directly from transaction fees, security concious users could in theory include a very high-paying fee to decrease the chances that their transaction is censored, but this fee would have to be impractically large to warrnat the increase in finality that the small proportion of total work supplied by it would add.

## Routing Bundles

Instead, users should be able to specify that their transaction must be a member of a bundle of a certain transaction fee threshold. A transaction bundle is considered invalid if it contains any transactions whose bundle fee threshold are not met by the actual total fees in the bundle.

### Economic Security

Transactions included in a bundle with a certain fee threshold individually adopt the security of the total bundle fee they are a member of. Since each transaction opting in to such a bundle is invalid when not included in a sufficiently fee-rich grouping, the cost to produce a competing block which censors a single transaction in the bundle is the cost to censor the entire bundle.

Users with access to large volume nodes who wish to increase the probablistics finality their transactions will have once published can author transactions which must be published with the shared economic security of a group of sufficiently high total fees.

### Efficiency

Transactions which do not require themselves to be bundled, can nevertheless, be members of bundles if routing nodes decide to include them as such. Both parties have a strong incentive to use or be bundled, as it can greatly reduce the bandwidth requirements for routing many individual transactions with the same routing path. When a transaction bundle is relayed, a single set of routing signatures is required for every transaction in the bundle.

Even looking forward to future innovations to reduce routing signature overhead in other ways, bundling will remain an efficient method to remove redundant data since it allows multiple transactions to share routing verification data.

#### Without Bundling

```
---------------------  ---------------------  ---------------------  ---------------------
| tx1               |  | tx2               |  | tx3               |  | tx4               |
| routing sig 1     |  | routing sig 1     |  | routing sig 1     |  | routing sig 1     |
| routing sig 2     |  | routing sig 2     |  | routing sig 2     |  | routing sig 2     |
| routing sig 3     |  | routing sig 3     |  | routing sig 3     |  | routing sig 3     |
| routing sig 4     |  | routing sig 4     |  | routing sig 4     |  | routing sig 4     |
---------------------  ---------------------  ---------------------  ---------------------

---------------------  ---------------------  ---------------------  ---------------------
| tx5               |  | tx6               |  | tx7               |  | tx8               |
| routing sig 1     |  | routing sig 1     |  | routing sig 1     |  | routing sig 1     |
| routing sig 2     |  | routing sig 2     |  | routing sig 2     |  | routing sig 2     |
| routing sig 3     |  | routing sig 3     |  | routing sig 3     |  | routing sig 3     |
| routing sig 4     |  | routing sig 4     |  | routing sig 4     |  | routing sig 4     |
---------------------  ---------------------  ---------------------  ---------------------
```

#### Bundled

```
---------------------  ---------------------  ---------------------  ---------------------
| tx1               |  | tx2               |  | tx3               |  | tx4               |
| bundle ID         |  | bundle ID         |  | bundle ID         |  | bundle ID         |
---------------------  ---------------------  ---------------------  ---------------------

---------------------  ---------------------  ---------------------  ---------------------
| tx5               |  | tx6               |  | tx7               |  | tx8               |
| bundle ID         |  | bundle ID         |  | bundle ID         |  | bundle ID         |
---------------------  ---------------------  ---------------------  ---------------------

+

---------------------
| bundle ID tx      |
| routing sig 1     |
| routing sig 2     |
| routing sig 3     |
| routing sig 4     |
---------------------

```

The cost of growing the membership size of a bundle is free, and the cost of adding routing signatures to a bundle is constant no matter how many transactions the bundle contains. Nodes who are able to compose larger bundles can offer users lower fees - at the limit, each transaction enjoys paying a fee which is fully discounted with respect to a routing signature set burden.

### Bundle ID Transactions

One benefit of forwarding individual transactions in real time is that all nodes along the routing path grow their mempool in real time - if all transactions were sent as bundles, which nodes would desire to maximize the size of before sending, then the propagation of network data would become more irregular and potentially delay block production.

For this reason it is important that the format of transaction bundles is ammendable to sending bundles before they are complete. This proposal suggests that a tranasction bundle be defined by a *bundle ID tx*, an individual transaction with standard routing signatures, where all transactions a member of the bundle are signed such that they reference this bundle id. Bundle transactions can then be routed without routing signatures, such that they reference a bundle id.

The bundle ID tx can be routed like a normal Saito transaction, collecting different routing signature sets down different routing paths. Any transaction which references this budle ID tx can then be accepted by any nodes along any path the bundle ID tx has taken, and verifiably impart their work on holders of the single ID tx, allowing all routing signatures to be safeuly stripped from its members. Bundle ID txs should of course be included in the same block as the transactions which it contains.

Since nodes are likely to route all or most of their transactions to all of their receiving nodes, these entrenched routing paths may be recognized by the routing reward scheme without every transaction explicitly carrying that data.

#### Bundle Signatures

Bundle signature are like normal routing signatures, with the exception that they reference the bundle ID tx - this prevents a transaction with the most recent routing 'signature' in its path being a bundle ID reference from having additional routing signatures added to it explicitly, and dictates that any additional routing routing work signatures are to be determined the the bundle ID tx it references.

This preserves the nice properties of the typical routing signature, where order is preserved and any node can add any signatures at will. The necessary change to consensus rules are also minimal - when a winning transaction is selected and its routing path points to a bundle ID tx, nodes must simply use the reference to that transaction to conjur up its data and compute the routing work just as before.

The changes to consensus code overall are very minimal: despite the special classification, *bundle ID txs* need not be anything but a standard Saito transaction with no additional data. The only requirement for bundles to work today is that transaction IDs may be used in place of a routing signature to define all subsequent routers by the signatures in the referenced transaction which is included in that block.
