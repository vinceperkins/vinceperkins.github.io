---
title: "The Death of Quantopian and My Journey in Algorithmic Trading"  
author: Vincent Perkins
date: 2022-04-29 19:25:00 +0800
categories: [Blogging, Trading]
tags: [Algo Trading, Alpaca, Trading, Python, Zipline]
math: true
mermaid: true
image:
  path: /posts/20220429/quantopian-sd.png
  width: 800
  height: 500
  alt: quantopian shutdown
---
## Introduction 
A common theme shared with traders and investors is the psychological dilemma of executing a trade. The question at hand is always "What if?". The question is prevalent before placing an order, and also during a trade when unrealized gains/losses are fluid. Bearing the weight of volatility in an active position is psychologically taxing and unproductive.

The psychological burden of active investing sparked my interest in algorithmic trading in late 2018. I began investing in early 2017 and from that point on I was hooked because of the stochastic nature of the market . I decided one night in November 2018 that I wanted to automate my investments and build a profitable strategy. I had no knowledge of the algo trading space and only a basic coding knowledge at the time, but I heard from friends well versed in computer science that Python could automate a lot of tasks. I honestly thought my idea to automate investing was novel because no one I personally knew was doing it, but it turns out I was late to the party. 

My quest to learn Python began in 2018 solely because of trading. I wanted to build a system from the ground up as a passion project, but after some research I discovered there was an end to end service already available. I immediately started writing algorithms on Quantopian.com and began to hone my coding skills, mainly learning from forums and documentation. 

Quantopian used a proprietary python library called Zipline which would interact with market data and simulate an investment portfolio through backtesting. Quantopian offered no restriction on over 10 years of market data for live forward testing and historical backtesting, at a 1 minute grain, all for free. For algorithms that needed multiple years of historical data to filter assets, I felt there was no better value than Quantopian.

I also felt that a service with so much value and free of charge could not last forever, and Quantopian went defunct as of November of 2020. I thank the service for introducing me to algo trading, and allowing me to learn python for a meaningful passion project which has continually evolved over the past 3 years. [Zipline](https://github.com/stefan-jansen/zipline-reloaded) has taken a life of its own; despite no longer being maintained by Quantopian, Stafan Jansen has updated the library for newer python versions, and created a new [community](https://exchange.ml4trading.io/) around using the library in tandem with machine learning techniques. In future posts I will detail the setup used for algorithms I run daily. 



