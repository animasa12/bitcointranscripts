---
title: "Chaincode Decoded: Mempool"
transcript_by: kouloumos via tstbtc v1.0.0 --needs-review
media: https://podcasters.spotify.com/pod/show/chaincode/episodes/Chaincode-Decoded-Mempool---Episode-12-evn0q1
tags: ['anchor-outputs', 'cpfp', 'package-relay', 'rbf']
speakers: ['Mark Erhardt']
categories: ['podcast']
summary: "The Chaincode Decoded segment returns and we jump into the deep end of the mempool."
episode: 12
date: 2021-04-26
additional_resources:
-   title: Child Pays for Parent (CPFP)
    url: https://bitcoinops.org/en/topics/cpfp/
-   title: Replace by Fee (RBF)
    url: https://bitcoinops.org/en/topics/replace-by-fee/
-   title: BIP 125
    url: https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki
-   title: Anchor Outputs
    url: https://bitcoinops.org/en/topics/anchor-outputs/
-   title: Package Relay
    url: https://bitcoinops.org/en/topics/package-relay/
-   title: blockchain.com implements Segwit
    url: https://github.com/blockchain/blockchain-wallet-v4-frontend/pull/1779
---
Speaker 0: 00:00:00

Welcome to the Chaincode podcast.
I'm here with Murch.
Hi there.
Today.
We're gonna jump into the mempool and That's a pun if you didn't get it.
Welcome to chaincode decoded the mempool The mempool an area you are more than familiar with.
The mempool whisperer you've been called.
Let's maybe start with, what's the relationship between the mempool and fees?

Speaker 1: 00:00:31

We often talk about the mempool, but there is no such thing as a global mempool.
Every full node has its own mempool and the mempool is basically just the queue of transactions waiting to get confirmed.
Where confirmed means included in a block.
So by default block template builders will just sort the waiting transactions by the highest effective fee rate, then pick from the top.
The juicier a transaction the quicker it gets confirmed.
Now especially in the last few months we've seen that there was very large queues because we had a huge run up in the price I haven't checked, but I think it's now about 120 days that we haven't cleared the mempool, Maybe 110 since 15th of December.
So mempools are limited and by default they are limited to 300 megabytes of deserialized data.
So that includes all the overhead structure, the previous UTXOs, maybe even the whole transaction that created UTXOs and so forth.
So roughly at about 80 blocks worth of data, the default of 300 megabyte gets exceeded, and at that point, a full node will automatically start purging the lowest fee rate transactions.
They just drop them and tell all their neighboring peers, hey, don't send me anything under this fee rate.
They start raising up their min fee rate.
So the problem that gets introduced here is if a parent transaction is no longer in the mempool, you cannot bump it.

## Child Pays for Parent (CPFP)

Speaker 1: 00:02:12

Because if you try to do a CPFP and the parent isn't there, the child is going to be invalid.
CPFP just for the uninitiated.
Child pays for parent.
Some things that are being done in the context of that is that people are working on package relay where you can send more than one transaction to appear as a package that they evaluate as a whole together, instead of looking at the parent and saying, OK, you're out.
And this child doesn't have a parent.
OK, you're out too.

Speaker 0: 00:02:45

And maybe you can just talk a little bit more about the mechanics of how CPFP actually works.

Speaker 1: 00:02:51

To get into a block, you bid on block space.
Transactions get serialized in a format where inputs are fairly big, outputs are smaller, there's a little bit of a transaction header that encodes how many inputs there are, outputs there are, and lock time and version.
So we already found out that when miners build blocks, they sort transactions by the highest fee rate.
So they first consider the transactions that paid the most Satoshis per byte of serialized data.

Speaker 0: 00:03:24

So what's the mechanics of CPFB?

Speaker 1: 00:03:27

When you try to get a transaction through, Sometimes they have a fee rate that is too low for it to be considered quickly.
And you can reprioritize your transaction by increasing its effective fee rate.
Now you cannot edit a transaction after you've submitted it to the network because the transaction itself is immutable.
But what you can do is you can spend one of the outputs of the transactions with another child transaction that has a very high fee.
And now The child transaction can only be valid by the parent getting included in the block.
So miners will look at transaction packages actually.
They sort the waitlist by their ancestor fee rate of transactions, not just by transactions in the singular.
So when you have a child that is super juicy, it basically pays for the parent to get included as well.
So literally, child pays for parent.

