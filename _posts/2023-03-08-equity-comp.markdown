---
layout: post
title: The Equity Compensation Hack
author: Dave
draft: true
---

It's layoff season, and lots of people are looking for jobs and/or getting hired right now. Today we're going to take a break from software and talk about equity-based compensation instead. Much has been written in this area; the [Holloway Guide](https://www.holloway.com/g/equity-compensation) for this is a great place to start if you want the details. However, I wanted to start at a much higher level, and answer foundational questions like

* Why is the company I'm talking to offering to pay me in stock in addition to cash?
* Why are they more willing to negotiate on stock awards than cash?

The short answer for both these questions is *because then they don't have to pay you*. But let's dig in a little bit and see what *that* means! 🙂

(Before we move on, since this touches on finance, I'm should to point out this is not financial advice. I'll do my best to give you helpful background info, but ultimately, if you need information to make a financial decision, you should always talk to a professional!)

## Background: The Origin of Stock

The history of stock is long and winding, and intertwines with several other developments in law and finance, as well as quite a bit of uh, *financial engineering* (read: fuckery). But, ultimately, stock started out as a tool for managing ownership in a company. So let's review what we know about companies.

A company is a legal construct &mdash; one that can do stuff like have bank accounts, own stuff, take on debt, pay taxes, get sued, and default on loans. If you plan to go into business, it's widely considered a good idea to do so as a company rather than doing business as yourself. That way, if you default on a loan or get sued, your company is on the hook to pay those bills, but *you* don't end up having to sell off *your* personal stuff to pay those bills!

Every company has owners: people who are ultimately responsible for the company, and are also entitled to its profits, should the company manage to make more money than it spends. Being an owner is not the same thing as being the CEO. The owners of a company elect, and delegate their authority to a CEO, who becomes in charge of making day-to-day decisions while the owners go off live their lives. The situation can be confusing, though, because early on, the CEO is often also the sole owner; however, as time goes on, most companies end up with more complex ownership scenarios involving many people.

Fundamentally, stock is just a tool for managing these complex ownership scenarios. When you have a large network of people who jointly own a company, are all responsible for that company, and are all entitled to the company's profits, how do you manage who-decides-what and who-gets-what? Stock is a system for answering those questions algorithmically.

In this context, **stock** refers, abstractly, to ownership of a company. A stock is split into **shares**, each of which represents a portion of the overall stock (ownership). Each share belongs to one of the company's owners (which can either be a person or a company); anyone who owns a share is called a **share holder**.

Ownership of a company mainly involves two things: making decisions and sharing profits. Each share of a stock represents an equal share in those two activites. So, if there are $N$ shares of a company's stock in existence, then each share represents $1/N$ of the total decision-making power, and is entitled to $1/N$ of the company's profits. In practice, this often works as follows:

* When the CEO needs the owners to make a decision, the company puts the decision to a vote, usually of the majority-rules kind. Each share is worth one vote, and each share holder can cast one vote per share they own. Thus, stock shares are a proxy for making decisions.

* When profits are distributed among the owners, the total profit to be distributed is split up equally among the shares. So, if there are $M$ dollars of profit and $N$ stock shares in existence, then each share is given $M/N$ dollars of the company's profit. Thus, stock shares are a proxy for profit-sharing too.

Although the profit-sharing aspect scales well as the number of share holders grows, the decision-making aspect breaks down once there are too many people: it becomes cumbersome to have so many people vote on so many decisions. To better scale the decision-making aspect, many large companies elect to set up a **board of directors**, which is a group of people who are elected by the share holders and make decisions on their behalf, just like how you elect members of congress to make laws for you. If you hold stock shares in a company that has a board, then generally you're only asked to vote on the "meta" questions, like ...

1. Who should be on the board of directors? You care since they represent you and your interests.
2. How many stock shares should exist? You care since increasing the number $N$ of stock shares in existence decreases the $1/N$ power of you share(s) proportionally

Once these are established, the board can decide things like what to do with profits (should they be shared with the share holders or reinvested?), long-term company vision / strategy decisions, hiring decisions for key employees like the CEO, and so on.

Now, so far, we haven't  talked about stock in terms of dollar amounts or share prices. That's because stocks aren't money. However, to many people, they can be *worth* money. Let's talk about that next.

## Trading Stock

A stock share is just a piece of paper (usually virtual, nowadays). You can sell it to someone else, if you can find a willing buyer. When you sell a share to someone, they take over both the decision-making aspect and the profit-sharing aspect of those share(s). Like any sale, the price is completely up to you and your buyer.

A **stock exchange** is a market to help facilitate buying and selling stock. An exchange is just somewhere potential buyers and potential sellers of stock come together and find each other, physically or virtually. The ultimate goals of a stock exchange are to (a) hook up each seller with a buyer, (b) help the two agree on a sale price, and (c) complete the exchange (cash for stock share(s)).

But how much is a share in some company worth? That turns out to be a very tricky question to answer!

The truth is, there's no rule for how much a share is worth. It depends on each person and how they feel about the stock they're trying to obtain or part with. So, instead of calculating prices a priori, exchanges help you *discover* the de facto price of the shares a posteriori, by telling you what shares of this company's stock went for on the most recent transaction. In theory, since the seller was trying to maximize the sale price and the buyer was trying to minimize it, whatever they agreed on must be the 'true' price.

This process, of monitoring and publishing the most recent sale price for a stock, is called **price discovery**. The dollar amount you see for a company in a stock ticker (like if you google "MSFT stock" right now), is the historical record of the sale prices over time.

In theory, the price of a stock should be bounded by (how much profit you expect the company to pay out per share every year) times (how many years you plan to keep the share); if you pay more than that much for a share, then you lost money in the long term. This provides a theoretical bound for how much a stock should be worth.

In practice, it's very common for stocks to break this bound. Many people who buy and sell stock on an exchange have zero interest in owning the company: they don't vote on decisions, and they don't really care how much profit the company pays out to share holders. Instead, they're tending to take advantage of fluctuations in the share's price, making money by buying the share when it's cheap and selling it when it's more expensive. This is an activity called **speculating**. You can speculate on anything that can be bought or sold, as long as there's fluctuation in price; stock shares are a common thing to speculate on because there's no physical object that has to be bought, sold, transported, warehoused, etc.

For speculators, the value of a stock share is a social construct: it's worth what everyone else thinks it's worth. What separates speculation on stock shares from speculation on crypto-scamcoin-of-the-week is some hope the theoretical basis of share price will anchor speculative prices to some kind of reality, coupled with heavy-handed regulation and intervention by the government. This generally helps keep shares from suddenly rocketing or cratering in price.

In a tail-wagging-the-dog kind of situation, many companies have stopped sharing profits with their share holders altogether, and instead reinvest the profits into growing the business; the vast majority of share holders are completely happy with that decision, because they care about the resale price of the stock share, not the profit-sharing aspect of the share. This further complicates any attempt to anchor the value of a stock share from anything other than guesswork.

## But then what is stock?

We've now arrived to a very weird place.

Stock exists as a way for people to own a company: make decisions that guide how the company grows and operates, and share in the company's profits. But that's not how people are really using the stock.

Share holders mostly are using their shares as an investment vehicle. They simply want someone else to believe the shares are worth a lot of money, so they can then go sell their shares for a lot of money. 

Interestingly, this happens to open up opportunties for companies themselves to do a little ... *financial engineering* to turn stocks into free money. We've found a few ways to make this work:

## Hack: Stock for Fundraising

Companies often need money up front to set up business operations that make money down the line. If they don't already have the money up front, they need to get it from somewhere. That's called **fundraising**. Common ways to source funds include taking out loans (usually from a bank) or selling ownership to venture capital, which are people or firms that have lots of money and invest that money in growing companies.

But here's a kind of funny idea: why not fundraise on a stock exchange?

Here's the plan:

1. Print more stock shares. This is easy, because you, the company, control how many shares of your stock exist. You don't have to ask anyone or pay anyone to make more shares.
2. Give the stock shares to a bank, and ask the bank to sell them on a stock exchange
3. Have the bank put whatever money they get by selling those shares into your bank account
4. Now you have money!

Infinite money glitch? Hardly! The amount of money you can fundraise this way is limited to your **valuation**, and even then, you can only capture a small portion of your valuation this way.

The "valuation" of a company is the total worth of its entire stock &mdash; the value of all its shares combined. It's impossible to actually calculate a company's valuation since there's no way to know the value of a company a priori, but you can estimate the valuation simply as (number of shares in existence) times (current share price on a stock exchange). Although stock prices are a somewhat social construct, we all agree that when a company prints more shares, doing so doesn't affect its valuation.

Say that

* $N_B$ ("number before") is the number of shares that existed before you printed more shares
* $P_B$ ("price before") is the price per share before you printed more shares
* $V_B$ ("valuation before") is the company's valuation before printing more shares
* $N_A$, $P_A$, $V_A$ are the corresponding values *after* you printed more shares

By defiition of a valuation, $V_B = P_B \times N_B$ and $V_A = P_A \times N_A$. Since the valuation doesn't change when you print more shares, we assume $V_B = V_A$. Then with a tiny bit of algebra, we see that:

$$P_B \times N_B = V_B = V_A = P_A \times N_A$$

$$P_B \times N_B = P_A \times N_A$$

$$P_A = P_B \times \dfrac{N_B}{N_A}$$

In other words, the price per share falls proportionally to how many new shares we printed. For anyone who currently holds a share, this means the value of their share drops in the short term; they're going to vote against printing more shares unless you can make a convincing argument that taking this hit now will lead to a much higher stock price in the future. This provides a natural limiting factor on how many new shares you get to print, which in turn limits how much money you can actually capture this way.

Even so, this strategy is interesting in that

* Whatever funds you raise this way are yours; there's no loan that has to be paid back
* You're not giving up all that much control of the company, assuming the investors who are buying your new stock shares are disinterested speculators; you're not talking to a VC who aims to exert their new shares to change how you do businesss

In a way, you're turning (possibly irrational) exuberance about your company into cold, hard cash with very few strings attached. Kind of neat!

## Hack: Stock for Compensation

Which finally leads us into the opening question for this post: why is the company I'm interviewing with offering me stock, and why are they more flexible to negotiate on stock dollars than salary dollars? It's because stock dollars don't come from the company you're talking to; they come from the investors on the stock exchange to whom you one day sell your stock. The company you're talking to is using their investors to pay you the same way we just saw them use their investors to fundraise. Namely, the idea is:

1. Print more shares
2. Give the new shares to employees
3. Negotiate with the employees as if the shares were money

Of course, a stock share *is not* money. But it's also not too many steps removed from money. Selling shares on an exchange is easy, and the exchange gives you a pretty good estimate of what the shares are worth right now. You may not be allowed to sell the shares right away, but historically, most stocks have increased in value over time, so you can reasonably hope to get *more* money from having to wait, rather than less.

Like before, the company only 'gets' to print more shares if the existing share holders agree to it, which means the number of shares you can print in step 1 is limited to how confident your share holders are that giving these new shares to new employees will result in better business outcomes and increased share prices. So once again, it's not an infinite money glitch.

However, once again, the stock "dollars" they put in your equity package aren't the company's dollars; you have to do the last step of selling those shares on a stock exchange, so really those are investors' dollars, not the company's. That's why companies are usually more willing to budge on stock dollars than salary dollars: salary dollars come from hard-earned funds currently in the company's bank account, whereas stock dollars come from shares the company can print at will, as if they were arcade tickets. There's a lot more flexibility because, really, those dollars don't exist yet anyways.

In fact, this strategy has become *especially* important for tech startups, which need to hire people who demand very large salaries, but by definition have almost no money in the bank. The only lever they can really pull early on in printing stock, so early employees in tech startups tend to be compensated mostly in stock, with small salaries.

Unfortunately for you, this leads to a big problem: what if you can't sell those shares yet?

## The Problem: When Stock isn't Public

