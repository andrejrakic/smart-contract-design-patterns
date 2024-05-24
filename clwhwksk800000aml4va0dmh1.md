---
title: "Integrating arbitrary ERC-20 tokens"
datePublished: Wed May 22 2024 14:12:03 GMT+0000 (Coordinated Universal Time)
cuid: clwhwksk800000aml4va0dmh1
slug: integrating-arbitrary-erc-20-tokens
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1716396319574/1ab5fff2-79fb-4527-83f8-0f134febb3ce.jpeg
tags: design-patterns, blockchain, solidity, smart-contracts

---

Special thanks to [Amine El Manaa](https://www.linkedin.com/in/amine-e-a7704b24/) and [Milos Bojinovic](https://www.linkedin.com/in/milos-bojinovic-765010202/) for helping with the draft. This article is inspired by the [https://github.com/d-xo/weird-erc20](https://github.com/d-xo/weird-erc20) repository.

## Preface

In the "Design Patterns: Elements of Reusable Object-Oriented Software" book by Gang of Four, design patterns are described as solutions to problems in a context. According to this famous book, each pattern has four essential elements:

1. Name
    
2. Problem (describes when to apply the pattern)
    
3. Solution
    
4. Consequences (because, in design, everything's a tradeoff)
    

Another quote from the same book says: "Despite the book's size, the design patterns in it capture only a fraction of what an expert might know. (...) It doesn't have any application domain-specific patterns. (...) Each of these areas has its own patterns, and it would be worthwhile for someone to catalog those too."

Having been in the industry for many years, I've learned that there is a lack of up-to-date catalogs addressing common smart contract development problems (I even [tweeted](https://x.com/andrej_dev/status/1590055846591229953) about it). Fast-forward two years, I decided to spend my spare time writing about common smart contract development patterns, which is why you are reading this newsletter. I hope you find it useful and enjoyable!

## Integrating arbitrary ERC-20 tokens

Since your smart contracts will very likely interact with ERC-20 tokens at some point, I've decided to start my first blog post with issues you may encounter during the integration of arbitrary ERC-20 tokens. You may write contracts for staking, escrow, vesting, swapping, lending, borrowing, cross-chain transfers, and many other use cases. But almost always, you will find yourself in a situation where you need to at least call the `transfer` or `transferFrom` functions, or both.

So let's pick the first use case from the list, staking, and consider designing smart contracts for it. Let's assume that the User can stake an arbitrary ERC-20 token, and in return, our system will mint some amount of our LP tokens (it can be a 1:1 ratio for simplicity). Upon unstaking, the User will burn those LP tokens, and our system will unlock the previously staked ERC-20 tokens.

The naive solution to this problem will probably look something like this:

```solidity
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract Staking {
    function stake(address tokenAddress, uint256 amount) external {
        IERC20(tokenAddress).transferFrom(msg.sender, address(this), amount);
        // mint LP tokens
    }

    function unstake() external {
        // burn LP tokens
        IERC20(tokenAddress).transfer(msg.sender, amount);
    }
}
```

At first glance, there is nothing wrong with this approach because it relies on the mandatory interface from the original EIP-20. However, if we try to stake the *USDT* token, for example, the transaction will revert. On the other hand, we might not be able to unstake some other tokens, leaving them locked inside our smart contract. The reason for this lies in the variations among different ERC-20 token implementations. In the upcoming sections, we will discuss trade-offs in designing such a smart contract system that supports arbitrary ERC-20 tokens.

## 1) Missing return values

The ERC-20 standard requires that `transfer()` and `transferFrom()` functions return a `boolean` indicating the `success` or failure of the call. Unfortunately, some tokens like *USDT* do not return a `boolean` on ERC-20 functions, which we can clearly see from the screenshot below. For reference, this is part of the *USDT* source code on the Ethereum Mainnet.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1713556801123/0ed2fa37-a914-4320-93c2-032cd9fd81a3.png align="center")

Having a function signature like this (without returning a `boolean`) is essentially the reason why our naive approach would not work with *USDT* and why the `stake` transaction would revert.

Some other tokens, such as *BNB*, may return a `boolean` for certain methods but not for all. Due to this behavior, a portion of *BNB* tokens got stuck in the initial version of the Uniswap protocol.

And then there are tokens that, no matter what, always return `false`, even on successful transfers. You can have some kind of wrappers or transfer abstractions around those tokens, but given the total number of tokens with that behavior, it is a reasonable trade-off, in my opinion, to note in the documentation that your design does not support this type of token. And that's what we are going to do for our little staking example.

The solution for "missing return values" behavior is to perform low-level calls using the CALL opcode instead of direct function calls and rely on the boolean outcome. If the return value of a low-level call is `false`, your design should always revert, ensuring that the contract does not silently fail. You can achieve this by implementing your own library, like Uniswap did with its [TransferHelper](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TransferHelper.sol), or by integrating already audited libraries like [OpenZeppelin’s](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol) `SafeERC20`, [Solmate's](https://github.com/transmissions11/solmate/blob/main/src/utils/SafeTransferLib.sol) or [Solady's](https://github.com/Vectorized/solady/blob/main/src/utils/SafeTransferLib.sol) `SafeTransferLib` and more. When working with Solmate, however, you should keep in mind that it does not check whether a token address is a smart contract at all, to be more gas efficient.

I would recommend OpenZeppelin's `SafeERC20` as a general solution. It wraps around "regular" `transfer` and `transferFrom` functions with `safeTransfer` and `safeTransferFrom` functions that perform low-level calls, validate the call's return data, check if a target address is a contract at all, and always revert with appropriate Solidity custom errors. So if we integrate it, our code will now become:

```solidity
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract Staking {
    using SafeERC20 for IERC20;

    function stake(address tokenAddress, uint256 amount) external {
        IERC20(tokenAddress).safeTransferFrom(msg.sender, address(this), amount);
        // mint LP tokens
    }

    function unstake() external {
        // burn LP tokens
        IERC20(tokenAddress).safeTransfer(msg.sender, amount);
    }
}
```

In Vyper, on the other hand, this behavior is already built into the language itself, so there is no need to import external libraries, but rather just use the `default_return_value` kwarg. For example: `assert token.transferFrom(msg.sender, self, amount, default_return_value=True), "transferFrom failed"`

## 2) Returning false vs. Reverting on failure

As mentioned in the previous section, tokens that strictly follow the ERC-20 standard return `false` after an unsuccessful transfer attempt. However, the majority of token implementations, including OpenZeppelin's, actually revert on failure instead and return only `true` after a successful transfer attempt. To create a system that works with arbitrary ERC-20 tokens, we must not make assumptions on this matter and simply support any implementation.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1713561060172/e5c0ec32-b24d-46d9-9ed6-979c5e95a231.png align="center")

Luckily, we have already addressed this issue in the previous section by integrating the `SafeERC20` wrapper. Another option would be to manually validate the returned `boolean` value and revert if it's `false`.

## 3) Tokens with hooks

Currently, our design assumes that the User will call the `approve` function of the token's smart contract before calling our `stake` function. Additionally, the User may want to decrease the approval to our smart contract back to zero. This results in additional transactions the User needs to perform, decreasing the User's experience and costing them more gas overall.

To mitigate this issue, the idea of standards that can perform both actions (`approve` + `transfer`) in a single transaction arose. That was possible because ERC-20 standard is simple enough to be extended with additional functionality using hooks. Examples of such standards are ERC-677, ERC-777, ERC-1363 and more. Although the motivation behind these tokens is to eliminate the approval mechanism, if not integrated correctly, these tokens are vulnerable to Reentrancy attacks.

The canonical advice when integrating standards that are extensions of the ERC-20 standard is to read the standard's EIP carefully and integrate its implementations according to the guidance present in the standard. There were multiple ERC-777 related exploits in the past, including the *imBTC* Uniswap V1 pool being drained, so in this section, we will focus on integrating this type of token.

So let's start with important quotes from the EIP-777 itself:

1. The token contract MUST call the `tokensToSend` hook before updating the state.
    
2. The token contract MUST call the `tokensReceived` hook after updating the state.
    

This tells us that besides the post-transfer hook, ERC-777 tokens also have a pre-transfer hook. The common assumption is that the Check-Effects-Interactions pattern (which will be covered in upcoming blog posts) is sufficient protection against Reentrancy attacks. However, due to the `tokensToSend` hook, our `stake` function MUST NOT follow the Check-Effects-Interactions pattern to integrate ERC-777 tokens correctly.

The `ERC1820Registry.sol` smart contract keeps track of the preferred hook receivers for ERC-777 tokens, known as interface implementers. This means if Alice sets Bob's address as the interface implementer, Bob will receive hooks when Alice sends or receives her ERC-777 tokens. But what if Alice is the malicious user and sets her attacker's contract as the preferred hook receiver instead?

Consider the following scenario. Note that the described ERC-777 token implementation is from OpenZeppelin.

Let's say that we refactored the `stake` function to follow the Checks-Effects-Interactions pattern by updating all storage variables, including minting our LP tokens, before making the external call to the token's `transferFrom` function.

```solidity
function stake(address tokenAddress, uint256 amount) external {
    // mint LP tokens
    IERC20(tokenAddress).safeTransferFrom(msg.sender, address(this), amount);
}
```

The attacker contract registers itself as an interface implementer in the `ERC1820Registry.sol` smart contract.

```solidity
IERC1820Registry(0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24)
        .setInterfaceImplementer(
            address(this), 
            keccak256("ERC777TokensSender"), 
            address(this)
        );
```

The attacker contract calls our `stake` function, which will eventually trigger the `_callTokensToSend` function.

```solidity
contract Attacker is IERC777Sender {
    function initiateAttack(uint256 amount) external {
        staking.stake(token, amount);
    }
}
```

Since the attacker contract is the preferred ERC-777 interface implementer for itself, it will receive the pre-transfer hook, meaning that it now has an opportunity for a Reentrancy attack.

```solidity
contract Attacker is IERC777Sender {
    // ERC777 hook
    function tokensToSend(address, address, address, uint256 amount, bytes calldata, bytes calldata) external {
        require(msg.sender == token, "Hook can only be called by the token");
        if(lpToken.balanceOf(address(this)) < targetAmount) {
            staking.stake(token, amount);
        }
    }
}
```

As a result, the attacker will receive more LP tokens than they should.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1713652518391/92809694-82ec-42a2-83e5-84bdceaf877a.png align="center")

To mitigate this issue, the correct implementation of the `stake` function would be:

* to first `transferFrom` tokens into our contract and then implement the rest of the function logic.
    
* to put a Reentrancy lock modifier around it.
    

Now let's consider attacking the `unstake` function. Unlike the previous scenario, our system is now at risk if this function does not follow the Check-Effects-Interactions pattern (which is something we would expect normally). The `_callTokensReceived` function will trigger the post-transfer `tokensToReceived` hook, which the attacker could reenter to withdraw more tokens than previously staked, potentially draining the entire contract.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716154633490/ed3c1a00-27bc-4a34-bd83-22234e6caacc.png align="center")

To mitigate this issue, the correct implementation of the `unstake` function would be:

* to follow the Check-Effects-Interactions pattern by burning LP tokens and performing other internal logic before making an external call to the token's `transfer` function.
    
* to put a Reentrancy lock modifier around it.
    

ERC-777 is also **more dangerous** than ERC-677 and ERC-1363 because its `transferFrom` function signature in the majority of implementations (including OpenZeppelin’s) is the same as in ERC-20, meaning that you can be unaware that you are interacting with ERC-777 (a token with hooks) instead of a regular ERC-20 token.

## 4) Fee on Transfer tokens

Fee on Transfer tokens, also known as deflationary tokens, involve a unique mechanism where a specific portion of the token amount is deducted each time they are transferred. This predetermined portion of the transfer is then either permanently removed (burned) or sent to another address, effectively reducing the total supply or redistributing the tokens.

There are also tokens that have this feature, but it is currently disabled, as is the case with *USDT* or *USDC*, for example. This means that even though they don't behave as Fee on Transfer tokens now, they may in the future (as shown in the screenshot below), and our design must account for that scenario.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715526571190/34ca58a1-fee5-4e08-a2ab-a083960b8022.png align="center")

Consider the current design for our Staking smart contract and the following scenario. Alice wants to stake 100 tokens, but the fee for that transfer is 5 tokens. This means that calling the token's `transferFrom` function will result in 95 tokens being transferred to our smart contract and 5 tokens being burned. However, because the `amount` provided to the `stake` function is 100, our current design will mint 100 LP tokens to Alice, even though only 95 tokens were actually staked. If Alice later tries to unstake 100 tokens, that transaction will fail because there are only 95 tokens available for withdrawal from the contract.

The solution to this issue is to rely on the actual amount of tokens received, not the amount specified. Therefore, to integrate Fee on Transfer tokens:

1. Check the current token balance of your smart contract.
    
2. Transfer tokens from the sender to your smart contract (ensure reentrancy lock is in place).
    
3. Check the token balance of your smart contract again.
    
4. Subtract the balance after the transfer from the balance before the transfer, and use that value instead of the amount used in the `transferFrom` function.
    

The implementation in our case can look like this:

```solidity
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract Staking {
    using SafeERC20 for IERC20;

    function stake(address tokenAddress, uint256 amount) external nonReentrant {
        uint256 balanceBefore = IERC20(tokenAddress).balanceOf(address(this));
        IERC20(tokenAddress).safeTransferFrom(msg.sender, address(this), amount);
        uint256 balanceAfter = IERC20(tokenAddress).balanceOf(address(this));
        uint256 actuallyReceived = balanceAfter - balanceBefore;
        // mint LP tokens
    }
}
```

The trade-off of this approach is that this sequence of function calls will inevitably result in higher gas consumption.

Be aware that there are two types of Fee On Transfer tokens:

* **Inclusive Fee On Transfer tokens**: These tokens burn or divert a small part of each transfer, causing the recipient to receive slightly less than what the sender sent.
    
    * To integrate, rely on the amount actually received as explained earlier.
        
* **Exclusive Fee On Transfer tokens**: These tokens send an extra transfer from the sender's address after the main transfer.
    
    * Depending on the scenario, the transaction should either fail completely or partially succeed (receive the main transfer but fail the extra transfer).
        

## 5) amount == type(uint256).max

Speaking of not relying on the `amount` provided, there are tokens, like *cUSDCv3*, that have a special case where if `amount == type(uint256).max` is true, the user's entire balance will be transferred.

Consider the following scenario. Alice wants to stake a *cUSDCv3*\-like token and provides `type(uint256).max` as the amount, indicating she wants to stake her entire balance. Our smart contract stores that amount in a mapping that tracks how many tokens each address has staked. When Alice decides to unstake tokens, the `amount` from the mapping, `type(uint256).max`, will be used, and our contract will be drained because its entire balance will be transferred to Alice. Let's say Alice staked 10 tokens and there were 100 tokens in the contract. After this transaction, Alice becomes 90 tokens richer.

To avoid this scenario, you should rely on the amount of tokens actually received, as described in the **4) Fee on Transfer tokens** section.

