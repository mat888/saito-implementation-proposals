# Saito Implementation Protocol -- 00x Merkle Completeness Proofs

## Abstract

Merkle Completeness Proofs (MCPs) are a new method of verification full nodes can offer lite clients which can prove that a set of transactions marked a certain way has been completely provided, or that no such transaction exists in the block at all. Transactions can opt-in and pay an extra fee for any additional computational complexity imparted on block producers, which most notably involves sorting each 'type' in any block; the computational overhead to verify the sorting or to provide a proof are trivial; the latter consists only of providing two additional Merkle Branches for any single group.

## 1. Merkle Proofs

A typical Merkle Proof is able to succintly prove that some peice of data in a Merkle Tree certainly is a member of it by providing its Merkle Branch. In UTXO blockchains this allows lite nodes and clients to confirm transactions without processing an entire block's worth of data. However, if a user is not provided these elements of the Merkle Tree, they can not know for sure what *does not* exist - i.e. they can not know what they are missing without a full block.

## 2. Ordered Merkle Trees

One crucial element which allows Merkle Completeness Proofs to function is that leaves in a Merkle Tree can be placed in a strict order from least to greatest. This is not possible in Merkle Tree implementations where parent nodes are created by hashing the *sum* of their children - it instead requires that parent nodes are computed by hashing the *concatenation* of their children. Since the order of concatenation affects the output hash, the Merkle Tree preserves the order of its input data.

The computation costs of concatenating preimages rather than summing should be trivial - other methods of ordering leaves are certainly possible. Transaction indices may serve to order leaves whilst summing their hashes. Whatever the method, it is important that the order of any two leaves can be compared to obtain a consistent result.

## 3. Transaction Intent

*Intent* is any arbitrary peice of identifying information included in the transaction which has opted in for MCP - the most obvious example being the receiving address, allowing that recepient to test for censorship of messages to them, but the intent can also be an application ID, layer-2 reference, or any other value. These mark the transaction and determine how they are ordered within the Merkle Tree, as well as providing users and nodes with an marker for transactions which they can use to request completeness proofs on.

## 4. Intent Ordered Merkle Trees

Merkle Completeness Proofs become possible for transactions which opt into a scheme such that all intent transactions are grouped adjecently into an *Intent-Ordered* Merkle Tree. Every unique intent group must place its associated transactions in an unbroken sequence until they are exhausted - and then again for the next intent group, and so on. All transactions opting in to the scheme must appear contiguously on one side the Merkle Tree, and each group must place all its members in an unbroken sequence until they are exhausted. 

Intent groups themselves must all be ordered by their intent's numeric value such that the value of each intent group strictly increases from left to right. Each intent group must have its elements placed in an unbroken sequence, and each seperate group must appear *sorted* by their intent value in an unbroken sequence. These rules must be enforced via consensus. While the requirement to sort does pose a computational burden on block producers, it is trivial to verify and can be reimbursed via an increased fee rate. This burden increases as new intent groups are added, and negligbly so as an existing intent group grows.

## 4. Merkle Completeness Proofs

Because transactions of like intent, and intent groups generally, must neighboor each other to satisfy consensus rules, concise proofs about the non-existence of any further elements of an intent group can be provided via the preceding and proceding elements of that group. As consensus rules dictate that transactions with intent must be placed contiguously, only an invalid block could possibly place a member of an intent group outside the bounds of the first and last intent transaction in any sequence - therefore, providing the first elements on either side which are outside the group produces a completeness proof.

This technique allows users to verify with consensus level certainty that a provided set of transactions of a certain intent includes all transactions of that intent from that block using standard Merkle Proofs, with the only additional information required being two transactions in the tree which 'sandwich' the intent group of interest. With this proof, users can be as certain as the block is valid that they know of every transaction of a certain 'intent.'

Note the exceptions when the intent group is the first intent group or the final one: respectively, the first intent group requires no leftside boundary to prove its completeness, and the final intent group requires the first transaction without any intent value, or if every transaction has intent, it is simply the final transaction in the block and requires no additional proof. 

## 5. Non-existence Proofs

The *Intent-Ordered Merkle Tree* can also provide non-existence proofs when *no* transactions of an intent exist at all in a block. Since intent groups must neighbor each other and are ordered numerically, providing the boundary between two groups (the final element of the leftside group, and the first element of the rightside group) proves that any intent values between those two groups can not exist in the block.

