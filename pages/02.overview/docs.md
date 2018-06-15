---
title: Overview
taxonomy:
    category:
        - docs
visible: true
---

Secrets are meant to be shared only between the two. Modern technology — mostly web protocols powered by cryptography — has all the means to pass message from one party to another without giving access to anyone besides these parties; most obvious example are e2e encrypted chat apps like Signal or Keybase. But if we want to decouple moments of passing a message and possibility to read it — all the existing ways require trusting third party at some moment.

Bitcannery creates cryptoeconomics system for secure trustless deferred message relay . It achieves secure and trustless delivery of encrypted messages which could not be read before particular event in the future. This trigger event should be tied to the state of a blockchain and be verifiable by Smart contract. To get this happen, system is built on top of the network of third-party agents — 'keepers'. Keepers don't have any access to original message, they hold parts of secret required to decrypt the message and are motivated to disclose their parts of secret only after trigger event in the future.
Bitcannery was built and designed with number of use-cases in mind. In particular, it's well-suited for postponed secret messages and passing a secret text with dead-man switch.

## Problem

We want to deliver encrypted message at some future moment of time. By means of asymmetric encryption one could pass a message so only it's recipient could read it. But to achieve time-locking one should postpone the very delivery of the message. If sender must send message now and not later, they must refrain to trusting some third party to deliver the message, with nothing but the trust and contract forcing thar third party to do the delivery job. There's no easy way to eliminate the trust, but there are solutions for different use-cases.

There are time-locked encryption techniques [^1]. The main idea is to seal original message with resource- and time-consuming encryption, designed to be decrypted not faster than some timeout. While being solution to some of the use-cases, this way requires big amount of resources from the recipient and doesn't ensure exact timeout.

The simple and centuries-tested solution is to trust attorney to check the event and pass the message then required. Drawbacks are quite obvious: in case something happens with attorney before event, they wouldn't be able to perform their part of the contract.

For more specific use-cases there could be automated trustless solutions. For instance, if we want to transfer Ethereum funds or smart contract ownership, we could build smart contract which transfers funds or ownership if the first owner fails to check in during predefined time period.

## Current solutions