## 6) Rebasing tokens

Rebasing, or elastic tokens, are tokens that adjust their supply automatically to maintain price stability. They achieve this through a process called a "rebase," which happens at regular intervals (e.g., every 24 hours).

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715530594685/04b4d1ed-a9c5-4e46-9377-e331b53acd4a.png align="center")

Lido’s *stETH* is an example of a rebasing token. According to Lido's documentation, its balances update once a day at 12 PM UTC when the oracle reports changes in Eth2 deposits and changes in *ETH* rewards from users who stake through Lido.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715530554163/202bdd7e-97bb-4f4a-a1da-a7d3d7d1b8fe.png align="center")

To integrate rebasing tokens, you can combine relying on the amount actually received with a separate mapping that tracks the total amount of tokens staked by token address and the percentage of that amount staked by a particular address. Then, upon withdrawal, you can use that percentage to calculate the amount of tokens to unstake based on the contract's overall balance of that token.

You should use basis points (BPS) as a unit of measurement for percentages, where 1 BPS equals 1/100th of 1 percent.

```solidity
/**
 * 0.01%  =  1 BPS
 * 0.05%  =  5 BPS
 * 0.1%   =  10 BPS
 * 0.5%   =  50 BPS
 * 1%     =  100 BPS
 * 10%    =  1 000 BPS
 * 100%   =  10 000 BPS
 */
```

