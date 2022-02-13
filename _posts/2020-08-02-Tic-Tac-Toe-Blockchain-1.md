---
title: Playing Tic-Tac-Toe as the world watches — An introduction to BlockChain and State Channels (Part 1)
date: 2020-08-02 09:00:00 +0530
categories: [Blockchain, Programming]
tags: []     # TAG names should always be lowercase
math: true
mermaid: true
toc: true
---

# Introduction

**This 2-part series is a brief introduction to the concepts of Ethereum and State Channels by building a**
**tic-tac-toe application.**

Stuck in the lock-down, you have been whiling away time playing the game of tic-tac-toe with your friend for months
now and it has become monotonous. To spice things up, you both have decided to bet some money and winner takes it all.
But, with money comes problems.

You cannot **trust** your friend to be able to afford the bet.

You cannot **trust** your friend to make a valid move.

You cannot **trust** your friend to complete the game.

You cannot **trust** your friend to **honor the bet** if you win.

You decide to solve each problem. Firstly, how do we prove that a player can afford a bet? You can ask him to submit
the funds before even the game begins. But,

He does not trust you to keep the funds safe and **distribute fairly**. Neither do you trust him with that responsibility.

You read up online for a solution to these problems and come across the following topics,

*Note: Feel free to skip the sections you are familiar with.*

# Blockchain

In layman terms, a Blockchain is a decentralized network that allows one to make transactions (i.e., transfer of funds
or data between users) in such a way that no single entity (a bank or a person) can control or tamper. In a Blockchain,
all transactions are completely public, so you can check the balance of your friend yourself and make sure that he’ll
be able to afford it.

