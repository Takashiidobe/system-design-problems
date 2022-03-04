# Market Maker

## Algorithm

> Write some code to create an algorithm that would maximize the amount of
> money, assuming that nobody else buys and sells.

some pseudocode:

```py
for order in orders:
  curr_asset_price = lookup_asset_price(order[asset])
  if order[buy] and order[price] >= curr_asset_price * 1.01:
    execute_sell_order(order)
  elif order[sell] and order[price] <= curr_asset_price * .99:
    execute_buy_order(order)
```

> What happens if this becomes an n-sided market, where there are many
> market makers? How does this change the problem?

If this becomes a n-sided problem, speed matters quite a lot -- if you
wait, someone else will snatch up the trade. Also, with more
competition, the spread between bid and ask might have to tighten, but
if there's less competition, the bid and ask spread can widen. You can
code up an algorithm that adjusts automatically to market volume in
this case.

> Assume you have a limited amount of capital, which you don't want to
> expend all at once (since there's volatility in asset prices, which may
> go up +-10% a day), and there might be a larger amount of sell offers
> than buy offers, for a particular asset. What happens if a stock goes up
> too much, or down too much in a given day?

If a stock goes up too much or too little in a given day, as a market
maker, you'd like to avoid the stock (since volatility decreases money
earned). You might try to calculate volatility and avoid volatile
assets.

## System Design

> Design a system that listens to a server that sends requests, records
> them to a database, and executes or passes on each offer, based on if it
> is being sold for a 1% discount or if someone is willing to buy at a 1%
> premium.

First, let's ask about durability guarantees. Since we're a taxable
financial entity, for tax purposes, we'll need to persist every
executed order to stable storage, like a database.

We can imagine that we would **receive** a potential offer, do some
logic to check if we'd like to execute, and if so, send an **execute**
back to the server. The server will then respond back to us with a
response that tells us whether or not we were fast enough. If we were,
we can record the result in the database. The database schema might
look like this:

```sql
create table transactions (
  id INTEGER PRIMARY KEY AUTO INCREMENT,
  created_at DATETIME NOT NULL DEFAULT NOW(),
  asset_id INTEGER FOREIGN KEY REFERENCES ASSETS.ID,
  price MONEY NOT NULL,
  action ENUM { BUY | SELL } NOT NULL,
);
```

The database doesn't need to be particularly fast, but it's beneficial
to be async (so we don't block the main thread). This would also be
write-only, so we can disable indexes and optimize for writes. We
also care a lot about durability, so we could have a cluster of nodes
taken care of by zookeeper, so writes are replicated.

To execute orders that are properly profitable for us, we also need to
keep the correct price. Let's assume that there is some other service
we can make RPC calls to that follows this interface:

```json
GET /{ticker}
// returns the current price of the asset, time, and asset.
{
    "price": 100.00,
    "time": 1646358424,
    "ticker": AAPL
}
```

We want to make sure we don't have stale data on the current price set
by the market, because if we do, we might buy or sell the asset at a
wrong price, losing us money. So we need to find a way to efficiently
be able to query and update our prices.

To do this, let's also assume we have an RPC endpoint that we can
query for all tickers we are going to buy or sell for the day.

```json
GET /all
// returns the assets to buy and sell.
[
  "AAPL",
  "AMZN",
  "FB",
  ...
]
```

On startup, we will query this endpoint, and make two copies of a
hashmap, where all of the keys are the tickers.
In the first hashmap, it will look like { Ticker -Datetime }, where
this hashmap displays the time a particular ticker had its time last
fetched.

For the second hashmap, it will look like { Ticker -Price }. We'll
want a HashMap type that scales well with updates, since we won't
insert at all into this hashmap, but will read and update. To do this,
we'll want to use an architecture with atomic swaps available, and lay
out our hashmap almost alphabetically, so we can index in quickly, and
quickly update values. Something like a `RwLock<HashMap<K, Mutex<V>>>` to allow for this.

We'll also need to handle some logic for when we should update the
asset -- if the asset has high trading, we should update its price
more often. If we notice that we are losing many offers, i.e. we are
slow/there is much contention for offers, we might lower our spread
in order to get contracts that are less lucrative, but still help our
bottom line.

If there is a particular asset that is extremely volatile, we may
choose to denylist it, since our strategy of money making is poor when
it comes to volatile assets.

## Back of the Envelope Numbers

Assuming we have 5000 tickers we care about, and there are 100 offers
every second, we need to be able to handle 500k read requests per second,
parse them, and choose whether or not to execute them.

We might choose to accept 1% of those offers, and win 10% of those, so
we can assume that our write throughput to our database would be close
to 500 requests per second, which should be pretty easy, given that we
are also appending only.

To scale reading and executing, we might decide to scale horizontally,
since our "execution" layer looks kind of stateless. But unfortunately,
it's not -- it needs to query for asset prices in the background, and we
want to prioritize speed in decisions. Also, there is some overhead in
maintaining these hashmaps (5000 \* 5B per ticker + 4B per price).

We'll have to buy some beefier machines, and try to scale that way,
instead of doing a purely stateless approach.

All writes to the database **must** be asynchronous, so they don't block
the main thread, because this would mean slower response times on
offers, which would lose us money. To feel better about this, we may
choose to also allocate each node a set amount of ids, which they can
use to generate UUIDs to send to a queue for processing. We could have
each node keep a cache of writes that it previously sent to the queue,
and using the id for idempotency on the queue, so we can send a message
multiple times, but the query would be rejected if the id was already
processed by the queue.

We would have to think more about calculating volatility, but one way of
doing so might be using another service, which would calculate the
volatility of assets by marking their price spread over rolling 5-10m
averages, and pub-subbing to the nodes any volatile assets which are
meant to be sold off. At that time, each node would only sell the
particular asset it holds, and may choose to lower its spread, so as to
get rid of the volatile asset quickly before it holds onto a price that
the market has moved past from.


