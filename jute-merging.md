
PASTEBIN
new paste
trends API tools faq
 
Guest User
-
Public Pastes

    Untitled2 sec ago
    Untitled5 sec ago
    Untitled8 sec ago
    Untitled11 sec ago
    Rhetorical Situati...12 sec ago
    dr tier list17 sec ago
    Untitled18 sec ago
    Untitled20 sec ago

 
Pastebin PRO Accounts SUMMER SPECIAL! For a limited time only get 40% discount on a LIFETIME PRO account! Offer Ends Soon!
SHARE
TWEET
Untitled
a guest Sep 8th, 2016 0 Never
AD-BLOCK DETECTED - Please Support Pastebin By Buying A PRO Account
For only $2.95 you can unlock loads of extra features, and support Pastebin's development at the same time.
pastebin.com/pro
rawdownloadcloneembedreportprint text 10.62 KB

# Construction
 
It will be easiest for me to explain Jute by starting with Bitcoin-style consensus and modifying it. And the first change to make is to allow blocks to have multiple parents. This changes our 'chain' into a directed acyclic graph, or 'rope' (also called a braid or a tangle). A block can have multiple parents, but no parent can be an ancestor of another parent. Banning ancestor inclusion is not strictly necessary, but makes implementation easier. Turning the blockchain into a rope means that multiple blocks can be mined in parallel, and then later merged into the chain. In Bitcoin style consensus, when two blocks are mined with the same parent, one will definitely become an orphan. By allowing a child block to point to multiple parents, we can enable both blocks to be included in the chain, meaning that both may be able to get block rewards.
 
