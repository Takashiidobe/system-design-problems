# Design a Market Maker

A Market Maker is a company that provides liquidity to the market, by
buying an asset when there is an offer to sell an asset for a certain
amount below the asset's real worth, and selling an asset when there is
an offer to buy an asset for a certain amount above the asset's real
worth.

For example, let's say AAPL is selling for $100, and we will buy any
AAPL sold for $99, and will sell AAPL to any buyer willing to buy AAPL
for more than $101.

Design an algorithm for a market maker, and design a system to scale it
properly.

## Algorithm

Write some code to create an algorithm that would maximize the amount of
money, assuming that nobody else buys and sells.

What happens if this becomes an n-sided market, where there are many
market makers? How does this change the problem?

Assume you have a limited amount of capital, which you don't want to
expend all at once (since there's volatility in asset prices, which may
go up +-10% a day), and there might be a larger amount of sell offers
than buy offers, for a particular asset. What happens if a stock goes up
too much, or down too much in a given day?


### Algorithm thoughts
We need a greedy algorithm that has fast access going through the existing data. We also need to concern ourselves with how are we getting access to this market data? 
We will be looking at the market one state at a time. I would say just find the maximum buy order at a given time and go for it, but keep in mind other people will be 
competing with us to sell to customers. So I reccomend we find the first existing buy order that gives us that 1% profit and execute it before our competition beats us. 
This will accomplish 2 things.
1.   we get to the resource faster than someone with a more greedy profit margin
2. we will have very fast turnaround time and allow us to execute more orders in each batch, thus making profit via quantity of sales, rather than one big payday. 

## System Design
Design a system that listens to a server that sends requests, records
them to a database, and executes or passes on each offer, based on if it
is being sold for a 1% discount or if someone is willing to buy at a 1%
premium.

# Step 1: Outline usecases, constraints, and assumptions
- Q1. Is this system more read or more write heavy?
- A1. probably pretty balanced in the ratio because we will show many stock prices to many users as well as store a history of the orders
- Q2. What is the best data representation of stock information? the scc will want to have access to all of our trades so i reccomend we split the data into two layers
1. The customer layer which stores a minimal amount of information to allow for speed
2. a more reliable timeseries based layer we can use to turn in tax information and have our quants do analysis of our profits and maybe tweak our algortihm accordingly



# Initial Thoughts
We need a very fast price comparison algorithm that also is greedy by nature. We want to grab the first stock that will replace our current stock.
Another concern we might want to raise is how do we index stocks and if we see a pattern of increase do we want to buy a large chunk of shares? What
can we do programatically to get larger than 1% of profits? Do we want to hold shares of every stock? or just select stocks? Do we want to buy stocks
at a higher volume?

# Step 2: Create High Level Design
1. Web Server Layer responsible for serving orders to our frontend
2. UI layer stateless retrieving data explicitly from the server
3. Customer Database layer (KV store)
4. Persistence layer(timeseries based datastore)




The web server needs to have 4 router compoents

1. Buy/Sell customer api responsible for reccomending stocks, aquiring a big enough stake for that trading day and executing the buy and sell orders.
2. Authentication. All information should only be accessed through a user account. 
3. transaction log. A durable history of all transactions our company has made.
4. Historical Stock Index for all stocks we trade.
5. Selection algorithm

The UI Layer just needs a couple sections
1. Search for stock
2. Query stocks available to buy as well as a graph of price history.


lets go over the scaling concerns of each one of our web server components.


### Buy sell Layer
Here we need to have a KV layer in which we serve a stock and its price on our market maker to our customer.
A stock object will look like

```python
@dataclass
class Stock:
  ticker: str
  price: int
  timestamp: datetime
```
For each stock we will do two things with it

#### View Concerns
Every time the price of a stock updates we can have a new cell price. The KV store will have a key for each stock that we are selling as well as a value. 
We will update this value every epoch second so we will want to have fast access to this value. Hash index is perfect for this. 
this way accessing the value of a stock that is queried is O(1) average case and O(log(n)) worst case. 
#### Buy Concerns
We only want to buy a stock when it has dipped in value then sell it at the next spike. So we want to check the existing key value store and see if our target stocks are at a low price.
Maybe some sort of target price can be generated where we also store the original buy price and only sell when the stock has gone up by 1%
```python
if buy stonk:
	transfer stock to new owner 
	then:
      if successful
		update our transaction ledger
      else;
		respond with error message
```
#### Sell Concerns
We want to sell expensive shares and try to scam our customers. 

If an asset is more popular we can buy more so also storing information on market cap could be useful.



### Transaction Log Layer
This layer needs to have multiple redundancies and high fault tolerance. Everytime we write to this layer, we need to make sure it successfully writes to at least 3 database shards before cconsidering it a success. This will mainly be used in a datascience purpose and for taxes so we dont need to have efficent reads just need to prioritize data reliablity.
