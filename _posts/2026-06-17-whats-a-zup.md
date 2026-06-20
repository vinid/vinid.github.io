---
layout: post
title: "What's a ZUP? On research, ego, and what actually gets cited"
date: 2026-06-17
category: Research
---

# Throwback

Back in 2023 [Patrick](https://patrickjohncyh.github.io/) and I started working on an idea: building an arena for LLMs to explore their capabilities in negotiation tasks. This was a fun, little project that was more explorative than grounded in an actual goal: if you read the paper, you see that we tried a few different experiments to understand models' behavior. While there is some overreaching story about reasoning capabilities, this paper was more about testing different negotiation scenarios and see what happened.

Our results were packaged into a paper, [NegotiationArena](https://arxiv.org/abs/2402.05863), that found its way at ICML2024. For what it's worth, NegotiationArena was my first ICML paper. 

The paper explores three different negotiation games done with LLM agents and describes the effects that different persona/prompting strategies have on the outcomes of the games. One interesting finding was that initializing one LLM with aggressive personalities would make them more likely to end the negotiation with the winning hand.

For this short blog post, I want to mention that one of the scenarios implements a buyer agent and a seller agent, negotiating the price of an object in a fictional currency ZUPs. Here is the first commit where we introduced ZUPs in the codebase.

<figure>
  <img src="/assets/img/zups/commit.png" alt="ZUPs commit" style="max-width: 100%; margin: 2rem 0;">
</figure>

# Scientific Angst

As for all my papers, I of course wanted NegotiationArena to be somewhat of interest. Indeed, as many scientists do, I aspire for my research to be of some minor usefulness. I am confident that my research is just a minuscule part of this joint scientific endeavor, but I like to think I am part of this great adventure, partially driven by general interest and partially driven by ego.

To be honest, this is a blessing and a curse at the same time. I am very excited and ambitious and I want the work I do to be relevant. However, that means that when things don't work out I feel the weight of the failure on my shoulders. Science is always more losses than wins, successes are often derived from days and months of unsuccessful experiments.[^1]

I always look above me at people that are more successful than me (in fame, citations, and so on and so forth) and often feel I should work harder, publish more, publish better; at the same time I should know more and study more, learn all the things they know and I don't. Comparison is the thief of joy, they say, but it's hard to have ambitions that don't make you look at what others are achieving. For better or worse, science is a strongly competitive game, you are always playing with other people.[^2]

The fun thing is that, working hard and publishing does not predict what people will cite you for.  I did not expect much out of NegotiationArena, it's a set of useful and interesting analysis, but there is no new modeling contribution and the benchmarking and evaluation aspect is not systematized enough to make for a full testing suite.

# ZUPs in the Wild

So, what gets passed on?

In the NegotiationArena paper we introduced this concept of a fictional currency `ZUP` for agents to exchange to avoid any bias coming from the word `dollar`. The idea came from the amazing Xu and Tenenbaum's [Word Learning as Bayesian Inference](https://pubmed.ncbi.nlm.nih.gov/17500627/) paper, which explains the usage of new words (e.g., fep) to test learning. Indeed, one of the first proposals for the name of this currency was actually "fep".

I remember Patrick and I discussed what the name of this currency should have been at length. We had a few different proposals: some were pretty weird names and initial experiments had been run with an uninspired `M` (for money) as the currency. Eventually, Patrick proposed `ZUP` and it was introduced in a commit around November 19, 2023.[^3]

`ZUP` was absolutely introduced for fun, there was no big meaning behind it if not being a means to an end in an experiment around agents negotiating: ZUPs sounds funny and it is still a running internal joke, appearing semi-frequently in our chats. See the following figure:

<figure>
  <img src="/assets/img/zups/zups.png" alt="Chats about ZUPs" style="max-width: 100%; margin: 2rem 0;">
</figure>

What I find even more funny is that a few other researchers started using ZUP in their experiments when extending or reproducing our work. Quite a few papers now (just for reference: [1](https://arxiv.org/pdf/2405.16588), [2](https://arxiv.org/pdf/2512.09254), [3](https://arxiv.org/pdf/2511.17990), [4](https://openreview.net/pdf?id=j7RkeNqSDo), [5](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=5837643), but there is more!) mention in their experiments the use of ZUPs.

It's surprising to see that the name was picked up by folks and people are building on it; that was never the intention or the aspiration. You never know what people will cite you for, but fun things somehow go a long way. NegotiationArena's most exciting contribution, might be a new word!


[^1]: For what it's worth you absolutely don't get used to it.
[^2]: I deliberately chose `with` instead of `against` here.
[^3]: For what I can remember ZUP is the compressed version of `what's up` which is one of the most common phrases in the exchanges between me and Patrick.