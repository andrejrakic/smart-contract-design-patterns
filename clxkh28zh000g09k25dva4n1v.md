---
title: "Commit-Reveal Scheme"
datePublished: Tue Jun 18 2024 14:00:45 GMT+0000 (Coordinated Universal Time)
cuid: clxkh28zh000g09k25dva4n1v
slug: commit-reveal-scheme
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1716924661899/66a2e827-947a-4d8c-8c12-077af90fe078.jpeg
tags: design-patterns, solidity, smart-contracts

---

This article is inspired by the [ENS' ETHRegistrarController](https://github.com/ensdomains/ens-contracts/blob/staging/contracts/ethregistrar/ETHRegistrarController.sol) smart contract.

# Problem

The Commit-Reveal Scheme is one of the oldest and most popular smart contract design patterns. I would even argue that, along with the Check-Effects-Interactions pattern, it is the most basic smart contract design pattern.

Its main purpose is to ensure fairness in time- and data-sensitive transactions, such as voting, auctions, quizzes, games and more.

Most blockchain clients' implementations have a publicly visible mempool of transactions that block proposers can include in upcoming blocks. There are exceptions, but we will assume this behavior throughout the article and design our contracts to address this issue. The problem with publicly visible mempools is that anyone monitoring them can see your transactions and, more importantly, your intentions with those transactions. Block proposers have a short window of opportunity to take advantage of that information and "front-run" you.

Let's say we want to design a smart contract for a simple quiz. The admin will post a question and fund the contract with some tokens during deployment. Whoever answers the question correctly first will get those tokens as a reward.

```solidity
contract Quiz {
    string public question = "What distributed ledger technology, introduced in a 2008 whitepaper by Satoshi Nakamoto, underpins the cryptocurrency Bitcoin?";
    bytes32 hashedAnswer = 0xf60b0e34c817371a0b36d9e1c96487f690df8a12a997d1512b34c3a0331fe204;

    function guess(string memory answer) external {
        if(keccak256(abi.encode(answer)) == hashedAnswer) {
            // distribute reward
        }
    }
}
```

Alice knows the answer, while Bob does not. Alice submits a transaction containing the answer.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717181871148/4516d3cc-b558-4d12-983a-d1b4729c3b8f.png align="center")

Bob, who was monitoring the mempool, now knows the correct answer. And there is still time for Bob to submit his transaction with the same answer. However, Bob will tip the block proposer significantly to outbid Alice's transaction and include his first to win the quiz reward.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717182216301/61c81ff2-c167-48aa-9851-4bb978e1bb65.png align="center")

The block proposer will choose the transaction with the higher tip from the mempool to maximize their gains. As a result, Bob will receive the reward from the Quiz smart contract, even though he didn't know the correct answer initially. This isn't fair, but fortunately, there is a solution called the Commit-Reveal Scheme.

By the way, the correct answer to the quiz question is "Blockchain" :)

# Solution: Commit-Reveal Scheme

> Commit-Reveal Scheme is a two-phase process where participants first commit to a value by submitting a hash, and later reveal the value along with a secret used to generate the hash.

To implement it for a quiz example, we would need to add:

* A function to commit a hashed guess
    
* A function to reveal both the guess and the secret used to generate the previously committed hash
    

To keep things fair, it is advisable to split these two processes by forcing some time to pass between the commit and reveal phases. This prevents calling both functions as part of the same transaction (for example, by using a smart contract).

```solidity
contract Quiz {
    string public question = "What distributed ledger technology, introduced in a 2008 whitepaper by Satoshi Nakamoto, underpins the cryptocurrency Bitcoin?";
    bytes32 hashedAnswer = 0xf60b0e34c817371a0b36d9e1c96487f690df8a12a997d1512b34c3a0331fe204;

    uint256 immutable guessingEndsTimestamp;
    mapping(address => bytes32) answers;

    constructor(uint256 guessingPeriod) {
        guessingEndsTimestamp = block.timestamp + guessingPeriod;
    }

    function commit(bytes32 guess) external {
        require(guessingEndsTimestamp > block.timestamp);
        answers[msg.sender] = guess;
    }

    function reveal(string memory answer, bytes32 secret) external {
        require(guessingEndsTimestamp <= block.timestamp);
        if(answers[msg.sender] == keccak256(abi.encode(answer, secret))) {
            if(keccak256(abi.encode(answer)) == hashedAnswer) {
                // distribute reward
            }
        }
    }
}
```

Now Alice can safely commit her guess without revealing it.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718658713237/b5c36b96-78cd-4c95-9e03-957bd0c0e3e7.png align="center")

Once the committing phase is over, Alice will reveal both her original guess and secret used to generate the previously committed hash. By splitting commit and reveal phases Bob can't "front-run" Alice even though he can now see the actual answer in the mempool.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718659756717/41c3da71-46f5-4cac-bc57-18bd6e6627c8.png align="center")

# Using Commit-Reveal Scheme in production

The Commit-Reveal scheme isn't just a theory. It's used in real applications that have been on the mainnet for years. Let's analyze how the [ENS' ETHRegistrarController](https://github.com/ensdomains/ens-contracts/blob/staging/contracts/ethregistrar/ETHRegistrarController.sol) smart contract uses this pattern.

According to the official ENS documentation, there are three phases in registering a new ENS handle in order to prevent frontrunning:

* Commit
    
* Wait (at least ~60 seconds)
    
* Reveal
    

This contract exposes a `makeCommitment` helper function that computes a keccak256 hash of all input parameters instead of the user.

Once the user calculates the commitment hash, they can pass it as an argument to the commit function. Because the hash is pre-image resistant, no one monitoring the mempool can learn which handle the user is attempting to register.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718660928428/4b6c1e2d-7c08-4f29-9292-82e70ef7f9a9.png align="center")

Now we enter the "Wait" phase, where the user must wait for at least the `maxCommitmentAge`, which is about 60 seconds.

Finally, in the "Reveal" phase, the user needs to call the `register` function with the same input parameters used in the earlier `makeCommitment` function call. Although you can wait longer than the `maxCommitmentAge`, the stored commitment hash must be between 1 and 24 hours old, otherwise the `register` function call will revert. Additionally, you cannot make another commitment before the current one expires.

# **Conclusion**

In this article, we analyzed the Commit-Reveal Scheme. Here's the cheat sheet for this design pattern.

* Name: ***Commit-Reveal Scheme***
    
* Problem: Ensuring fairness in time- and data-sensitive transactions.
    
* Solution:
    
    * **Commit phase**: Participants first commit to a value by submitting a hash.
        
    * **Wait**: Split the commit and reveal phases by forcing some time to pass between the two.
        
    * **Reveal phase**: Reveal the value along with a secret used to generate the hash.
        

My name is [**Andrej**](https://twitter.com/andrej_dev) and I hope you enjoyed reading this article. To receive the next one, subscribe to the [**Smart Contract Design Patterns newsletter**](https://andrej.hashnode.dev/newsletter). Thank you!