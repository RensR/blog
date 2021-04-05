---
title: 'Work in progress'
date: 2021-04-05T15:16:05+01:00
draft: false
tags: ["blockchain", "Code", "C#"]
---

This post is a placeholder for the actual blog post.

Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s, when an unknown printer took a galley of type and scrambled it to make a type specimen book. It has survived not only five centuries, but also the leap into electronic typesetting, remaining essentially unchanged. It was popularised in the 1960s with the release of Letraset sheets containing Lorem Ipsum passages, and more recently with desktop publishing software like Aldus PageMaker including versions of Lorem Ipsum.

```csharp
public class Clause
{
    string to { get; }
    decimal value { get; }
    string data { get; }
}
```

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