Read more on Blockchain [here](https://medium.com/swlh/a-simple-guide-to-blockchain-technology-4589971e6d03).

# Multi-Signature Wallets

A Blockchain transaction requires the owner of the account/wallet to prove that they are the one making the
transaction, similar to how one has to enter the UPI PIN on any UPI app before making a payment. Unlike traditional
blockchain wallets that belong to a single person, a Multi-Signature wallet is owned by multiple people. Before making
a transaction from the wallet, all the owners of the wallet need to agree to make the transaction.

Read more on Multi-Signature Wallets [here](https://www.antiersolutions.com/how-can-a-multisig-wallet-secure-your-transactions/).

A **Multi-Sig wallet** can solve the responsibility of keeping funds! How? You create a Multi-Signature wallet between
the players and both of you can send your bets to the shared wallet. Now, no single player can unilaterally withdraw
funds. Unless the withdrawing transaction created is accepted by both participants, it will not be executed. Thus,
the funds are safe.

There is still the problem of players making an invalid move. You do further reading and stumble upon some more topics,

# Ethereum

Ethereum is yet another Blockchain BUT it does not limit itself to monetary transactions like Bitcoin. It can execute
any programmable function in such a way that no one entity can tamper the output. The output of the function is
verified by the entire network of Ethereum users.

The Ethereum Blockchain can be viewed as a large **computation engine** which is guaranteed to give correct results every
time. But, it is not free. You are expected to pay a fee (called gas) to do the computation. The gas is directly
proportional to the number of instructions of the function you are trying to execute.

Read more on [Ethereum](https://medium.com/@micheledaliessi/what-is-ethereum-f4c5e566ff77) here.

# Smart Contracts

Functions to be computed by the Ethereum network is written in the form of Smart Contracts. Consider a Smart Contract
to be a literal replacement for real life contracts. Once a real life contract is in place, the specified rules are
strictly adhered to, and in case of deviations, law enforcement is involved. Similarly, smart contracts define a set of
rules in the form of a program, and also makes it impossible to make any deviation from the rules. It can go above and
beyond a real-life contract and act as a third-party entity.

A Smart Contract can store funds (Ether in Ethereum) and distribute them. Hence, it can act as a multi signature wallet
but also allow a single or set of owners to withdraw unilaterally when specific conditions are met according to the
contract. Every smart contract has a state to store variables and use them, such as the current tic-tac-toe board
state.

*Note: Going further, you will be called Alice and your friend will be called Bob. So, hello Alice!*

This is perfect! You can now solve the problem of trusting Bob to always make a valid move and completing the game
using a smart contract. Smart Contracts are written in the Solidity language. You can write and deploy a Smart
Contract that looks like this,

```solidity
contract TicTacToe {

  // currentState contains the current board state
  // i.e., List of the 3*3 tiles and which ones have
  // been marked by whom
  State currentState;

  // Wallet addresses of Alice and Bob
  Address alice;
  Addess bob;
  
  int betAmount = 10;

  // This stores who has previously made a move.
  Address previouslyMoved;
  
  // Time at which last move was done.
  Time lastMoveTime;
  
  bool aliceHasDepositedFunds = false;
  bool bobHasDepositedFunds = false;
  bool isGameOver = false;
  
  // This constructor is called when the contract is created.
  TicTacToe(Address _alice, Address _bob) {
    alice = _alice;
    bob = _bob;
    
    // Since the first move is to be done by alice,
    // We will set bob to have moved previously
    previouslyMoved = _bob;
  }

  depositFunds() {
    // If a require condition fails,
    // the function terminates and does not proceed
    require(amountSent == betAmount);
    
    // sender is an inbuilt variable that has the
    // address of the person calling the function.
    if (sender == alice) {
      aliceHasDepositedFunds = true;
    } else if (sender == bob) {
      bobHasDepositedFunds = true;
    }
  }
  
  move(State nextState) {
    require(aliceHasDepositedFunds && bobHasDepositedFunds);
    require(!isGameOver);
    
    // Check if the same person is not making a move twice.
    require(isCorrectPersonMakingTheMove(sender, previouslyMoved));
    
    // Check if given two consecutive states is a valid transition.
    require(isValidMove(currentState, nextState));
    
    currentState = nextState;
    previouslyMoved = sender;
    lastMoveTime = getCurrentTime();
    
    if (isGameTied(currentState)) {
      // No more valid moves are left
      sendAmount(alice, betAmount);
      sendAmount(bob, betAmount);
      isGameOver = true;
    } else if (hasGameEnded(currentState)) {
      // sender has made the last move, and won the game
      sendAmount(sender, betAmount + betAmount);
      isGameOver = true;
    }    
  }
  
  // Function to be called when the other player
  // is refusing to make a move.
  reportInactivity() {
    if ((getCurrentTime() - lastMoveTime) > "10 minutes") {
      sendAmount(previouslyMoved, betAmount + betAmount);
      isGameOver = true;
    }
  }
  
  isValidMove(State currentState, State nextState) {
    // You can define a set of rules to validate
    // two consecutive tic-tac-toe moves
  }
  
  sendAmount(Address recipient, int amount) {
    // Sends amount to recipient.
  }
}
```

> It is not required to understand the code completely to follow the remaining sections.
>
> Note: The above code is not a proper syntax of any language. It is meant as a cross between C++, pseudocode and
> Solidity for familiarity.

You have now created a smart contract and deployed it on the Ethereum network using tools such as
[Remix](https://remix.ethereum.org/) and [Metamask](https://metamask.io/).

Your friend needs to be briefed about how it works as well. This is how it should work when both players are honest.

![](https://miro.medium.com/max/700/1*fRHea1FxE-AuIkb0mKn10Q.jpeg)

When Bob tries to make an invalid move,

![](https://miro.medium.com/max/700/1*UF8wNzCiyS3mkibPLl0vDg.jpeg)

When bob is not willing to make the next move, delays the next move or tries to lock the funds forever in the contract,

![](https://miro.medium.com/max/681/1*_FgXGMfwsXpHbbB-LNIhMQ.jpeg)

You have ended up solving the following problems,

1. You need not trust that the opponent has the required funds because the game does not start without both players
   depositing.
2. You can always expect the opponent to make a valid move. If the opponent does not make a valid move, he loses all
   his bet.
3. You are guaranteed to be paid by the opponent (if you win), since the funds are locked before the game even begins.
4. You no longer need to entrust a third party to fairly distribute the funds, since the smart contract, whose rules
   are written in stone, handles the distribution.

- - -

You have solved the initial problems and have started playing the game. But, there are other major problems!

**Speed:** An Ethereum transaction can take anywhere from 15 seconds to 5 minutes to process a transaction.
Imagine waiting for 5 minutes after every move on a simple tic-tac-toe game.

**Cost:** Computation on Ethereum is not free. As already mentioned, we need to pay a certain amount of transaction fee
and every move is a transaction. The average Ethereum transaction fee is 1.59$. This will be incurred on every turn,
including the initial deposit!

More problems to solve :D

*Thanks for reading!*

*In the next part, we’ll be elucidating about a concept called State Channels that make game moves instantaneously*
*with just one or two transactions to Ethereum while maintaining all the existing benefits.*

*Also, the code has its logical flaws. I had to make a trade-off between correctness and simplicity.*
