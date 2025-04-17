---
draft: true
date: 
  created: 2025-04-13
categories:
  - Reading
tags:
  - Transfer Attack
  - ICLR 2024
authors:
  - zhazi
---

# 翻译：Rethinking Model Ensemble in Transfer-based Adversarial Attacks

!!! info "文献"

    - [Rethinking Model Ensemble in Transfer-based Adversarial Attacks](https://arxiv.org/abs/2303.09105)

> 阅读最新论文，找点灵感

## Abstract

深度学习模型对对抗样本缺乏鲁棒性。对抗样本可以在不同的模型之间迁移，这使得攻击可以在目标模型未知的情况下进行。提高可迁移性的一种有效策略是攻击**集成的模型**。然而，以前的研究只是对不同模型的输出进行平均，缺乏对模型集成方法如何以及为何能够大大提高可迁移性的深入分析。

在本文中，我们重新思考了对抗性攻击中的集成，并用两个属性定义了模型集成的共同弱点：1）损失景观的平坦性；2）与每个模型局部最优值的接近程度。我们通过经验和理论表明这两个属性都与可迁移性密切相关，并提出了一种共同弱点攻击 (CWA)，通过提升这两个属性来生成更多可迁移的对抗性样本。

在图像分类和物体检测任务上的实验结果验证了我们的方法在提高对抗性可迁移性方面的有效性，尤其是在攻击对抗性训练的模型时。我们还成功地将我们的方法应用于攻击黑盒大型视觉语言模型——谷歌的 Bard，展示了实际的有效性。代码可在 https://github.com/huanranchen/AdversarialAttacks 上找到。

## Introduction

根据攻击者对受害者模型的了解程度，对抗性攻击通常可分为白盒攻击和黑盒攻击。在模型信息有限的情况下，黑盒攻击要么依赖查询反馈，要么利用可迁移性来生成对抗性示例。

基于迁移的攻击使用针对白盒代理模型生成的对抗性示例来误导黑盒模型，而不需要访问黑盒模型，这使得它们在许多实际场景中更为实用。随着对抗性防御和多样化网络架构 的发展 ，现有方法的可迁移性可能会大大降低。

通过将对抗性示例的可迁移性与深度神经网络的泛化进行类比，许多研究人员通过设计高级优化算法来避免不良的最优值或利用数据增强策略来防止过度拟合，从而提高可迁移性。

同时，为多个代理模型生成对抗性示例可以进一步提高可迁移性，类似于在更多数据上进行训练以提高模型泛化能力。早期方法只是在损失函数或 logits  中对多个模型的输出进行平均，但忽略了它们的不同性质。Xiong 等人引入了 SVRG 优化器，以在优化过程中降低不同模型的梯度方差。然而，最近的研究表明，梯度方差并不总是与泛化性能相关。 从模型角度来看，增加替代模型的数量可以减少泛化误差，一些研究建议从现有模型中创建替代模型。然而，由于计算成本高，对抗攻击中的替代模型数量通常有限，因此有必要开发新的算法，以利用更少的替代模型，同时提高攻击成功率。

![](./images/CWA-commom-weakness.png)

图 1： 共同弱点的说明。 泛化误差与损失景观的平坦度以及解决方案与每个模型最近的局部最优值之间的距离密切相关。我们将模型集成的共同弱点定义为位于平坦景观且接近训练模型局部最优值的解决方案，如 (d) 所示。
{ .caption }

在本文中，我们重新思考了对抗攻击中的集成方法。通过对受害者模型的预期攻击目标进行二次近似，我们观察到二阶项涉及损失函数的 Hessian 矩阵和每个模型到局部最优值的平方 ℓ2 距离，这两者都与对抗可转移性密切相关（如图 1 所示），尤其是当集成中只有少数模型时。

基于这两个术语，我们将模型集成的共同弱点定义为处于平坦景观且接近模型局部最优值的解决方案。为了生成利用模型集成共同弱点的对抗性示例，我们提出了一种共同弱点攻击（CWA）， 由两个子方法组成，分别称为锐度感知最小化（SAM）和余弦相似性鼓励器（CSE），旨在分别优化共同弱点的两个属性。由于我们的方法与以前的方法正交，我们的方法可以与它们无缝结合以提高性能。

我们进行了大量实验，以确认我们的方法生成的对抗性示例的卓越可迁移性。我们首先在图像分类中针对 31 个受害者模型进行验证，这些模型具有各种架构（ 例如 CNN、Transformers ）和训练设置（ 例如标准训练、对抗性训练、输入净化）。令人印象深刻的是，在攻击最先进的防御模型时，在黑盒设置下，MI-CWA 比采用 logits 集成策略的 MI 高出 30%。我们将方法扩展到物体检测，生成一个通用对抗补丁，在 8 个现代检测器上实现了 9.85 的平均 mAP，超越了最近的工作。消融研究还证实，我们的方法确实使损失图景平坦化，并增加了不同模型梯度的余弦相似度。此外，我们基于我们的方法展示了对最近的大型视觉语言模型的成功黑盒攻击，其表现远超基线方法。

## Related Work

在本节中，我们简要回顾了现有的基于迁移的对抗攻击算法。总体而言，这些方法通常将对抗性示例的可迁移性和深度学习模型的泛化性进行类比，我们将它们分为以下 3 类。

基于梯度的优化。 将对抗样本的优化与深度学习模型的训练进行比较，研究者引入了一些优化技术来增强对抗样本的可迁移性。动量迭代（MI）方法 （Dong et al.， 2018 ） 和 Nesterov 迭代（NI）方法 （Lin et al.， 2020 ） 引入动量优化器和 Nesterov 加速梯度，以防止对抗样本陷入不良的局部最优值。增强动量迭代（EMI）方法 （Wang et al.， 2021 ） 累积前次迭代梯度方向上采样的数据点的平均梯度，以稳定更新方向并摆脱不良的局部最大值。方差调整动量迭代（VMI）方法 （Wang & He， 2021 ） 通过使用前一个数据点邻域中的梯度方差来调整当前梯度来降低优化过程中梯度的方差。

输入转换。 这些方法在将输入图像输入分类器之前对其进行变换，以实现多样性，类似于深度学习中的数据增强技术。多样化输入 (DI) 方法 (Xie et al., 2019 ) 对输入图像进行随机调整大小和填充。平移不变 (TI) 攻击 (Dong et al., 2019 ) 推导了一种有效的算法，用于计算平移图像的梯度，这相当于对输入图像进行平移。尺度不变 (SI) 攻击 (Lin et al., 2020 ) 基于模型在缩放图像上具有相似性能的观察结果，使用不同的因子对图像进行缩放。

模型集成方法。 Huang 等人 ( 2023 ) 指出，增加对抗攻击中分类器的数量可降低经验风险最小化 (ERM) 中的泛化误差上界，类似于增加深度学习中的训练示例数量。Dong 等人 ( 2018 ) 提出对替代模型的损失、预测概率或对数取平均值以形成集成。Li 等人 ( 2020 ) 和 Huang 等人 ( 2023 ) 也提出了新方法来生成替代模型的大量变体，然后取损失或对数的平均值。Xiong 等人 ( 2022 ) 将 SVRG 优化器 (Allen-Zhu & Yuan, 2016 ) 引入集成对抗攻击，以减少优化过程中梯度的方差。然而，实践中可用的替代模型数量通常较少，这无法保证 ERM 理论中令人满意的泛化上界，而仅仅关注方差并不一定能获得更好的泛化效果 (Yang et al., 2021 ) 。此外，自组装替代模型的有效性通常低于标准替代模型，导致其性能受限。

## Methodology

在本节中，我们基于二阶近似提出了常见弱点的公式，并提出了由尖锐感知最小化 (SAM) 和余弦相似性鼓励器 (CSE) 组成的常见弱点攻击 (CWA) 。

### Preliminaries

我们让 \( \mathcal{F} = \{ f \} \) 表示给定任务的可能图像分类器的集合，其中每个分类器 \( f : \mathbb{R}^D \to \mathbb{R}^K \) 输出输入 \( x \in \mathbb{R}^D \) 的 K 个类别的 logits。给定自然图像 \( x_{\text{nat}} \) 和相应的标签 \( y \)，基于迁移的攻击旨在构造一个对抗性示例 \( x \)，使得 \( \mathcal{F} \) 中的模型可能会错误分类它。它可以被表述为一个约束优化问题：

$$
\min_{x} \mathbb{E}_{f \in \mathcal{F}} [L(f(x), y)], \quad \text{s.t. } \|x - x_\text{nat}\|_\infty \leq \epsilon,
\tag{1}
$$

其中 \( L(f(x), y) \) 是分类器 \( f \) 在输入 \( x \) 上对标签 \( y \) 的损失函数，\(\epsilon\) 是允许的最大扰动量。

其中 \( L \) 是损失函数，例如负交叉熵损失 \( L(f(x), y) = -\log(\text{softmax}(f(x))) \)，我们研究 \( l_\infty \) 范数。公式 1 可以通过几个“训练”分类器 \( f_i, i=1,\ldots,n \)（称为集成）来近似，如 \( L(f(x), y) \) 的平均值或 \( f_i(x) \) 的对数和的平均值。之前的工作提出了基于这种经验损失的梯度计算或输入变换方法以提高可转移性。例如，动量迭代（MI）方法执行梯度更新（如图 2(a) 所示）作为

$$
\mathbf{m} = \mu \cdot \mathbf{m} + \frac{\nabla_\mathbf{x} L(\frac{1}{n} \sum_{i=1}^{n} f_i(\mathbf{x}), y)}{\Vert \nabla_\mathbf{x} L(\frac{1}{n} \sum_{i=1}^{n} f_i(\mathbf{x}), y)\Vert_1};\quad 
\mathbf{x}_{t+1} = \text{clip}_{\mathbf{x}_{nat},\epsilon}(\mathbf{x}_t + \alpha \cdot \text{sign}(\mathbf{m})),
$$

### Motivation of Commom Weakness

虽然现有方法可以在一定程度上提高可迁移性，但最近的一项研究指出，对抗样本的优化符合经验风险最小化（ERM），训练模型的数量有限可能导致较大的泛化误差。在本文中，我们考虑式（1）中目标的二次近似。

形式上，我们让 \( \mathbf{p}_i \) 表示模型 \( f_i \in \mathcal{F} \) 到 \( x \) 的最近最优解，\( \mathbf{H}_i \) 表示在 \( \mathbf{p}_i \) 处的 \( L(f(x_i), y)) \) 的Hessian矩阵。我们对每个模型 \( f_i \) 使用二阶泰勒展开来近似方程 1 中的 \( \mathbf{p}_i \)。