There are several trade-offs with this approach that you should carefully consider:

* Although the above calculation works for all non-rebasing tokens as well, it is much more complex. It requires more storage accesses, which means more gas is needed, and there is room for rounding errors when calculating shares.
    
* There is no way to prevent anyone from sending ERC-20 tokens to your smart contract, and the above approach suggests relying on `balanceOf(address(this))` when calculating the number of tokens to unstake. If not implemented correctly, this step can become a potential exploit.
    

Integrating rebasing tokens is quite tricky, and many projects state that they do not support rebasing tokens, which is a completely valid approach, in my opinion. Others, like Uniswap V2, have dedicated `sync` and `skim` functions to support rebasing tokens and give their users an opportunity to profit from the rebases.

## 7) Approval Race protections

Consider the following scenario: Alice wants to buy a T-shirt from Bob using a middle-man smart contract to handle the payment in tokens. The price of the T-shirt is 100 tokens.

Alice approves 100 tokens to be spent on her behalf by the middle-man contract.

But then Bob says, "Hey Alice, the tax is 20 tokens, so you need to send me 120 tokens."

Alice doesn't mind and approves 120 tokens to be spent on her behalf by the middle-man contract. What Alice doesn't know is that Bob is an MEV searcher and plans to take advantage of the situation by including both of her approval transactions in the next block.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715531823060/ee672cf7-924f-4c5c-b72b-163d100ed700.png align="center")

