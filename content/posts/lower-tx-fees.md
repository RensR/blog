---
title: 'How to lower L1 transaction fees on any blockchain'
date: 2021-04-11T15:14:05+01:00
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

These values can be seen on most [blockchain explorers](https://etherscan.io/tx/0xabb28019cb67085bc676a23d9d1511516b0ab75e39da424fdeeab7953626e95c)  in the 'Input Data' field. The transaction above specifies 3000000000000000000000 as sender amount, as most tokens have an 18 decimal precicion this translates to 3.000 actual tokens. Addresses are padded to 64 bytes with leading zero's while the \_amount parameter specifies uint256 an unsigned 64 byte integer. This transaction cost structure can be seen in the table below.


```
                   | non-zero bytes    | zero bytes    | formula           | cost  
-------------------|----------------   |------------   |------------------ |--------
 function          | 4                 | 0             | 4 x 68            | 272  
 address           | 40                | 24            | 40 * 68 + 24 * 4  | 2816 
 amount            | 14                | 50            | 14 * 68 + 50 * 4  | 1152 
-------------------|-------------------|---------------|-------------------|-------- +
 total             | 58                | 74            | 58 * 68 + 74 * 4  | 4240 gas
```

## What can we save?

We have seen that gas costs depends on the transaction data field, but how can we leverage it without materially affecting our transaction? We are used to sending rounded numbers, sending exactly 100 tokens feels better than 99.98895 tokens. Machines also appriciate round numbers but they do differ in what they deem round. Since all values are base 256 instead of our humanly accepted [decimal](https://en.wikipedia.org/wiki/Decimal), or base 10, computers don't agree that 100 tokens is a round number.

Take the example above where sending 3.000 tokens with a 18 decimal precision results in the hex string ```A2A15D09519BE00000```. When we transfer exactly 3.000,045870175091687424 tokens it would be equivalent to ```A2A200000000000000```. Increasing the number of tokens to transfer by 0.001529006% reduces the number of non-zero bytes in the string by 5, saving 320 gas! At a gas price of 200 gwei that is 64.000 gwei, or 0.000064 ether. At the current price of $2150 dollar per ether this converts to **over $0.13 saved**.


```
                   | amount            | alternative | % change in value  | savings in gas    | $ savings  
-------------------|----------------   |------------ |------------------- |------------------ |--------
 small             | 1                 | 0           |                    |                   | $   
 large             | 50.450.100        | 50.450.100  |                    |                   | $  
 best case         | EFFFF....         | F0000....   | <1 / 10^77         | 4032              | $1.73

 The dollar savings calculation assumes the current values of $2150 per eth and 200 gwei per gas.
```

EFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
108555083659983933209597798445644913612440610624038028786991485007418559037439

F000000000000000000000000000000000000000000000000000000000000000
108555083659983933209597798445644913612440610624038028786991485007418559037440


table showing % cheaper per dif allowed

actual $ saved using technique table

## To the extreme

redeploy contacts until 0x000000...

contract method names that hash to 0x0000....


## In summary

The transaction cost of contract calls is directly correlated to the number of tokens we transact. Because of this, we are able to save some of the gas cost of transactions if we allow for a slight variation in the number of tokens to transact with. The savings scale with the variance that is allowed e.g. allowing a 1% token value change will yield a cheaper transaction than a 0.1% value change. 

Token amounts are encoded in base 256, therefore, using values [256^x] for any x will yield the cheapest gas cost. Since this is also true for referencing smart contract addresses, when expecting many references to your contract it might be beneficial to redeploy it until you get a contract with a lot of zero's. This would reduce the gas cost of any transaction that references the contract in the future. This would be beneficial if your contract handles transactions that need a token to be approved with e.g. the [ERC-20 standard 'approve'](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/) function.