Speaker 0: 00:04:31

Got it.
Every parent's dream to have their children pay for them.

## How miner evaluate fee rates

Speaker 0: 00:04:34

You said that when miners evaluate these fee rates, is that built into Bitcoin Core or are they writing custom software for that?

Speaker 1: 00:04:43

Bitcoin Core has a get block template call, which allows you to exactly do that, just generate a block template.
But I believe that most miners are probably running custom code because, for example, they accept out of band payments to reprioritize transactions, or they run their own wallet service on the side and always prioritize their own transactions or they might have some sort of other solver that optimizes block template building further.
So I think that I haven't looked at this in detail, but I think that at least they're not running default values, because by default, blocks created by Bitcoin Core would leave a little space, I think about six kilobytes.
And blocks are full, if you look at them.

Speaker 0: 00:05:32

So they must have at least tweaked it a little bit.
And when we say miners, we're talking about pools.

Speaker 1: 00:05:38

Yes, right.
So most miners, as in the people running ASICs or whatever, they just join a pool who does the coordination of the work and they basically the pool operator picks the block template that is being worked on and the miner just gets a separate workspace that they iterate over in order to try to find a random block.

## Why is it hard to estimate fee rates?

Speaker 0: 00:06:02

This problem sounds hard.
Why is it hard to estimate phe rates?

Speaker 1: 00:06:06

So block discovery is a random process.
Think of decay of radioactive isotopes.
What we do there is we can give you a half time.
It usually takes around this much of time for half of the atoms to dissipate.
But we can't tell you if we look at a single atom when it's actually going to dissipate.
It might be immediately.
It might be at the half time.
It might take decades.
With blocks it's the same thing.
They're in average coming in at I think about 9.7 minutes, but when the next block is going to be found is up to this random Poisson process.
Actually, it is such that since there is no memory to the process, every draw just has a chance to succeed.
At every point in time, the next block is about 10 minutes away, on average.

Speaker 0: 00:07:02

Yeah, it's really unintuitive to think about that.
Right.
Even if you're 18 minutes into not finding a block, the next block will be found in 10 minutes?
Yes, exactly.

Speaker 1: 00:07:11

You don't know when the next block is going to be found, so you don't know what transactions you will be competing against.
You might be competing against the transactions that are currently in the mempool plus the transactions that get added in the next one minute.
You might be competing against the transactions in the mempool plus 10 minutes or plus 60 minutes because about once a day there's a block that takes 60 minutes.
Really, you have this one shot to pick exactly the right V rate to slide in at the bottom of the block that you want to be in.
Because if you don't slide in at the bottom of the blot, you're overpaying.
And if you underestimate, you're not going to get confirmed in the time that you were aiming to be confirmed.

Speaker 0: 00:07:52

And so how do exchanges usually do this?
Are they overpaying?
Are they just estimating the upper end?
Maybe like who's paying those fees?

## How do exchanges estimate fees?

Speaker 1: 00:08:01

Right.
So there's different scenarios.
Some exchanges have different tiers, like low time preference and high time preference or whatever, and they treat those differently.
But generally, Most exchanges by now batch their withdrawals, which gives them a way to leverage their scale.
So if you're sending to 20 people every minute, making one transaction out of that is a lot cheaper than making 20 separate payments.
It's also much easier to manage your UTXO pool that way.
And then they just tend to very conservatively estimate their fees, just be in the next two blocks and maybe rather overpay slightly Because it's so much less work to deal with all the customer complaints over step transactions than to pay like, sure, we're overpaying by 30% to be in the next block.

Speaker 0: 00:08:56

It's not them that's overpaying though.
Usually that gets passed on to the customer.

Speaker 1: 00:09:01

There's different models.
I think in most, actually, the exchange pays, but they take a flat fee for withdrawal.

Speaker 0: 00:09:10

Oh, really?

Speaker 1: 00:09:10

Yeah.
So like Bitstamp for a very long time, for example, had like, I think a 90 euro cent flat withdrawal fee, but then they'd batch every few minutes only.

## Will the mempool empty again?

Speaker 0: 00:09:23

