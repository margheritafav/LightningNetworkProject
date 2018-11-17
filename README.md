# Lightning Network Project: Açai Protocol
This page aims to give an overview of my Master thesis project, but also is an open space to share ideas and knowledge useful to improve the current protocol. Any toughts/feedback would be really appreciated to proceed in the wisest way and find a solution that can also cover all the needs of community. 
If you are interested to this research topic, please do not hesitate to contact me for a possible collaboration.

# The problem
The problem that I'm focusing on is the recovering mechanism of false positive in the Lightning network. The new mechanism [Eltoo](https://blockstream.com/eltoo.pdf) permits to two nodes of each channel to share the last common status, so if one of the two nodes loses some information, it can simply ask to the other node to share the most recent status. 
This is a very useful solution, but unfortunately, presents some vulnerabilities. For example, this mechanism doesn't include the case that the other node doesn't share the last transaction, but instead an older one, more favorable for own balance. My project aims to solve this particular issue, to make the protocol more solid to face completely the case of a false positive node in the network. 

# Watchtower 
The main idea of the design is using watchtowers as back-up of the recent status channel. Suppose to have the network shown below, where Alice is connected to Bob, Charlie, Diana and to 3 watchtowers: W0, W1 and W2. 

![alt text](/Diagram.jpg)


Watchtower are currently using  is to handle breaches while client is offline. For example, if Alice is offline, Bob can send an older status channel to the blockchain, more favourable for him. In this case, the watchtower can discover the malicious act and intervene on time, even if Alice is offline. 
To make aware the watchtower of all last status channels, Alice has to send after every change of status a hint and a blob to the watchtower. 

1. hint = tixd[:16]
2. blob = Enc(data, txid[16:])
3. Send (hint, blob) to watchtower.

where the txid is an information on the channel status between Alice and Bob.

# Açai Protocol
The solution project, called Açai Protocol, aims to use the watchtower for backup. To this aim, it utilizes the three concepts of txid, hint and blob, but with the scope to be used for the backup service and not to monitor the channel status. To distinguish the two services, I'll introduce txidç, hintç and blobç. 
* txid, hint and blob as define in https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-April/001196.html
* txidç, hintç=txidç[:16] and blobç=Enc(dataç,txidç[16:]) for the backup. 

Every time that a status channels changes, Alice sends to one of the three watchtowers the channel status information (hint, blob), but also the pair (hintç, blobç) contained the backup information. Infact, dataç contains an array of txids: [txid_Bob, txid_Charlie, txid_Diana], where txid_Bob is the the transmission between Alice and Bob. The watchtowers will stores both pairs (hint, blob) and pairs (hintç, blobç), without distinguishing them. To maintain this features, dataç and data have to have the same dimension.

In this way the watchtower cannot distinguish the pair (hint, blob) for monitor channel status from (hintç, blobç).

In the following sections, I'm going to list all the steps on how the node Alice sends the data to the watchtower and how she can request the backup.

# Send the backup to the watchtower
1. Create a deterministic wallet address, *x* using the current Blockheight e.g.m/108/0(mannet)/(number account)/0/Current_Blockheight (we use 108 as purpose and O as account)
2. Calculate the txidç
txidç_0=2SHA256(*x*)
If Alice has another transaction with the same BlockHeight, there could be a conflict in the watchtower: two hintç with two different blobç. To mitigate this problem, everytime there is already a transaction in the same block, Alice will calculate the new txidç based on the txidç of the previous transaction: txidç_1= SHA256(txidç_0). Note that is not necessary to use 2SHA256, but it is enough SHA256. If there will be another change of channel status: txidç_2= SHA256(txidç_1), and so on...
Now, we consider the case that this change status channel is the first one of the block. 
- Split the txidç_0 in: hintç_0= txidç_0[:16] and blobç_0= Enc(dataç_0,txidç_0[16:])
- Alice sends hintç_0 and the blobç_0 to one of three watchtowers. Note that everytime, Alice can choose one of the watchtower randomly.



# How to request backup to the watchtowers
Scenario: Alice can requests the backup when she losees all her data(e.g. the software doesn't work properly) or otherwise she can simply request the backup to the watchtower to check that the data stored in the watchtower mirrors the current status of all channels.
1.  Ask to the connected nodes the current BlockHeight. Note that the nodes cannot cheat, otherwise they are benned from the network. Moreover, if 51% of them send the same Blockheight, Alice'll consider it.
2. Calculate the deterministic wallet address m/108/0(mannet)/(number account)/0/Current_Blockheight. 
3. Calculate txidç_0 =2SHA256(address)and the hintç_0=txidç_0[:16]
4. Ask to the three watchowers who contains the hintç_0. 
  **case a**: If one of the watchtowers contains the hintç_0,it will also contain hintç_1 (caso of more than one transaction in the same Block). So, calculate txidç_1=SHA256(txidç_0) and ask to the watchtower who contains the hintç_1. If one of the watchtower has the hintç_1, calculate txidç_2 and hintç_2. If no one of the watchtowers has hintç_2, it is possible to conclude that the txidç_1 contains the last status channels and request to the watchtowers the blolbç_1. If, one of the watchtower contains  hintç_2, calculate txidç_3 and so on..
  **case b**:  If no one of the watchtower has the hintç_0,  means that Alice doesn't have any transaction in the current Block. So, in this case we have to calculate the address utilizing m/108/0(mannet)/(number account)/0/(Current_Blockheight-1), calculating hintç_0 and proceed as in the case a.
5. When Alice finds her last backup, she requests to the watchtower to send the correspond blobç. The blobç is composed by: dataç and txidç[:16], where dataç=[txid_Bob, txid_Charlie, txid_Diana]
6. From dataç Alice can recover the txid_Blob, calculate the hint_Bob, ask to witchtower who has that, and requests the correspond blob_Bob. Since Alice knows txid_Blob, she is able to get the data inside blob_Bob and recover the status channel. The same can happen with Charlie and Diana.
7. In this way Alice can check that the data in the watchtower are correct or otherwise just collect data for the backup and recovery all own data.


# Assumptions in the solution
To achieve this solution, I've assumed : 
* Fee: Nodes pays the service every months. Since Lightning Network is based on onion routing, the watchtowers cannot know which nodes are paying the service, so which nodes are using the service.
* Memory: The data stored in the watchtower are never deleted.
* This protocol doesn't cover the case in which the node loses all information, I presume that a node maintains the most important information (public key, secret and list of nodes to which is connected) in a safe place.

