---
layout: post
title: "Learning to Discover at Test Time"
date: 2026-09-27
category: Research
description: "How TTT-Discover trains a model at test time to find a TriMul GPU kernel faster than every human entry."
---

This blog post is about our method TTT-Discover from our ICML paper "Learning to Discover at Test Time".[^1] Instead of presenting the paper in the usual order I am going to present it through an incremental story. Please note that what I am going to show is in some cases a bit handwavy as I will be conflating numbers from different experiments for the sake of giving a coherent story.[^2]

# Scientific Discovery

How can we make new discoveries with AI? How can we find something no one has ever found before? The big issue is how large the space of possibilities to explore is (think all algorithms you could write to solve a problem). We need to efficiently explore the search space.

For example, assume you have a piece of python code that solves a problem. Let's say we want to make it faster or maybe more accurate. How would you go about that? You would probably need to think about different techniques to use, optimizations and so on and so forth.

There has been a lot of successful work in automating this direction, [autoresearch](https://github.com/karpathy/autoresearch) is a prime example of this.

<video src="/assets/img/discovery/01-edit-loop.mp4" autoplay loop muted playsinline style="max-width: 100%; margin: 2rem 0;"></video>

Many modern systems for automated discovery work like this: you have a problem, you ask the LLM to solve the problem. It finds an initial solution. You use that solution again, ask the LLM to edit again, and keep doing this loop over and over. The LLM provides edits on top of that piece of code, and you assume the edits are going to be better.[^3]

So if these systems work and discover, why would we want to do anything else? Well, for one, they rely on frontier closed models and second, there is a surprising amount of things you can do once you enable the model to learn (in the weight space) at test time.[^4]

# Story of one Kernel: TriMul

Let's start with a little goal. Let's say we want to find a new version of the TriMul kernel, an operation in AlphaFold.[^5] If you don't know what a kernel is: it's some code that tells hardware how to compute the result of a math operation. You often want to build faster kernels so the output comes out sooner.

TriMul takes an $$N \times N$$ grid of pair features $$z_{ij}$$ and updates each pair $$(i, j)$$ by summing over every third point $$k$$ of the triangle $$(i, j, k)$$:

$$
\mathrm{TriMul}(z)_{ij} = \sum_{k} a_{ik} \odot b_{jk}
$$

where $$a$$ and $$b$$ are gated linear projections of the normalized input (the real op also wraps the result in another LayerNorm, projection and gate). 

Our goal for today would be to see if we can make this faster. As a reference we can use the GPUMode Leaderboard.[^6] These are the human scores we need to beat (lower is faster):

| Rank | Runtime on H100 (µs) |
|:-----|---------------------:|
| 1 (best human) | 1371 |
| 2 | 2368 |
| 3 | 2546 |
| 4 | 3655 |
| 5 | 4233 |

The easiest thing we can do is to just sample from a language model Best-of-N style. Language models have seen algorithms — tons of kernels, they've read GitHub, they've read textbooks, they have access to a lot of this information during training. It's reasonable to assume they can just combine this into a new kernel. 

We ask "create a TriMul implementation", we generate many samples and look at those. Some will be good, some will be bad, we will just pick the best. We know from the literature that sampling many times is useful — test-time scaling. How do we pick the best? Through a verifier.

For kernels, we take a bunch of matrices, throw them on the hardware, check for correctness, estimate how fast this kernel is.[^7] 

So we are combining exploration (sampling from the LLM) with a guide (the verifier).

<video src="/assets/img/discovery/02a-best-of-n.mp4" autoplay loop muted playsinline style="max-width: 100%; margin: 2rem 0;"></video>

Let's take gpt-oss-120B, a fully open model, so we can actually train it later. How fast is a TriMul kernel sampled from it? 

<img src="/assets/img/discovery/02-gpt-oss-distribution.png" alt="Distribution of 25,600 TriMul kernels sampled from gpt-oss" style="max-width: 100%; margin: 2rem 0;">

The distribution on the left shows 25,600 samples, with the best one marked by the orange line. Even that best sample is very far from the best human result. We update our scoreboard, but we're still far away (best of 25,600: 5352µs).

## Reuse States

Every sample starts from zero, so we waste a lot of compute resampling from the same system, and the distribution stays concentrated. An easy solution is to extend the horizon by reusing the best kernel so far: take the best one, ask gpt-oss to improve it, sample again, pick the best, repeat.

<video src="/assets/img/discovery/03-state-reuse.mp4" autoplay loop muted playsinline style="max-width: 100%; margin: 2rem 0;"></video>

Anyone who knows Monte Carlo tree search knows what comes next: this has an over-exploitation problem. 

To balance it, we use an MCTS-inspired approach based on PUCT from AlphaZero, with some minimal changes. Instead of only exploring the best branch, we also select samples that are nearly as good but barely explored, to balance exploration and exploitation.[^8]

<video src="/assets/img/discovery/04-puct.mp4" autoplay loop muted playsinline style="max-width: 100%; margin: 2rem 0;"></video>

The kernel indeed gets faster: 2061µs... but it's not yet close to the best human. There's still a pretty big distance.

<img src="/assets/img/discovery/03b-reuse-result.png" alt="Best kernel found by search with PUCT reuse (2061µs) against the human leaderboard" style="max-width: 100%; margin: 2rem 0;">

# Training 

Even if search brings us far, the model keeps making the same mistakes over and over. It never gets a chance to learn anything; we just keep trying to one-shot solutions.[^9]

And if you think about it, no one really solves a problem on the first try. Think back to being a student, given textbook problems. Textbook problems are designed so you can't just copy what you studied. You need to try things, adapt, see what failed, recover, and eventually get to the right solution. 

You want to learn at test time, during the test itself. 

For each sample we know how fast the kernel is, and which ones don't compile (zero reward). So let's do reinforcement learning via GRPO and combine it with PUCT. This should allow the model to get enough feedback to correct some unwanted or unproductive behaviour.[^10]

<img src="/assets/img/discovery/03c-rl-result.png" alt="Best kernel found by standard RL plus Reuse (1985µs) compared to search and the human leaderboard" style="max-width: 100%; margin: 2rem 0;">

We get to around 1985µs. It's good, but not yet good enough.

# RL for Discovery

The question is, why doesn't it work? Here we have a bit of a conceptual shift. 

Standard reinforcement learning maximizes the expected reward — it raises the typical rollout, makes the policy more likely to generate good-on-average results. But that's not the goal of discovery. Discovery needs one good rollout. You don't need a policy that writes good kernels on average — you need a policy that eventually produces one good rollout, which becomes the new discovery. Different focus.

<video src="/assets/img/discovery/05a-standard-vs-discovery-rl.mp4" autoplay loop muted playsinline style="max-width: 100%; margin: 2rem 0;"></video>

Let me say this again, because the change is kind of important: the artifact we get out of the policy is the goal of the discovery process. In standard RL, you train and deploy the policy because you want a good policy. In discovery, all we want is the best artifact — we're going to train a model, use it to generate the kernel we want, then throw the model away, because we don't need it anymore.


# The Entropic Objective

We go from the standard RL loss to an entropic objective, controlled by a beta parameter. In practice: beta controls how much weight goes to the highest-reward actions. Small beta: behavior close to standard, even minor differences weighted about the same. Spike up beta: the best actions get more weight.

$$
J_\beta(\theta) = \mathbb{E}_{s \sim \mathcal{H}}\Big[\log \mathbb{E}_{a \sim \pi_\theta(\cdot \mid s)}\big[e^{\beta R(s, a)}\big]\Big]
$$

where $$\mathcal{H}$$ is the history of generated solutions, and the starting state $$s$$ is picked from it with PUCT. Standard RL maximizes $$\mathbb{E}[R]$$. Here the reward is exponentiated before averaging, so as $$\beta \to \infty$$ the objective tends to the max reward instead of the mean. The video below shows how the weight is going to pile up on the action that got the best reward as we increase beta.

<video src="/assets/img/discovery/05-entropic-objective.mp4" autoplay loop muted playsinline style="max-width: 100%; margin: 2rem 0;"></video>

The entropic objective comes from risk-sensitive RL.[^11] Choosing beta well matters a lot for discovery: too high early and training becomes unstable, too low later and there's almost no learning signal. In the paper we adapt it automatically. See there for details.

Once you run this you get 1161µs, about 15% faster than the best human. This is the only method that found a kernel faster than every human entry on the leaderboard at time of submission. The following figure shows the distribution of TTT-Discover samples at different training steps, with a label for what the model has learned to do to improve the solution (the gist is that it learns to do better and deeper fusion).

<img src="/assets/img/discovery/06-distribution-shift.png" alt="Distribution of kernel runtimes shifting across training steps" style="max-width: 100%; margin: 2rem 0;">

# Scientific Discoveries in Different Fields

We have applied TTT-Discover on a few different tasks, new math upper bounds (beating AlphaEvolve), new algorithmic results (beating ShinkaEvolve), new biology results (beating MAGIC, the best method on the OpenProblems denoising benchmark). For the Erdős minimum overlap problem, funnily enough now the result is on Wikipedia![^12] Our results are validated by external experts that discuss successes and caveats.

I will not spend much time on the results, the method is general enough to work well in different domains. However, in the paper we report all problems we tried and for some we get very close to either AI-best or human-best (which is still pretty cool considering that this is an automated process), but we don't beat them. See details in the paper.

# Discovering by Learning at Test Time

The main message: the shift in focus that learning to discover brings to the table. Normal reinforcement learning focuses on the policy, a policy that's good on average. That is not what discovery needs. Discovery needs one single rollout that gives you the best possible artifact: the new discovery.

In practice, this is very simple to use: [code is online](https://github.com/test-time-training/discover). Define your reward in Python and your environment, run the discoverer. Applies to any algorithm you want (if you have a reasonably good verifier).


[^1]: [Learning to Discover at Test Time](https://openreview.net/pdf?id=96zNuQrH9Y). ICML 2026.
[^2]: The impact should be minimal, but it's important to refer to the paper for the details.
[^3]: This is more or less how very cool systems like AlphaEvolve work. Of course they use many more heuristics (e.g., reuse newly found good solutions to extend the horizon of your system).
[^4]: There is also related work that does training. Read the paper; there is an extensive discussion of related work!
[^5]: You can replace TriMul with any algorithm and any property you might want to improve, but TriMul is the only case where I have the full numbers of the entire process so we will stick with that.
[^6]: These numbers came from the time of submission.
[^7]: Ok yes, I am simplifying, there is lots that goes into verifying behavioral correctness of a kernel (e.g., not cheating through caching), but that's not important right now.
[^8]: One key difference from AlphaZero: a state is scored by the best reward among its children, not the average, because in discovery we only care about the best outcome (the same idea behind the entropic objective below).
[^9]: We found gpt-oss on TriMul only generates correct code 46% of the time (this doesn't even count slow kernels). Most samples from gpt-oss are wasted. Moreover, the model never learns what doesn't work: if it's used to writing some slow code it thinks is useful, it never learns not to do that.
[^10]: Skipping this but I am going to tell you that RL alone does not bring you anywhere, it's again around 5000µs.
[^11]: [Risk-Sensitive Reinforcement Learning for Alleviating Exploration Dilemmas in Large Language Models](https://openreview.net/forum?id=7kC8ORye4l). ICLR 2026.
[^12]: My paper being on Wikipedia was not on my 2026 bingo card etc etc