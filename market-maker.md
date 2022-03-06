Design a Market Maker
A Market Maker is a company that provides liquidity to the market, by buying an asset when there is an offer to sell an asset for a certain amount below the asset's real worth, and selling an asset when there is an offer to buy an asset for a certain amount above the asset's real worth.

For example, let's say AAPL is selling for $100, and we will buy any AAPL sold for $99, and will sell AAPL to any buyer willing to buy AAPL for more than $101.

Design an algorithm for a market maker, and design a system to scale it properly.

Algorithm
Write some code to create an algorithm that would maximize the amount of money, assuming that nobody else buys and sells.

What happens if this becomes an n-sided market, where there are many market makers? How does this change the problem?

Assume you have a limited amount of capital, which you don't want to expend all at once (since there's volatility in asset prices, which may go up +-10% a day), and there might be a larger amount of sell offers than buy offers, for a particular asset. What happens if a stock goes up too much, or down too much in a given day?

System Design
Design a system that listens to a server that sends requests, records them to a database, and executes or passes on each offer, based on if it is being sold for a 1% discount or if someone is willing to buy at a 1% premium.