When a user seeks proof that no transactions of a certain intent exist within a block, a full node need only to find the boundary between two intent groups for which the user's intent value would have been required to be placed in between. By showing that the intent group in the Ordered Merkle Tree is progressed in a single group *past* the intent value in question, that intent group can be shown not to exist in the block using at most only two additional Merkle Proofs.

## 6. Use Cases

* Efficient Partial UTXO Set Generation

Consider transaction logic which constrains a UTXO such that any transactions spending it as an input must have an intent value which references that UTXO's hash. This UTXO can then be proven to be unspent by providing a non-existence proof, two Merkle Leaves, for every following block up to the tip of the chain.

This allows lite clients to confirm specific transactions are unspent without processing the entire blockchain. Such subsets of UTXO data may be useful for verification nodes which don't run the whole chain, proving ownership of a non-fungible token, and more generally enchancing possible smart contract logic.

* Censorship Checks

Users expecting sensitive messages which full nodes may choose not to reveal to them can ask for succint and consensus-backed proofs of non-existence to quickly rule out data-withholding. Nodes which refuse to offer such a trivial service may be assumed censorious by users sensitive to such situations.

* Man-in-the-Middle Detection

Specific use cases may include public rooms designed to bootstrap key exchange. Users offer basic identifying information and use the transaction intent "key_exchange". They are thus protected from a full node copying the bootstrapping transactions of others and covertly withholding it from its lite clients in order to secretly impersonate users.

If lite clients are not provided with every transaction marked with the intent "key_exchange" they will be able to assume such ommissions have taken place when they do not receive a completeness proof - they can then seek a new network provider which can produce such proofs. When a node finally does provide them with complete information, they will have a valid completeness proof. As certain as the block is valid, they will have all "key_exchange" transactions.

While malicious nodes are still free to spoof the identities of other users, they cannot, without detection, prevent the interested party(s) from seeing every claim to a single identity in the key exchange intent domain. Users which are being 'attacked' in such a way can resolve the situation by connecting to each individual claim to an identity via a secure key exchange, and determining the verity of each seperately.

<!--
** Man-in-the-Middle Detection

A man-in-the-middle (MITM) attack is possible when an intermediating relay is able to intercept key exchange information between two parties, replace that data with their own spoofed data, and prevent each party from seeing the originals, thus convincing them to trust the spoofed messages. Blockchain, in theory, prevents this, because the blockchain itself is robustly censorship resistance.

Lite-clients, however, do not usually have the same guarantees of censorship resistence, since they cannot tell when transactions are withheld from them. If a scalable blockchain is going to exist then users must be able to rely on lite clients, as full nodes may be prohibitively expensive. A MITM attack can occur without detection between two lite clients using the same full node provider, or two which collude. Consider Alice and Bob are using the same compromised network provider to connect to the chain, want to initiate a key exchange, and do not yet know each others' public key.

Alice sends the transaction "Hey, Bob. It's Alice," and Bob sends "Hey, Alice. It's Bob." Both transactions are published on chain and the proof of it is given to each respective sender. But rather than supplying Alice's lite client with Bob's transaction, the full node creates and publishes a spoof transaction which claims to be from Bob and supplies Alice's lite client only with that transaction; then gives Bob similar treatment with a spoofed message supposedly from Alice. Each unknowingly connects to the full node's spoof, who can then relay messages between the two with full ability to read and edit.

It is not the fact that spoofed identities can be levied (this is possible on any open network), but rather that the node can strategically and covertly withhold certain information from the key exchange participants, such that they never have any proof or access to more than a single claim to an identity they are interested in communicating with. It is certainly true that if Alice or Bob were to discover another network provider who was no corrupt, or at least two non-coordinating (but perhaps corrupt) providers, that they key exchange, in the worst case, would fail, rather than becoming compromised.

But considering users will only realistically connect to one full node at a time, and may very well fall into using large, popular providers, it should also be the case that they enjoy the ability to detect a single node performing MITM attacks on lite clients rather than relying on an abstract incentive not to snoop - after all, if nodes have an incentive to share transactions pertaining to users in order to attract their transaction flow, they can attract it all the same by sharing with them spoofed messages and failing to relay the genuine ones.
-->
* Complete Virtual Machine State Inputs

Layer-2 machines which accumulate state from Saito transactions may require that a valid input must have an intent matching the ID of the layer-2. This makes it so any node or user wishing to rebuild the full state of the layer-2 virtual machine needs only two additional Merkle Leaves in order to verify they are being fed the complete set of inputs and not building an incorrect state at the whim of what full nodes choose to reveal to them. Again, nodes refusing to provide such succinct proofs may be assumed to be withholding data.