$$
\mathbb{E}_{f_i \in \mathcal{F}} \left[ L(f_i(\mathbf{p}_i), y) + \frac{1}{2} (\mathbf{x} - \mathbf{p}_i)^T \mathbf{H}_i (\mathbf{x} - \mathbf{p}_i) \right].
\tag{2}
$$

在下面的讨论中，为了简化起见，我们将省略式（2）中期望的下标。根据式（2），我们可以看出 \( E[L(f(\mathbf{p}), y)] \) 和 \( E[(\mathbf{x} - \mathbf{p})^T \mathbf{H} (\mathbf{x} - \mathbf{p})] \) 的较小值意味着测试模型的损失较小，即更好的可转移性。第一项表示每个模型在其自身最优解 \( \mathbf{p}_i \) 处的损失值。一些先前的工作已经证明了局部最优与全局最优在神经网络中几乎具有相同的值。因此，进一步改进这一项的空间可能不大。因此，我们把主要精力放在第二项上，以实现更好的可转移性。

在下面的定理中，我们推导出第二项的上限，因为直接优化它（需要三阶导数）是难以解决的。

!!! info "定理 3.1."

    假设 \(\Vert\mathbf{\mathbf{H}}\Vert_F\) 和 \(\|\mathbf{\mathbf{p}}_1 - \mathbf{x}\|_2\) 之间的协方差为零，我们可以得到第二项的上界如下：

    $$
    \mathbb{E}[(\mathbf{x} - \mathbf{p}_i^\star)^T \mathbf{H}_i(\mathbf{x} - \mathbf{p}_i)] \leq \mathbb{E}[\Vert\mathbf{H}_F\Vert] \cdot \mathbb{E}[\Vert(\mathbf{x} - \mathbf{p}_i^\star)\Vert_2^2].\tag{3}
    $$