1. Attorneys: we trust them to act as we asked for, but couldn't really enforce this.
2. Email to the future (centralized web service)
3. Time-locked encryption: only one time-out, no enforcement, chances the message
will be decrypted almost instantly, doesn't fit dead-man-switch use-case. [^1]
4. 'Heritable' Ethereum Smart contract: well suited for passing funds and rights in form
of Smart contract ownership; doesn't fit free-form message relay use-case. [^2]
5. Specialized blockchain-based solutions:
- Mywish ([mywish.io](http://mywish.io)): implemets good traits of 'heritable' Smart contract + checks ownership transfer automatically with a network of actors; doesn't cover free-form message relay use-case.
- Keep network ([keep.network](http://keep.network)): supports robust suite of use-cases, but doesn't cover free-form message relay use-case. Keep was designed with more broad applicability in mind and seems not so well defended against different attacks. That being said, the project seems most viable of all exisiting blockchain-based options.
- Safe Heaven ([safehaven.io](http://safehaven.io)): uses key sharing methods, specializes on passing funds, doesn't cover free-form message relay use-case, has technical restrictions.
- Enigma ([enigma.co](http://enigma.co)): builds privacy layer for Smart contracts as a Layer 2 solution. Probably could be used for dead-man-switch use-case, but doesn't seem production-ready yet.

## Bitcannery

Bitcannery is a secure trustless deferred free-form message relay.
Bitcannery system leans on secret sharing techniques and the network of third-party actors keeping parts of secrets and incentivised to reveal them only in time of future event — be it timeout or passing canary checkin.

The main idea is to store message in the blockchain, but encrypt it so neither Bob nor anyone else could read it before trigger event and only Bob could after that event. As all the data in blockchain is open to read for anyone, we need to encrypt message at least two times: first to prevent anyone but Bob to read it (use asymmetric crypto), second — to prevent Bob from accessing his message before trigger event. To open the message after trigger event, Bitcannery uses a network of `keepers` — agents with access to parts of the decryption key. The key is divided with Shamir's secret sharing method [^3], so we don't need every single key part to be able to decrypt Bob's message. After saving encrypted key partsand twice-encrypted message to blockchain smart contract, Alice performs canary checkins within predefined time interval. After missed Alice's checkin keepers decrypt their key parts and send them to smart contract. Once enough keepers had revealed their key parts, Bob restores decryption key from keepers' shards, and then decrypts message from smart contract twice, receiving the Alice's message.

Note that Bitcannery could be used for both timed-out message delivery and dead-man-switch-like mechanics. The main difference would be trigger event for keepers to reveal their parts of the secret.
Some parts of current Bitcannery implementation are dependent on specific blockchain features. For reference implementation we've used Ethereum Smart contracts. 1) Smart contract couldn't call its own methods; 2) There's no timeouts for Ethereum Smart contracts; 3) There's no private data store for Ethereum Smart contracts.

1)+2) yields we need someone to call Smart contract to check whether Alice has missed her canary checkin or not; 3) yields the very impossibility to store only once-encrypted message and reveal it after timeout without any extra devices or actors.
That being said, one should note that Bitcannery protocol could be implemented on top of any blockchain with Smart contracts with comparable to Ethereum's properties.

There are finer details:
1. To get list of keepers to choose from, Alice starts her contract in `call for keepers` state. Actors interested in being a keeper for Alice contract send proposals with their payment rate and public keys. All Bitcannery contracts are registered in registry-contract, so keepers could find new ones to send proposals to.
After enough keeper proposals are gathered, Alice chooses which ones to accept. With keepers list accepted, she saves the message and key parts in smart contract and starts canary checkins.
2. Due to the fact that Ethereum Smart contracts couldn't initiate actions, we need third party actor to check whether trigger event has come or not. In current design this actors are contract keepers.
3. It's very likely not all the keepers will stay active at all times. If a keeper misses couple checkin periods in a row, he will no longer receive a keeping fee.
4. In case number of active keepers drops below reasonable level (for instance, is lower than a threshold for Shamir's secret sharing algorithm), Alice performs rotation procedure: opens new contract and marks existing one as closed and migrated.

## Domain of applicability

If we could get on by trusting some third-party like using attorney's service — we could be good without any complicated blockchain setup. This way the message could be kept secret, but we need to rely on the third-party to deliver the message. One could use asymmetric encryption to make the message unreadable for the third-party, but the very delivery still depends on it.

If we want to pass ownership of assets which could be put in the blockchain and don't need to make them hidden — for instance, that's the case for passing some quantity of ether — we could store this assets in smart contract and seal ownership with canary checkin scheme. If original owner hadn't checked in (sent transaction to method in smart contract) in predefined timeout, ownership could be claimed by 'successor' (reference implementation could be found in OpenZeppelin framework's Heritable Smart contract [^2]).

Bitcannery is designed to combine blockchain trustlessness and unstoppability with the message being free-form and unreadable to anyone before trigger event (and readable only to recipient after this event). This use-case suits well for passing personal notes or access keys for any real-world assets.

## Incentives

We need keepers to 1) keep their parts of the secret encrypted and unreachable before trigger event; 2) disclose their parts of the secret after trigger event.

Basic motivation for being a keeper is keeping fee. It's the ether (or any other value transmittable through blockchain) keeper gets for a promise to disclose his part of the secret after a trigger event. As we need some actor to check whether Alice has missed her canary checkin or not, we motivate keepers to check up a Smart contract during Alice's canary checkin period. On every keeper checkin keeper receives part of his keeping fee proportional to time from last checkin. If a keeper hadn't checked in for two consequent periods, he wouldn't get any further keeping fees.

As Bitcannery has all the history of operations for every keeper stored on blockchain, in later point of time it could incorporate reputation tracking in some form. Keepers with best reliability would get better chances to be accepted to new contracts or would be able to set bigger keeping fee.

Another technique under consideration is stakes-based incentive balancing. In order to function properly, Bitcannery should demotivate keepers from selling their key parts to Bob. One of the ways to prevent this from happening is to ask keepers to put some amount of assets to the stake before being accepted as keeper in any contract. In case it becomes known that keeper has revealed their part of the key to anyone before trigger event, one holding the decrypted key claims the stake. Using system-specific token gives another edge of motivation, as keeper wouldn't want to decrease the value of his stake by his bad actions.

## Risks and attacks
Bob wants to get his message before trigger event
1. Bob's address couldn't be found anywhere in the contract or in message saved to
blockchain. To receive the encrypted message, he attempts to decrypt every message with enough already submitted keeper key parts.

Keeper wants to sell his decrypted key part to Bob before trigger event
1. To reveal who Bob is, keepers must decrypt the message. It's impossible to keeper to
do this without gathering 66% of keepers' decrypted key parts.
2. Only way to prove Bob you're a real keeper and could decrypt your key part is to
reveal that key part. Once the key part is revealed, Bob has no motivation to pay for it.

Sybil attack by keeper
1. Alice chooses keepers by some algorithm (could be random, could be with some
optimisation technique — knapsack problem, more or less), so there's no strict way
to get threshold amount of keepers in the single contract.
2. Bitcannery encryption scheme is designed so that one needs to get decrypted key parts of
66% of keepers to be able to decrypt the message. That's bigger percentage than is needed to perform 51% attack on the blockchain (though of course these are non-comparable numbers).
3. Stakes system makes it expensive to get hold of enough key parts to be able to decrypt the message

Keepers become inactive and miss their checkins
1. Bitcannery uses Shamir's secret sharing method for the key Alice handles to keepers. This
method allows to decrypt message without every single keeper's key. Default redundancy level for Bitcannery contracts is 66% — we need two thirds of all the key parts to decrypt the message.
2. If number of active keepers comes close to minimal required, Alice could close original contract and create another one with the same message. Bitcannery calls this procedure “rotation”, and it's automated by client applications.

## Status and plans

We have: barebone implementation on top of Ethereum
We want in nearest future: 1) explore options of building stakes system; 2) explore options of bringing our own token into the system; 3) explore other blockchains for possible use. Our next steps: 1) deploy to Rinkeby; 2) mass-test there; 3) go for mainnet deployment

### References

[^1]: Gwern Branwen, Time-lock encryption. https://www.gwern.net/Self-decrypting-files

[^2]: OpenZeppelin, Heritable contract. https://github.com/OpenZeppelin/zeppelin-solidity/blob/master/contracts/ownership/Heritable.sol

[^3]: Shamir, Adi (1979), "How to share a secret", Communications of the ACM, 22 (11): 612–613, doi:10.1145/359168.359176.