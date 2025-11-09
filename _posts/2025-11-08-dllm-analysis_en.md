---
layout: post
title: "A Deep Dive into Diffusion LLMs 1.0"
date: 2025-11-08
description: "A comprehensive overview of my understanding of DLLMs."
tags: [life, entrepreneur, en]
lang: en
---

### What are the advantages of DLLMs? What are their fundamental problems? And do they even work?

When discussing the motivation for DLLMs, several points are usually mentioned:

* First, **speed**. It's a parallel generation process. Generating 32 tokens at once is definitely faster than generating one token at a time.
* Second, **quality**. The quality comes from better training data augmentation and higher utilization. Theoretically, masking provides a more complex training paradigm for the data. Originally, the model only needed to learn to predict the next token based on the previous ones. Now, it must also learn to predict previous tokens based on subsequent ones. This inherently constitutes more training tasks and a fuller utilization of the existing, abundant data. There's also a possibility that using generative data augmentation produces data that aligns in meaning but differs in pattern from the training data, thereby enhancing the model.
* Third, **flexibility**. Different tasks have different levels of difficulty. Many tasks are simple, but due to token throughput limitations, even a smart model can only speak one sentence at a time. The flexible nature of DLLMs offers the potential to output everything at once.
* Fourth, **representation power**. Due to their bidirectional attention mechanism and unique training mode, DLLMs have a stronger individual hidden space than decoder-only architectures. They naturally achieve LLM2Vec-level capabilities, which means that representation learning also has the potential to scale.

### What are the problems with DLLMs?

* First, **DLLMs aren't actually fast**. Traditional Autoregressive (AR) models are fast because they can use KV Cache. The parallel nature of DLLMs makes them inherently anti-KV Cache. Even if I generate 32 tokens at once, I need multiple steps. This means the KV Cache from the previous iteration is already changed in the next step and cannot be reused. The result from LLaDa is that generating 32 tokens requires 32 steps, and each step has zero KV Cache reuse.
    Fast DLLM proposed a solution: approximate KV Cache. The idea is that although the cache changed, it didn't change much, so the previous step's KV Cache can still be used. Another idea is similar to discrete diffusion forcing: I can first use block diffusion to generate tokens. For all completed tokens, I allow them to have a KV cache. Next, I can try to build a hierarchical KV cache structure. What's already generated can use the cache, and new generation can happen in two parallel blocks, with some approximate KV cache between the blocks. This is the idea behind Fast DLLM v2.
    So, how slow is the original LLaDa? Let's put it this way: discrete diffusion forcing claims their model is 50x faster than the original LLaDa, but in reality, it's only about 2x faster than AR. Fast DLLM claims a 27x speedup, which in other words, is still slower than AR.
    Then why do the Gemini Diffusion models look so fast? The reason is that a standard AR 7-30B model might have an online inference speed of 300-500 tokens/s in OpenRouter. Theoretically, if I train a model of the same size and apply a series of fancy optimizations, I should be able to achieve a generation speed of 1000-2000 tokens/s. This is the speed currently being demonstrated by diffusion models.

* Second, **DLLMs have a higher peak memory (VRAM) usage**. Under the same conditions, DLLMs hit a peak in memory usage because the attention within a block is calculated in parallel. This results in a peak activation size that is a multiple of the block size. This means the maximum text length that can be run on a single card is very limited. Current benchmarks for the Fast DLLM series are mostly tested at a 512 context length, not in scenarios like 10K or 100K.

* Third, **we might lose even more in production scenarios**. When we serve multiple users on the same GPU, we find that as the batch size increases, the generation speed of DLLMs gradually falls behind AR models. This is because as the batch size increases, the inherent parallelism within the AR batch is better utilized.

* Fourth, **another acceleration technique, speculative sampling**, can theoretically bring a 2-3x speedup to AR models. This gain is similar to the gains from DLLMs.

### Current Research Directions for DLLMs:

1.  Information from previous predictions is lost after re-masking. This previously predicted information is an intermediate state; it's incorrect but potentially useful. We want to be able to use this intermediate state, turning it into a continuous state.
2.  Existing (already generated) text cannot be 100% trusted. Although traditional LLaDa can modify already generated text, it wasn't actually trained to do so. It was only trained to modify masks.
3.  Due to the inherent limitations of KV cache, DLLMs will ultimately take the form of block diffusion. So, how can we build an optimal generation scheduling mechanism? When should it decide to be AR, and when to use a block? How to determine the block size? And how to consider KV Cache optimization when determining this optimal generation order?
4.  How can DLLMs perform insertion and deletion operations, and how can they account for KV Cache reuse while doing so?

### Current Analytical Questions for DLLMs:

1.  Does fine-tuning from an AR model to a DLLM have scaling properties? Can I get a comparable DLLM at the 100B parameter level? How many tokens would it take, and what characteristics must these tokens have?
2.  Is there any difference between the AR component of a DLLM and a standard AR model? Does it have higher compressibility? If using KV Cache, can a larger amount of the cache be pruned?
3.  If a DLLM can achieve an intermediate state of latent embedding, does it mean that problems like "large concept models" and "latent reasoning" can work under this architecture?
4.  Are the natural acceleration properties of DLLMs the final piece of the puzzle for Agents? The characteristics of agents are long context, high latency requirements, and high intelligence requirements. If DLLMs can do compression and acceleration well, could this be the solution?
5.  Why does latent reasoning not work, but converting images into continuous-space tokens and feeding them into the network *does* work?