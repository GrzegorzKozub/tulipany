# Tulips

This is an example architecture design for online Dutch auction system.

I will focus on the design process while trying to highlight the considerations, explain the trade-offs and argue decisions we've taken. I will ignore the obvious matters.

I will use the following sources for the functionality I would like to provide:

* [Wikipedia article](https://pl.wikipedia.org/wiki/Aukcja_holenderska)
* [Clemens example](https://clemens.pl/aukcje-holenderskie)

Hope you like it!

## Glossary

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

## Exploring the domain

I started from the problem space, meaning the domain. In this section I'm aiming to show you which functionality I have selected and how I understand they should work.

I think these are the more interesting functionalities that are worth exploring in depth.

* Auctioneer manages their auctions
* Auction starts
* Bidder lists and views auctions
* Current price is decreased by the increment as the clock progresses
* Bidder sets up a proxy bidding
* Bidder offers a bid and auction ends with transaction

In order to simplify our analysis I will skip the search, moderation, transaction processing and commissions for now.

Since they're rather commonly implemented, I've chosen to ignore things like user registration and authentication.

### Auctioneer manages their auctions

![](https://github.com/GrzegorzKozub/tulipany/raw/master/manage-auctions.png)

Auctioneers can decide if the minimal price is visible to the bidders. If that's what they opt for, maybe we also want to show the clock to the bidders.

How we store auctions really depends on how fast and to how broad bidder audience we want to show them.

Publishing an auction makes it visible to the bidders.

**New challenges**

* I will need a scheduler, possibly with a centralized clock

**Changes when iterating**

* In case the auction should start immediately we should ~~also start the clock immediately~~ defer to some kind of schedule anyway.

### Auction starts

![](https://github.com/GrzegorzKozub/tulipany/raw/master/start-auction.png)

Auction can start immediately after it was polished or at a specific start date and time. That depends on how the auctioneer set it up.

**New challenges**

* Updating the currently open auction page for all the current viewers will likely require an ongoing connection via something similar to Web Sockets so that the server can send the updates immediately. This will put a burden on the server when there's many concurrent viewers. 

*Changes when iterating*

* When handling a scheduled start auction command, the scheduler creates a new schedule to decrement the current price. Is this idea error-prone? Would it be better to setup the whole schedule for all increments up front and potentially remove it when the auction ends? Probably it would also simplify the code and allow better SRP. Let's remove Decrement Scheduled then and redefine Auction Scheduled from the previous section.

### Bidder lists and views auctions

![](https://github.com/GrzegorzKozub/tulipany/raw/master/view-auctions.png)

Most of the auction page is static really. Some text and some pictures. But the current price and the clock are often changing.

We probably want to let the bidders know about price changes and that an auction has ended immediately while they're still on the auction page.

**New challenges**

* As the current price decreases, more bidders will most likely be interested in that particular auction and so that page will get visited more. Did we just find a potential peak period?

### Current price is decreased by the increment as the clock progresses

![](https://github.com/GrzegorzKozub/tulipany/raw/master/decrement-current-price.png)

For simplicity and to make things clearer for the users, we always increment ever hour. The increment amount will differ.

If the previous current price was indeed a minimal price, the auction ends without any bids. If there's an eligible proxy bidding, the auction ends immediately with a transaction. Otherwise a new current price gets displayed on the auction page.

In all these cases the bidders, or people who are viewing the auction page, should probably be notified.

**New challenges**

* For this system to remain fair, all the bidders should see the new price as close to the same time as possible. This adds to the argument for synchronizing the clock.
* Minimal price and proxy bidding checks should be quick but will still require a non-zero amount of time. Maybe it's a good idea to lock the auction at every scheduled increment. This lock should not be noticed by the users.

### Bidder sets up a proxy bidding

![](https://github.com/GrzegorzKozub/tulipany/raw/master/submit-proxy-bidding.png)

Only possible when the auction is in progress.

The bid should not be lower than the minimal price. If the minimal price is hidden there's a possibility for the bidders to sniff it with this option, if incorrectly implemented.

### Bidder offers a bid and auction ends with transaction

![](https://github.com/GrzegorzKozub/tulipany/raw/master/buy-now.png)

Once the auction ends we should immediately disable the option to submit a bidding.

Besides properly selecting the first one who decided to make a purchase we need to gracefully inform the bidders who managed to hit Buy now but still did not make it on time.

**New challenges**

* There is a race condition when possibly multiple bidders will hit the Buy now button at (close to) the same time. Adds to the argument for syncing the clock.
* The API handling the Buy now button should expect peaks close to the scheduled current price increments for auctions with clock visible.

## User interface

I would like to have mobile and desktop clients which makes me want to put as much business logic as possible on he server and expose it via the APIs to the UIs that are only responsible for displaying the data really. Here's how I would imagine the UI would roughly look:

![](https://github.com/GrzegorzKozub/tulipany/raw/master/ui.png)

The challenges that I was able to see from drawing the UI were already identified, namely:

* Clock synchronization that shows the same current price and time remaining to all the bidders 
* Disabling the purchase options as soon as the auction ends
* Potential high number of concurrent viewers which will impact the server load and increase the burden on the clock sync solution

## Possible abuse

If I trust [these conclusions](https://clemens.pl/aukcje-holenderskie#bezpieczenstwo), in contrast to classic auctions, Dutch auctions are less prone to the common unfair practices.

## Drafting the contexts

Looking at the above I will now aim to distil the contexts. I will mainly consider the following factors:

* Potential change to our system should ideally affect just one of the contexts
* Load and scalability requirements should be consistent within a single context

My guts suggest me these initial contexts:

### Management

**Functionality**

* Creating, editing and removing auctions by the auctioneer
* Publishing and unpublishing auctions
* Calculating auction schedules

**Discussion**

Deserves to be there because of a distinct actor, the auctioneer, and because managing auctions will happen considerably less often than viewing and bidding (different scalability).

Groups events that always happen together: management, publishing and scheduling.

**Ideas**

An auction will be stored as a set of relational data including its basic properties and a schedule. On top of that we will store the pre-rendered, static version for fast access. Might consider a light version for the list.

### Pages

**Functionality**

* Listing auctions by the bidders
* Viewing auction pages by the bidders
* Subscribing to the page updates automatically
* Exposing the bidding and notification buttons

**Discussion**

We're servicing a specific event here: a bidder views the auction page. Considering the previously identified challenges, we can expect high load not only in peak times but also simply because we want to have much more bidders than auctioneers. The other source of increased load will come from refreshing the page by a single bidder just to make sure the price did not drop yet. This will drive how we scale this one.

**Ideas**

This context is mainly interested in the pre-rendered static content that it will display on the client. That's really its distinct perspective on the auction entity. We should maybe provide it from memory and distribute it geographically?

When the page it will fetch the current price and the clock from the server and subscribe to the updates of these details in (close to) real time.

### Updates

**Functionality**

* Keeping pages up to date regarding current price and the clock

**Discussion**

This will need to scale similarly to Pages. On the other hand it's not the bidder who usually initiates the action here. It's the clock in most cases.

Initially this context was merged with Notifications. Since there may be considerably less bidders subscribed to notifications than bidders who simply view the page, we may need to scale differently.

**Ideas**

I think this will be just some kind of API that the clients will connect to to and subscribe to the updates via Web Sockets.

**New challenges**

* How will Updates know about the actual changes it needs to forward?
* There will be multiple users keeping auction pages open for long time. Auctions can be configured to last for over a month. We will have multiple open connections that are very long lasting.

### Notifications

**Functionality**

* Sending notifications to the subscribers

**Discussion**

Very similar to Updates but was extracted due to potentially different scalability needs.

### Scheduling

**Functionality**

* Creating schedules
* Starting auctions
* Calculating the increment
* Decrementing current price
* Checking proxy biddings
* Checking minimum price
* Creating transactions
* Removing schedules

**Discussion**

Seems like there's multiple unrelated functionalities in this context. I decided to put all of them into one because most of them happen at the same time, when the next current price increment needs to happen. Also, probably better to keep schedule logic in one place due to potential changes.

For now there's no compelling reason for further split. Later, when we consider transaction handling it may be smarter to move the related logic to that context.

**Ideas**

This seems like a background task to me. It can employ an existing scheduler library like [quantum](https://hex.pm/packages/quantum) that's using familiar crontab metaphor. 

We probably don't want to put all auctions into a single schedule. We should find a way to partition.

After starting an auction and decrementing the current price we need to update the pages and notify the bidders via Updates context. Maybe we could use events here? Would this address the challenge we noticed with the Updates API?

### Bidding

**Functionality**

* Accepting proxy biddings
* Processing buy now requests

**Discussion**

Scales similarly to Updates but the initiator is the bidder.

Proxy biddings have been put here due to domain similarities but I can't find any compelling scalability reason. Might want to change it later.

**Ideas**

Tackling the challenge of making sure we make the indeed first bidder who hits the Buy now button successful, maybe we could do something very light when accepting that request, like saving the winner ID. This would allow us to lock this operation so the others would wait and then quickly get the negative response.

If we go this route, we would then handle actions that are not urgent (updating pages, sending notifications, creating transactions) asynchronously.

We need to consider that if we update the pages quickly enough, then we actually get less Buy now requests in the first place.

## Syncing the clock

Time seems to be important from the point of view of fairness and user experience. 

### Ensuring every node's time is the same

When we scale different contexts of the system, it becomes important so that each node has the same time. This is especially important for the Scheduling context.

A node is not something that would live forever but maybe long enough to consider potential discrepancies over time. Ensuring proper use of [NTP](https://en.wikipedia.org/wiki/Network_Time_Protocol) should alleviate this problem.

### Syncing user's clocks

For the scenario when the clock is displayed on the auction page, we can refresh it when updating the current price. This means every hour which is more than enough.

### Time zone conversion

To avoid problems with the browser settings I would initially put user's time zone as a configuration item on their profile.

## Architecture

...

## Conclusions

* Maybe it would be better to distinct between an auction viewer and a bidder.
* It's hard to do this in a one man team. I tried to challenge my own ideas constantly. This is tiring and my skills and knowledge into goes this far. These designs should be team efforts to increase the chance of success.
