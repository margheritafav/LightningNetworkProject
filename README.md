# Açai: a backup protocol for Lightning Network wallets
This page aims to give an overview of my Master thesis project at the Technical University of Denmark, but also is an open space to share ideas and knowledge useful to improve the current protocol. Any thoughts/feedback would be really appreciated, to help me proceeding in the wisest way and find a solution that can also cover all the needs of Lightning Network users.

This work has been accepted for the Scaling Bitcoin Workshop, which will take place September 11th-12th in Tel Aviv. The presentation video will be added on this page as soon as available.

If you are interested in this research topic, please do not hesitate to contact me for a possible collaboration at s170065@student.dtu.dk.


# The problem
The problem I'm focusing on is the recovery of the unspent bitcoins stuck inside the Lightning Network after a wallet failure (e.g. lost or corrupted transaction data put into the wallet storage).


The [Lightning Network](#) provides higher speeds and confidentiality for bitcoin transactions, but the absence of the underlying distributed ledger makes impossible the recovery of unspent transactions through the traditional cryptographic seed and the BIP32 address derivation.


I have seen an opportunity within the new mechanism [Eltoo](https://blockstream.com/eltoo.pdf), which expects the two nodes of an open channel to share the same state(or ‘commitments’) for their unspent bitcoins. By leveraging Eltoo one of the two nodes could try to recover missing data from the counterpart, since its unspent bitcoins are stored inside the same ‘commitment’.
This could be a very useful solution, but, unfortunately, presents key vulnerabilities. The biggest one, comes from the case that an adversarial node (Eve) doesn't respond with the latest channel state to the user (Alice), providing instead a previous one. The LN-Penalty fee, involuntary triggered by Alice, represents an economic incentive for Eve to act adversarially, increasing the risks for Alice to ask for help. My project aims to solve this particular issue, and to make the protocol less vulnerable to involuntary triggers of LN-Penalty fees as false positives.

Moreover, my idea of leveraging Eltoo to mitigate the issue is still missing a remediation comparable to the mentioned seed + BIP32 recovery which is provided today by any Bitcoin deterministic wallet. 
 


# Watchtower 
Further exploring upcoming Lightning Network features, the Watchtowers, as have been outlined so far, will provide a protection service to offline nodes, by storing their encrypted ‘commitments’. Watchtowers are designed to broadcast, on the blockchain, the latest channel state if an adversarial node tries to spend an old commitment, exploiting the offline state of its counterpart.

As an example, let's suppose to have a circumstance where Alice has open channels with Bob, Charlie, Eve and is using 3 watchtowers: W0, W1 and W2. 

Eve, exploiting the situation of Alice being offline or unable to broadcast her ‘commitments’ could try to broadcast an older channel state to the blockchain, essentially stealing the last payment done to Alice. In this event, a Watchtower is capable to identify this malicious transaction from the mempool and intervene in defense of Alice, even if she is offline. 

Alice, in order to keep the watchtowers aware of the latest channel state has to send a [**hint**](https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-April/001196.html) and a [**blob**](https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-April/001196.html) after every update to her ‘commitments’ with Bob, Charile or Eve.


For example, the sequence of events after a Lightning payment between Alice and Bob is:

1. Change of the channel state between Alice and Bob (a new Eltoo commitment is signed).
2. Alice calculates the variable ‘hint’ by truncating the commitment hash (which is also the txid): hint = tixd[:16]
3. Alice calculates the payload ‘blob’ by encripting the commitment itself, using the hash/txid as the simmetric key: blob = Enc(data, txid[16:])
4. Alice sends (hint, blob) to  one of the watchtowers W0, W1 or W2.


Therefore, some of the Watchtowers are storing the information that Alice needs in case of failure, without the adversarial risks from asking this data to Bob, Charlie or (worse) Eve.


If we imply a process where Alice can probe Watchtowers availability, by randomly retrieving her blobs through the list of hints, we only need a way to backup and restore the list of all of the hash/txid still unspent inside the Lightning Network.

Açai Protocol stores this list of commitments inside a properly crafted ‘blob’, using ‘hints’ as unique data pointers and the Watchtowers for the backup itself.
The key to decrypt this special ‘blob’ is a derived BIP32 public-key, generated by using the latest ‘blockheight’ as a rough timestamp.
This final step enables the advantages of [BIP39](#) passphrases to recover funds stored also inside the Lightning Network.

# Açai Protocol
This solution , named Açai Protocol after the controversial berry fruit, aims to use the watchtowers not just for monitoring the channels, but also as a backup service. 
This protocol adopts the same concepts of txid, hint and blob. In order to distinguish the two types of data payload, I'll use a different notation: txidç, hintç and blobç. 


Therefore:
*  txid, hint and blob represent the standard format used by Alice to interact with the watchtowers
*  txidç,hintç and blobç represent the format used by Alice’s wallet to store unspent txid inside an Açai backup. 

Technically, Açai Protocol leverages the same entpoints:
* hintç = txidç[:16]

* blobç = Enc(dataç,txidç[16:])

In the Açai protocol, dataç contains an array of txid: [txid_Bob, txid_Charlie, txid_Eve], where (e.g.) txid_Bob is the hash of the Eltoo commitment between Alice and Bob.
Therefore, once Alice is capable to retrieve and decode her Açai blob, she can also identify the list of txid stored inside the watchtowers.
Every time that a channel changes its state, Alice sends to one of the three watchtowers W0, W1 or W2 (randomly chosen) two different payloads: the channel state information (hint, blob),and the Açai blob(hintç, blobç) containing the updated list of txid. Note that the watchtowers will store both the standard blobs (hint, blob) and the Açai blob (hintç, blobç), without being able to distinguishing them. 
We can imply that, an Açai’s blobç has length and size comparable to standard Watchtowers blobs.

In the following sections, I'm going to list all the steps needed by Alice’s wallet to  send commitments data to the watchtowers and how she can request the backup.


## Send the backup to the watchtower
**Scenario:** after a payment to Bob, the channel state changed, so Alice has to send the new channel information (hint, blob) and the new backup (hintç, blobç) to watchtower W0.

**Steps:**
1. To generate txidç, which represent the key to identify and decrypt blobç, Alice uses its deterministic wallet address function, applying the current Block height as the derivation variable to generate the pub-key.

    e.g. *pub-key*= m’/108’/0(mainnet)/(account number)’/0/Current_Blockheight 

    Where 108 is an arbitrary purpose number, and account number is needed to match the wallet subject to restore process.
    
2. Calculate the txidç_n. 

   Since it's possible for Alice to perform many payments in the same Blockheight, we must enumerate each new state of the Açai backup inside the same Blockheight as: txidç_0, txidç_1, txidç_2, txidç_3...
   
    Therefore, we can assume that we need to change the hintç while keeping the same Blockheight in the BIP32 derivation function, so:
    
    txidç_0= 2SHA256(*pub-key*)
    
    As soon as Alice needs to update her Açai backup, but the Blockheight is not yet changed, we can apply the following rule: the txidç_n is calculated as the hash function of the previous txidç_n-1.
    
    For example: 
    
                 txidç_1= SHA256(txidç_0)
    
                 txidç_2= SHA256(txidç_1)
             
                 txidç_3= SHA256(txidç_2)
             
                 .....

3. Truncate the txidç_n into hintç_n= txidç_n[:16] and encrypt blobç_n= Enc(dataç_n,txidç_n[16:])
4. Send hintç_n and the blobç_n to watchtower W0, with the information of the new channel status (hint,blob).


## How to request backup to the watchtowers
**Scenario:** Alice can request the backup when she has lost all her data accidentally or otherwise she can simply request the backup to the watchtower every time she is online. In this way, she can check that the watchtowers are storing real data and they are providing the promised service.

**Steps:**

1. Ask to the connected nodes the current Blockheight. Note that the nodes cannot cheat, otherwise they could be excluded by the network.

2. Use the Blockheight and the seed to calculate the deterministic pub-key:

    address= m/108’/0(mainnet)’/(account number’)/0/Current_Blockheight. 

3. Calculate txidç_0 = 2SHA256(pub-key) and the hintç_0=txidç_0[:16]

4. Ask to W0, W1 and W2 watchtowers to retrieve hintç_0. 

    **case a**: If one of the watchtowers contains the hintç_0, it could also contain hintç_1 (case of more than one transaction in the same Block).
    Therefore, Alice’s wallet calculates txidç_1=SHA256(txidç_0) and asks the watchtowers if one of them contains the hintç_1.      If one has the hintç_1, calculate txidç_2 and hintç_2 and so on.
    If no one has hintç_2, we can assume that  txidç_1 contains the latest channel state so Alice can decrypt blolbç_1 and extract the list of txid.

    **case b**: If no one of the watchtowers provides  hintç_0, we can assume  that Alice doesn't have any transactions at the current Blockheight. 
    In this case, Alice’s wallet generates a new pub-key, decreasing the blockheight by one: pub-key = m/108’/0(mainnet)’/(account number)’/0/(Current_Blockheight-1). Once generated, Alice’s wallet calculates hintç_0 and proceed as in the case a.
  
5. When Alice’s wallet finds her hintç_n, she requests to the watchtower to send the correspond blobç_n. The blobç is composed by dataç and txidç[:16], where dataç=[txid_Bob, txid_Charlie, txid_Eve]

6. From dataç, Alice can recover the txid_Blob. then she can calculate the hint_Bob=txid_Blob[:16] and ask the watchtowers which one has that value and requests the corresponding blob_Bob. Since Alice knows txid_Blob[16:], she is able to get the data inside blob_Bob and recover the status channel with Bob. The same happens with the recovery of the status channel with Charlie and Eve.

7. At this point, Alice is able to check that her data matchest the ones stored in the watchtowers, or otherwise recovery her data.


# Assumptions in the solution
To achieve this solution, I assumed : 
* Fees: Watchtowers model still needs a form of trustless payment to be economically viable, e.g. to cover the extra storage and bandwidth for the service (both for the normal service and the Açai backup)
* Memory: The Açai data stored in the watchtowers are not deleted or substituted/tampered.
* Açai Protocol doesn't cover the case where Alice has no idea of which watchtowers she used and when. 






