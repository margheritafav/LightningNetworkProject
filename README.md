# Lightning Network Project: Açai Protocol
This page aims to give an overview of my Master thesis project, but also is an open space to share ideas and knowledge useful to improve the current protocol. Any thoughts/feedback would be really appreciated to proceed in the wisest way and find a solution that can also cover all the needs of the community. 
If you are interested in this research topic, please do not hesitate to contact me for a possible collaboration.

# The problem
The problem that I'm focusing on is the recovering mechanism of false positive in the Lightning network. The new mechanism [Eltoo](https://blockstream.com/eltoo.pdf) permits to two nodes of each channel to share the last common status, so if one of the two nodes loses some information, it can simply ask to the other node to share the most recent status. 
This is a very useful solution, but unfortunately, presents some vulnerabilities. For example, this mechanism doesn't include the case that the other node doesn't share the last transaction, but instead an older one, more favourable for own balance. My project aims to solve this particular issue, to make the protocol more solid to face completely the case of a false positive node in the network. 

# Watchtower 
The Açai Protocol aims to solve the problem above using watchtowers for backup service. Let's suppose to have a network where Alice is connected to Bob, Charlie, Diana and to 3 watchtowers: W0, W1 and W2. 

The Watchtowers, as we've defined so far, are currently using to handle breaches while a client is offline. For example, if Alice is offline, Bob can send an older status channel to the blockchain, more favourable for him. In this case, the watchtower can discover the malicious behaviour and intervene on time in favour of Alice, even if Alice is offline. 

To make aware the watchtower of all last status channels, Alice has to send to it a [**hint**](https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-April/001196.html) and a [**blob**](https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-April/001196.html) after every change of status channel.

1. Change of a status channel.
2. Calculate hint = tixd[:16]
3. Calculate blob = Enc(data, txid[16:])
4. Send (hint, blob) to a watchtower.

where the txid is the information of the channel status between two nodes (e.g. Alice and Bob).

# Açai Protocol
The solution project, called Açai Protocol, aims to use the watchtowers not just for monitoring the channels, but also as a  backup service. To this aim, the protocol adopts the three concepts of txid, hint and blob, and extends their use. To distinguish the two services, I'll use a different notation: txidç, hintç and blobç. 

*  txid, hint and blob for monitor channel status
*  txidç, hintç=txidç[:16] and blobç=Enc(dataç,txidç[16:]) for the backup service. 

In the Açai protocol, dataç contains an array of txids: [txid_Bob, txid_Charlie, txid_Diana], where e.g. txid_Bob refers to the transmission between Alice and Bob.

Every time that a status channel changes, Alice sends to one of the three watchtowers (randomly chosen) two pairs: the channel status information (hint, blob), but also the pair (hintç, blobç) contained the backup after the change. Note that the watchtowers will store both pairs (hint, blob) and pairs (hintç, blobç), without distinguishing them. 
To this aim, dataç and data have to have the same dimension of 32 bit.

In the following sections, I'm going to list all the steps on how the node Alice sends the data to the watchtowers and how she can request the backup.


## Send the backup to the watchtower
**Scenario:** one of the channel status has changed, so Alice has to send the new channel status information (hint, blob) and the new backup (hintç, blobç) to one of three watchtowers.

**Steps:**
1. To create txidç, Alice uses a deterministic wallet address based on the current Block height. 

    e.g. *address*= m/108/0(mannet)/(number account)/0/Current_Blockheight 

    where we use 108 as purpose and O as the account.
    
2. Calculate the txidç_n. 

   Since it's possible to have more change status channel in the same Block, we can enumerate each txid inside the same       Block: txidç_0,txidç_1, txidç_2, txidç_3...
   
    Assume to have the first change channel status of the current block, so calculate txidç_0. 
    
    txidç_0= 2SHA256(*address*)
    
    For the next status channel changes of the block, we apply the following rule: the txidç_n is calculated as the hash function of the previous txidç_n-1.
    
    For example: 
    
                 txidç_1= SHA256(txidç_0)
    
                 txidç_2= SHA256(txidç_1)
             
                 txidç_3= SHA256(txidç_2)
             
                 .....

3. Split the txidç_n into hintç_n= txidç_n[:16] and blobç_n= Enc(dataç_n,txidç_n[16:])
4. Send hintç_n and the blobç_n to one of three watchtowers, with the information of the new channel status (hint,blob).


## How to request backup to the watchtowers
**Scenario:** Alice can request the backup when she has lost all her data accidentally or otherwise she can simply request the backup to the watchtower every time she is online. In this way, it is possible to check that the data stored denotes the current status of all own channels.

**Steps:**
1. Ask to the connected nodes the current BlockHeight. Note that the nodes cannot cheat, otherwise they are banned from the network. If 51% of them send the same Block height, Alice'll consider that number.

2. Calculate the deterministic wallet address:

    address= m/108/0(mannet)/(number account)/0/Current_Blockheight. 

3. Calculate txidç_0 =2SHA256(address)and the hintç_0=txidç_0[:16]

4. Ask to the three watchtowers if one of them contains the hintç_0. 

    **case a**: If one of the watchtowers contains the hintç_0, it could also contain hintç_1 (case of more than one transaction in the same Block). So, calculate txidç_1=SHA256(txidç_0) and ask the watchtower if one of them contains the hintç_1. If one has the hintç_1, calculate txidç_2 and hintç_2. If no one has hintç_2, it is possible to conclude that the txidç_1 contains the last status channels and so Alice can request to the watchtowers the blolbç_1. If, one of the watchtowers contains hintç_2, calculate txidç_3 and so on.
  
    **case b**: If no one of the watchtowers has the hintç_0, means that Alice doesn't have any transactions in the current Block. So, in this case, we have to calculate the address utilizing m/108/0(mannet)/(number account)/0/(Current_Blockheight-1), then calculate hintç_0 and proceed as in the case a.
  
5. When Alice finds the hintç_n, she requests to the watchtower to send the correspond blobç_n. The blobç is composed by dataç and txidç[:16], where dataç=[txid_Bob, txid_Charlie, txid_Diana]

6. From dataç, Alice can recover the txid_Blob. then she can calculate the hint_Bob=txid_Blob[:16] and ask the watchtowers which one has that value and requests the corresponding blob_Bob. Since Alice knows txid_Blob[16:], she is able to get the data inside blob_Bob and recover the status channel with Bob. The same happens with the recovery of the status channel with Charlie and Diana.

7. In this way, Alice can check that the data in the watchtower are correct or otherwise recovery of all own data.


# Assumptions in the solution
To achieve this solution, I've assumed : 
* Fee: Nodes pays the service every month. Since Lightning Network is based on the onion routing, the watchtowers cannot know which nodes are paying the service, to which nodes are using the service.
* Memory: The data stored in the watchtower are never deleted or substituted.
* This protocol doesn't cover the case in which the node loses all information, I presume that a node maintains the most important information (seed and list of nodes to which is connected) in a safe place.