Some tokens, however, have built-in protection against this type of attack. For example, *USDT* does not allow approving an amount `M > 0` when an existing amount `N > 0` is already approved. Instead, you must decrease the approval back to 0 before approving the new amount `N`.

No further action is needed for our Staking contract besides acknowledging this behavior. However, if your smart contract logic includes calling the `approve` function, be aware that:

* It may revert in the *USDT* scenario described above.
    
* There are also implementations (like OpenZeppelin's) that have `safeIncreaseAllowance` and `safeDecreaseAllowance` functions.
    

## 8) Number of decimals

You must not assume that the number of token decimals is 18 (or any other value). The `decimals()` function **is optional** in the ERC-20 standard.

Most ERC-20 tokens have 18 decimals, meaning that if you want to:

* transfer 5 tokens, the amount should be `5 * 10^18`,
    
* transfer 0.5 tokens, the amount should be `5 * 10^17`, etc.
    

However, some tokens can have a low number of decimals, like *USDC*, which has 6 decimals. Others can have a high number of decimals, 24 for example. Therefore, to transfer 5 *USDC*, the amount should be `5 * 10^6`.

And then, there is a special case where *USDT* has 6 decimals on Ethereum but 18 decimals on Binance Smart Chain.

Performing basic math operations with a lower number of decimals can result in precision loss, and with a higher number of decimals can result in potential overflows. One other canonical piece of advice is to always round against the User and in favor of the Protocol to reduce the risk of exploits.

When integrating an arbitrary ERC-20 token:

* Don't assume the number of decimals is 18.
    
* Don't assume the `decimals()` function is implemented, because it's optional.
    
* Rely on the actual amount received. If you must perform any decimals-related computation, integrating arbitrary ERC-20 tokens is not possible.
    
* When doing cross-chain transactions, don't assume the number of decimals of the same token is equal on both blockchains.
    

## 9) WETH9

Wrapped Ether is probably one of the most used ERC-20 tokens. However, you must not rely on its `totalSupply()` function if you ever need such logic. Let's analyze this function.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715548232400/328ed133-621e-449f-b342-7c27b635d659.png align="center")

When the `deposit` function is called, the amount of native coins (*ETH* on Ethereum) gets locked inside the `WETH9` contract, and the same amount of *WETH* tokens gets minted to the sender's address. The reverse happens when the `withdraw` function is called. However, the `totalSupply()` function returns the total balance of coins inside the `WETH9` contract, not the amount of *WETH* tokens in existence.

The problem is that a contract cannot prevent anyone from sending native coins to it.

You can put reverts inside your contract's `receive()` and `fallback()` functions, but that still wouldn't be enough to prevent anyone from sending you native coins (an attacker contract can `selfdestruct` itself and send its native coins to the victim contract), and the same applies to the `WETH9` contract.

Moreover, if you analyze the `WETH9` contract on Ethereum Mainnet, you will see that it holds more *ETH* than there are *WETH* tokens in existence, which means that the `totalSupply()` function returns an inaccurate value.

In general, contracts should not have invariants that rely on `address(this).balance`.

## 10) Flash-mintable tokens

Some tokens, like *DAI*, have a flash-mint feature, which allows a high amount of tokens (there is usually a cap) to be minted for the duration of one transaction, as long as the same amount of tokens gets burned at the end of that transaction.

The idea is similar to flash-loans, to democratize arbitrage, but saves users' gas because they don’t need to get flash-loans from somewhere else. What this means for our architectural design is that it must work with the amount actually received and support huge amounts, up to `type(uint256).max`, which is already the case.

In general, since flash-mints are similar to flash-loans, when integrating flash-mintable tokens, you need to:

* Be aware of this concept and design contracts accordingly (there are both good and bad MEV)
    
* Defend against flash loan-like exploits, as these will address common issues with flash-mints as well
    

We will discuss flash-loans in more detail in upcoming blog posts.

## 11) Upgradable tokens

Some tokens, like *USDT* or *USDC*, are upgradable, meaning they can switch to a newer version without you being aware of it. Additionally, changes to the token's behavior can break any smart contract that relies on its previous behavior. For example, *USDT* or *USDC* can enable the Fee on Transfer feature, and if your contract doesn't account for the actual amount of tokens received, it could become vulnerable to exploits. To mitigate that issue, revisit the **4) Fee on Transfer tokens** section.

