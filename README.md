# Tornado factory
A bitcoin channel factory that works without a soft fork

# How it works

Have n people prepare and sign a coinjoin that funds an n of n multisig with Q sats apiece. It is important that Q be the same for every user and that every user has one of the keys to the multisig. Before the users share their coinjoin sigs with one another, they generate n presigned txs, to which each user gets a copy. Each presigned tx defines a “round” (e.g. round 1, round 2...round n).

The first round has a number of outputs equal to k + 3, where the value of k will be explained shortly. The first output is worth 240 sats and is a p2a output. The second is worth Q - 240 - 330 sats and is locked to a “midstate” address to be described momentarily. The third is worth ( ( Q * n ) - Q ) - ( 330 * n ) and returns almost all of the remaining money to the multisig, ready for use in the next round. The amount not returned is equal to 330 * n and gets divided into k “connector creation outputs.” The connector creation outputs are each an n of n multisig. Presigned transactions allow anyone to spend these outputs to create a "connector tree" of additional n of n multisig outputs. The leaves of this tree are n "connector outputs" with a value of 330 sats apiece.

Every other round only has 3 outputs which are identical to the first three of the first round, except the third returns a smaller amount to the factory address each time (smaller by Q - 330), until, in the last round, there is no third output because there’s nothing to return to the factory address.

In each round, the withdrawer spends a connector creation output and, if necessary, its children, til he or she creates a particular leaf of the connector tree that is, from the start, uniquely pertinent to him or her. Then the withdrawer spends the first output created in that round (i.e. the p2a output) to pay the mining fee for withdrawing from the factory address. The withdrawer must wait til at least 2 blocks go by to finalize his or her withdrawal, because the transaction doing so spends his or her leaf of the connector tree as an input, and the signatures for spending that leaf are all signed with a sequence number of 2.

Once two blocks go by, the withdrawer spends the second output created in that round (i.e. the midstate output) to finalize his or her withdrawal, in a v3 transaction that does three things: (1) it consumes two inputs (the midstate and the withdrawer's leaf), (2) it allows the withdrawer to pay the fee in a subsequent transaction, and (3) it creates two outputs. One is an address chosen by the user in advance, before anyone deposited any money to the multisig, and the other is a new n-of-n connector output (worth 246 sats) whose existence constitutes proof that the user exited the channel factory, and whose purpose will be explained later. I call this connector the proof-of-exit connector.

Note that it is possible for another user to wait til a user creates the midstate and then initiate a race condition whereby two people both try to withdraw using the same midstate. However, they can only do this if their connector leaf was created in a block equal to or less than the "legitimate" withdrawer's connector leaf. If that circumstance pertains and both users try to withdraw using the same midstate, the winner of the race is up to miners. If the "original" withdrawer loses, he or she can try again in the next round. However, it is sad that he or she lost the value of the transaction fee he or she paid to create the midstate, and will now have to try to exit again, paying a new transaction fee to create a new midstate.

If the circumstance described above pertains, it is sad because it means the midstate-thief got away with using a midstate without paying for it. But that person probably had to pay extra anyway in order to "bribe" miners to let him or her win. Nonetheless, to mitigate this circumstance, if someone creates their connector, it seems to me that a presigned transaction should burn their funds and their connector if they do not finalize their withdrawal within 4032 blocks (4 weeks). That way, there is no reason to create your connector unless you are planning to withdraw in this round, and if a user sees that another user already created their connector, they can just wait til that user withdraws or their money gets burned before attempting to withdraw themselves and risking paying to make a midstate in vain.

The midstate output is called that because it is locked to a “midstate address,” i.e. an address used when the user is in a state \*between\* initiating a withdrawal and finally withdrawing. The midstate address has n script paths, each of which lets a different user spend that utxo, but only with n-of-n signatures (i.e. a signature from every other party; these signatures are also generated before anyone deposits money into the multisig) and only if they also consume a particular one of the connectors. Consuming the connector ensures that user cannot withdraw in a subsequent round; their connector is gone, so the signatures which would have allowed them to withdraw in that round are invalid, because they commit to that connector as a second input, and it cannot be spent twice.

Thanks to the existence of the connector, a user can only withdraw from a single midstate address, and in order to do so they must supply the n-of-n signatures for their script path in the midstate and supply the n-of-n signatures for their connector, using both utxos as inputs. The n-of-n sigs force the user to send the money to two outputs: one is locked to a p2a output and the other is locked to an address the user chose in advance, before anyone deposited money into the multisig.

This scheme allows for the txid of the final “exit” transaction in each round to be known in advance, and thanks to that, the utxo info for each user’s final output is known; or rather, it is known that it will have one of n “utxo info options.” This means you can do lightning stuff with it. E.g. each user can pick a “channel counterparty” in advance and, before sharing their coinjoin sigs, the channel counterparty can ensure he has all the info he needs, in any round, to put the user’s funds in the address they chose in advance, which (he may ensure) is a 2 of 2 multisig (i.e. a channel) where he has one of the keys.

The user can ensure that he doesn’t share his coinjoin sigs until he has all the info he needs from his channel counterparty to exit that multisig in the initial state of a lightning channel, regardless of what round he exits in. And the user can then, without interacting with anyone in the pool other than his channel counterparty, update the state of his lightning channel to send and receive payments.

Note: I have not yet explained the purpose of the proof-of-exit connector created when a user exits. The reason it exists is to enable the following mechanism which allows the routing nodes to disperse the funds from the multisig to one another even if some people get ejected via the sad path. They do this by having everyone create two sets of signatures during the signing ceremony, called "proof-of-exit signatures" and "disappearance signatures." Both sets of signatures send all of the money left after any round to the routing nodes, including any money in the most recently created midstate (if any), but they only work if the total number of proof-of-exit signatures and disappearance signatures used is equal to the number of users.

The validity of a proof-of-exit signature constitutes proof that a user exited because it spends that user's exit connector as an input; if that input exists, that user departed with their money, so they can safely create the proof-of-exit signature without risking the loss of their own money. The validity of a disappearance signature constitutes proof that a user disappeared because it spends that user's leaf of the connector tree as an input, and only after 2016 blocks go by upon its creation. If that input exists after such a long time, the protocol considers that user to have disappeared, and it is better if the routing nodes get their money than for it to stay stuck in the pool forever.
