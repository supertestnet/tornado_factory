# Tornado factory
A bitcoin channel factory that works without a soft fork

# How to try it

Just click here: https://supertestnet.github.io/tornado_factory/

Note: this idea builds on [a previous idea](https://github.com/supertestnet/hurricash) I had called hurricash. If you find tornado factory difficult to grok, consider starting there.

Also note: despite saying this is a "channel factory," it's only a POC and does not give users actual channels. It just gives them a unilaterally withdrawal utxo whose txid, vout, amount, and address they know in advance. But since that info is reliably available from the start, devs can make a channel using that info and treat them as "real" even before they go on chain (and even if they \*never\* go on chain). I am working on software to demonstrate this feature [here](https://github.com/supertestnet/hedgehog_factory).

# How it works

Have n people prepare and sign a coinjoin that funds an n of n multisig with Q sats apiece. It is important that Q be the same for every user and that every user has one of the keys to the multisig. Before the users share their coinjoin sigs with one another, they generate n presigned txs, to which each user gets a copy. Each presigned tx defines a “round” (e.g. round 1, round 2...round n).

The first round has a number of outputs equal to k + 3, where the value of k will be explained shortly. The first output is worth 240 sats and is a p2a output. The second is worth Q - 240 - 330 sats and is locked to a “midstate” address to be described momentarily. The third is worth ( ( Q * n ) - Q ) - ( 330 * n ) and returns almost all of the remaining money to the multisig, ready for use in the next round. The amount not returned is equal to 330 * n and gets divided into k “connector creation outputs.” The connector creation outputs are each an n of n multisig. Presigned transactions allow anyone to spend these outputs to create a "connector tree" of additional n of n multisig outputs. The leaves of this tree are n "connector outputs" with a value of 330 sats apiece.

Every other round only has 3 outputs which are identical to the first three of the first round, except the third returns a smaller amount to the factory address each time (smaller by Q - 330), until, in the last round, there is no third output because there’s nothing to return to the factory address.

In each round, the withdrawer spends a connector creation output and, if necessary, its children, til he or she creates a particular leaf of the connector tree that is, from the start, uniquely pertinent to him or her. Then the withdrawer spends the first output created in that round (i.e. the p2a output) to pay the mining fee for withdrawing from the factory address. The withdrawer must wait til at least 2 blocks go by to finalize his or her withdrawal, because the transaction doing so spends his or her leaf of the connector tree as an input, and the signatures for spending that leaf are all signed with a sequence number of 2.

Once two blocks go by, the withdrawer spends the second output created in that round (i.e. the midstate output) to finalize his or her withdrawal, in a v3 transaction that does three things: (1) it consumes two inputs (the midstate and the withdrawer's leaf), (2) it allows the withdrawer to pay the fee in a subsequent transaction, and (3) it creates two outputs. One is an address chosen by the user in advance, before anyone deposited any money to the multisig, and the other is a new n-of-n connector output (worth 246 sats) whose existence constitutes proof that the user exited the channel factory, and whose purpose will be explained later. I call this connector the proof-of-exit connector.

It is possible for another user to wait til a user creates the midstate and then initiate a race condition whereby two people both try to withdraw using the same midstate. However, they can only do this if their connector leaf was created in a block equal to or less than the "legitimate" withdrawer's connector leaf. If that circumstance pertains and both users try to withdraw using the same midstate, the winner of the race is up to miners. If the "original" withdrawer loses, he or she can try again in the next round. However, it is sad that he or she lost the value of the transaction fee he or she paid to create the midstate, and will now have to try to exit again, paying a new transaction fee to create a new midstate.

If the circumstance described above pertains, it means a midstate-thief got away with using a midstate without paying for it. But I wonder if game theory makes this circumstance unlikely, for the following reason: the thief has no greater chance of winning the race condition than the legitimate user unless they pay extra in fees to "bribe" miners to favor their transaction, in which case, they've proven their willingness to pay extra to leave, so why didn't they just pay to create their own midstate and avoid a race that they might lose?

Nonetheless, to mitigate the chance of theft, if any user creates a connector, it seems to me that a presigned transaction should allow anyone to burn their funds and their connector if they do not finalize their withdrawal within 4032 blocks (4 weeks). That way, there is no reason to create your connector unless you plan to withdraw as soon as possible; and if a user wants to withdraw but sees that another user already created their connector and hasn't withdrawn yet, they can avoid a race condition and the risk of falling victim to a midstate-thief by waiting to withdraw til after the other user withdraws or their money gets burned.

The midstate output is called that because it is locked to a “midstate address,” i.e. an address used when the user is in a state \*between\* initiating a withdrawal and finally withdrawing. The midstate address has n script paths, each of which lets a different user spend that utxo, but only with n-of-n signatures (i.e. a signature from every other party; these signatures are also generated before anyone deposits money into the multisig) and only if they also consume a particular one of the connectors. Consuming the connector ensures that user cannot withdraw in a subsequent round; their connector is gone, so the signatures which would have allowed them to withdraw in that round are invalid, because they commit to that connector as a second input, and it cannot be spent twice.

Thanks to the existence of the connector, a user can only withdraw from a single midstate address, and in order to do so they must supply the n-of-n signatures for their script path in the midstate and supply the n-of-n signatures for their connector, using both utxos as inputs. The n-of-n sigs force the user to send the money to two outputs: one is locked to a p2a output and the other is locked to an address the user chose in advance, before anyone deposited money into the multisig.

This scheme allows for the txid of the final “exit” transaction in each round to be known in advance, and thanks to that, the utxo info for each user’s final output is known; or rather, it is known that it will have one of n “utxo info options.” This means you can do lightning stuff with it. E.g. each user can pick a “channel counterparty” in advance and, before sharing their coinjoin sigs, the channel counterparty can ensure he has all the info he needs, in any round, to put the user’s funds in the address they chose in advance, which (he may ensure) is a 2 of 2 multisig (i.e. a channel) where he has one of the keys.

The user can ensure that he doesn’t share his coinjoin sigs until he has all the info he needs from his channel counterparty to exit that multisig in the initial state of a lightning channel, regardless of what round he exits in. And the user can then, without interacting with anyone in the pool other than his channel counterparty, update the state of his lightning channel to send and receive payments.

I have not yet explained the purpose of the proof-of-exit connector created when a user exits. The reason it exists is to enable the following mechanism which allows the routing nodes to disperse the funds from the multisig to one another even if some people get ejected via the sad path. They do this by having everyone create two sets of signatures during the signing ceremony, called "proof-of-exit signatures" and "disappearance signatures." Both sets of signatures send all of the money left after any round to the routing nodes, including any money in the most recently created midstate (if any), but they only work if the total number of proof-of-exit signatures and disappearance signatures used is equal to the number of users.

The validity of a proof-of-exit signature constitutes proof that a user exited because it spends that user's exit connector as an input; if that input exists, that user departed with their money, so they can safely create the proof-of-exit signature without risking the loss of their own money. The validity of a disappearance signature constitutes proof that a user disappeared because it spends that user's leaf of the connector tree as an input, but only when 2016 blocks go by after its creation. If that connector utxo exists after such a long time, the protocol considers that user to have disappeared, and it is better if the routing nodes get their money than for it to stay stuck in the pool forever.

Note that, when this protocol is used to create channels, it might seem inadvisable to allow a user's funds + connector to be burned if they fail to withdraw within 4032 blocks; burning their funds would prevent their disappearance signature from being usable when routing nodes wish to disperse the funds, so routing nodes would have to broadcast every "sad path" transaction to disperse the remaining funds, which is inefficient. But I do not consider this to be a significant danger, for two reasons: a user would hardly choose to eject themselves and then wait 4032 blocks for their money to get burned, because they would lose all their money by doing so; and even if they chose to do so, their routing node would probably finalize their withdrawal for them so as not to lose *their* money.

Therefore, a user's funds are likely only ever to be at risk of getting burned if *the routing nodes* eject them; and the routing nodes would likely only do so in preparation for sweeping all of the funds from the multisig by a combination of proof-of-exit signatures and disappearance signatures. Since disappearance signatures become valid *before* anyone-can-burn signatures becomes valid, this means a user who disappears will probably just stay in the channel factory until the routing nodes are ready to sweep the funds from the channel factory, and if they are ready to do that, there's no danger of the funds getting burned -- instead, the user who disappeared will just lose custody of them due to his or her disappearance.

Upon reflection, I no longer think the disappearance sigs and proof-of-exit connectors are very useful. They multiply the number of signatures that everyone has to create and only really work if precisely one user disappeared or used a sad path to exit. In order to make them work if \*any number\* of users disappeared or exited via the sad path, the number of signatures necessary seems to blow up into n! proportions, because you have to sign for the case where all 15 people disappear or use the sad path, and then, in case 1 person did *not* do so, you have to sign *again* for the case where only 14 people disappeared or used the sad path, and so forth. So you end up needing to create more than n! signatures to make this work for all possible cases.

There might be a way to make it work for only a *subset* of cases (e.g. every user could provide a proof that they've worked happily with their routing node in the past, and if the routing nodes trust one another, they could skip making signatures that assume such users will disappear, and only do so for people they've never worked happily with before). But I don't want to think about that any more so let this be the end for now.
