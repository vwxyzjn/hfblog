---
title: "如何成功将 🤗 API 客户的 transformer 模型推理速度加快 100 倍"
thumbnail: /blog/assets/09_accelerated_inference/thumbnail.png
translators:
- user: MatrixYao
- user: zhongdongy
  proofreader: true
---

# 如何成功将 🤗 API 客户的 transformer 模型推理速度加快 100 倍 


🤗 Transformers 已成为世界各地数据科学家用以探索最先进 NLP 模型、构建新 NLP 模块的默认库。它拥有超过 5000 个预训练和微调的模型，支持 250 多种语言，任君取用。无论你使用哪种框架，都能用得上它。

虽然在 🤗 Transformers 中试验模型很容易，但以最高性能将这些大模型部署到生产中，并将它们用可扩展的架构管理起来，对于任何机器学习工程师来说都是一个 **艰巨的工程挑战**。

100 倍性能提升及内置可扩展性是用户选择在我们托管的 [Accelerated Inference API](https://huggingface.co/pricing) 基础上构建自己的 NLP 模块的原因。尤其是为了实现 **最后那 10 倍性能** 提升，我们需要进行底层的、特定于模型且特定于目标硬件的优化。

本文分享了我们为用户充分榨干每一滴计算资源所使用的一些方法。 🍋

## 获取首个 10 倍加速

优化之旅的第一站相对来讲是最容易的，主要涉及到 [Hugging Face 库](https://github.com/huggingface/) 提供的所有平台无关的优化技术。

我们在 Hugging Face 模型的 [流水线 (`pipeline` )](https://huggingface.co/transformers/main_classes/pipelines.html) 中集成了能有效减少每次前向传播计算量的最佳方法。这些方法因模型架构和目标任务不同而不同，例如，对基于 GPT 架构的模型的文本生成任务，我们通过缓存过去时刻的注意力矩阵，而仅计算每一轮中最后一个新词元的注意力，来减小参与计算的注意力矩阵的维度:

-| 原始版 | 优化版 |
-|:---------------------------------------------------------------------------------------------------------:|:-------------------------------------------------------------------------------------------------------:|
-|![](/blog/assets/09_accelerated_inference/unoptimized_graph.png)|![](/blog/assets/09_accelerated_inference/optimized_graph.png)|

分词常常成为推理效率的瓶颈。我们在 [🤗 Tokenizers](https://github.com/huggingface/tokenizers/) 库中实现了高效的算法，用 Rust 来实现模型分词器并与智能缓存技术相结合，获得了高达 10 倍的端到端延迟加速。

利用 Hugging Face 库的最新功能，在相同的模型及硬件上，与开箱即用的部署相比，我们稳定达到了 10 倍加速。由于 Transformer 和 Tokenizer 通常每月都会发版，因此我们的 API 客户无需不断适配新的优化，即可让自己的模型越跑越快。

## 为了胜利而编译: 10 倍加速硬核技术

现在到真正棘手的地方了。为了获得最佳性能，我们需要修改模型并针对特定硬件进行编译以优化推理速度。选择什么硬件取决于模型 (内存大小) 和需求情况 (对请求进行组批)。即使是使用相同的模型来进行预测，一些 API 客户可能会更受益于 CPU 推理加速，而其他客户可能会更受益于 GPU 推理加速，而每种硬件会涉及不同的优化技术以及库。

一旦为针对应用场景选定计算平台，我们就可以开始工作了。以下是一些可应用于静态图的针对 CPU 的优化技术:

- 图优化 (删除无用节点和边)
- 层融合 (使用特定的 CPU 算子)
- 量化

使用开源库中的开箱即用功能 (例如 🤗 Transformers 结合 [ONNX Runtime](https://github.com/microsoft/onnxruntime)) 很难得到最佳的结果，或者会有明显的准确率损失，特别是在使用量化方法时。没有什么灵丹妙药，每个模型架构的最佳优化方案都不同。但深入研究 Transformers 代码和 ONNX Runtime 文档，星图即会显现，我们就能够组合出适合目标模型和硬件的额外的 10 倍加速方案。

## 不公平的优势

从 NLP 起家的 Transformer 架构是机器学习性能的决定性转折点，在过去 3 年中，自然语言理解和生成的进展急剧加快，同时水涨船高的是模型的平均大小，从 BERT 的 110M 参数到现在 GPT-3 的 175B 参数。

这种趋势给机器学习工程师将最新模型部署到生产中带来了严峻的挑战。虽然 100 倍加速是一个很高的标准，但惟有这样才能满足消费级应用对实时性的需求。

为了达到这个标准，作为 Hugging Face 的机器学习工程师，我们与 🤗 Transformers 和  🤗 Tokenizers 维护人员 😬 相邻而坐，相对其他机器学习工程师而言当然拥有不公平的优势。更幸运的是，通过与英特尔、英伟达、高通、亚马逊和微软等硬件及云供应商的开源合作建立起的广泛合作伙伴关系，我们还能够使用最新的硬件优化技术来优化我们的模型及基础设施。

如果你想感受我们基础设施的速度，可以 [免费试用](https://huggingface.co/pricing) 一下，我们也会与你联系。
如果你想在自己的基础设施实施我们的推理优化，请加入我们的 [🤗 专家加速计划](https://huggingface.co/support)。

---

>>>> 英文原文: <url>https://huggingface.co/blog/accelerated-inference</url>
>>>>
>>>> 原文作者: Hugging Face
>>>>
>>>> 译者: Matrix Yao (姚伟峰)，英特尔深度学习工程师，工作方向为 transformer-family 模型在各模态数据上的应用及大规模模型的训练推理。
>>>>
>>>> 审校/排版: zhongdongy (阿东)