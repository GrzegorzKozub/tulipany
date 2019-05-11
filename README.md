# Tulips

This is an example architecture design for online Dutch auction system.

I will focus on the design process while trying to highlight the considerations, explain the trade-offs and argue decisions we've taken. I will ignore the obvious matters.

I will use the following sources for the functionality I would like to provide:

* [Wikipedia article](https://pl.wikipedia.org/wiki/Aukcja_holenderska)
* [Clemens example](https://clemens.pl/aukcje-holenderskie)

Hope you like it!

## Exploring the domain

I started from the problem space, meaning the domain. In this section I'm aiming to show you which functionality I have selected and how I understand they should work.

### The language

These are the terms I will be using throughout:

* Auctioneer - person/user who offers the goods
* Bidder - person/user interested in buying the goods
* Starting price - the price at which the auction starts, usually much higher than the expected one
* Current price 
* Minimal price - the price at which the auction ends when there's no bids
* Increment - the amount of money deducted from the current price every (in my example) hour
* Bid - when a bidder decides to purchase the goods at the current price
* Proxy bidding - declared explicitly while the auction is in progress, maximum price a bidder is willing to pay for the goods
* Transaction - the actual fact of making the purchase
* Commission - paid by the auctioneer when an auction ends with transaction

### Functionality in general

I think these are the more interesting functionalities that are worth exploring in depth. I'm trying limit myself to "what" here but sometimes it's impossible not to point out the potential challenges.

1. Auctioneer manages their auctions

    Auctioneers can decide if the minimal price is visible to the bidders. If that's what they opt for, maybe we also want to show the clock to the bidders.

    How we store auctions really depends on how fast and to how broad bidder audience we want to show them.

2. Auction starts

    Wait, so there's a concept of a clock here? Needs to be snchronized? Central?

3. Bidder lists and views auctions

    Most of the auction page is static really. Some text and some pictures. But the current price and the clock are often changing.

    We probably want to let the bidders know about price changes and that an auction has ended immediatelly while they're still on the auction page.

    As the current price decreases, more bidders will most likely be insterested in that particular auction and so that page will get visited more. Did we just find a potential peak period?

    What about searching?

4. Current price is decreased by the increment as the clock progresses

    If there's an eligible proxy bidding, the auction ends immediately with a transaction. If the previous current price was indeed a minimal price, the auction ends without any bids. Otherwise a new current price gets displayed on the auction page.

    In all these cases the bidders, or people who are viewing the auction page, should probably be notified.

5. Bidder sets up a proxy bidding

    Only possible when the auction is in progress.

    The bid should not be lower than the minimal price. If the minimal price is hidden there's a possibility for the bidders to sniff it with this option, if incorrectly implemeted.

6. Bidder offers a bid and auction ends with transaction

    Once the aution ends we should immediatelly disable the option to submit a bidding.

    The challenge here is a race condition when possibly multiple bidders will hit the Buy now button at (close to) the same time. Adds to the argument for syncing the clock.

    Besides properly selecting the first one who decided to make a purchase we need to gracefully inform the bidders who managed to hit Buy now but still did not make it on time.

Since they're rather commonly implemented, I've chosen to ignore things like user registration and authentication. I'm still debating if I should talk about processing the transaction and commissions.

### Thinking events

...

### Possible abuse

If I trust [these conclusions](https://clemens.pl/aukcje-holenderskie#bezpieczenstwo), in contrast to classic auctions, Dutch auctions are less prone to the common unfair practices.

## User's point of view

...