Integrating arbitrary ERC-20 tokens includes any future implementations because, technically, they are just other token smart contracts. However, you will likely care about the specific implementation. In that scenario, you can introduce a mapping that would, for a given token proxy address, contain the implementation address approved by the Governance (via an additional function).

And then you will have logic when implementation addresses do not match. You have two choices:

* Be optimistic: If a mapping returns `address(0)` as the implementation address, assume the token is not upgradable and proceed
    
* Or strict: Token is supported only after Governance approves its implementation
    

## 12) Tokens with Blocklists

Some tokens, like *USDT* or *USDC*, have contract-level address blocklists controlled by the contract owners. If an address is on the blocklist, transfers `to` and `from` that address are not possible, and the transaction will revert.

This behavior creates a problem for our Staking contract due to a specific edge case. If an address is on the token-level blocklist, it will never be able to stake that token initially. However, if an address that has already staked the token gets added to the blocklist later, it will never be able to unstake the previously staked tokens, resulting in locked tokens inside our Staking smart contract.

There is no standard way to check for token blocklists, and there is no universal solution for integrating them. In our example, we are building a staking app, which means we encourage locking tokens inside our smart contract, so no further action is needed from our side. However, sometimes you might want to include a feature to withdraw any stuck tokens from a contract. If you are building contracts for token escrow, you might want to add a `cancelEscrow` function to withdraw tokens if the counterparty is on a token-level blocklist, and so on.