??? note "定理 3.1. 的证明"

    **TODO**

直观地说，$\Vert\mathbf{H}\Vert_p$ 代表损失地形的尖锐度/平坦度，而 $\|\mathbf{x}-\mathbf{p}_i\|^2$ 描述了损失地形的平移。这两项可以假设是独立的，因此它们的协方差可以被假定为零。如定理 3.1 所示，$\mathbb{E}[\Vert\mathbf{H}\Vert_F]$ 和 $\mathbb{E}[\Vert\mathbf{x}-\mathbf{p}\Vert^2]$ 的较小值提供了较小的边界，并且会导致测试模型的损失更小。之前的研究指出，Hessian 矩阵的范数较小表明目标的地形较为平坦，这与泛化能力有很强的相关性。至于 $\mathbb{E}[\|\mathbf{x}-\mathbf{p}_i\|^2]$，它代表了从 $\mathbf{x}$ 到每个模型的最优解 $p_i$ 的平方欧几里得距离，我们在第 D.2 节中经验性地展示了它与泛化和对抗可转移性的紧密联系，并在第 A.2 节的一个简单情况下理论上证明了这一点。图 1 提供了一个示例，说明了损失地形的平坦度和不同模型局部最优解之间的接近程度如何有助于对抗可转移性。

根据上述分析，对抗样本的优化变成了寻找一个接近受害模型最优点的点（即最小化原始目标），同时追求该点的景观平坦且与每个最优点的距离较近。后两个目标是我们的主要发现，我们在这里定义了**共同弱点**的概念，这两个术语作为局部点 \( \mathbf{x} \)，其具有小的 \( E[\Vert\mathbf{H}_i\Vert_F] \) 和 \( E[\Vert(\mathbf{x}-\mathbf{p}_i)\Vert^2] \)。请注意，共同弱点和非共同弱点之间没有明确的界限——这两个术语越小，\( \mathbf{x} \) 越可能是共同弱点。最终的目标是找到一个具有共同弱点属性的例子。正如定理 3.1 所指出的那样，我们可以通过分别优化这两个项来实现这一点。