You said that the mempool hasn't really been empty for almost four months.

Speaker 1: 00:09:27

Yeah, that's correct.

Speaker 0: 00:09:28

Is it ever going to empty again?
As we go to the moon, what happens to the mempool?

Speaker 1: 00:09:34

Yeah, that's a great question.
I think we'll eventually see a mempool empty again, but there should probably be a long tail end to it emptying Because now in this four months, a lot of the exchanges that usually would do consolidations to keep the UTXO pool size manageable, they haven't been able to get any of those through.
So when the fee rates go down now, I think that we'll see more people put in their consolidation transactions at like three to five satoshis per byte.
And that I think we, we might not see an empty men pool for multiple months still, even if the top fee rates get a lot more relaxed now.
Generally, the competition to be in blocks seems to correlate with volatility and especially price rises when the market heats up and people are more excited to trade, there's more transaction volume on the network.
And now we've seen in the past four weeks or so, the price has been going more sideways.
There might have been even a small dips here and there.
And the top fee rates have come down.
The, on the weekends, it's dropped first to seven satoshi per byte, then six, and now last weekend six was cleared completely.
I don't think that getting a one satoshi per byte transaction through will be possible any time soon, but it'll be very possible to wait to the weekend to get a 10 Satoshi per byte transaction frame.

## Miner/pools and the need for a high fee environment

Speaker 0: 00:11:04

Maybe from like a more meta view, don't the miners like this?
Don't they like having high fees?
Because one, it's revenue for them.
But also, as we sort of zoom out, we think about the decreasing block reward over time, don't we have to have a high fee environment in order for this system to work?

Speaker 1: 00:11:23

On the one hand, you have to also consider that the exchange rate 10xed in the last year.
So the same fee rates represent a 10x purchasing value in cost for getting essentially the same service, a transaction into a block.
So while the fee rates are similar, the cost of getting a transaction through has actually increased.
The miners do love it because I think fee rates make about 17% or so of the block reward right now.
So that's a nice little tip, right?
But there is definitely a concern that when we continue to reduce the block subsidy and every four year have a reward schedule, that eventually the system will have to subside just on transaction fees.
And if the transaction fees are too low, it will basically not be economic for miners to provide security to the Bitcoin system.
So there is a good argument for not increasing the block space to a degree where it's always going to be empty.
If you want to do that, you essentially have to also switch to an endless block subsidy.
Otherwise, there is no economic incentive for miners to continue mining if there's not enough fees.
Unless your minimum fee rate at some point becomes so valuable that even at minimum fee rate, any transactions are some sort of sufficient revenue for miners to continue their business.

Speaker 0: 00:13:02

Maybe we can sort of circle back to what happens when transactions are evicted from the mempool and talk about what problems that could introduce, especially for fee bumping and lightning channel closing.
Right.
When a mempool fills up, as we said earlier, the node will start dropping the lowest fee rate transactions.

Speaker 1: 00:13:22

And especially for people or services that use unconfirmed inputs, that can be a problem at times because you cannot spend an input that is unknown to other nodes.
So if all other nodes on a network have dropped a transaction, your follow-up transaction that spends the output from that dropped transaction will not be able to relay on the network.
So you cannot only not spend your funds but you can also not reprioritize the prior transaction.
One thing that this solves basically is RBF because you can just rewrite a replaceable transaction and submit a transaction with a higher fee rate.

## Replace by Fee (RBF)

Speaker 0: 00:14:09

Alright, so we went over CPFP, can we go over RBF?

Speaker 1: 00:14:11

