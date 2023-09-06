# Saito Implementation Protocol -- 00x Merkle Completeness Proofs

## Abstract

Merkle Completeness Proofs are a new method of verification full nodes can offer lite clients which can prove that a set of transactions marked a certain way has been completely provided, or that no such transaction exists in the block at all. Transactions can opt-in and pay an extra fee for any additional computational complexity imparted on block producers, which most notably involves sorting each 'type' in any block; the computational overhead to verify the sorting or to provide a proof are trivial; the latter consists only of providing two additional Merkle Branches for any single group.

## 1. Merkle Proofs

A typical Merkle Proof is able to succintly prove that some peice of data in a Merkle Tree certainly is a member of it by providing its Merkle Branch. In UTXO blockchains this allows lite nodes and clients to confirm transactions without processing an entire block's worth of data. However, if a user is not provided these elements of the Merkle Tree, they can not know for sure what *does not* exist - i.e. they can not know what they are missing without a full block.

## 2. Ordered Merkle Trees

One crucial element which allows Merkle Completeness Proofs to function is that leaves in a Merkle Tree can be placed in a strict order from least to greatest. This is not possible in Merkle Tree implementations where parent nodes are created by hashing the *sum* of their children - it instead requires that parent nodes are computed by hashing the *concatenation* of their children. Since the order of concatenation affects the output hash, the Merkle Tree preserves the order of its input data.

The computation costs of concatenating preimages rather than summing should be trivial - other methods of ordering leaves are certainly possible. Transaction indices may serve to order leaves whilst summing their hashes. Whatever the method, it is important that the order of any two leaves can be compared to obtain a consistent result.

## 3. Transaction Intent

*Intent* is any arbitrary peice of identifying information included in the transaction - the most obvious example would the receiving address, so that recepient can test for censorship of messages to them, but can also be an application ID, layer-2 reference, etc. These mark the transaction and determine how they are ordered within the Merkle Tree, as well as providing users and nodes with an marker for transactions they desire completeness proofs for.

## 4. Intent Ordered Merkle Trees

Merkle Completeness Proofs become possible for transactions which opt into a scheme where all transactions of specific 'intent' are grouped adjecently into an *Intent-Ordered* Merkle Tree - each grouping of intent transactions must also be placed adjacently to other intent groups. All transactions opting in to the scheme must appear contiguously on one side the Merkle Tree, and each group must place all its members in an unbroken sequence until they are exhausted. 

Intent groups themselves must all be ordered by their intent's numeric value such that the value of each intent group strictly increases from left to right. Each intent group must have its elements placed in an unbroken sequence, and each seperate group must appear *sorted* by their intent value in an unbroken sequence. These rules must be enforced via consensus.

## 4. Merkle Completeness Proofs

Because transactions of like intent, and intent groups generally, must neighboor each other to satisfy consensus rules, concise proofs about the non-existence of any further elements of an intent group can be provided via the preceding and proceding elements of that group. Since consensus rules dictate that transactions with intent must be placed contiguously, only an invalid block could possibly place a member of an intent group outside the bounds of the first and last transaction in any sequence - therefore, providing the first elements on either side which are outside the group produces a completeness proof.

This technique allows users to verify with consensus level certainty that a provided set of transactions of a certain intent includes all transactions of that intent from that block using standard Merkle Proofs, with the only additional information required being two transactions in the tree which 'sandwich' the intent group of interest. With this proof, users can be as certain as the block is valid that they know of every transaction of a certain 'intent.'

Note the exceptions when the intent group is the first intent group or the final one: respectively, the first intent group requires no leftside boundary to prove its completeness, and the final intent group requires the first transaction without any intent value, or if every transaction has intent, it is simply the final transaction in the block. 

## 5. Non-existence Proofs

The *Intent-Ordered Merkle Tree* can also provide non-existence proofs when *no* transactions of an intent exist at all in a block. Since intent groups must neighbor each other and are ordered numerically, providing the boundary between two groups (the final element of the leftside group, and the first element of the rightside group) proves that any intent values between those two groups can not exist in the block.

When a user seeks proof that no transactions of a certain intent exist within a block, a full node need only to find the boundary between two intent groups for which the user's intent value would have been required to be place in between. By showing that the intent group in the Ordered Merkle Tree is progressed *past* the intent value in question, that intent can be shown not to exist in the tree with at most only two standard Merkle Proofs.

## 6. Use Cases

* Efficient Partial UTXO Set Generation

Consider transaction logic which constrains a UTXO such that any transactions spending it as an input must have an intent value which references that UTXO's hash. This UTXO can then be proven to be unspent by providing a non-existence proof, just two Merkle Leaves, for every following block up to the tip of the chain.

This allows lite clients to confirm specific transactions are unspent without processing the entire blockchain.

* Censorship Checks

Users expecting sensitive messages which full nodes may choose not to reveal to them can ask for succint and consensus-backed proofs of non-existence to quickly rule out data-withholding. Nodes which refuse to offer such a trivial service may be assumed censorious by users sensitive to such situations.

* Complete Virtual Machine State Inputs

Layer-2 machines which accumulate state from Saito transactions may require that a valid input must have an intent matching the ID of the layer-2. This makes it so any node or user wishing to rebuild the full state of the layer-2 virtual machine needs only two additional Merkle Leaves in order to verify they are being fed the complete set of inputs and not building an incorrect state at the whim of what full nodes choose to reveal to them. Again, nodes refusing to provide such succinct proofs may be assumed to be withholding data.