### Sharpness Aware Minimization

为了应对损失地形的平坦性，我们对集成中的每个模型最小化 \( \Vert\mathbf{H}_i\Vert_F \)。然而，这需要对 \( \mathbf{x} \) 的三阶导数进行计算，这在计算上是昂贵的。有一些研究旨在缓解模型训练中损失地形的尖锐度。锐度感知最小化（SAM）是一种有效的算法，用于获取更平坦的地形，它被表述为一个极小极大优化问题。内最大化旨在找到一个方向，沿着这个方向损失变化更快；而外层问题则在这个方向上最小化损失以改善损失地形的平坦性。

在由 \( \ell_\infty \) 范数约束生成的对抗性示例的背景下（如公式 1 所示），我们的目标是在 \( \ell_\infty \) 范数的空间内优化地形的平坦度，以增强其可转移性，这与原始 SAM 算法在 \( \ell_2 \) 范数空间内的做法不同。因此，我们推导出一种适合 \( \ell_\infty \) 范数的改进 SAM 算法（见第 B.1 节）。如图 2 所示，在第 t 次迭代中的对抗性攻击时，SAM 算法首先对当前对抗性示例 \( x_t \) 执行一个梯度上升步骤，步长为 \( \epsilon \)，然后执行另一个梯度下降步骤，步长为 \( r \)。

$$
\mathbf{x}_t^r = \text{clip}_{x_{nat}, \epsilon} \left( \mathbf{x}_t + r \cdot \text{sign} \left( \nabla_\mathbf{x} L \left( \frac{1}{n} \sum_{i=1}^{n} f(\mathbf{x}_i), y \right) \right) \right).
\tag{4}
$$

