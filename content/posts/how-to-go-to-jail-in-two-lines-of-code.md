---
title: 'How to go to jail in two lines of code'
date: 2021-05-23T16:47:05+01:00
draft: true
tags: ["blockchain", "mining", "short"]
---

__*Why does my laptop sound like a jet engine?*__

While there are an infinite number of code snippets that would land you in jail, today we'll be talking about crypto miners. It only takes two lines of code to start a crypto-miner in your browser. This is called cryptojacking.

```js
var miner = new CoinHive.Anonymous('<site-key>');
miner.start();
```

The website doesn't have to inform you that it's running, as was the case in 2017 with [The Pirate Bay](https://torrentfreak.com/the-pirate-bay-website-runs-a-cryptocurrency-miner-170916/). They ran a [Monero](https://www.getmonero.org/) miner without asking for consent from their users, gaining monetarily from their visitors' CPU cycles.  

## What does it do?

An embedded crypto miner uses your computer to mine cryptocurrencies for someone else. In most cases, they choose Monero as it is an anonymous currency that cannot be tracked. If they chose Bitcoin you would be able to trace the coins you mined using a [bitcoin explorer](https://www.blockchain.com/explorer), as all transactions are public. The miner uses your CPU for their gain: mining them valuable cryptocurrencies. Most miners offer a maximum CPU utilization parameter, the miner then uses, for instance, at most 20% of the CPU's capacity. This reduces that impact for the user, in most cases making it go completely unnoticed while running at 20% mining speed.

In the case of a website that mines while the tab is active it is easy to circumvent; simply close the browser tab. Cryptojacking, however, comes in many forms and most of them are harder to get rid of. Cryptojacking malware has existed for [multiple years](https://www.zdnet.com/article/brutal-cryptominer-crashes-your-pc-when-discovered/) and [botnet mining](https://www.investopedia.com/tech/what-botnet-mining/#:~:text=Cryptocurrency%20mining%20botnets%20are%20making,various%20devices%20across%20the%20globe.&text=The%20working%20mechanism%20of%20such,and%20now%20controls%20their%20system.) has been a lucrative application for botnet owners, with the Smominru miner botnet successfully mining 9.000 Monero (currently worth $1,475,100). 

Most modern ad-block plugins will block crypto miners as they have a distinct signature. While a malware-infected PC might be more difficult  to cure, modern anti-virus tools are aware of the threat and will take immediate action. Case in point, while writing this post, my computer flagged this file as a 'Trojan:JS/CoinHive.B' because it contains the CoinHive miner code. It doesn't seem to understand that JavaScript doesn't run well in markdown files. Luckily was able to convince it that this file is rather harmless.

## All bad?

Some see in-browser mining as a replacement for ads and welcome the mining innovation. A website might be able to supplement income without having to resort to displaying ads. A modern desktop CPU like the AMD 5600X gets about [6.6 kH/s](https://www.hashrates.com/cpus/), bringing in about [$0.41](https://www.cryptocompare.com/mining/calculator/xmr?HashingPower=6.6&HashingUnit=KH%2Fs&PowerConsumption=1200&CostPerkWh=0&MiningPoolFee=1) a day at full power. The average cost per 1000 impressions of advertisements is [between $3 and $10](https://www.topdraw.com/insights/is-online-advertising-expensive/) depending on the platform. When running at 20% capacity the crypto miner brings in $0.0034 per hour. With one single impression costing between $0.003 and $0.010, a miner would have to run between one and three full hours to equal a single ad impression. As most pages have multiple ads running at the same time, the math skews even more in favor of ads.

Currently, ads are orders of magnitude more profitable than running in-browser miners. There is little indication that this will ever change as the [Monero hash rate](https://2miners.com/xmr-network-hashrate#:~:text=Currently%2C%20Monero%20network%20hashrate%20is,Network%20Difficulty%20and%20Hashrate%20Explained.), and therefore mining difficulty, keeps increasing.

## Conclusion

Cryptojacking is a real thing that you might encounter on the web. It doesn't seem like a valid replacement for ads as the monetary benefits are significantly lower. Modern anti-virus or ad-blockers will most likely protect you but when a Chome tab suddenly uses exorbitant CPU power you might want to check it out. Although it could very well be Chrome doing Chrome things. 