Sure.
So BIP 125 introduces rules by which you are allowed to replace transactions.
You have to explicitly signal that a transaction is replaceable.
And in that case, before a transaction is confirmed, the sender may issue an updated version of the transaction, which can completely change the outputs.
The only restriction is that it has to use one of the same inputs.
Otherwise it wouldn't be a replacement and wouldn't be.
So it has to be a conflicting transaction, essentially.
And additionally, it has to pay enough fees to replace the prior transaction and all the transactions that chained off of them in the mempool so if you had like three transactions you have to pay more fees and the replacement and those three transactions together got it so blessed double spending so what we're saying.
I do not like the term double spending in that context.
So, the problem with that is a successful double spend means that either you actually got two transactions that were in conflict confirmed, which could basically only happen if you have two competing blocks where one block had a prior version and the second block had another and then the second block eventually becomes part of the best chain.
Or when you at least convince somebody that they had been paid but then actually managed to spend the funds somewhere else.
But Here in this case, RBF transactions are explicitly labeling themselves as replaceable.
Basically they're running around with a red lettered sign on front of their chest, do not trust me, right?
And most wallet software just doesn't show you RBF transactions until they are confirmed.
Once confirmed in the blockchain, they're exactly the same and same reliability as any other transaction.
But while queuing, they are explicitly saying, look, I could be replaced.
Do not consider yourself paid.
So calling this a double spend is really just saying that, well, somebody made extremely unreasonable assumptions about the reliability of a transaction that explicitly warned them that it's not reliable.
So I like conflicting transactions more in this context.

Speaker 0: 00:16:35

And maybe why do we need two ways to bump fees?
Why do we need RBF and CPFP?

Speaker 1: 00:16:40

Right.
So they have slightly different trade-offs.
CPFP allows any recipient of a transaction to bump it.
That could be a recipient in the sense that the person that got paid, or the sender if there was a change output on the transaction.
It also doesn't change the Txid because you're just chaining other transactions on it.
And it, it takes more block space, right?
Because you now have to send a second transaction in order to, to increase the effective fee rate of the first.
So more block space, easier to keep track of, and more flexibility as in there's more parties that can interact with it.
RBF on the other hand allows you to completely replace the transaction, which means that it is more flexible, but you potentially have to pay more fees, especially if somebody else chained off of your transaction already.
It changes the TXID and a lot of wallets and services have been tracking payments by the TXID rather than looking at like what address got paid, what the amount was, or whatever, as in like treating Bitcoin addresses as invoices as they should be used.
They built a whole system around TXIDs. So RBF transactions that they change the inputs or outputs, right?
Otherwise they couldn't change the fees.
And that means that they have a new TX ID and it is not trivial to keep proper track of that and to update your UX and UI to make that easily accessible to your users.
Right.
Then also only the sender can bump a transaction doing that because they have to reissue the updated variant of the transaction, given that it is a little more difficult to interact with RBF transactions, a lot of services only see them once they're confirmed, once they're reliable.
So if you're trying to get a service to give you something very quickly, You might want to choose to not do an RBF transaction in the first place so that they can reasonably assume, oh, this has a high enough fee rate and we know the user.
We can trust them that they're sending us these $3 actually and give them access to that song or whatever.

## Mempool eviction and the problems it can cause

Speaker 0: 00:19:04

Got it.
Okay.
So we asked the question, like what problems do does mempool eviction cause for fee bumping?
And also maybe the lightning channel closing use case.

Speaker 1: 00:19:15

We talked a bit generically about how parent transactions being gone stops you from being able to spend those unconfirmed outputs.
But this is especially a problem in the context of Lightning, because when you close a Lightning channel, It's either the collaborative case where you have no problem because you can renegotiate the closing transaction with your partner, but where you really need it, you're trying to unilaterally close because your channel partner hasn't shown up in all.
Then you have to fall back to the transaction that you had negotiated some time in the past when you last updated the commitment transaction.
So let's say that was in a low fee rate time and now the fee rates have exploded And you can't actually even broadcast the commitment transaction to the network because it's too low fee rate.
Now the problem is the party that is closing the Lightning Channel under the LN penalty update mechanism, their funds are actually locked with a CSV.
So they can't do CPFP because the output is only spendable after the transaction is confirmed for multiple blocks.
So you can't chain a transaction to a output that is not spendable while it's still unconfirmed.
Especially for lightning, this introduces the volatility in the block space market, introduces a headache because you can literally come into a situation where you can't close your Lightning channel due to the fee rate.

## Anchor Outputs

Speaker 1: 00:20:49

So one approach I've heard about is to, to introduce anchor outputs, which are depending on the proposal, either spendable by either side or spendable under certain conditions, but they're immediately spendable so they can be CPFPed.

## Package Relay

Speaker 1: 00:21:05

