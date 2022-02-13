---
title: Playing Tic-Tac-Toe as the world watches — An introduction to BlockChain and State Channels (Part 2)
date: 2020-08-15 09:00:00 +0530
categories: [Blockchain, Programming]
tags: []     # TAG names should always be lowercase
math: true
mermaid: true
toc: true
---

# Introduction

**This two-part series is a brief introduction to the concepts of Ethereum and State Channels by building a Tic-Tac-Toe**
**application. Read Part 1 [here](https://medium.com/@saihemanth9019/playing-tic-tac-toe-as-the-world-watches-an-introduction-to-blockchain-and-state-channels-part-78aa2b01f76b).**

You have solved all the conventional issues of playing a bet match of Tic-Tac-Toe with the use of Smart Contracts on
the Ethereum network. Now, you are facing foundational problems of a Blockchain itself which are speed and cost of
transactions.

## Why are transactions on Blockchain slow?

A Blockchain is fundamentally a set of blocks and each block is made up of transactions. Only transactions included
in blocks are considered to be valid. A block can consist of a predefined number of transactions only. In Ethereum,
the average number of transactions in a block is 70 and the average block creation time is around 13 seconds. One
may ask, why not just create blocks instantly? To create a block i.e., validate a set of transactions, one needs to
solve an extremely difficult cryptographic puzzle which takes a considerable amount of time. This process is known as
mining.

When a Blockchain experiences high traffic, the transactions are destined to wait for long periods of time by the design
itself. Even if there are no other transactions waiting to be processed, there is a minimum time for the transaction to
be included into a block. To make things even worse, the decentralized nature of a Blockchain can cause multiple miners
(ones who create blocks) from different parts of the world to create blocks containing similar transactions. Only one
of these blocks will be accepted into the Blockchain and the other blocks get discarded.

Read more on resolving forks [here](https://www.mangoresearch.co/blockchain-forks-explained/).

To make sure that a block B containing a transaction is definitely included in the blockchain and accepted by the
network, it is advised to wait for a minimum number of blocks (6 blocks in Bitcoin, which takes around 1 hour) to be
created after B. The affirmation that a block will not be revoked once added to the blockchain is known as Finality.
Ethereum has a finality time of 2.5 minutes.

## Why are transactions on Blockchain expensive?

As mentioned before, the miners need to solve a cryptographic puzzle using their computers to create a block and
include your transaction. Computers need electricity and hardware, neither of which are free. To make the act of
mining profitable, the transactions require a transaction fee (gas in Ethereum) which the miner gets rewarded. Hence,
the transaction fee depends on various factors such as average electricity and hardware costs to perform mining.

> **Note:** The earlier technical details are not required to understand the following sections.

# The Solution

**Don’t use a Blockchain to solve petty problems.**

Not every piece of information needs to be on the Blockchain. Identifying what needs to go on a Blockchain is
crucial. Do we really need the world to watch "everything"?

The stages in which we need the intervention of a smart contract are,

1. **Initial Funding** — The smart contract needs to store the bets before the game begins to ensure that even if one
   player is malicious, he/she is penalized.
2. **Invalid move** — When a player makes an invalid move, such as making multiple actions (crossing off multiple
   boxes) in one turn, or not making a move at all, the smart contract needs to be able to act as a judge and penalize
   the act.
3. **Inactivity** —The smart contract needs to take the correct decision when a player refuses to make the next move.
4. **Withdrawing** — At the end of the game, based on the verdict, the funds need to be distributed.

Not so surprisingly, these were the issues we began with! We need to design a Smart Contract to solve only these
problems and nothing more.

Since the Smart Contract will no longer track and validate every state transition of the Tic-Tac-Toe board, how do the
players know what the current state of the board is? We use a messaging channel between players to exchange player
moves and updated board states.

This brings up a crucial problem, how do we prove that there is no impersonation? How do we prove that if one receives
a message from Alice, the message is actually created and sent by Alice?

## Digital Signatures

Every Ethereum account is identified by a private/public key. These keys have one peculiar property. It is possible
to encrypt any data with the public key and this encryption can only be reversed with the private key. This is known
as [asymmetric key encryption](https://en.wikipedia.org/wiki/Public-key_cryptography). Interestingly, the reverse is
also possible.

We can encrypt data with the private key which can only be reversed by the corresponding public key. This forms the
basis of digital signatures. One can encrypt a publicly known message using the private key (which is only known to
the owner of the account). Anyone can now decrypt the encrypted message using the sender’s public key and verify that
the sender possesses the private key i.e., it is actually the owner of the public key.

![](https://miro.medium.com/max/700/1*pQT2HqmNALvJR3ubFK6alQ.png)

Before sending any move as a message, the player can simply attach a digital signature (i.e., encryption of message
using private key) of the move to prove that he is not being impersonated.

For convenience, the game can be divided into a set of states,

## Intent State

Before a game even begins, the players need to share an intent to play a game together. Alice and Bob will each create
a message of the following format and send to their opponent.

```
state: "INTENT"
playerAddress: "<Player's Ethereum Public Key Here>"
betAmount: 10
state: "<Initial state of the tic-tac-toe board>"
contractAddress: "<Address at which the smart contract is hosted>"
```

In this message, only the address is allowed to be different for the player intents. Otherwise, the intent is
discarded. contractAddress is the agreed upon smart contract to be used. Once a player receives and sends an
intent he agrees with, it is decided that the game is happening.

## Depositing Funds

Players will now call `deposit()` function on the smart contract and deposit the bet amounts.

```cpp
deposit(Intent senderIntent, Intent opponentIntent) {
  require(hasValidSignature(senderIntent) && hasValidSignature(opponentIntent));

  // Check if the intents are matching in all fields except playerAddress
  require(areValidIntents(senderIntent, opponentIntent));
  require(sender == senderIntent.address);
  require(sender.amountSent == senderIntent.betAmount);
  
  if (!aliceHasDepositedFund) {
    // The first player depositing will be called Alice
    aliceHasDepositedFunds = true;
    alice = senderIntent.address;
  } else {
    // The second player will be called Bob 
    bobHasDepositedFunds = true;
    bob = senderIntent.address;
  }
}
```

The first player depositing the betAmount is assigned as Alice, and the next player will be Bob. The function requires
intents of both players before accepting any funds.

## Move State

Once both players see that the bets have been deposited by checking the state of the Smart Contract, Alice makes her
first move and sends a message of the following format to Bob along with the digital signature through the messaging
channel.

```
type: "MOVE"
state: "<Board state after making the first move>"
turn: 1
player: "<Alice's public key>"
```

> Note: Reading the state (variables) of a Smart Contract do not need a transaction. Only state modifications
> require a transaction.

Bob makes the next move in a similar way.

```
type: "MOVE"
state: "<Board state after making the second move>"
turn: 1
player: "<Bob's public key>"
```

Since, both players are friendly, the game continues in a fair manner.

> Note: One turn consists of two moves, first by Alice and then by Bob.

The game proceeds with incrementing turn numbers. At the final turn, considering Alice wins, she makes the last move.

## Withdraw

Either of the players can call the `withdraw()` function (once the final move of the game is done) on the smart
contract by sending the last two moves of the game.

```cpp
withdraw(Move preFinalMove, Move finalMove) {
  // Before executing,
  // 1. Check if signatures are correct
  // 2. Check if finalMove has a final board state
  //    i.e., no more move can be performed.
  // 3. Check if preFinalMove -> finalMove is a valid transition

  require(isGameOver == false);
  
  isGameOver = true;
  
  if (aliceHasWon(finalMove)) {
    sendAmount(alice, betAmount + betAmount);
  } else if (bobHasWon(finalMove)) {
    sendAmount(bob, betAmount + betAmount);
  } else {
    // Game has tied.
    sendAmount(alice, betAmount);
    sendAmount(bob, betAmount);
  }
}
```

The last two moves, one by each player, are required to avoid a single player from unanimously creating a final move
and withdrawing funds.

## What happens when a player makes an invalid move or stays inactive?

Players can always reject invalid moves by the opponent. Hence, it can be considered to be a special case of opponent
inactivity, where the other player refuses to make a valid move. In these scenarios, the active player can call the
`reportInactivity()` function of the Smart Contract by sending the move made by him and the last move made by opponent.

```cpp
// Assigning a dummy address 0 initially.
address inactivePlayer = address(0);
Move lastMove = null;

reportInactivity(Move currentMove, Move lastOpponentMove) {
  // Check if,
  // 1. Signatures are valid.
  // 2. lastOpponentMove -> currentMove is a single valid move
  
  inactivePlayer = lastOpponentMove.player;
  lastMove = currentMove;
}
```

If Bob decides to make a valid move now, he can call the respond() function with the next valid move.

```cpp
Move respondMove = null;

respond(Move nextMove) {
  // Check if
  // 1. Signature is valid.
  // 2. The sender is the inactivePlayer.
  require(sender == inactivePlayer);
  // 3. Valid single transition from lastMove -> nextMove.
  
  respondMove = nextMove;
  inactivePlayer = address(0);
  lastMove = null;
}
```

Alice can now read Bob’s next move from the respondMove variable in state. If Bob does not make a move even after a
predefined waiting period (say 10 minutes), Alice can withdraw both players bets.

```cpp
withdrawInactivity() {
  // Check if game is not already over i.e., funds have
  // not already been withdrawn.
  // Check if it's been atleast 10 minutes since inactivity
  // has been reported.
  
  isGameOver = true;

  if (inactivePlayer == alice) {
    sendAmount(bob, betAmount + betAmount);
  } else {
    sendAmount(alice, betAmount + betAmount);
  }
}
```

> Note: reportInactivity can be generalized to accept intents as well, to handle the situation where one player
> deposits funds, but the other does not.

The following flowchart summarizes the state transitions of the game.

![](https://miro.medium.com/max/691/1*qrxLenmoYg5mmh3haxczTg.png)

Hence, the Smart Contract acts like a third-party which only gets involved to start the game, end the game and
handle any disagreements between the players.

- - -

When players are fair to each other, only three transactions in total need to be made to the Blockchain.
1. Deposit of bet by Alice before the game begins.
2. Deposit of bet by Bob before the game begins.
3. Withdrawing of bets once the game is over.

This drastically improves the performance of the game.

**Speed:** Every move is almost instantaneous, since we do not need to make the move on Ethereum.

**Cost:** The players only need to pay the transaction fee for three transactions, which is a significant
improvement than the earlier approach.

This concept is not a new idea in the Blockchain space. You have developed what is known as a State Channel.
A State Channel involves creating a channel between two or more parties where state updates can be exchanged
directly and a smart contract only handles the disputes.

Read more on State Channels [here](https://education.district0x.io/general-topics/understanding-ethereum/basics-state-channels/).

This specific example of tic-tac-toe is inspired by a State Channel protocol called Force Move.
Read [here](https://magmo.com/force-move-games.pdf).

*Thanks for reading!*