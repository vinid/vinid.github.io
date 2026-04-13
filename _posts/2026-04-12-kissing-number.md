---
layout: post
title: "The night we (almost) found a new bound for the kissing number problem"
date: 2026-04-12
category: Research
---

In March we released a platform to let agents solve open mathematical problems called [EinsteinArena](https://einsteinarena.com/). You can think of it as a combination of the now very popular Moltbook and Kaggle: EinsteinArena offers a message board and several open math problems to optimize. Everything happens via API, so AI agents can interact and communicate on a message board and work together to solve problems.

EinsteinArena is an experiment to check if agents can collaborate asynchronously on hard math problems. We released a [blog post](https://www.together.ai/blog/einsteinarena) introducing the platform. Here I want to describe what happened two nights before the kissing number breakthrough — or almost-breakthrough. While I use 'we', the discoveries belong to the agents and the community; my role was building the stage they performed on.

# Prelude

The Kissing Number is a very famous mathematical problem. How many spheres in $$\mathbb{R}^n$$ can simultaneously touch a central sphere?

Some intuition is given by the following figure, where we can see the kissing number for $$\mathbb{R}^1$$ and $$\mathbb{R}^2$$.
<figure>
  <img src="/assets/img/kissing/kissingnumbers.svg" alt="Kissing Numbers" style="max-width: 100%; margin: 2rem 0;">
</figure>

<p><strong>Definition (Kissing Number).</strong> 
The kissing number \(k(n)\) in \(n\)-dimensional Euclidean space \(\mathbb{R}^n\) is:
$$
  k(n) = \max \left\{ |S| : S \subset \mathbb{R}^n,\ 
  \forall\, \mathbf{x} \in S,\ \|\mathbf{x}\| = 2,\ 
  \forall\, \mathbf{x} \neq \mathbf{y} \in S,\ 
  \|\mathbf{x} - \mathbf{y}\| \geq 2 \right\}
$$
</p>

The problem is very hard in higher dimensions, and more importantly for most dimensions we only know lower and upper bounds (we know we can fit 40 spheres in 5 dimenions, but there is no proof that that is the final number); the process has been incremental over the course of the centuries (Isaac Newton himself worked on this problem).

In early 2025, AlphaEvolve found a groundbreaking new lower bound on dimension 11, finding a set of 593 spheres.

How do you know if your lower bound is certifiable? You can evaluate how good a solution by looking at how much the spheres overlap. More formally, given a set of non-zero vectors in $$\mathbb{R}^{11}$$, each vector $$\mathbf{x}_i$$ defines a sphere center at $$\mathbf{c}_i = 2\mathbf{x}_i/\|\mathbf{x}_i\|$$. The overlap loss is

$$\text{loss} = \sum_{i < j} \max(0,\ 2 - \|\mathbf{c}_i - \mathbf{c}_j\|)$$

A score of exactly 0 means no two spheres overlap — a valid kissing configuration and a certifiable lower bound.[^1]

## April 8th, 10PM

I came back home on April 8th after dinner with friends to find a suspicious set of submissions for the kissing number problem on EinsteinArena. I was seeing a submission with a full set of zeros (which would have meant a new lower bound of 594, beating the previous record). There were three possible options: 

1. A new lower bound had been found
2. The platform was not showing enough decimal digits
3. There was a bug in the code

I am not new to writing bugs in my code but this was really suspicious, these verifiers are usually well-defined. I was pretty confident this was just a rendering issue and more digits would clarify. Even in that scenario, if the solution were at 1e-7 that would have been pretty huge.

<figure>
  <img src="/assets/img/kissing/first.png" alt="Kissing Number Leaderboard" style="max-width: 100%; margin: 2rem 0;">
</figure>

As soon as I saw that, I sent a friendly and totally not panicked message to my colleague Yongchan.

<figure>
  <img src="/assets/img/kissing/sir.png" alt="" style="max-width: 100%; margin: 2rem 0;">
</figure>

I double checked the solution and it was not yet a valid bound, but it was still a huge leap forward: 3.8e-10. I didn't check this particular problem in the last few days but my memory was telling me that we had been stuck around 0.15 for a while, we really did not expect any kind of progress on this particular problem. 

Anyway, the platform was showing all zeros, so first PR of the night, let's fix the visualization, then we can look at the actual submissions. Fix is simple but it was hardcoded in and it is still hardcoded there, basically transforming numbers with `.toExponential()`; I felt like I did not have time to do things the right way and check testing. In retrospect, I had all the time in the world, but the excitement took the best of me. 

## The Next Few Hours

Next step was studying the solution. What were these agents submitting? 

While we were studying the solutions to better understand what was happening, I did some tracking of the events: at 4pm an agent, `alpha_omega_agents`, submitted a set of spheres that got 0.0119. This solution was the structural key to the final solution. After that `CHRONOS` took the lead.[^2] The next few hours were a sequence of improvements from the two above and `JSAgent`. Agents kept taking the lead from each other and they eventually got to 1e-15. We were getting so close. 

While I was pestering the poor Yongchan with tons of messages, new solutions started to flow in from other agents. I cannot describe how exciting this was! Agents were surpassing each other on the leaderboard in real time at the same time as  we were checking things! It gave me the same feeling of watching those exciting football matches where the stakes are high and all can change in a few minutes.[^3] 

Look at how excited I was:
<figure>
  <img src="/assets/img/kissing/pending_submission.png" alt="" style="max-width: 100%; margin: 2rem 0;">
</figure>

We thus decided to update the rate limits, unfortunately, this required another PR. It would have been better to have this controlled by some DB flag but I never had the time to implement them. 

After that, we were just watching the progress live. I was looking at our logs on Vercel, and suddenly I saw that the submission was:
<figure>
  <img src="/assets/img/kissing/vercel_logs.png" alt="" style="max-width: 100%; margin: 2rem 0;">
</figure>

One of these new submitted solutions, was 0. Not 1e-18. Exactly 0. For a moment, we thought it was over: one of the agents had just found the kissing number. They did it! JSAgent had just won! This was a new lower bound!

We both downloaded the solution and found that it gave 0s, but when we used the stricter verifier it was not valid. We had a bug.

The verifier we were using was not able to support the precision required. We were at numpy error level and the accumulation of floating point errors was being problematic.[^5] 

Agents were submitting, we had to fix the verifier live without breaking the platform[^4]. Yongchan prepared a new verifier using the decimal library in python and 80 digits of precision. Third PR of the night, we improved the verifier. We tested it, it did not work. Is there any difference between local and prod? is this a typescript thing?

Absolutely not, I forgot to update the DB so that the new verifier is propagated to all requests. Lessons learned: never have multi-step processes to modify one part of the pipeline.[^6]

Not only we had to do that, but the submissions of the last 2 hours all had been evaluated with the wrong verifier. Thus, we also had to update the database for all those agents that submitted - which is always a pretty scary `UPDATE` operation to send on prod even if this is a research tool.[^7]

The performance of these submission had regressed to 1e-13, far from the 1e-15. We woke up our agents and tried to push it a little bit but we could not make it. We decided we would try to get it done the next day.

I could not sleep, I had to watch a few anime episodes to get relaxed enough to sleep, my adrenaline was too high.

## The Next Days

The morning of the day after we found more problems and more fires to put out. More 1 liner PRs also YOLO'ed in because my typescript knowledge is not enough to understand that.  Even with increased precision, our agents could not find a way to snap those coordinate in place. An agent got a 1e-50 submission; it was not there yet, we needed exact zero.

Two days later, we found a new lower bound 594. 

More surprisingly, this lower bound evenutally unlocked a new lower bound of 604 with a solution living in $$\mathbb{Z}[\sqrt{2}]$$ — we never expected the answer to hide in an algebraic structure like that rather than something purely rational or integer-valued. However, that's a story for another time.

This was such a fun turn of events, so much emotion and so much exploring, debugging and fixing. The joy of discovering is all in the sweat you put in. We were not expecting to find a new bound, but we also were not expecting to have to improve things at 11.58PM because agents were making incredible progress, we were just there to watch (and fix bugs).

[^1]: There are a few important considerations here with respect to 0 and the precision of computing that value using floating point arithmetic. See [Georgiev et al.](https://arxiv.org/pdf/2511.02864).
[^2]: This is a good moment to explain that agents can submit at any time but my evaluator runs every 15 minutes.
[^3]: Roma 3-0 Barcelona comes to mind from the 2017/2018 Champions League.
[^4]: We didn't want to stop the agents from working, a failure might mean the agent leaves the platform.
[^5]: float64 gives ~16 significant digits. With thousands of pairwise distance computations, the minimal overlaps live at 1e-15 or smaller where floating point dominates. Switching to Python's `decimal` with 80-digit precision makes those values unambiguous.
[^6]: My code that handles API leaderboard and submission is looking at me asking to be refactored.
[^7]: Also folks we were so close at the acutal new lower bound!!!
