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

I think these are the more interesting functionalities that are worth exploring in depth.

1. Auctioneer manages their auctions
2. Auction starts
3. Bidder lists and views auctions
4. Current price is decreased by the increment as the clock progresses
5. Bidder sets up a proxy bidding
6. Bidder offers a bid and auction ends with transaction

In order to simplify our analysis I will skip the search, moderation, transaction processing and commissions for now.

Since they're rather commonly implemented, I've chosen to ignore things like user registration and authentication.

### Looking closer at what happens

Now I'm going to zoom in to find out what events take place within our Dutch auction system. I'm trying limit myself to "what" here but sometimes it's impossible not to point out the potential challenges.

#### Auctioneer manages their auctions

![](https://github.com/GrzegorzKozub/tulipany/raw/master/manage-auctions.png)

Auctioneers can decide if the minimal price is visible to the bidders. If that's what they opt for, maybe we also want to show the clock to the bidders.

How we store auctions really depends on how fast and to how broad bidder audience we want to show them.

Publishing an auction makes it visible to the bidders.

**New challenges**

* I will need a scheduler, possibly with a centralized clock

**Changes when iterating**

* In case the auction should start immediately we should ~~also start the clock immediately~~ defer to some kind of schedule anyway.

#### Auction starts

![](https://github.com/GrzegorzKozub/tulipany/raw/master/start-auction.png)

Auction can start immediately after it was polished or at a specific start date and time. That depends on how the auctioneer set it up.

**New challenges**

* Updating the currently open auction page for all the current viewers will likely require an ongoing connection via something similar to Web Sockets so that the server can send the updates immediately. This will put a burden on the server when there's many concurrent viewers. 

**Changes when iterating**

* When handling a scheduled start auction command, the scheduler creates a new schedule to decrement the current price. Is this idea error-prone? Would it be better to setup the whole schedule for all decrements up front and potentially remove it when the auction ends? Probably it would also simplify the code and allow better SRP. Let's remove Decrement Scheduled then and redefine Auction Scheduled from the previous section.

#### Bidder lists and views auctions

![](https://github.com/GrzegorzKozub/tulipany/raw/master/view-auctions.png)

Most of the auction page is static really. Some text and some pictures. But the current price and the clock are often changing.

We probably want to let the bidders know about price changes and that an auction has ended immediately while they're still on the auction page.

**New challenges**

* As the current price decreases, more bidders will most likely be interested in that particular auction and so that page will get visited more. Did we just find a potential peak period?

#### Current price is decreased by the increment as the clock progresses

![](https://github.com/GrzegorzKozub/tulipany/raw/master/decrement-current-price.png)

If the previous current price was indeed a minimal price, the auction ends without any bids. If there's an eligible proxy bidding, the auction ends immediately with a transaction. Otherwise a new current price gets displayed on the auction page.

In all these cases the bidders, or people who are viewing the auction page, should probably be notified.

**New challenges**

* For this system to remain fair, all the bidders should see the new price as close to the same time as possible. This adds to the argument for synchronizing the clock.
* Minimal price and proxy bidding checks should be quick but will still require a non-zero amount of time. Maybe it's a good idea to lock the auction at every scheduled increment. This lock should not be noticed by the users.

#### Bidder sets up a proxy bidding

![](https://github.com/GrzegorzKozub/tulipany/raw/master/submit-proxy-bidding.png)

Only possible when the auction is in progress.

The bid should not be lower than the minimal price. If the minimal price is hidden there's a possibility for the bidders to sniff it with this option, if incorrectly implemented.

#### Bidder offers a bid and auction ends with transaction

![](https://github.com/GrzegorzKozub/tulipany/raw/master/buy-now.png)

Once the auction ends we should immediately disable the option to submit a bidding.

Besides properly selecting the first one who decided to make a purchase we need to gracefully inform the bidders who managed to hit Buy now but still did not make it on time.

**New challenges**

* There is a race condition when possibly multiple bidders will hit the Buy now button at (close to) the same time. Adds to the argument for syncing the clock.
* The API handling the Buy now button should expect peaks close to the scheduled current price increments for auctions with clock visible.

### User interface

I would like to have mobile and desktop clients which makes me want to put as much business logic as possible on he server and expose it via the APIs to the UIs that are only responsible for displaying the data really. Here's how I would imagine the UI would roughly look:

![](https://github.com/GrzegorzKozub/tulipany/raw/master/ui.png)

The challenges that I was able to see from drawing the UI were already identified, namely:

* Clock synchronization that shows the same current price and time remaining to all the bidders 
* Disabling the purchase options as soon as the auction ends
* Potential high number of concurrent viewers which will impact the server load and increase the burden on the clock sync solution

### Possible abuse

If I trust [these conclusions](https://clemens.pl/aukcje-holenderskie#bezpieczenstwo), in contrast to classic auctions, Dutch auctions are less prone to the common unfair practices.

