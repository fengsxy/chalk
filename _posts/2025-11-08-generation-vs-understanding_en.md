---
layout: post
title: "Why Are Generation and Understanding So Difficult to Unify?"
date: 2025-11-08
description: "Generation, Understanding, and Representation Learning"
tags: [life, entrepreneur, en]
lang: en
---

### Why are diffusion models bad at representation? Why does their latent space look so much like pixel space?

Because the amount of information in a generated output is far greater than the amount of information required for understanding.

Looked at another way, "understanding" only requires you to provide general contour information. When we talk about "understanding," we mean that after linear probing, we can get a good classification result. For this kind of classification, I don't need high-frequency pixel information.

But reconstruction *does* need it. So why can MAE (Masked Autoencoders) produce a good representation?
MAE isn't doing a generation task, but a *restoration* task. Therefore, it actually has enough capacity to store all the fine-grained details. Even with a mask ratio of 80-95%, it can still recover well because the 5%-15% of information it *does* have is complete, containing both high and low-frequency information.

So, what I need is a representation that, first, has compressed, abstract information to help with probing, but at the same time, also stores detail information to allow me to perform recovery (reconstruction).
A simple joint training doesn't seem to make the model's channels split (e.g., the first 512 for understanding, the last 512 for generation). Instead, they are mutually influencing each other.
This gives the impression that the method for storing representations for "understanding" is inconsistent with the method for storing representations for "generation."

### The default task of diffusion is denoising. The essence of denoising is to generate from noise, which requires generating both contours and fine details well.

What does a VAE do? VAE compresses this space of contours and details *together*. This space still retains both contour and detail information, but the drawback of VAE is that its semantic capability is very poor.

Theoretically, I should have a decoupled representation, perhaps split into three parts [256] [256] [256]: the first for pure understanding, the second for the overlap between generation and understanding, and the third for the detail information for generation. In traditional diffusion, we put the "understanding" part into the "condition," which is also a form of decoupling.

This is also why representation tasks in diffusion-like models are actually done at the condition level. The papers SODA (Bottleneck Diffusion Models for Representation Learning) and RCG (Return of Unconditional Generation: A Self-supervised Representation Generation Method) are two sides of the same coin. They both discuss figuring out the representation of the *condition*.
* One paper states that I can use the condition as a medium to *incidentally* acquire representation capabilities during the diffusion training process.
* The other states that I *already* have a good representer (encoder), I can use it to generate countless embeddings, then use diffusion to *simulate* these embeddings, and finally use these approximate embeddings to guide generation.

### In my view, all subsequent work like RAE, RCG, and LD without VAE is essentially trying to decouple the representation space.

Regarding Repa, I think it's forcing the network to learn abstract semantics in the early stages of denoising, while leaving space in the later stages of the network to learn how to predict high-frequency detail information. There is an optimal solution space for denoising. One such solution is the one I mentioned, where the semantic space and the detail-generation space are well-decoupled. When they are well-decoupled, the model can perform well on understanding tasks.

LD without VAE uses residuals to add DINO features and fine-tuned features (which can be understood as decoupling the detail features). This is easy for the diffusion model to understand, and it's "just okay" for probing.
What RAE does is use DINO, but it tries to "stretch" the intermediate denoising space as much as possible to create extra room to store this detail information.

### So, the question arises: why can text models handle both representation and generation well? My understanding is that text models have ample hidden space to aid decision-making, and each step only requires predicting the next single token.

Without this condition, text models would likely also fail to unify generation and understanding.

* First, let's assume we have an excellent representation embedding—an "invincibly good" embedding for, say, a 1024-length text. It performs incredibly well on classification tasks. Can I recover the original text from it? Currently, the answer is no. The experiments by John Morris only proved that one can *sort of* recover it to 128 dimensions. And even with that 128-dimension recovery, I suspect it's actually just doing retrieval. (Meaning, taking the embedding, calculating its similarity with all other texts, finding the closest one, and "recovering" it that way.) **This shows that an "understanding" embedding is not, by nature, a "generative" embedding.**

* Second, let's assume we have an excellent LLM decoder. If we take its final hidden state and use it for probing, how does it perform? The answer is, it's "just okay," but it gets crushed by encoder-only models. It needs a series of changes to get better, like bidirectional attention or specific contrastive learning training.

**This demonstrates that the optimal space for generation is not the optimal space for understanding.**

The solution, then, is to inject an **inductive bias** that allows the generative representation and the understanding representation to have some degree of decoupling. This decoupling is good for linear probing because it can easily mask out the detail-oriented parts of generation. For generation, it still retains enough information to produce the output.

Alternatively, we need an "invincibly good" training method that results in a final model that naturally conforms to this inductive bias.