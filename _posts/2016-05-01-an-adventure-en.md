---
layout: post
title: "Diffusion LLM分析"
description: "an adventure in English"
tags: [life, entrepreneur, cn]
lang: cn
---
Dllm的优点是什么？Dllm目前fundamental的问题是什么？ Dllm work了吗？
提到做dllm motivation一般可以提到几点，
第一，快，他是一个并行生成过程，32token一起生成是肯定比一个token一个token生成要更快的，
第二，好，好来自于训练数据的增强，利用程度更高。
	理论来说我mask给予了数据更复杂的训练模式，原本我只需要学会根据前面预测后面，现在我还要学会根据后面预测前面，这本身是更多的训练任务，对已经有的足量的数据的更充分的利用，其实在使用生成数据增强模型的时候也有一种可能是生成出和训练时候在模式上不相符合的数据，但意思上符合的数据用于增强。
第三，灵活，其实不同的任务是不同的任务难度的，有很多任务其实很简单，但是我受限于token的吞吐我再聪明我也只能一句话一句话说，而dllm的灵活属性让我有机会能一下子全吐出来。
第四， dllm的表征能力，dllm由于其双向注意力特点以及特殊的训练模式，导致了他单个hidedn space要比decoder only架构要强，天然达到了的llm2vec级别的能力，意味着表征这件事情也有了scale的潜力


dllm的问题是什么？
第一，**dllm其实并不快**，传统AR模型之所以快是因为可以使用KV Cache技术，而Dllm的并行决定了他先天是反KV Cache的，而我就算是32个token一起生成，但是我也需要多轮，这就意味着之前的KV Cache在下一步迭代的时候就已经改变了，并不能复用。 目前LLaDa的结果就是如果我需要生成32个token那就需要32个step,每一个step都是0KV Cache复用。
Fast Dllm大概提出了一种解决方法就是近似KV Cache就是虽然变了但是变得不多，所以我可以使用上一步的KV Cache，其次是类似discrete diffusion forcing的idea就是我可以先block diffusion 生成 所有生成完成的我让他有kv cache，其次是我可以尝试建立hierarchical kv cache的结构，已经生成完成的可以用kv cahce的，然后生成可以是两个block并行的，block间可以有一些近似的kv cache，这样就是fast dllm v2的idea。
那么原版本的llada究竟有多慢呢？这么说吧discrete diffusion forcing 号称他们的model比原版本的llada快50x倍，但实际上也就是比AR快2倍，而fast dllm声称快27倍，换句话说速度不如ar快。
那么为啥Gemini Diffusion的model看起来飞起来了呢？原因是本身AR 7-30B之间的模型上线可能是1s 300 -500 token在openrounter中，理论上我训一个同等大小的模型，做一些列fancy的优化最后应该是可以收益到一个1000-2000 token/s的生成速度，这也就是目前展示diffusion model的速度。

第二，dllm其实有**更高的峰值显存占用**，dllm在同等条件达到显存的峰值，原因是因为block内的attention在并行计算，他会有block size倍数 activation峰值，这也就是说单卡能跑的最长文本其实很有限，目前fast dllm系列的评测基本上都是在512长度测试，并没有在类似10K，100K的场景测试。

第三，实际上生产场景我们可能损失了更多，那我们给同一个GPU多个用户的时候，我们会发现随着batch的上升，Dllm的生成速度会逐渐落后于AR model，这是因为当batch上升的时候，我们batch内本身在并行的，这种并行性随着batch上升被利用起来了。

第四，另一种技术加速技术，投机采样，理论上可以给ar模型带来2-3倍的加速收益，这个收益和dllm的收益是近似的。

当前DLLM的科研改进路线：
1. remask之后丢失之前预测的信息，之前预测的信息理论上是一种中间状态，他是错误的但是是有用的。我们希望能够把这个中间状态用上，让他变成连续状态。
2. 已经存在的文本不值得百分之百的信任，虽然传统的LLaDa可以改动已经生成好的文本，但是他在训练的过程中实际上并没有训练过改动已经生成好的文字的能力。他只训练了改Mask的能力。
3. dllm由于kv cahce的先天限制她最后就一定是block diffusion 的形式，那么他如何构建一个最优生成顺序机理，何时决定AR，何时决定block，如何决定block大小，并且如何在决定最优生成顺序的时候考虑到KV Cache的优化。
4. dllm如何做到插入删除操作，并且做这些操作的时候如何考虑到KV Cache复用等问题

当前DLLM分析类问题：
1. 从AR model fine-tune到dllm 是否具备scale的特性，我能否在100B level获得一个同等级别的dllm，需要用多少token，这些token需要具备什么特性。
2. dllm自带的ar和原本的ar有没有什么区别，是否具备更高的压缩性，如果是KV Cache的话能否删除更大量的KV Cache
3. dllm如果能实现一个laltent embedding的中间状态，是不是意味着large concept model，latent reasoning等问题能够在这种架构下work
4. dllm天然加速类问题是不是是Agent最后一块拼图，agent 的特性是长文本，高latency要求，高智能要求，dllm能够做好压缩和加速的话
5. 为什么latent reasoning是不work的，但是我把图像转成连续空间的token塞到网络就又是work的？



