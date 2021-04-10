---
title: 'What you learn from building a blockchain SDK'
date: 2021-04-05T15:16:05+01:00
draft: false
tags: ["blockchain", "Code", "C#"]
---

__*The best way to learn is to do.*__

So what better way to learn more about the inner workings of a blockchain than to build an SDK for one? The goal when coding was to create a functional blockchain SDK and learn some things along the way. Here follows a brief highlight reel of what I’ve encountered while building an SDK for a blockchain. This post goes into the guts of what a transaction is and comes out wielding a funcional SDK.

*This posts assumes basic knowledge of blockchains.*

At the end of 2018, there was no C# SDK available for the VeChain blockchain. There is a [Java SDK](https://github.com/vechain/thor-client-sdk4j) and there was support for Golang, as the chain is built in Go. Luckily VeChain is very similar to Ethereum; it was heavily inspired by it and the developers even credited the Ethereum team in the [genesis block](https://twitter.com/sunshinelu24/status/1012858429822496768). There are, however, some key differences and that’s why we cannot simply use existing Ethereum libraries as is. 
The first key difference is that a [VeChain transaction](https://vechainofficial.medium.com/introducing-the-vechainthor-blockchain-transaction-model-b9944a0b6703) consists of  one or more clauses. A clause can be seen as a transaction on most other chains.

```csharp
public class Clause
{
    string to { get; }
    decimal value { get; }
    string data { get; }
}
```

The main benefit of having a single transaction potentially containing many clauses is that you can ensure the transactions are either all mined or none are. For example, within DeFi you may want to execute a neutral trade that would trade asset A for asset B on a certain exchange and B for A on another. This arbitrage could potentially be lucrative if both transactions succeed but could leave you with a position that is heavily tilted towards one asset if only one of the transactions gets mined. Other parts of the transaction model are mostly the same, with the notable difference that the nonce is not an [ever incrementing value](https://docs.nethereum.com/en/latest/nethereum-managing-nonces/) but a value that the user can set themselves.


```csharp
public class Transaction
{
    public Network chainTag { get; set; }
    public ulong blockRef { get; set; }
    public uint expiration { get; set; }
    public List<Clause> clauses { get; set; }
    public byte gasPriceCoef { get; set; }
    public ulong gas { get; set; }
    public string dependsOn { get; set; }
    public ulong nonce { get; set; }
    private static readonly byte[] Reserved = {0xc0};
    public string signature { get; set; }
    public virtual string id { get; set; }
}
```

## Where do you start?

What are the features that a basic SDK should have? My first requirement was sending a transaction. Seems pretty straightforward. First, run an instance of the [VeChain blockchain](https://github.com/vechain/thor). After that we create a basic implementation of the types needed to talk to the chain.

First time talking to the chain:

```csharp
HttpClient Client = new HttpClient();

public static async Task<Account> GetAccount(string address)
{
    var streamTask = Client.GetStreamAsync($"http://localhost:8669/accounts/{address}");
    var serializer = new DataContractJsonSerializer(typeof(Account));
    return serializer.ReadObject(await streamTask) as Account;
}
```

That seemed to work! This might be the point where we think about the structure of the final solution. We chose a client interface and an implementation that was tailored to talk with the RESTful API of the [official implementation](https://github.com/vechain/thor) of the chain. 

VeChain uses a dual token modal, Vet is the native token that passively generated the ‘gas’ token, VeThor. VeThor is implemented as a VIP-180 token, the equivalent of an ERC-20 token on Ethereum. To send a transaction we spend VeThor and if we want to transfer VeThor we have to call the token smart contract. This means that our SDK should at least be capable of using smart contracts.

We have our goal: building an SDK that is able to send Vet and call smart contracts.

## Built upon

Since transactions need to be signed we will need some algorithms, namely Blake2B and Keccak256. Fortunately, the [BouncyCastle packages](https://www.bouncycastle.org/) contain many mathematical classes and algorithms. They offer support for both of the required algorithms and we only have to create a facade to expose the relevant parts of BouncyCastle. Another topic where BouncyCastle makes things a lot easier is their [BigInteger](https://github.com/bcgit/bc-csharp/blob/master/crypto/src/math/BigInteger.cs) class. We'll often have to deal with values that won’t fit in a (u)long, for these we use the BigInteger type as this .

Another project that our SDK depends on is Nethereum, the .NET implementation of Ethereum. Specifically, the ABI, Hex, RLP, Signer and Util packages are used at this time. 

The existing Java SDK and the Golang code of the chain itself were used as inspiration and some calculations are directly taken from those, like the gas cost calculations for transactions.

## RLP

Recursive Length Prefix, or RLP, encoding is a way to store the values of a transaction in a single hex string.  Each value, the recipient that we see in [codeblock 2] is padded to proper length, encoded as hexadecimal and combined in a list. This encoded list is a fully complete and signed transaction, ready to enter the transaction pool.

As I mentioned, we use the RLP engine from Nethereum. Since the transaction model for Ethereum is different we cannot use the Encode function provided to us since it wouldn’t know what to do with clauses. This is why I thought it was a good idea to create the RLP engine myself. This was incorrect. The individual functions from Nethereum.RLP like EncodeByte, EncodeElement and EncodeList are now used to be able to parse the VeChain clauses.

The final code to get all RLP elements for a vechain transaction is as follows

```csharp
public byte[] RLPData => RLP.EncodeList(GetRlpDataElements());

public byte[][] GetRlpDataElements()
{
    return new[]
    {
        RLP.EncodeByte((byte) chainTag),
        RLP.EncodeElement(blockRef.ToBigEndianBytes().TrimLeading()),
        RLP.EncodeElement(((long) expiration).ToBytesForRLPEncoding().TrimLeading()),
        RLP.EncodeList(clauses.Select(c => c.RLPData).ToArray()),
        RLP.EncodeByte(gasPriceCoef),
        RLP.EncodeElement(gas.ToBigEndianBytes().TrimLeading()),
        RLP.EncodeElement(dependsOn == "" ? null : dependsOn?.HexToByteArray().TrimLeading()),
        RLP.EncodeElement(nonce.ToBigEndianBytes().TrimLeading()),
        Reserved,
        RLP.EncodeElement(signature?.HexToByteArray())
    };
}
```
It is used like this in the client.
```csharp
var stringOutput = ByteArrayToString(txn.RLPData);
var totalProperEncoding = Encoding.UTF8.GetBytes("{\"raw\":\"0x" + stringOutput + "\"}");

var bytes = new ByteArrayContent(totalProperEncoding);

bytes.Headers.ContentType = new MediaTypeHeaderValue("application/json");

var response = await SendPostRequest("/transactions", bytes);
```

## Gas calculations

Because of the clauses the gas cost calculations are somewhat different from Ethereum. Where normally sending multiple transactions would cost [ #tx * price ] with VeChain it is actually cheaper to bundle multiple clauses in a single transaction. This is because of the intrinsic transaction price that is 5.000 gas per transaction plus 16.000 per clause. This results in a single clause Vet transfer transaction costing 21.000 gas, or 21 VeThor at the moment. Adding another clause, that could be unrelated, would only add 16.000 gas, saving over 10% compared with two single clause transactions. Since any action other than sending Vet requires the use of the data field all other transactions are more expensive than 21.000 gas for a transaction or 16.000 gas per clause. There is a cost for each value between 0-255 that is included in the data field. As data is stored as hexadecimal this means that each two characters have a certain cost associated with them. If they are exactly 00 then it adds 4, any other value adds 68 to the cost.


```csharp
public static ulong CalculateDataGas(this IClause clause)
{
    const uint zgas = 4;
    const uint nzgas = 68;

    uint totalGas = 0;

    var data = clause.data;

    for (var i = 2; i < data.Length; i += 2)
    {
        var hexPair = data.Substring(i, 2);
        totalGas += hexPair == "00" ? zgas : nzgas;
    }

    return totalGas;
}
```

When submitting a transaction you need to specify a maximum amount of gas that it is able to use, this means that a correct gas estimation algorithm is required to have a functional SDK.

## ABI

After not working on the project for some months the need arose to support ABI contract definitions. I wanted to automate certain processes on the VeChain blockchain and why not dust off my own code? 

ABI stands for Application Binary Interface, it is the description of the ways to interact with a smart contract. As VeChain uses Solidity smart contracts there was no chain specific implementation needed. 
Importing an ABI string can be done like this.

```csharp
deFiContract = new Contract
{
    AbiDefinition = AbiContractDefinition.ContractBuilder(" [{
        "type":"function",
        "inputs": [{"name":"amount","type":"uint256"}],
        "name":"trade",
        "outputs": []
        }]"),
    Address = "0x1aD91ee08f21bE3dE0BA2ba6918E714dA6B45836",
    Name = "deFiContract"
};
```

The following code block displays a multi clause transaction with an approval of a DeFi token and the spending of that token at a different DeFi contract. 


```csharp
private Transaction DeFiTrade(int amount, ulong nonce, ulong gas)
{
    // Approve the spending of the defi token
    var tokenContractCode = tokenContract.Execute(
            "approve",
            deFiContract.Address,
            new DeFi(amount).ToHex())
        .ToHex(true);

    // Trade with the approved funds
    var defiContractCode = deFiContract.Execute(
            "trade", 
            new DeFi(amount).ToHex())
        .ToHex(true);

    return new Transaction(Network.Main, BlockRef, _settings.Expiration, new List<Clause>
    {
        new ArbitraryClause(DeFi.ContractAddress, 0, tokenContractCode),
        new ArbitraryClause(deFiContract, 0, defiContractCode)
    }, nonce, byte.MinValue, gas, "");
}
```

## Final thoughts

What did we set out to do? Create a functional blockchain SDK and learn some things along the way. Did we succeed? Yes! We have an SDK that supports sending transactions, both simple and more complex contract calls. We’ve seen how to work around differences in Ethereum and VeChain while leveraging existing Ethereum libraries like with the RLP example. 
This post skims over a lot of details (chaintags, getting the right blockref, ..) encountered while constructing the code. I should probably also mention that there is an official .NET release from Vechain since quite some time so the project is deprecated. I would advise anyone looking to use their solution. 

I have to thank [TylerAP](https://github.com/TylerAP) for forking and eventually merging his additions back to the source. I have never met Tyler but it was nice to know that my code helped someone that wanted to code on VeChain in C#. 

I have learned a lot from doing this, I had never known how RLP encoding worked, how the exact cost of a transaction is calculated and how ABI worked internally. The project was purely something for the weekend and the code is *not* production ready. Feel free to browse or use it but it does not come with any guarantee whatsoever. All the code can be found on my [Github](https://github.com/RensR/VeChainCore).
