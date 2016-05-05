Paper: Eclipse Attacks on Bitcoin’s Peer-to-Peer Network
From Usenix Security '15

## Summary
Eclipse is an attack on the Bitcoin distributed network which can allow adversaries to double spend Bitcoins, fork the blockchain and selfishly mine Bitcoins by dropping other's discovered blocks. The eclipse attack is performed by flooding the target node with updates of attacker-controlled IP addresses of other nodes in the network. The purpose of the list of nodes on the target is for being able to communicate with other nodes to read or write to the blockchain.

### What systems are vulnerable to the attack?

Any node in the Bitcoin network that uses the official Bitcoin library is vulnerable to the eclipse attack. The Bitcoin library is the core library that each node uses to communicate with one another and to work with the blockchain. This includes both miners, sellers and buyers of Bitcoin.

### What is the nature of the vulnerability?

An attacker is interested in manipulating the messages that are sent and received to the target node. When an attacker controls all of the nodes that the target is connected to, the attacker is able to prevent a miner from claiming that they've 'solved the hash', double spending on transactions such that only one transaction succeeds but the attacker still gains the results of both transactions, and partitioning the mining power to eliminate their abilities from the network thereby giving the attacker the ability to perform a 51% mining attack, ie. controlling what goes on the blockchain. An adversary would most likely attack the Bitcoin network for monetary gains because it could net them large sums of money.

### What is the the exploit? In particular, what is its technical core?

Essentially, the eclipse attack is performed by hundreds to thousands of nodes located at different IPs on the internet. These nodes that belong to the attacker are then directed to update a target node's 'tried' and 'new' tables of a Bitcoin miner or a normal user with the IP addresses of the nodes that are controlled by the attacker. The more IPs that are added to the target node's 'tried' and 'new' tables results in the previous IPs being removed and replaced with the attacker controlled nodes. When the target node restarts their Bitcoin client the node consults the 'tried' and 'new' tables to connect to nodes in the Bitcoin network. Because all of the nodes represented by IPs in the 'tried' and 'new' tables belong to the attacker, the attacker can perform a man in the middle attack between the target node and the rest of the Bitcoin network.

### How reproducible is the exploit?

In the paper, the attack is proven to be successful 85% of the time when the attacker can control 4600 nodes. The author goes on to say that even controlling 400 nodes can succeed at attacking a target. The easiest way to have a distributed set of nodes is to control a botnet, or to rent time on a botnet. Two botnets mentioned in the paper control over 160,000 IPs and 420,000 IPs, respectively, are easily able to cater to performing an eclipse attack on a target. Due to the possible financial gains in attacking the Bitcoin network, paying for time on a botnet can be a small expense.

### Are there likely to be many similar exploits, in the targeted system or other systems?

The Bitcoin community is more likely to see more attacks due to the popularity and rising prices of the Bitcoin network.

### How difficult will it be mitigate/fix the vulnerability in targeted systems?

The paper presented multiple fixes that reduces the success of the attack from it's original 85% claim to 0%. At time of the writing multiple changes have already been integrated into the Bitcoin library, thus reducing the success probability to around 50%. Some patches that haven't made it into the Bitcoin library yet further reduce the attack success rate to 0. These patches are likely to make it in to the Bitcoin library, but take more time because of the large code changes and testing around it.

### Did you like the paper?

I enjoyed reading this paper. Bitcoin has been a fun topic to learn about previously before reading this paper because it involves cryptography. This paper shows an attack that I never knew about before.

### Was it easy to understand, or was it hard to read?

The paper was hard to understand at first. One of the more complicated sections was the part explaining the node's 'tried' and 'new' tables. The way that the node splits IPs into different buckets based on the first two octets of the IP address, and evicting old IPs when the buckets get full were hard to understand at first. In the end, some of the patches made only required a general understanding of how the 'tried' and 'new' tables worked.

### Did you learn much from the paper?

I learned that 51% attacks on the Bitcoin network could be made much easier if an attacker used the eclipse attack on a large Bitcoin miner pool that controlled a large portion of the mining pool.

### How surprised were you by the result?

I was surprised that the attacker could get their attack to work 85% of the time. I was also very surprised that the attack could be completely mitigated if a few patches were added to the Bitcoin library codebase.
