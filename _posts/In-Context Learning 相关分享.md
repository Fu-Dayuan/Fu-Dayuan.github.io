---

layout:     post
title:      "In-Context Learning（上下文学习）相关分享"
subtitle:   大模型相关
date:       2023-02-09 18:00:00
author:     "Kabi"
tags:

  - 大模型
  - In Context learning
---
详见
https://zhuanlan.zhihu.com/p/603650082/edit

## 1. 前言

随着大模型（GPT3，Instruction GPT，ChatGPT）的横空出世，如何更高效地提示大模型也成了学术界与工业界的关注，因此In-context learning的方法在NLP领域十分火热。
从时间线上看，它的演变历程大约是从Prompt learning（2021年初） 到 Demonstration learning （2021年底） 再到 In-cotnext learning（2022年初），但从方法原理上，他们却有很多相似之处。
本文对一篇有代表性的in-context learning论文：Rethinking the Role of Demonstrations: What Makes In-Context Learning Work? 进行阅读，之后我也会做其他ICL论文的阅读笔记。

## 2. 什么是In-Context Learning：

在这篇综述论文https://arxiv.org/pdf/2301.00234.pdf 给出了详细的定义：
​
![1](https://github.com/dongguanting/dongguanting.github.io/blob/master/img/in-context/1.png)

In Context Learning（ICL）的关键思想是从类比中学习。上图给出了一个描述语言模型如何使用ICL进行决策的例子。首先，ICL需要一些示例来形成一个演示上下文。这些示例通常是用自然语言模板编写的。然后ICL将查询的问题（即你需要预测标签的input）和一个上下文演示（一些相关的cases）连接在一起，形成带有提示的输入，并将其输入到语言模型中进行预测。
值得注意的是，与需要使用反向梯度更新模型参数的训练阶段的监督学习不同，ICL不需要参数更新，并直接对预先训练好的语言模型进行预测（这是与prompt，传统demonstration learning不同的地方，ICL不需要在下游P-tuning或Fine-tuning）。我们希望该模型学习隐藏在演示中的模式，并据此做出正确的预测。
详细介绍见论文

## 3. 论文分析：

Rethinking the Role of Demonstrations: What Makes In-Context Learning Work? 中我们已经发现Demonstration，ICL与大规模语言模型结合（LMs）在零样本条件下的许多任务上取得了很好的效果，但人们对它如何工作以及演示的哪些方面有助于最终任务的执行知之甚少。本文主要以实验为主，探究以上影响ICL的因素。

实验设置：
作者采用12个模型进行了实验。我们包括6种语言模型（表1），所有这些模型都是仅限解码器的dense LM。LMs的大小从774M到175B不等。

![2](https://github.com/dongguanting/dongguanting.github.io/blob/master/img/in-context/2.png)

对于每个模型，作者采用了两种应用方式，即direct和channel：

![3](https://github.com/dongguanting/dongguanting.github.io/blob/master/img/in-context/3.png)

Direct：直接计算给定input x条件下，label y的概率P(y|x)。

Channel：与上面恰好相反，给定y的条件下计算x的概率P(x,y)∝P(x|y）。

作者在如下数据集上进行实验，包括情感分析，段落检测，自然语言推理，仇恨言语检测，问答，句子补全等任务。

![4](https://github.com/dongguanting/dongguanting.github.io/blob/master/img/in-context/4.png)

## 结论1：ICL中 Ground Truth 信息无关紧要
No Demos：LMs直接进行零样本预测，无提示

Demos w gold：依赖于K个标注的examples进行提示，进行预测

Demos w random labels：抽样K个examples提示，但样本labels在标签集中随机采样，而非groundtruth。

![5](https://github.com/dongguanting/dongguanting.github.io/blob/master/img/in-context/5.png)
我们发现，用随机标签替换黄金标签只会轻微影响性能。这一趋势在几乎所有的模型上都是一致的：模型看到的性能下降在0-5%的绝对范围内。在多选择任务中（平均1.7%）替换标签的影响小于在分类任务中（2.6%的绝对标签）。

这一结果表明，地面真实值输入标签对并不是实现性能提高的必要条件。这是违反直觉的，因为正确的配对训练数据在典型的监督训练中是至关重要的——它通知模型执行下游任务所需的期望输入-标签对应。尽管如此，这些模型在下游任务上确实取得了非常重要的性能。
作者在以下3个维度上进一步做了消融实验：正确演示占总的百分比（下图1）与演示样本数量K（下图2），演示的模板样式（下图3）

![6](https://github.com/dongguanting/dongguanting.github.io/blob/master/img/in-context/6.png)
![7](https://github.com/dongguanting/dongguanting.github.io/blob/master/img/in-context/7.png)
![8](https://github.com/dongguanting/dongguanting.github.io/blob/master/img/in-context/8.png)

可以得到相似的结论，在演示正确与否影响并不大。

## 结论2：ICL的性能收益主要来自独立规范的 输入空间 和 标签空间 ，以及正确一致的演示格式
作者分别从以下四个维度探究In-Context Learning效果增益的影响
1. The input-label mapping：即每个输入xi是否与正确的标签yi配对
2. The distribution of the input text：即x1...xk的分布是否一致
3. The label space：y1...yk所覆盖的标签空间
4. The format：使用输入标签配对作为格式。

![9](https://github.com/dongguanting/dongguanting.github.io/blob/master/img/in-context/9.png)

输入文本分布实验：
下图中，青绿色的柱子为用（从外部语料中）随机采样的句子替换输入句子的设置。可以看到，模型表现明显下降。因此，in-context learning中，演示中的分布内输入极大地有助于提高性能。这可能是因为已IND（in-distribution）文本的条件使任务更接近于语言建模，因为LM在此期间总是以IND文本为条件进行推理标签。

![10](https://github.com/dongguanting/dongguanting.github.io/blob/master/img/in-context/10.png)

标签空间实验：
下图中，青绿色的柱子为用随机英语词汇替代展示样本中的标签。可以看到，模型表现明显下降。因此，in-context learning中，标签空间的一致性显著有助于提高性能。

![11](https://github.com/dongguanting/dongguanting.github.io/blob/master/img/in-context/11.png)

演示格式实验：
下图中，分别用labels only（深紫）和no labels（深绿）来探索演示模式的差异对模型表现的影响。可以看到，模型相对于上面两图的OOD setting而言，都有了进一步的下降。这可以表明ICL中保持输入-标签对的格式是关键的。

![12](https://github.com/dongguanting/dongguanting.github.io/blob/master/img/in-context/12.png)

## 有意思的讨论：
作者还进行了个有意思的讨论，即模型是否在Test阶段学习到了知识？
作者认为如果我们对学习进行严格的定义，即学习在训练数据中给出的输入标签对，那么lm在测试时不学习新的任务。然而，学习一项新任务可以更广泛地解释：它可能包括适应特定的输入和标签分布以及演示的格式，并最终更准确地做出预测。有了这个学习的定义，该模型确实可以从演示中学习任务。我们的实验表明，该模型确实利用了演示的各个方面，并实现了性能的提高。

## 4. 总结：
本文从多个角度探究了演示是如何让In-context learning在不同的任务中产生性能增益的，而且随着fine-tune阶段的黑盒化，很多文章也提出fine-tune阶段可能让模型丧失了泛化性，那么ICL这种不fine tune的方法既节省时间与资源开销，且能提升效果，应该会在大模型林立的时代被人关注，并迅速火起来。

