---
title: 'How to lower L1 transaction fees on any blockchain'
date: 2021-04-10T15:14:05+01:00
draft: true
tags: ["blockchain", "Code"]
---

__*Cheaper transactions for everyone*__

## Ever-increasing transaction costs

Transaction fees on blockchains like Ethereum are once again [prohibitively high](https://etherscan.io/chart/gasprice). Gas prices routinely approach 200 gwei, making even the most basic transaction cost multiple dollars. Sadly we cannot change the gas price, but what if we could change the amount of gas we use? There is a simple method that anyone can use on any blockchain to reduce the gas cost of (smart contract) transactions, without relying on layer 2 scaling or waiting for EIP-1559-like proposals. 

Our method proposed can reduce transaction costs for most smart contract calls and, therefore, almost every token interaction. Why isn't anyone using this method? We have to give up some liberties on what amounts we work with, e.g. sending a fraction of a token more than we want to send. Savings are likely in the range of 0.2% to 1.0% of the total transaction cost. 

The method is not limited to simple transactions; at the end of this article, we will look into applying the same technique in other situations to save gas, like contract deployments. 

![Historical transaction fees on Ethereum](/images/eth-gas-prices.png)

## How gas works

In my [previous post](/posts/learn-from-building-sdk/), we went over gas price calculations for on-chain transactions. While the example was from VeChain, the same holds for Ethereum and most Ethereum based blockchains. In summary, transactions have an intrinsic cost and an execution cost. The intrinsic cost for Ethereum is 21.000 gas plus the cost of the data field. As can be seen in the [yellow paper](http://paper.gavwood.com/) (appendix H. Virtual Machine Specification), data field cost is increased by 4 for every zero byte and 68 for every non-zero byte. When we replace non-zero bytes with zero bytes, the transaction cost will decrease. How will we change the number of non-zero bytes while still maintaining a valid and sensible data field?

![Gas cost for adding bytes to the data field](/images/gas-price-per-byte.png)

We will use the example of sending ERC-20 tokens to demonstrate the technique. The ERC-20 standard for sending a token is called [transfer](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/#body), it has two mandatory arguments: an address and an amount. The amount and address values will have to be included in the data field of the transaction and, therefore, increase the total gas cost. 

```
Function: transfer(address _to, uint256 _amount)

MethodID: 0xa9059cbb
[0]:  000000000000000000000000D850942eF8811f2A866692A623011bDE52a462C1
[1]:  0000000000000000000000000000000000000000000000A2A15D09519BE00000
```

The example tranasction above translates to the following.

```
    transfer
0   _to         address         0xD850942eF8811f2A866692A623011bDE52a462C1
1   _amount     uint256         3000000000000000000000
```

These values can be found on most [blockchain explorers](https://etherscan.io/tx/0xabb28019cb67085bc676a23d9d1511516b0ab75e39da424fdeeab7953626e95c) in the 'Input Data' field. The transaction above specifies 3000000000000000000000 as sender \_amount, as most tokens have a precision of 18 decimals, this translates to 3,000 actual tokens. Both of the arguments are padded to a fixed length, 64 bytes. Addresses are simply padded with leading zero's while the \_amount parameter is a hexadecimal string of the [unsigned 256-bit integer](https://en.wikipedia.org/wiki/256-bit_computing). The contents of the data field consists of the concatenation of the MethodID, the \_to, and the \_amount field. This results in 4 bytes for the method id and 64 bytes for both the address and amount.

```
                   | non-zero bytes    | zero bytes    | formula            | cost  
-------------------|-------------------|---------------|--------------------|-----------
 methodId          | 4                 | 0             | 4 x 68             | 272  
 _to               | 40                | 24            | 40 * 68 + 24 * 4   | 2816 
 _amount           | 14                | 50            | 14 * 68 + 50 * 4   | 1152 
-------------------|-------------------|---------------|--------------------|-------- +
 total             | 58                | 74            | 58 * 68 + 74 * 4   | 4240 gas
```

## What can we save?

We have seen that gas cost depends on the transaction data field, but how can we leverage it without materially affecting our transaction? We are used to sending rounded numbers; sending exactly 100 tokens feels better than 99.98895 tokens. Machines also appreciate round numbers but they do differ in what they think is 'round'. Since all values are base 256 instead of our humanly accepted decimal, or base 10, computers don't see 100 tokens as a round number. The number of zero-bytes directly correlates to the roundness of a value in base 256.

Take the example above where sending 3.000 tokens with an 18 decimal precision results in the hex string ```A2A15D09519BE00000```. Slightly changing the number of tokens to 3,000.045870175091687424 would result in ```A2A200000000000000```. Sending 0.00153% additional tokens reduces the number of non-zero bytes in the string by 5, saving 320 gas! At a gas price of 150 gwei per gas that results in 48,000 gwei, or 0.000048 ether. With the current price of $2,300 per ether, this converts to **over $0.11 saved**.


```
             | amount       | alternative          | change in value | savings in gas  | $ savings
-------------|--------------|----------------------|-----------------|-----------------|-----------
 small       | 1            | 1.00000032790413312  | < 0.0001%       | 192             | $0.066
 large       | 50,450,100   | 50,450,111.047652..  | < 0.0001%       | 384             | $0.132
 best case   | EFFFF....    | F0000000000000000..  | < 1/10^77       | 4032            | $1.391

 The dollar savings calculation assumes the current values of $2300 per eth and gas price of 150 gwei.
```

We see that changing a transaction amount by less than 0.0001% saves between $0.066 and $0.132 while in the absolute best case a potential $1.391 can be saved by changing 63 non-zero bytes to zero's. A basic token transfer transaction costs about 42.000 gas, saving 192 would be **0.46%** while saving 384 would amount to **0.91%** of the **total transaction**, including execution cost. The theoretical best-case scenario saves almost **9%**!

## To the extreme

We have seen that addresses also influence the transaction price; sending tokens to 0x000000000000000000000000 is cheaper than sending the same amount to any other address. If you expect to receive an extreme amount of transfer transactions it might be worth it to generate an address with a lot of zero bytes. The same concept works for smart contracts, although it is more expensive to iterate through addresses as the deployment costs have to be paid for each attempt to get an address with many zero bytes.

One step further, we're able to use method names within the solidity smart contract that hash to values with zero bytes so any transaction using that method will be cheaper. Since this method reduces the gas amount of a transaction it even allows for more transactions to fit in a single block.


## In summary

We have seen that we can realistically reduce transaction costs by up to 1% by changing the values we use in those transactions. Since numbers are encoded in base 256 and zero-bytes are cheaper than non-zero bytes, using values that, when converted to base 256, result in zero bytes will reduce transaction costs. The required changes are minimal, yielding most of the results with changes smaller than 0.0001%. 

Because of this, we can save some of the gas cost of transactions if we allow for a slight variation in the number of e.g. tokens we send. The savings scale slightly with the variance that is allowed; allowing a 0.1% token value change will often yield a cheaper transaction than a 0.001% value change. 

Since this is also true for referencing addresses, it is less expensive to send tokens to an address with zero bytes than one without. Deploying a smart contract on an address with zero bytes would reduce the gas cost of any future transaction that references the contract. This would be beneficial if your contract handles transactions that e.g. need a token to be approved with the [ERC-20 standard](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/) 'approve' or transfer function.