Or another idea is to have package relay, right?
Because if the channel closing transaction has a low fee rate And you can then relay it together with a second transaction.
That'll work, except if you're unilaterally closing, because the CSV issue still pertains to that.
But either way, if you get package relay, you would be able to do away with the fee estimate and commitment transactions altogether because we talked about how fee estimation is hard for regular transactions.
Fee estimation for commitment transactions is even much harder because you have no clue when you will want to use the transaction.

Speaker 0: 00:21:46

Yeah, that dependency seems very scary.

Speaker 1: 00:21:49

Right, you have absolutely no clue what the fee rates will be like when you actually try to use it.
So having package relay in combination with anchor outputs would allow you to always have a zero fee on the commitment transaction.
And then basically always bring your own fee when you broadcast it in the CPFP child transaction.

Speaker 0: 00:22:10

Got it.
So we've sort of talked about some specifics, but maybe we can zoom out.

## Ways to use the blockspace more efficiently

Speaker 0: 00:22:15

And what are some ways that we could be using our block space more efficiently?
What are some things that make us optimistic about the future?

Speaker 1: 00:22:22

We still have only about, I think 40% or so SegWit inputs.
Now about 55% of all transactions use SegWit inputs, but the majority of inputs are still non-SegWit.
Once more people start using SegWit or even Taproot, once Taproot comes out, the input sizes will be smaller.
So naturally there will be more space for more transactions.
So recently a major service, wallet service provider, announced on 1st of April nonetheless, that they would be switching to native SegWit addresses.
And they had been a long holdout.
So blockchain.com has probably around 33 percent of all transaction creations among their user base.

Speaker 0: 00:23:06

Yeah, I mean that dependency is, we're shaking our heads simultaneously.
It's not great.

Speaker 1: 00:23:14

SegWit activated on 24th of August in 2017, right?
That's three and a half years ago.
And until recently, I think they, they weren't even able to send to native segwit addresses and now they announced that not only that they'll actually default to native SegWit addresses altogether.
I think they claimed this month, but I'm hoping that they'll come through with that, because we have a huge backlog of all these outputs that they created over the years.
It has been one of the most popular Bitcoin wallets for almost a decade.
And it will take forever for all of these non-SegWit outputs to eventually get spent.
But the observation is that most inputs are consuming just very young outputs.
So funds that got moved are much more likely to move again soon.
So seeing that blockchain.com will hopefully switch to native SegWit outputs soon, I would assume that even while the UTXO set will have a lot of non-SegWit outputs living there for a very long time, the transactions that get built will much quicker become SegWit transactions to a high degree.
If 33% of all transactions, let's say 80% of them become SegWit inputs and literally more than half their input sides, that would be, I want to say, like 15% of the current block space demand going away overnight.
Yeah.
Yeah, That would be nice.
I should do the calculation more thoroughly, but...
Are there other holdouts?
Bitmex recently switched to native SegWit, I think, for deposits.
There's still quite a few services that use wrapped SegWit rather than native SegWit, which already gets most of the efficiency, but clearly not all.
It was actually expecting that the high fee rates might get more people moving.
I think that the Taproot rollout might get a huge block space efficiency gain, because Taproot introduces a bunch of new features that are only available through Taproot, and Taproot outputs and inputs are about the size of pay to witness public key hash in total.
So smaller than a lot of the multi-sig constructions these days even in native SegWit and definitely smaller than everything non-SegWit.
So any any wallets that switch to taproot will bring down the block space use a lot quickly.

Speaker 0: 00:25:53

Yeah, the multi-SIG savings are pretty significant.
And hopefully it'll bring in a new era of multi-SIG being more standard.
I think that's a more exciting thing.

Speaker 1: 00:26:02

It'll take quite some time because to do the public key aggregation that will bring the biggest efficiency gain, people will actually have to implement music or another aggregation algorithm.
And until that gets into regular wallets will be a while.
I think maybe first it gets into libraries and especially for services with multi-sig wallets, there would be a huge efficiency gain there and they should have great incentives to roll it out very quickly.
Great.

Speaker 0: 00:26:40

Thanks for listening to another episode of Chain Code Decoded and we're going to keep it rolling.
We'll have another one next week.

Speaker 1: 00:26:46

Yeah, let's talk about maybe how the blockchain works.

Speaker 0: 00:26:49

Going back to basics.
See you next time.

Speaker 1: 00:26:51

Top 10 Hits.
Top 10 Hits.
