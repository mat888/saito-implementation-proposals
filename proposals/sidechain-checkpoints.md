# SIP - Sidechain Checkpoints

A mechanism for providing re-org security for a blockchain with low fee-volume, designed for simplicity and to be easy to turn off once fee-volume sufficiently increases.

The goals of the mechanism are:

* Simplicity of the mechanism
* Easy offboarding from the mechanism
* Decent decentralization
* Enable simple social-consensus options

The mechanism does not need to be perfectly secure - the reliance on social-consensus in certain edge cases allows the design to be greatly simplified, and this should be considered when reviewing it. The primary goal is to prevent cheap and disruptive chain re-orgs.

## Overview

Sidechain checkpoints label a secure (no smart contracts required) chain as the *Host Chain* and Saito as the *Guest Chain*. The relationship between the two is that Guest Chain blocks must periodically publish their block headers (every $n$ blocks) onto the Host Chain in order to remain a valid fork. This is coupled with the requirement that block producers on Saito, the Guest Chain, must stake Saito tokens to produce blocks.

This forces a block producer potentially working on a long re-org to announce their fork on the Host Chain within a certain period. The data published to the Host Chain identifying the fork on Saito is called a 'checkpoint.' If the Host Chain has no competing checkpoints for a certain length of blocks then the previous checkpoint, assuming it references a valid Saito fork, can be finalized.

Even without specifying any 'finality rules' for the checkpoints, the requirement to announce the intent to publish any fork in a timely manner lets all users transacting currency whether or not a re-org is possible past a certain time, thus preventing themselves from double-spends - it does not prevent cheap and indefinite delays by attackers. The rules defined below address that.

### Finality Rules

Ground Rules:

* Successive Checkpoints (committing the current checkpoint to the last) must be spaced a minimum number of Host Chain blocks apart
* Checkpoints must include all, or a random sample of block headers between the checkpoint they commit to and themselves; these block headers must prove that a Saito Staker had the 'right' to produce those blocks.

Intervals: every sequence of blocks which exists between two checkpoints is called an interval. Intervals always occur at the same Saito block depth i.e. if the interval length is $100$ Saito blocks, then checkpoint $1$ is the first interval at block $100$, checkpoint $2$ at Saito block $200$, etc.

Multiple competing checkpoints for the same interval are allowed to exist on the Host Chain at once:

![tiny-sidechain-checkpoints-step1](https://github.com/mat888/saito-implementation-proposals/assets/22969119/567add35-003d-4f90-a3b8-b52c27b74426)

The above diagram shows three competing checkpoints on the Host Chain for interval $2$. The leftside color indicates which previous checkpoint they have built on top of - so these are three separate Saito forks which share the same parent block defined by the orange color. The rightside colors indicate unique forks.

Note that each checkpoint is published onto the Host Chain at different blocks, but this shouldn't necessarily be used to prioritize one fork over another, as an attacker can be hasty.

If there are no attackers, then the checkpoints will eventually reflect the consensus of the network to settle on a fork:

![tiny-sidechain-checkpoints-step2](https://github.com/mat888/saito-implementation-proposals/assets/22969119/a5e14297-f676-42ff-9a5f-eeba8778c6c4)

The diagram above demonstrates how the orange checkpoint becomes finalized. Once the checkpoint for interval 3 is published on the Host Chain, finality rules state that any checkpoints posted afterwards which reference interval 2 or earlier are no longer valid and to be ignored.

Because the third interval checkpoint 'closed' the ability to submit more checkpoints between $1$ and $2$, and all checkpoints labelled $2$ commit to the same (orange) checkpoint previously, the orange interval is finalized.

![tiny-sidechain-checkpoints-step2a](https://github.com/mat888/saito-implementation-proposals/assets/22969119/e8af56e8-deff-4085-bb3f-1c36bf6adb9a)

This mechanism does not offer perfect security, but it does offer true finality without attackers. Even if the low-fee Saito Chain operated without attackers, users could not be sure that a long re-org was not brewing in the background. This mechanism offers true finality for all periods without attackers.

### Dealing With Attackers - Social Consensus

Attackers can relatively easily publish checkpoints which meet the Host Chain deadlines, reference valid forks, and continiously compete with honest forks.

![tiny-sidechain-checkpoints-step1a](https://github.com/mat888/saito-implementation-proposals/assets/22969119/ea96d2e8-8851-40bd-95f3-31bd3affe9c7)


This can be used to delay finality indefintely (though it at least publicizes such attacks). There are some options in dealing with this, but they all require subjectivity (social consensus):

1. Use total stake or a random number in the checkpoint to determine a winner.

Since checkpoints must prove the blocks within their referenced fork were validly produced with staked Saito, the total stake can be compared between competing checkpoints. If more total stake between all checkpoints commits references one in majority, that majority commitment can determine the winner.

## Withholding Attacks

There is an important reason the solution above should not be automated: data withholding. An attacker may produce checkpoints with valid stake on the Host Chain that follow all rules, yet simply withhold the actual Saito Blocks from the network, which may contain invalid transactions. A fork which is finalized on the Host Chain yet unavailable or invalid to the Saito Network is clearly an attack.

For this reason, simply looking at the Host Chain is insufficient to determine if a checkpoint is valid. A checkpoint must follow the finality rules while also being having its Saito Blocks be available and valid. This is where social consensus must be invoked, and why choosing between competing checkpoints shouldn't be automated.

If multiple checkpoints are competing, but the Saito Network can only validate a subset of the chains those checkpoints represent, then the checkpoints referencing unavailable chains must be manually ignored. If an attacker does decide to extend their disruption this far, they risk havint their stake made inert or even having it slashed, because the checkpoints they created were required to bind stake.

This ultimately has to be decided via social-consensus. Another large benefit of the sidechain-checkpoint mechanism is that it simplifies social consensus and forces all potential block producers to identify their stake.

## Implementation Details

### Stake and Stake Lockup

Since stake must able to hold block producers to account under social consensus, it is necessary that block producers are not allowed to spend their stake before a checkpoint is finalized. The simplest possible rule is that stake may not be transferred (though may still be used to produce blocks) until the transaction which locked the stake is part of a finalized fork.

Basic rules for staking, taken for granted above, should be stated:

* Any stake found producing different blocks at the same depth or checkpoints at the same interval, is punished.
* Stake should buy block producers the ability to produce blocks at a certain rate.

### Minimum Host Chain Interval Length

To prevent attackers from finalizing an interval before honest nodes have a chance to compete, a minimum length of Host Chain blocks must pass before two checkpoints can 'link up.' This parameter depends heavily on the target Saito block time, as well as the block time of the Host Chain.

### Random Sampling

It is taken for granted that checkpoints must either include every block header or a random sample from the interval it reflects - the former is likely excessively expensive and the latter requires a fair random-sampling mechanism.

The simplest possible way to ensure checkpoint publishers provide an unbiased random sample is to split checkpoints into a commit then publish process. The checkpoint publisher first commits their checkpoint by publishing a hash representation of it. A random number from the Host Chain block (like Host Chain block header) is then used to define the random sample of block headers which the actual checkpoint must include. Once the checkpoint publisher sees their commitment and the random sample it defines, they can then publish a valid checkpoint along with the random sample.

The random sample, while not as precise as every block header, can still provide onlookers with an estimate for the relative stake between competing checkpoints.

Requiring checkpoint publishers to produce two transactions per checkpoint on the Host Chain may influence the decision of the Host Chain, as it doubles the Host Chain fees required. The most important quality of the Host Chain is that it is secure, but in order to keep costs low while Saito fees are low, consideration should also be taken in choosing a chain which can publish data for a decent price.
