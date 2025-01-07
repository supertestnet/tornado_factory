# Tornado factory
A bitcoin channel factory that works without a soft fork

# How it works

Have n people prepare and sign a coinjoin that funds an n of n multisig with Q sats apiece. It is important that Q be the same for every user and that every user has one of the keys to the multisig. Before the users share their coinjoin sigs with one another, they generate n presigned txs, to which each user gets a copy. Each presigned tx defines a “round” (e.g. round 1, round 2...round n).

The first round has n + 3 outputs. The first is worth 240 sats and is a p2a output. The second is worth Q - 240 - 330 sats and is locked to a “midstate” address to be described momentarily. The third is worth ( ( Q * n ) - Q ) - ( 330 * n ) and returns almost all of the remaining money to the multisig, ready for use in the next round. The amount not returned is equal to 330 * n and gets divided into n “connector outputs,” each worth 330 sats, and each one locked to an n-of-n multisig.

Every other round only has 3 outputs which are identical to the first three of the first round, except the third returns a smaller amount to the factory address each time (smaller by Q - 330), until, in the last round, there is no third output because there’s nothing to return to the factory address.

In each round, the withdrawer spends the first output created in that round (i.e. the p2a output) to pay the mining fee for withdrawing from the factory address. When those transactions confirm, the user spends the second output created in that round (i.e. the midstate output) to finalize his or her withdrawal, in a v3 transaction whereby they pay the fee in a subsequent transaction. Note that it is possible for another user to wait til a user creates the midstate and then initiate a race condition whereby two people both try to withdraw using the same midstate. If that happens, who wins the race is up to miners. If the "original" withdrawer loses, he can try again in the next round. However, it is sad that he lost the value of paying a transaction fee and will now have to redo it, and someone else effectively stole the value of a transaction fee from him. (But that person probably had to pay extra anyway in order to "bribe" miners to let him win.)

The midstate output is called that because it is locked to a “midstate address,” i.e. an address used when the user is in a state \*between\* initiating a withdrawal and finally withdrawing. The midstate address has n script paths, each of which lets a different user spend that utxo, but only with n-of-n signatures (i.e. a signature from every other party; these signatures are also generated before anyone deposits money into the multisig) and only if they also consume a particular one of the connectors. Consuming the connector ensures that user cannot withdraw in a subsequent round; their connector is gone, so the signatures which would have allowed them to withdraw in that round are invalid, because they commit to that connector as a second input, and it cannot be spent twice.

Thanks to the existence of the connector, a user can only withdraw from a single midstate address, and in order to do so they must supply the n-of-n signatures for their script path in the midstate and supply the n-of-n signatures for their connector, using both utxos as inputs. The n-of-n sigs force the user to send the money to two outputs: one is locked to a p2a output and the other is locked to an address the user chose in advance, before anyone deposited money into the multisig.

This scheme allows for the txid of the final “exit” transaction in each round to be known in advance, and thanks to that, the utxo info for each user’s final output is known; or rather, it is known that it will have one of n “utxo info options.” This means you can do lightning stuff with it. E.g. each user can pick a “channel counterparty” in advance and, before sharing their coinjoin sigs, the channel counterparty can ensure he has all the info he needs, in any round, to put the user’s funds in the address they chose in advance, which (he may ensure) is a 2 of 2 multisig (i.e. a channel) where he has one of the keys.

The user can ensure that he doesn’t share his coinjoin sigs until he has all the info he needs from his channel counterparty to exit that multisig in the initial state of a lightning channel, regardless of what round he exits in. And the user can then, without interacting with anyone in the pool other than his channel counterparty, update the state of his lightning channel to send and receive payments.