![Block Ropes - Blocks with Multiple Parents](http://i.imgur.com/IHMN24h.jpg)
 
 Unmerged chaintips are called 'threads', and may consist of multiple blocks.
 
![Threads of the Rope](http://i.imgur.com/GO98Ncz.jpg)
 
Preventing double spends requires being able to arrive at an exact ordering for transactions. Also required is a certainty that changing the ordering and inclusion of transactions is difficult. In Bitcoin, certainty around the difficulty of reorganizing transactions is provided by the heaviest chain rule. A set of transactions that have been confirmed in a Bitcoin block cannot be removed or double-spent unless another block is found. A set of transactions that have three confirmations cannot be reorganized without having a parallel chain that has at least three different blocks (starting from the parent of the first confirmation or earlier).
 
The properties of exact ordering and difficulty of reorganization can be brought to Jute by using a specific algorithm to convert the DAG into a linked-list. First we define a relative weight for a block. The relative weight of a block is equal to the sum of its own weight plus the weight of all ancestor blocks back to the genesis block. When merging two parent chains, first all identical ancestry, then the heaviest parent chain, then the next heaviest parent, and so on, until finally the lightest parent chain is included. If two parents have the same weight, a deterministic random die roll is used derived from the hash of the child to pick the parent ordering. A set of examples helps to illustrate:
 
![Merging](http://i.imgur.com/FZO72DM.jpg)
 
Block 'E' merges two threads, thread 'ABC', and thread 'AD'. To determine the exact ordering, first all identical ancestry is included. In this case, that is only 'A'. After the common ancestry, the heaviest parent chain is included next. 'BC' is heavier than 'D', so 'BC' is included followed by 'D'. Finally, the child is included, to end up with a final exact ordering of 'ABCDE'.
 
We now explore a more complicated example:
 
![Merging 2](http://i.imgur.com/IC180aL.jpg)
 
There are several merges in this image. The first merges to notice are the merges between 'ABC' and 'AD'. The identical ancestry is 'A', which gets included first. Then, 'BC' is heavier than 'D', so 'BC' is included first. The result of the merges are the two threads 'ABCDE' and 'ABCDF'. 'G' merges each of these threads. First, the identical ancestry is included. The head of this thread is therefore 'ABCD'. Then, because both of the parent threads are the same height, the hash of 'G' is used to randomly select between the threads. In this case, the result was 'ABCDEFG'. A different block 'G\*' could achieve the result 'ABCDFEG\*'.
 
A final example is used to highlight the difference between 'common ancestry' and 'identical ancestry':
 
![Merging 3](http://i.imgur.com/1P8RqvD.jpg)
 
Prior to block 'J', there are two threads, 'ABCDEF' and 'AGHID'. They each have common ancestry 'AD', however they only have identical ancestry 'A', because 'D' has different strictly-ordered parents in each thread. When performing the merge with 'J', you start with the identical ancestry 'A'. Then you merge the heavier parent, which would be 'BCDEF'. Finally, you merge the lighter parent 'GHID'. An important note here is that 'D' is already in our chain, so we exclude it when merging. The result is 'ABCDEFGHIJ'.
 
To reiterate, the purpose of this merging algorithm is to give a deterministic method of converting any DAG into a strictly ordered linked-list. Furthermore, a heaviest-chain rule is used to choose between parents, providing some certainty against reorganizations. In fact, the same guarantee exists in this setup as in Bitcoin: a block cannot be reorganized (removed or given new ancestry) unless at least as many blocks new blocks are added as the block in question has confirmations. For example, if we have 'ABCDE', block 'D' cannot be reorganized unless at least one new block is added, and block 'B' cannot be reorganized unless at least three new blocks are added.
 
Reorganizations can happen in two ways. The first is that an alternate, unmerged chain appears which is longer than the current chain. Like in Bitcoin, the chain will be reversed to a common parent, then the new chain will be applied. The second method is that multiple alternate chains are merged into a single larger chain, such that the current chain is given additional strictly-ordered parents (only possible if some combination of the external threads ended up being longer than the current thread).
 
As a quick summary, we have a directed acyclic graph of blocks, and an algorithm for converting the DAG into a linked list. The merge algorithm provides a guarantee that a block 'A' cannot get new ancestors unless a merge occurs where at least X new blocks are introduced, where X is total number of descendants of block 'A' in the longest known chain.
 
If blocks are allowed to be merged from arbitrary points, potential long-range attacks become available. A simple denial-of-service attack can be constructed by mining a large number of blocks over a long period of time off of an earlier, lower difficulty block in the chain. After creating a large number of these blocks, the blocks are flooded to the network all at once, causing a potentially disruptive amount of congestion and computational load. The general mechanism is to store up blocks and release them all at once. This mechanism can also be used to facilitate double spends. Observe the example below:
 
![Double Spend Attempt](http://i.imgur.com/1HxX4tr.jpg)
 
Initially there are two chains. There is a public chain that is 2,000 blocks long, and there is a secret chain maintained by a minority hashrate attacker which is 200 blocks long. The attacker then decides to attempt a double spend. The attacker releases 'B', containing a transaction that the attacker will reorganize later. 'B' is allowed to get 100 confirmations. After 100 confirmations, The attacker releases 'B*', which has the same parents as 'B', but also has the 200 blocks in the secret chain. When block 'C' merges everything together, 'B*' is given priority over 'B', resulting in a successful double spend.
 
This attack can be prevented by limiting the depth of a reorganization. And we can do this by observing a block's relative height vs. its absolute height. The relative height is the height that the block would appear if it was at the tip of the longest known thread. The exact height is the height that the block appears at in the longest known thread. The block's 'gap' is the difference between the relative height and the exact height:
 
![Rope Heights and Gaps](http://i.imgur.com/d9U1QPu.jpg)
 
We can prevent long range attacks by putting a limit on how large the gap is allowed to be. In the attacker example, the 200 secret blocks which are used to execute the double-spend will all have gaps of 2,000 blocks, as their relative height puts their immediate parent as 'A', however their absolute height puts them as children of the 2,000 block public chain.
 
For a maximum gap X, an attacker with 40% hashrate can consistently build a secret chain 0.4X long, which means that an attacker of that size can consistently double-spend transactions with 0.4X confirmations. Because mining is probabilistic, there is a chance that the attacker can double spend transactions with greater than 0.4X confirmations as well. We will revisit this later.
 
The goal of Jute is to eliminate unfairness between miners. Because Jute creates a system where an attacking miner is always able to get blocks ahead, we want to set up the consensus rules such that the exact ordering does not favor paying out to miners that are consistently putting their blocks ahead in the canonical ordering. Ideally, so long as miners are able to get their blocks into the chain at all, they will have equal payouts to the miners that have higher hashrates, faster network connections, or are engaging in behaviors that manipulate the linked-list block ordering.
 
If we ignore transaction fees, we can achieve this easily. So long as a miner's block is not orphaned, we can honor all block subsidies even if the blocks include invalid or conflicting transactions (remember we are ignoring fees for now). And because blocks have a strict ordering after applying the merge algorithm, we can achieve consensus around transactions despite conflicts -  within the strict ordering, only acknowledge the first valid transaction.
 
Consider the example below:
 
![Consensus Ordering](http://i.imgur.com/UT1nSdr.jpg)
 
We start with a DAG, where each block has different transactions. Block 'D' has a transaction '2\*', which is in conflict with transaction '2' of block 'B'. To prevent double spends, only one of the transactions can be allowed in the chain. According to the strict ordering, block 'B' appears first, which means that transaction '2\*' of block 'D' is ignored, tossed from consensus. Note that the valid an non-conflicting transaction '5' in block 'D' is still kept. The consensus ordering will only throw out the transactions that are not valid.
 
As a side note, this breaks naive SPV, because a transaction is able to be added to the blockchain even though it is invalid. We are however able to easily restore SPV with some additional modifications that will be explained later.
 

    Following these rules, assuming no transaction fees, and assuming that no miners ever have any blocks orphaned, the payouts are fair and are independent of hashrate or network connectivity. The safety of miners is closely related to the maximum allowed block gap. As a reminder, the gap is the difference between a block's relative height and its absolute height. Longer allowed gaps mean greater protection against being orphaned, but also mean that an adversary has more opportunity to do long-range reorgs, which means confirmation times must be higher.

RAW Paste Data
# Construction

It will be easiest for me to explain Jute by starting with Bitcoin-style consensus and modifying it. And the first change to make is to allow blocks to have multiple parents. This changes our 'chain' into a directed acyclic graph, or 'rope' (also called a braid or a tangle). A block can have multiple parents, but no parent can be an ancestor of another parent. Banning ancestor inclusion is not strictly necessary, but makes implementation easier. Turning the blockchain into a rope means that multiple blocks can be mined in parallel, and then later merged into the chain. In Bitcoin style consensus, when two blocks are mined with the same parent, one will definitely become an orphan. By allowing a child block to point to multiple parents, we can enable both blocks to be included in the chain, meaning that both may be able to get block rewards.

![Block Ropes - Blocks with Multiple Parents](http://i.imgur.com/IHMN24h.jpg)

 Unmerged chaintips are called 'threads', and may consist of multiple blocks.

![Threads of the Rope](http://i.imgur.com/GO98Ncz.jpg)

Preventing double spends requires being able to arrive at an exact ordering for transactions. Also required is a certainty that changing the ordering and inclusion of transactions is difficult. In Bitcoin, certainty around the difficulty of reorganizing transactions is provided by the heaviest chain rule. A set of transactions that have been confirmed in a Bitcoin block cannot be removed or double-spent unless another block is found. A set of transactions that have three confirmations cannot be reorganized without having a parallel chain that has at least three different blocks (starting from the parent of the first confirmation or earlier).

The properties of exact ordering and difficulty of reorganization can be brought to Jute by using a specific algorithm to convert the DAG into a linked-list. First we define a relative weight for a block. The relative weight of a block is equal to the sum of its own weight plus the weight of all ancestor blocks back to the genesis block. When merging two parent chains, first all identical ancestry, then the heaviest parent chain, then the next heaviest parent, and so on, until finally the lightest parent chain is included. If two parents have the same weight, a deterministic random die roll is used derived from the hash of the child to pick the parent ordering. A set of examples helps to illustrate:

![Merging](http://i.imgur.com/FZO72DM.jpg)

Block 'E' merges two threads, thread 'ABC', and thread 'AD'. To determine the exact ordering, first all identical ancestry is included. In this case, that is only 'A'. After the common ancestry, the heaviest parent chain is included next. 'BC' is heavier than 'D', so 'BC' is included followed by 'D'. Finally, the child is included, to end up with a final exact ordering of 'ABCDE'.

We now explore a more complicated example:

![Merging 2](http://i.imgur.com/IC180aL.jpg)

There are several merges in this image. The first merges to notice are the merges between 'ABC' and 'AD'. The identical ancestry is 'A', which gets included first. Then, 'BC' is heavier than 'D', so 'BC' is included first. The result of the merges are the two threads 'ABCDE' and 'ABCDF'. 'G' merges each of these threads. First, the identical ancestry is included. The head of this thread is therefore 'ABCD'. Then, because both of the parent threads are the same height, the hash of 'G' is used to randomly select between the threads. In this case, the result was 'ABCDEFG'. A different block 'G\*' could achieve the result 'ABCDFEG\*'.

A final example is used to highlight the difference between 'common ancestry' and 'identical ancestry':

![Merging 3](http://i.imgur.com/1P8RqvD.jpg)

Prior to block 'J', there are two threads, 'ABCDEF' and 'AGHID'. They each have common ancestry 'AD', however they only have identical ancestry 'A', because 'D' has different strictly-ordered parents in each thread. When performing the merge with 'J', you start with the identical ancestry 'A'. Then you merge the heavier parent, which would be 'BCDEF'. Finally, you merge the lighter parent 'GHID'. An important note here is that 'D' is already in our chain, so we exclude it when merging. The result is 'ABCDEFGHIJ'.

To reiterate, the purpose of this merging algorithm is to give a deterministic method of converting any DAG into a strictly ordered linked-list. Furthermore, a heaviest-chain rule is used to choose between parents, providing some certainty against reorganizations. In fact, the same guarantee exists in this setup as in Bitcoin: a block cannot be reorganized (removed or given new ancestry) unless at least as many blocks new blocks are added as the block in question has confirmations. For example, if we have 'ABCDE', block 'D' cannot be reorganized unless at least one new block is added, and block 'B' cannot be reorganized unless at least three new blocks are added.

Reorganizations can happen in two ways. The first is that an alternate, unmerged chain appears which is longer than the current chain. Like in Bitcoin, the chain will be reversed to a common parent, then the new chain will be applied. The second method is that multiple alternate chains are merged into a single larger chain, such that the current chain is given additional strictly-ordered parents (only possible if some combination of the external threads ended up being longer than the current thread).

As a quick summary, we have a directed acyclic graph of blocks, and an algorithm for converting the DAG into a linked list. The merge algorithm provides a guarantee that a block 'A' cannot get new ancestors unless a merge occurs where at least X new blocks are introduced, where X is total number of descendants of block 'A' in the longest known chain. 

If blocks are allowed to be merged from arbitrary points, potential long-range attacks become available. A simple denial-of-service attack can be constructed by mining a large number of blocks over a long period of time off of an earlier, lower difficulty block in the chain. After creating a large number of these blocks, the blocks are flooded to the network all at once, causing a potentially disruptive amount of congestion and computational load. The general mechanism is to store up blocks and release them all at once. This mechanism can also be used to facilitate double spends. Observe the example below:

![Double Spend Attempt](http://i.imgur.com/1HxX4tr.jpg)

Initially there are two chains. There is a public chain that is 2,000 blocks long, and there is a secret chain maintained by a minority hashrate attacker which is 200 blocks long. The attacker then decides to attempt a double spend. The attacker releases 'B', containing a transaction that the attacker will reorganize later. 'B' is allowed to get 100 confirmations. After 100 confirmations, The attacker releases 'B*', which has the same parents as 'B', but also has the 200 blocks in the secret chain. When block 'C' merges everything together, 'B*' is given priority over 'B', resulting in a successful double spend.

This attack can be prevented by limiting the depth of a reorganization. And we can do this by observing a block's relative height vs. its absolute height. The relative height is the height that the block would appear if it was at the tip of the longest known thread. The exact height is the height that the block appears at in the longest known thread. The block's 'gap' is the difference between the relative height and the exact height:

![Rope Heights and Gaps](http://i.imgur.com/d9U1QPu.jpg)

We can prevent long range attacks by putting a limit on how large the gap is allowed to be. In the attacker example, the 200 secret blocks which are used to execute the double-spend will all have gaps of 2,000 blocks, as their relative height puts their immediate parent as 'A', however their absolute height puts them as children of the 2,000 block public chain.

For a maximum gap X, an attacker with 40% hashrate can consistently build a secret chain 0.4X long, which means that an attacker of that size can consistently double-spend transactions with 0.4X confirmations. Because mining is probabilistic, there is a chance that the attacker can double spend transactions with greater than 0.4X confirmations as well. We will revisit this later.

The goal of Jute is to eliminate unfairness between miners. Because Jute creates a system where an attacking miner is always able to get blocks ahead, we want to set up the consensus rules such that the exact ordering does not favor paying out to miners that are consistently putting their blocks ahead in the canonical ordering. Ideally, so long as miners are able to get their blocks into the chain at all, they will have equal payouts to the miners that have higher hashrates, faster network connections, or are engaging in behaviors that manipulate the linked-list block ordering.

If we ignore transaction fees, we can achieve this easily. So long as a miner's block is not orphaned, we can honor all block subsidies even if the blocks include invalid or conflicting transactions (remember we are ignoring fees for now). And because blocks have a strict ordering after applying the merge algorithm, we can achieve consensus around transactions despite conflicts -  within the strict ordering, only acknowledge the first valid transaction.

Consider the example below: 

![Consensus Ordering](http://i.imgur.com/UT1nSdr.jpg)

We start with a DAG, where each block has different transactions. Block 'D' has a transaction '2\*', which is in conflict with transaction '2' of block 'B'. To prevent double spends, only one of the transactions can be allowed in the chain. According to the strict ordering, block 'B' appears first, which means that transaction '2\*' of block 'D' is ignored, tossed from consensus. Note that the valid an non-conflicting transaction '5' in block 'D' is still kept. The consensus ordering will only throw out the transactions that are not valid.

As a side note, this breaks naive SPV, because a transaction is able to be added to the blockchain even though it is invalid. We are however able to easily restore SPV with some additional modifications that will be explained later.

Following these rules, assuming no transaction fees, and assuming that no miners ever have any blocks orphaned, the payouts are fair and are independent of hashrate or network connectivity. The safety of miners is closely related to the maximum allowed block gap. As a reminder, the gap is the difference between a block's relative height and its absolute height. Longer allowed gaps mean greater protection against being orphaned, but also mean that an adversary has more opportunity to do long-range reorgs, which means confirmation times must be higher.
create new paste  /  dealsnew!  /  api  /  trends  /  syntax languages  /  faq  /  tools  /  privacy  /  cookies  /  contact  /  dmca  /  advertise on pastebin  /  scraping  /  go
Site design & logo © 2016 Pastebin; user contributions (pastes) licensed under cc by-sa 3.0 -- Dedicated Server Hosting by Steadfast
My Alerts
Top
