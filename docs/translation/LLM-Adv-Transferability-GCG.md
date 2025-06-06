---
draft: true
date: 
  created: 2025-05-14
categories:
  - Reading
tags:
  - Transfer Attack
authors:
  - zhazi
---

# 翻译 & 思考记录：Universal and Transferable Adversarial Attacks on Aligned Language Models

> 调研一下

## Abstract

由于“开箱即用”的大型语言模型能够生成大量令人反感的内容，因此最近的工作侧重于对齐这些模型，以防止产生不需要的内容。虽然在规避这些措施（针对 LLM 的所谓“越狱”）方面取得了一些成功，但这些攻击需要人类的大量聪明才智，并且在实践中很脆弱。 自动生成对抗性提示的尝试也取得了有限的成功。在本文中，我们提出了一种简单有效的攻击方法，该方法会导致对齐的语言模型产生令人反感的行为。具体来说，我们的方法找到了一个后缀，当它附加到 LLM 的广泛查询以产生令人反感的内容时，旨在最大限度地提高模型产生肯定回答（而不是拒绝回答）的可能性。然而，我们的方法不是依赖手动工程，而是通过结合贪婪和基于梯度的搜索技术自动生成这些对抗性后缀，并且还比过去的自动提示生成方法有所改进。

> 威胁模型如下：
> 1. 动机：出于个人欲望，或出于引流变现的目的，利用 LLM 的生成敏感内容（黄色、暴力、涉政等）
> 2. 威胁来源：有不法意图的用户
> 3. 攻击路径：通过在输入后面添加对抗性后缀，使模型产生政策上禁止的内容。
> 4. 攻击目标：LLM
> 5. 可能的后果：敏感内容广泛传播、LLM 伦理问题等
> 创新点是：
> 不是手工寻找而是自动化搜索？

令人惊讶的是，我们发现我们的方法生成的对抗性提示是高度可转移的，包括转移到黑盒、公开发布的生产 LLM。具体来说，我们在多个提示（即请求许多不同类型的不良内容的查询）以及多个模型（在我们的例子中是 Vicuna-7B 和 13B）上训练对抗性攻击后缀。这样做时， 由此产生的攻击后缀会在 ChatGPT、Bard 和 Claude 以及 LLaMA-2-Chat、Pythia、Falcon 等开源 LLM 的公共接口中诱导令人反感的内容。有趣的是，与基于 GPT 的模型相比，这种攻击转移的成功率要高得多，这可能是因为 Vicuna 本身就是根据 ChatGPT 的输出进行训练的。总的来说，这项工作显着推进了针对对齐语言模型的对抗性攻击的最新技术，提出了有关如何防止此类系统产生令人反感的信息的重要问题。代码可在[github](https://github.com/llm-attacks/llm-attacks)获取。

## Introduction