然后在 \(\mathcal{x}_t^r\) 处执行外梯度下降步骤，步长为 \(\alpha\)，如下所示：

$$
\mathbf{x}_t^f = \text{clip}_{x_{nat}, \epsilon} \left( \mathbf{x}_t^r - \alpha \cdot \text{sign} \left( \nabla_\mathbf{x} L \left( \frac{1}{n} \sum_{i=1}^{n} f(\mathbf{x}_i^r), y \right) \right) \right).
\tag{5}
$$

请注意，在公式4和公式5中，我们在模型集成上应用了 SAM 而不是每个训练模型。因此，我们不仅可以提高反向传播过程中的并行计算效率，还可以利用logits 集成策略获得更好的结果。

此外，我们可以将 SAM 的反向步骤和正向步骤合并为一个更新方向 $\mathbf{x}_t^f-\mathbf{x}_t$，并将其集成到现有的攻击算法中。例如，MI 与 SAM 的集成可以得到 MI-SAM 算法，其更新方式如下（ 如图 2(c) 所示）：

$$
\mathbf{m} = \mu \cdot \mathbf{m} + \mathbf{x}_i^f - \mathbf{x}_i; \quad \mathbf{x}_{i+1} = \text{clip}_{\mathbf{x}_{nat}, \epsilon}(\mathbf{x}_i + \mathbf{m}),
$$

其中 \( \mathbf{m} \) 以衰减因子 \( \mu \) 累积梯度。通过迭代重复此过程，对抗性示例将收敛到一个更平坦的损失景观，从而提高可转移性。

![](./images/CWA-MI-SAM.png)

图 2： MI、SAM 和 MI-SAM 示意图。符号介绍见公式 (4)-(6)
{ .caption }

### Cosine Similarity Encourager

然后我们开发了一种算法，使对抗性示例收敛到每个模型的局部最优解附近。我们不直接优化 $ \frac{1}{n} \sum_{i=1}^{n}\left\Vert\mathbf{x}-\mathbf{p}_{i}\right\Vert_2^2$ ，因为计算相对于 $\mathbf{x}$ 的梯度是困难的，所以我们推导出这个损失的上界。


!!! info "定理 3.2."

    \(\frac{1}{n} \sum_{i=1}^{n}\left\Vert\mathbf{x}-\mathbf{p}_{i}\right\Vert_2^2\) 的上限与所有模型的梯度的点积相似度成正比：

    \[
    \frac{1}{n} \sum_{i=1}^{n} \| (\mathbf{x} - \mathbf{p}_i) \|_2^2 \leq \frac{2M}{n} \sum_{i=1}^{n} \sum_{j=1}^{i-1} \mathbf{g}_i \mathbf{g}_j,
    \]

??? note "定理 3.2. 的证明"

    **TODO**

根据定理 3.2 ，最小化每个模型到局部最优值的距离转化为最大化不同模型梯度之间的点积。为了解决这个问题， Nichol 等人 提出了一种基于一阶导数近似的有效算法。我们应用该算法生成对抗样本，方法是使用从集成模型 $\mathcal{F}_t$ 中采样的每个模型 $f_i$，以较小的步长 $\beta$ 依次执行梯度更新。更新过程如下：

$$
\mathbf{x}_t^i = \text{clip}_{x_{nat}, \epsilon} \left( \mathbf{x}_t^{i-1} - \beta \cdot \nabla_\mathbf{x} L(f_t(\mathbf{x}_t^{i-1}), y) \right),
$$

其中 \(\mathbf{x}_t^0 = \mathbf{x}_t\)。每个模型的更新完成后，我们使用更大的步长 
\(\alpha\) 计算最终更新，如下所示：

$$
\mathbf{x}_{t+1} = \text{clip}_{x_{nat}, \epsilon}(\mathbf{x}_t + \alpha \cdot (\mathbf{x}_t^n - \mathbf{x}_t)).
$$