## 13) Multiple token addresses

Some tokens, like *TrueUSD*, can have multiple token addresses. Again, as stated in section **11) Upgradable tokens**, integrating arbitrary ERC-20 tokens includes any future implementations because, technically, they are just other token smart contracts.

Besides acknowledging this fact, you can also introduce a mapping that would, for a given token address, contain the implementation address approved by the Governance (via an additional function).

## 14) Pausable tokens

Some tokens, like *BNB*, can be paused by contract owners. When a smart contract is paused, you cannot interact with the functions protected by `whenNotPaused`\-like modifiers.

This is similar to the blocklist issue described in the **12) Tokens with Blocklists** section, and there is not much we can do besides acknowledging that this behavior is possible.

## 15) Zeros, zeros everywhere

Some tokens, like OpenZeppelin’s implementation, will revert if you call `approve(address(0), amount);`.

---

Some tokens, like *BNB*, will revert if you call `approve(spender, 0);`.

---

Some tokens will revert if you attempt to transfer `0` amount of a token.

---

Some tokens, like OpenZeppelin’s implementation, will revert if you attempt to burn tokens by transferring them to the `address(0)`.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715883856178/d77a69f2-96d4-48ac-aefc-78c67658af46.png align="right")

---

Most of these issues are not critical unless:

