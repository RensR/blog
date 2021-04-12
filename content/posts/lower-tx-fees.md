---
title: 'How to lower L1 transaction fees on any blockchain'
date: 2021-04-10T15:14:05+01:00
draft: true
tags: ["blockchain", "Code"]
---

__*Cheaper transactions for everyone*__

## Ever increasing transaction costs

Transaction fees on blockchains like Ethereum are once again [prohibitively high](https://etherscan.io/chart/gasprice). Gas prices routinely approach 200 gwei, making even the most basic transaction cost multiple dollars. But what if I told you that we currently don't use transactions optimally? There is a simple method that anyone can use on almost any blockchain to reduce the gas cost of transactions, without relying on layer 2 scaling or waiting for EIP-1559-like proposals. 

The method proposed is able to reduce transaction costs for most smart contract calls and, therefore, almost every token. This might sound too good to be true but there is one downside, you have to be willing to change the amount you transact with slightly in order for it to work. The savings are also marginal, most likely not saving more than one percent of the total transaction cost. At the end of this post we will look into applying the same technique in other situations to safe gas, like contract deployments. 

![Historical transaction fees on Ethereum](/images/eth-gas-prices.png)

## How gas works

In my [previous post](/posts/learn-from-building-sdk/) I went over how the gas price of a transaction is calculated. While the example was from VeChain, the same holds for Ethereum and most Ethereum based blockchains. In summary, transactions have an intrinsic cost and an execution cost. The intrinsic cost for Ethereum is 21.000 gas plus the cost of the data field. As can be seen in the [yellow paper](http://paper.gavwood.com/) appendix H. Virtual Machine Specification, data field cost increases by 4 for every zero byte and 68 for every non-zero byte. If we can replace non-zero bytes with zero bytes the transaction cost will decrease. But is it possible to change the number of non-zero bytes while still sending a desired number of tokens?

![Gas cost for adding bytes to the data field](/images/gas-price-per-byte.png)

On Ethrerum, a transaction that does not require data costs 21.000 gas. The ERC-20 standard for sending a token is called [transfer](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/#body) and required an address and an amount. The ```amount``` and ```address``` values will have to be included in the data field of the transaction and, therefore, increase the cost. 

```
Function: transfer(address _to, uint256 _amount)

MethodID: 0xa9059cbb
[0]:  000000000000000000000000D850942eF8811f2A866692A623011bDE52a462C1
[1]:  0000000000000000000000000000000000000000000000A2A15D09519BE00000
```

The example tranasction above translates to the following.

```
0   _to         address         0xD850942eF8811f2A866692A623011bDE52a462C1
1   _amount     uint256         3000000000000000000000
```

These values can be seen on most [blockchain explorers](https://etherscan.io/tx/0xabb28019cb67085bc676a23d9d1511516b0ab75e39da424fdeeab7953626e95c)  in the 'Input Data' field. The transaction above specifies 3000000000000000000000 as sender amount, as most tokens have an 18 decimal precicion this translates to 3,000 actual tokens. Addresses are padded to 64 bytes with leading zero's while the \_amount parameter specifies uint256 as its type, an unsigned 256 bit integer. The contents of the data field is simply the concattenation of the MethodID, the \_to, and the \_amount field. This results in 4 bytes for the method id and 64 bytes for the address and amount respectively.

```
                   | non-zero bytes    | zero bytes    | formula            | cost  
-------------------|-------------------|---------------|--------------------|-----------
 methodId          | 4                 | 0             | 4 x 68             | 272  
 address           | 40                | 24            | 40 * 68 + 24 * 4   | 2816 
 amount            | 14                | 50            | 14 * 68 + 50 * 4   | 1152 
-------------------|-------------------|---------------|--------------------|-------- +
 total             | 58                | 74            | 58 * 68 + 74 * 4   | 4240 gas
```

## What can we save?

We have seen that gas costs depends on the transaction data field, but how can we leverage it without materially affecting our transaction? We are used to sending rounded numbers, sending exactly 100 tokens feels better than 99.98895 tokens. Machines also appriciate round numbers but they do differ in what they deem round. Since all values are base 256 instead of our humanly accepted decimal, or base 10, computers don't agree that 100 tokens is a round number.

Take the example above where sending 3.000 tokens with a 18 decimal precision results in the hex string ```A2A15D09519BE00000```. When we transfer exactly 3,000.045870175091687424 tokens it would result in ```A2A200000000000000```. Increasing the number of tokens to transfer by 0.00153% reduces the number of non-zero bytes in the string by 5, saving 320 gas! At a gas price of 200 gwei that is 64,000 gwei, or 0.000064 ether. At the current price of $2,150 per ether this converts to **over $0.13 saved**.


```
             | amount       | alternative          | change in value | savings in gas  | $ savings
-------------|--------------|----------------------|-----------------|-----------------|-----------
 small       | 1            | 1.00000032790413312  | < 0.0001%       | 192             | $0.083
 large       | 50,450,100   | 50,450,111.047652..  | < 0.0001%       | 384             | $0.165
 best case   | EFFFF....    | F0000000000000000..  | < 1/10^77       | 4032            | $1.734

 The dollar savings calculation assumes the current values of $2150 per eth and 200 gwei per gas.
```

We see that changing a transaction amount by less than 0.0001% saves between $0.083 and $0.165 while in the absolute best case a potential $1.734 can be saved by changing 63 non-zero bytes to zero's. A basic token transfer transaction costs about 42.000 gas, saving 192 would be **0.46%** while saving 384 would amount to **0.91%** of the **total transaction**.

## To the extreme

We have seen that addresses also unfluence the transaction price; sending tokens to 0x000000000000000000000000 is cheaper than sending the same amount to any other address. If you expect to receive an extreme amount of transfer transactions it might be worth it to generate an address with a lot of zero-bytes. The same concept works for smart contracts, although it is more expensive to iterate through adresses as the deployment costs have to be paid for each attempt to get an address with many zero-bytes.

One step further, we're able to use method names within the solidity smart contract that hash to values with zero-bytes so any transaction using that method will be cheaper.


## In summary

The transaction cost of contract calls is directly correlated to the number of tokens we transact. Because of this, we are able to save some of the gas cost of transactions if we allow for a slight variation in the number of tokens to transact with. The savings scale with the variance that is allowed e.g. allowing a 0.1% token value change will yield a cheaper transaction than a 0.01% value change. 

Token amounts are encoded in base 256, therefore, using values [256^x] for any x will yield the cheapest gas cost. Since this is also true for referencing smart contract addresses, when expecting many references to your contract it might be beneficial to redeploy it until you get a contract with a lot of zero's. This would reduce the gas cost of any transaction that references the contract in the future. This would be beneficial if your contract handles transactions that need a token to be approved with e.g. the [ERC-20 standard 'approve'](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/) function.