虽然直接应用该算法可以取得良好的效果，但由于梯度范数的尺度不同，它与 SAM 不兼容。为了解决这个问题，我们在每次更新时用它们的 \(\ell_2\) 范数对梯度进行归一化。我们发现修改后的版本实际上最大化了梯度之间的余弦相似性（证明在下面）。因此，我们将其称为余弦相似性激励器 (CSE)，它可以进一步与 MI 结合成为 MI-CSE。MI-CSE 涉及一个内部动量来累积每个模型的梯度（伪代码在下面）。

??? 证明

    **TODO**

??? 伪代码

    **TODO**

### Common Weakness Attack

鉴于各个算法都在优化损失景观的平坦度和不同模型局部最优值之间的接近度，因此我们需要将它们组合成统一的公共弱点攻击 (CWA)，以实现更好的可迁移性。考虑到并行梯度反向传播的可行性和时间复杂度，我们用 CSE 代替 SAM 的第二步，得到的算法称为 CWA。我们还将 CWA 与 MI 结合起来得到 MI-CWA，伪代码如算法 1 所示。CWA 还可以与其他强对抗攻击算法结合使用，包括 VMI、SSA 等 。


$$
\begin{array}{ll}
& \textbf{Algorithm 1.} \quad \text{MI-CWA 算法} \\
& \textbf{Input. } \text{自然图像 } \mathbf{x}_{nat}, \text{ 标签 } y, \text{ 扰动预算 } \epsilon, \text{ 迭代次数 } T, \text{ 损失函数 } L, \text{ 模型集合 } \\
& F_t (f_i, i \in [1, n]), \text{ 衰减因子 } \mu, \text{ 步长 } r, \beta \text{ 和 } \alpha. \\
& \textbf{Output. } \text{对抗样本 } \mathbf{x}_T. \\
& \textbf{Method. } \\
& 1 \quad \mathbf{m}_0 \gets 0, \hat{\mathbf{m}}_0 \gets 0, \mathbf{x}_0 \gets \mathbf{x}_{nat}; \\
& 2 \quad \textbf{for} t = 0 \textbf{to} T-1 \textbf{do} \\
& 3 \quad \qquad \text{ 计算 } \mathbf{g} \gets \nabla_\mathbf{x} L \left( \frac{1}{n} \sum_{i=1}^n f_i(x_t), y \right); \\
& 4 \quad \qquad \text{ 更新 } \mathbf{x}_t \gets \text{clip}(\mathbf{x}_{nat}, \epsilon (x_t + r \cdot \text{sign}(g))); \\
& 5 \quad \qquad \textbf{for} i = 1 \textbf{to} n \textbf{do} \\
& 6 \quad \qquad \qquad \text{ 计算 } \mathbf{g} \gets \nabla_\mathbf{x} L(f_i(\mathbf{x}_t^{i-1}), y); \\
& 7 \quad \qquad \qquad \text{ 更新内部动量 } \hat{\mathbf{m}} \gets \mu \cdot \hat{\mathbf{m}} + \frac{g}{\Vert\mathbf{g}\Vert_2}; \\
& 8 \quad \qquad \qquad \text{ 更新 } \mathbf{x}_t^i \gets \text{clip}_{\mathbf{x}_{nat}, \epsilon} (x_t^{i-1} - \beta \cdot \hat{m}); \\
& 9 \quad \qquad \textbf{end for} \\
& 10 \quad \qquad \text{ 计算更新方向 } \mathbf{g} \gets \mathbf{x}_t^n - \mathbf{x}_t; \\
& 11 \quad \qquad \text{ 更新外部动量 } \mathbf{m} \gets \mu \cdot \mathbf{m} + \mathbf{g}; \\
& 12 \quad \qquad \text{ 更新 } \mathbf{x}_{t+1} \gets \text{clip}_{\mathbf{x}_{nat}, \epsilon} (\mathbf{x}_t + \alpha \cdot \text{sign}(\mathbf{m})); \\
& 13 \quad \textbf{end for} \\
& 14 \quad \textbf{return } \mathbf{x}_T. \\
\end{array}
$$

## Experiments