* Attempting to `approve` or `transfer` a zero value (which is calculated or retrieved from storage) causes the contract to get stuck because this attempt will always revert. For example, if we have a loop to transfer staking rewards to all eligible holders, but for some addresses, the reward is zero, then all rewards are stuck.
    
* The `burn` function must not revert for any reason (e.g., a trade position must be closed). If the `burn` function is not implemented, we can always send tokens to other burn addresses like `0xdead` or lock them inside our smart contract that doesn't have a withdrawal function.
    

## 16) Revert on large approvals and transfers

Some tokens, like *UNI* or *COMP*, revert if the value passed to `approve` or `transfer` functions is larger than `uint96`. If your smart contracts rely on the actual received amount, no further action is needed other than acknowledging that this behavior is possible.

**⚠️** However, if you support rebasing tokens with a separate mapping that tracks the total amount of tokens staked by token address and a percentage of that amount staked by a particular address, as described in the **6) Rebasing tokens** section, your design won't support rebasing tokens with units represented by a smaller data type than `uint256`.

This tradeoff can be accepted for our Staking example, but it must be clearly documented, because:

* The likelihood is small - there isn’t any major token implemented like this.
    
* The impact is high but not permanent - the calculated amount to unstake can potentially be greater than `type(uint96).max`, resulting in locked tokens. However, with the next rebase, it can go below this value, making withdrawal possible again.
    

## 17) Non-standard permits

Some tokens, like *DAI* or *WETH*, have a `permit()` implementation that does not follow the [EIP-2612](https://eips.ethereum.org/EIPS/eip-2612).

This means we cannot rely on `permit()` reverting for arbitrary tokens.

The `permit()` function never reverts for tokens that:

* do not implement `permit`
    
* have a (non-reverting) `fallback` function
    

As a solution, consider using Uniswap's Permit2 library or implementing your own.

## Conclusion

In this blog post, we analyzed how arbitrary ERC-20 tokens can be integrated into your smart contracts. Finally, here's the cheat sheet for this design pattern.

* Problem: ***Integrating arbitrary ERC-20 tokens***
    
* Solution:
    
    * Use OpenZeppelin's `SafeERC20` library.
        
    * Use reentrancy locks.
        
    * Rely on actually received tokens, instead of the provided amount.
        
    * Consider rebasing tokens.
        
    * If you need a number of token decimals for any type of calculation, you must provide it. Don't assume it's 18. Don't rely on the `decimals()` function because it's optional in the ERC-20 standard.
        
    * There is no way to prevent anyone from sending tokens or native coins to your smart contract. Your contract invariants must not rely on the `balanceOf` or `address(this).balance` functions.
        
    * Protect your contract against attacks that include flash-mint tokens.
        
    * Be aware that there are tokens that:
        
        * Are upgradable
            
        * Are pausable
            
        * Have blocklists
            
        * Have multiple contract addresses
            
    * Be aware that transferring or approving zero values `from` or `to` addresses can revert, and ensure your system won't halt because of that.
        
    * Be aware that some token implementations don't use `uint256` as a data type for an `amount` variable and act accordingly.
        
    * Use Uniswap's `Permit2` library to support non-standard permits.
        
* Consequences: Due to all the additional checks we added, the gas cost will be higher.
    

My name is [Andrej](https://twitter.com/andrej_dev) and I hope you enjoyed reading this article. To receive the next one, subscribe to the [Smart Contract Design Patterns newsletter](https://andrej.hashnode.dev/newsletter). Thank you!