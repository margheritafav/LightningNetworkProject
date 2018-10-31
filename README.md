# LightningNetworkProject
This page aims to give an overview of my Master thesis project, but also is an open space to share ideas and knowledge useful to improve the current protocol. Any toughts/feedback would be really appreciated to proceed in the wisest way and find a solution that can also cover all the needs of community. 
If you are interested to this research topic, please do not hesitate to contact me for a possible collaboration.

# The problem
The problem that I'm focusing on is the recovering mechanism of false positive in the Lightning network. The new mechanism [Eltoo](https://blockstream.com/eltoo.pdf) permits to two nodes of each channel to share the last common status of the channel, so if one of the two nodes loses some information, it can simply ask to the other node to share the most recent status. 
This is a very useful solution, but unfortunately, presents some vulnerabilities. For example, this mechanism doesn't include the case that the other node doesn't share the last transaction, but instead an older one, more favorable for own balance. My project aims to solve this particular issue, to make the protocol more solid to face completely the case of a false positive node in the network. 

# Safe nodes protocol
The main idea of the design is using the other connected nodes as back-up of own recent status. Suppose to have the network  shown below. 

![alt text](/Diagram.jpg)

We can formalize the idea as follows: *If a node A is connected to a node B and a node C, for each transaction between A and B, A sends an encrypted information, regarding the last commitment transaction with B, to C. For each commitment transaction with C, A sends an encrypted information, regarding the last commitment transaction with C, to B.*

In this way, if A loses the last transactions, she can ask the information to the other connected nodes and update the status.
Moreover, for each request of information A has to pay a small fee to her connected nodes for the service. 

# Privacy and costs
This solution has two main disadvantages: privacy and expensive costs. 

First, this solution can be considered against privacy. In fact, the other nodes may know when the "unlucky" node has lost some data. To solve this lack of privacy, we can ask regularly to the other nodes an update of the status, within a random temporal interval. So, Alice will ask the other nodes information not just to recover her status, but also as simply update. 

As regarding the second disadvantage, to decrease the costs, it is not necessary to ask information to all nodes every time. For example, Alice can ask Bob the status with Carol, to Carol the status with Diana, to Eric the status with Bob and every time change this order. On the other hand, it is necessary to send information every time to all nodes, so if one of the channels will be closed, there is no risk to lose some information. 

# Assumptions in the solution
To achieve this solution, I've assumed : 
* A node has to be connected to at least two nodes. 
* This protocol doesn't cover the case in which the node loses all information, I presume that a node maintains the most important information (public key, secret and list of nodes to which is connected) in a safe place.

