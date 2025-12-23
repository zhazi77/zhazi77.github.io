---
draft: false
date:
  created: 2024-09-06
  updated: 2024-09-08
  updated: 2024-11-24
  updated: 2025-01-02
  updated: 2025-04-12
  updated: 2025-04-13
categories:
  - Summary
tags:
  - Rethingking
  - Adversarial Attack
authors:
  - zhazi
---

# 三维点云对抗攻击反思录(一) —— C&W Attack

>  随着三维视觉的发展，对抗攻击研究已然向三维点云发展。目前已经有工作对隐蔽性、可迁移性、最小生成代价等对抗样本经典问题进行了探讨，并出现了一些面向真实世界场景的对抗攻击方法。然而，在我研究学习过程中，我发现相关三维点云对抗攻击（之后简称点云对抗）的代码相比于图像中的对抗攻击要复杂的多。这导致我往往需要花很多的精力在那些与论文核心方法无关的代码上，而不能直入主题。我还时常对现有研究的评估体系感到疑惑，在我看来，一些工作的评估方式存在漏洞，甚至可能会导致错误的认识。在经历了数次困惑到理解的循环后，我最终决定整理并记录下学习过程中的一些想法，也算是给短暂的研究生涯画上一个句号。

在对抗攻击研究中，**[C&W Attack](https://arxiv.org/abs/1608.04644)** 是一项具有里程碑意义的工作。作者结合多目标优化的思想设计了全新的对抗攻击框架 C&W Attack，突破了（当时）最新的防御方法：防御性蒸馏，改变了社区对神经网络鲁棒性的认识。在此之后，有许多工作都建立在 C&W Attack 上。点云对抗领域的首篇[工作](https://arxiv.org/abs/1809.07016)就是 C&W Attack 在三维视觉上的精神续作，之后一系列的点云对抗研究也大都沿用了这一算法框架。

> 个人认为这一现象的背后有两方面因素：一是研究的惯性。在已经被证明有效的工作上改进是一种低门槛、高收益的科研方式。在前人的工作上改进可以避开很多前人踩过的坑，从而专注于自己的研究目标。二是点云模态的特殊性。人类对形变的感知比对颜色变化的感知更为敏感，因此点云中对抗样本需要设置更小的扰动，而 **C&W Attack** 中正好包含了许多优化扰动大小的设计。此外，C&W Attack 不仅是一种攻击方法，更是一个算法框架，这给后续工作提供了更多的改进空间 ~~适合水论文~~。

总之，**C&W Attack** 在点云对抗中具有重要地位。然而，各研究在借鉴和改进该方法时，通常会根据自身需求对算法的设计进行取舍与调整。这导致尽管许多论文都声称基于 C&W Attack，但在具体实现上存在差异，为初学者的学习带来了挑战。因此，本系列的第一篇以 C&W Attack 开始，旨在梳理 C&W Attack 在点云对抗中的应用。具体而言，本文将从算法设计的角度讲解 C&W Attack 的关键点，主要包含两部分内容：

1. 理论层面，怎么理解 C&W Attack? 
2. 实践层面，C&W Attack 代码的哪些地方需要关注? 

希望读者阅读结束后，能对 C&W Attack 有一个比较清晰的理解。

??? info "参考文献"

    - [Towards Evaluating the Robustness of Neural Networks](https://arxiv.org/abs/1608.04644)
<!-- more -->

## 背景知识

### 对抗样本
对抗样本是指经过人为修改或添加微小扰动后的输入数据，尽管这些修改对人类而言几乎不可见，但却足以使机器学习模型产生错误判断。例如，在图像识别任务中，添加几乎不可见的噪声可能会让模型将一只猫错误地识别为一只狗。

从对抗样本的经典定义中，可以提炼出如下两个要求：

1. **对抗性（Adversarial）**：对抗样本 $x^{\text{adv}}$ 能够误导目标模型，使其输出错误的预测结果。
2. **不可感知性（Imperceptible）**：对于人类而言，很难判断输入是否被扰动，即对抗样本与原始输入在感知上几乎无法区分。

从设计时的思路来看，对抗样本的生成方法的设计思路可以粗略的分为两类：基于优化的思路和基于分布的思路。前者从最优化的角度思考问题，将对抗样本建模为各种优化问题，然后设计算法来求解优化问题，所得到的解就是对抗样本。这类方法所设计的求解算法通常基于迭代的对抗样本搜索算法；后者从概率论的角度思考问题，将对抗样本建模为概率分布，先设法获取对抗样本分布，然后采样对抗样本分布来获得对抗样本。

**注**：C&W Attack 是基于优化的方法，本系列也主要关注于基于优化的方法，不会涉及基于分布的方法。

### 基于优化的对抗样本搜索方法

基于优化的方法依据对抗样本的定义构建一个优化问题，然后通过求解优化问题的方法来制作对抗样本。主流的优化问题构建方式可以分为两种：**建模为盒约束问题**和**建模为双目标优化问题**。

#### 盒约束问题

在这一类方法中，攻击者将对抗样本建模为一个具有盒约束的优化问题：

$$
\begin{align}
    \underset{\mathbf{x}^{\text{adv}}}{\arg\max}\quad &{\mathcal{J}(f(\mathbf{x}^{\text{adv}}), y)} \\
    \textrm{s.t.}\quad &\Vert \mathbf{x}^{\text{adv}} - \mathbf{x}^{\text{ori}}\Vert_{\infty} < \epsilon,
\end{align}
\tag{1}
$$

其中 $f$ 是目标模型，$\mathcal{J}(\cdot, \cdot)$ 表示交叉熵损失函数，$\mathbf{x}^{\text{ori}}$ 是具有真实标签 $y$ 的干净样本。攻击者的目标是在 $\mathbf{x}^{\text{ori}}$ 的 $\ell_\infty$ 邻域中找到一个对抗样本 $\mathbf{x}^{\text{adv}}$ 使得 $\mathbf{x}^{\text{adv}}$ 能误导 $f$ 。

在实践中，通常将扰动 ($\Delta = \mathbf{x}^{\text{adv}} - \mathbf{x}^{\text{ori}}$) 作为优化变量，并通过迭代算法来求解这个优化问题。例如，I-FGSM 中的扰动迭代公式为：

$$
  \Delta_{i+1} = \Delta_i + \text{Clip}_\epsilon \left\{ \alpha \cdot \text{Sign} \left( \nabla \mathcal{J}(f(\mathbf{x}^{\text{ori}} + \Delta_i), y) \right) \right\}.
$$

其中，$\text{Clip}_{\epsilon}(\cdot)$ 表示将超出预算 $\epsilon$ 的部分裁剪到 $\epsilon$ 的操作（也被称为投影），$\alpha$ 是迭代步长。对于一个 $T$ 步的迭代过程，通常设置 $\alpha=\epsilon / T$

#### 双目标优化问题

另一种方式是将对抗样本的两方面要求都视为优化目标，将对抗样本建模为一个双目标优化问题。这就是 C&W Attack 所考虑的，作者将对抗性目标建模为：

$$
\mathcal{L}_{adv} = \left(\max_{i\neq t}(f(\mathbf{x}^{\text{adv}})_{i})-f(\mathbf{x}^{\text{adv}})_{t}\right)^{+}
$$

其中，$f(\cdot)_{i}$ 表示 $f$ 输出的预测分数中 $i$ 类别的置信度。$(\cdot)^{+}$ 是一个分段函数，将小于 0 的部分置为 0（等效于 ReLU)。另一方面，不可感知性目标被刻画为 $p$ 范数距离：

$$
\mathcal{L}_{dis} = \Vert \mathbf{x}^{\text{adv}} - \mathbf{x}^{\text{ori}} \Vert_{p}
$$

通常（图像中）取 $p=2$。综上，对抗样本被表述为如下双目标优化问题：

$$
\begin{align}
\min_{\mathbf{x}^{\text{adv}}} & \quad \left(\max_{i\neq t}(f(\mathbf{x}^{\text{adv}})_{i})-f(\mathbf{x}^{\text{adv}})_{t}\right)^{+} \\
\min_{\mathbf{x}^{\text{adv}}} & \quad \Vert \mathbf{x}^{\text{adv}} - \mathbf{x} \Vert_{p} \\
\end{align}
\tag{2}
$$

与盒约束限制扰动必须在一定范围内不同，这种问题建模方式允许在对抗样本难以找到时增大扰动大小以实现攻击，因此这一设计也被称为**软约束**，相应地，前面的盒约束则被称为**硬约束**。

背景知识补充完毕，接下来正式进入 C&W Attack 的介绍。

## C&W Attack
C&W Attack 求解上述双目标优化问题的思路是常规的：**通过加权将双目标转化为单目标，然后通过超参搜索技术寻找最好的权值。**

### 问题转化
C&W Attack 引入了一个权重系数 $c$ 将问题转化为如下形式：

$$
\begin{align}
&\underset{\mathbf{x}^{\text{adv}}}{\arg\min} \quad c \mathcal{L}_{adv} + \mathcal{L}_{dis} \\
\Longrightarrow \quad &\underset{\mathbf{x}^{\text{adv}}}{\arg\min} \quad c \cdot \left(\max_{i\neq t}(f(\mathbf{x}^{\text{adv}})_{i})-f(\mathbf{x}^{\text{adv}})_{t}\right)^{+} + \Vert \mathbf{x}^{\text{adv}} - \mathbf{x}^{\text{ori}} \Vert_{p}
\end{align}
\tag{3}
$$

!!! warning "注意"

    这里权重 $c$ 加在对抗性目标上，后面将提到的点云对抗中的设置与此不同。

在解决上述方程时，较小的 $c$ 会使得所得到的解 $\mathbf{x}^{\text{adv}}$ 更具不可感知性，但同时也使得优化问题可行集更小，增大了求解失败的概率。相反，较大的 $c$ 提供了更大的可行集，代价是放宽了对不可感知性的约束。不同的干净样本 $\mathbf{x}^{ori}$ 对应的最佳权重 $c$ 不同，需要依据求解结果确定。因此 C&W Attack 引入了**超参数搜索技术**，使用**二分搜索**来寻找最优的 $c$。

### 超参二分搜索

一个理想的 $c$ 应该满足：如果减少它，则公式 3. 中的优化问题无解，换言之，**理想的 $c$ 的值是使得公式 3. 中问题有解的最小值**。此时，得到的样本具有最好的不可感知性，且能够实施攻击。

为了找到最优的 \( c \)，C&W Attack 采用了二分搜索技术。这里先简述超参搜索的过程，然后再结合代码看具体如何实现。为了理解方便，不妨先考虑单个样本的情况。

首先，固定 $c$，求解公式 3. 所示的优化问题，得到当前问题的解。然后，检查解的有效性（即样本是否能够攻击模型）。如果本轮攻击成功，则放大 $c$，让下一轮攻击目标向对抗性目标倾斜；反之，如果本轮攻击失败，则缩小 $c$，让下一轮攻击目标向不可感知性目标倾斜。最后，重复上述过程，寻找一个最佳的 $c$。算法流程如下：

1. **初始化搜索范围**：设定 \( c \) 的初始搜索范围，例如 \( c_{\text{min}} = 0.1 \) 和 \( c_{\text{max}} = 10^5 \)。
2. **计算中值**：在当前搜索范围内计算 \( c \) 的中值 \( c_{\text{mid}} = \frac{c_{\text{min}} + c_{\text{max}}}{2} \)。
3. **评估中值**：令 \( c = c_{\text{mid}} \)，求解公式 3. 中的优化问题生成对抗样本，并评估其是否成功欺骗模型。
4. **更新搜索范围**：
    - 如果对抗样本攻击成功，则减小 \( c \) 的搜索上界，令 \( c_{\text{max}} = c_{\text{mid}} \)。
    - 如果对抗样本攻击失败，则增加 \( c \) 的搜索下界，令 \( c_{\text{min}} = c_{\text{mid}} \)。
5. **重复步骤 2-4**：直到满足特定的结束条件。

而对公式 3. 中优化问题的求解可以使用梯度下降法，算法流程如下：

1. **初始化对抗样本**：初始化对抗样本 $\mathbf{x}^{adv}_0$，通常设置为原始样本 $\mathbf{x}^{ori}$。
2. **计算梯度**：计算损失函数 $c \mathcal{L}_{adv} + \mathcal{L}_{dis}$ 关于 $\mathbf{x}^{adv}$ 的梯度 $\nabla_{\mathbf{x}} (c \mathcal{L}_{adv} + \mathcal{L}_{dis})$。
3. **更新对抗样本**：根据梯度下降的更新规则，更新对抗样本：

    $$
    \mathbf{x}^{adv}_{t+1} = \mathbf{x}^{adv}_t - \eta \cdot \nabla_{\mathbf{x}} (c \cdot \mathcal{L}_{adv} + \mathcal{L}_{dis})
    $$

    其中，$\eta$ 是学习率，控制着更新的步长。

4. **重复步骤 2-3**：直到满足特定的结束条件。

## C&W Attack 代码剖析

现在我们走出单个样本输入带来的舒适区，考虑多个样本的情况。实际上，C&W Attack 的[代码实现](https://github.com/carlini/nn_robust_attacks/tree/master)就考虑了 batch 输入，这也是其复杂性的来源之一。下面结合代码仓库下 `l2_attack.py` 的代码(见 `#!python attack_batch()` 方法) 进一步的剖析 C&W Attack。 

### 双循环
C&W Attack 由内外两个循环组成：内循环固定 $c$ 的值对公式 3. 中的优化问题进行求解（梯度下降法）。外循环进行超参数搜索，依据内循环的求解结果调整 $c$ 的值（二分搜索）。如下所示：

```python title="l2_attack.py" linenums="172" hl_lines="1"
        for outer_step in range(self.BINARY_SEARCH_STEPS):
            print(o_bestl2)
            # completely reset adam's internal state.
            self.sess.run(self.init)
            batch = imgs[:batch_size]
            batchlab = labs[:batch_size]
            # ......

```
```python linenums="192" hl_lines="1"
            for iteration in range(self.MAX_ITERATIONS):
                # perform the attack 
                _, l, l2s, scores, nimg = self.sess.run([self.train, self.loss, 
                                                         self.l2dist, self.output, 
                                                         self.newimg])
            # ......
```

这里，`MAX_ITERATIONS` 参数控制梯度下降的迭代次数，`BINARY_SEARCH_STEPS` 参数控制超参搜索的搜索次数。

为了后续表述方便，这里下两个定义：

1. **全局最优解**：全局最优解是指在所有可能的解中，满足对抗性目标的前提下，不可感知性（通常通过 $L_2$ 距离衡量）最佳的解。在 C&W Attack 的上下文中，全局最优解是指在整个二分搜索过程中，所有尝试的权重系数 $c$ 下生成的对抗样本中，具有最小 $L_2$ 距离且成功欺骗模型的样本。

2. **局部最优解**：局部最优解是指在特定的权重系数 $c$ 下，通过梯度下降法在当前搜索空间内找到的，满足对抗性目标的前提下，不可感知性最佳的解。在 C&W Attack 的上下文中，局部最优解是指在每个固定的 $c$ 值下，通过梯度下降法生成的对抗样本中，具有最小 $L_2$ 距离且成功欺骗模型的样本。

C&W Attack 的目标是获得全局最优解，而为了获得全局最优解，C&W Attack 维护了两部分记录：**全局最优解记录**和**局部最优解记录**。

### 外循环：维护全局最优解

外循环维护了全局最优解，即在整个搜索过程中找到的最佳对抗样本。这包含两部分逻辑：

1. **记录初始化**：在开始二分搜索之前，初始化全局最优解记录，包括最优的 $L_2$ 距离（`o_bestl2`）、模型预测分数（`o_bestscore`）和对抗样本（`o_bestattack`）。

```python title="l2_attack.py" linenums="167" hl_lines="2-4"
        # the best l2, score, and image attack
        o_bestl2 = [1e10]*batch_size
        o_bestscore = [-1]*batch_size
        o_bestattack = [np.zeros(imgs[0].shape)]*batch_size
        
        for outer_step in range(self.BINARY_SEARCH_STEPS):
            print(o_bestl2)
            # completely reset adam's internal state.
            self.sess.run(self.init)
            batch = imgs[:batch_size]
            batchlab = labs[:batch_size]
    
            bestl2 = [1e10]*batch_size
            bestscore = [-1]*batch_size
```

2. **最优解更新**：对于每个固定的 $c$，内循环会生成一个对抗样本。如果这个样本的 $L_2$ 距离小于当前全局最优解的 $L_2$ 距离，并且成功欺骗了模型，则更新全局最优解记录。

``` python title="l2_attack.py" linenums="213" hl_lines="6-9"
                # adjust the best result found so far
                for e,(l2,sc,ii) in enumerate(zip(l2s,scores,nimg)):
                    if l2 < bestl2[e] and compare(sc, np.argmax(batchlab[e])):
                        bestl2[e] = l2
                        bestscore[e] = np.argmax(sc)
                    if l2 < o_bestl2[e] and compare(sc, np.argmax(batchlab[e])):
                        o_bestl2[e] = l2
                        o_bestscore[e] = np.argmax(sc)
                        o_bestattack[e] = ii
```

代码中 `e` 表示当前样本的序号，`l2` 表示当前样本的 $L_2$ 距离，`sc` 表示模型对当前样本的预测结果，`ii` 表示当前样本的对抗样本。`#!python batchlab[e]` 表示当前样本的标签，`compare(sc, np.argmax(batchlab[e]))` 判断模型对当前样本的预测结果是否与当前样本的标签相同。


### 内循环：维护局部最优解

??? question "为什么要维护局部最优解"

    维护局部最优解是二分搜索的需要，在二分搜索时会依据本轮内循环得到的局部最优解是否攻击成功来更新搜索区间。

内循环维护了局部最优解，即在当前 $c$ 值下找到的最佳对抗样本。同样包含两部分逻辑：

1. **记录初始化**：在本轮梯度下降迭代开始前，初始化局部最优解记录，包括最优的 $L_2$ 距离（`bestl2`）和模型预测分数（`bestscore`）。对抗样本本身就是在内循环中迭代更新的，因此不需要额外记录。

```python title="l2_attack.py" linenums="167" hl_lines="13-14"
        # the best l2, score, and image attack
        o_bestl2 = [1e10]*batch_size
        o_bestscore = [-1]*batch_size
        o_bestattack = [np.zeros(imgs[0].shape)]*batch_size
        
        for outer_step in range(self.BINARY_SEARCH_STEPS):
            print(o_bestl2)
            # completely reset adam's internal state.
            self.sess.run(self.init)
            batch = imgs[:batch_size]
            batchlab = labs[:batch_size]
    
            bestl2 = [1e10]*batch_size
            bestscore = [-1]*batch_size
```
2. **最优解更新**：在梯度下降的每一步，生成一个新的对抗样本。如果这个样本的 $L_2$ 距离小于当前局部最优解的 $L_2$ 距离，并且成功欺骗了模型，则更新局部最优解记录。

``` python title="l2_attack.py" linenums="213" hl_lines="3-5"
                # adjust the best result found so far
                for e,(l2,sc,ii) in enumerate(zip(l2s,scores,nimg)):
                    if l2 < bestl2[e] and compare(sc, np.argmax(batchlab[e])):
                        bestl2[e] = l2
                        bestscore[e] = np.argmax(sc)
                    if l2 < o_bestl2[e] and compare(sc, np.argmax(batchlab[e])):
                        o_bestl2[e] = l2
                        o_bestscore[e] = np.argmax(sc)
                        o_bestattack[e] = ii
```

### 搜索区间的更新

在 C&W Attack 实施过程中，为了找到最佳的权重 \( c \)，算法需要自适应地调整每个样本的搜索区间。初始时，所有样本的搜索区间均被统一设置为 [0, 1e10]，而权重 \( c \) 则被初始化为 1。见代码 162 行：

```python title="l2_attack.py" linenums="162"  hl_lines="2-4"
        # set the lower and upper bounds accordingly
        lower_bound = np.zeros(batch_size)
        CONST = np.ones(batch_size)*self.initial_const
        upper_bound = np.ones(batch_size)*1e10
```

这里 `lower_bound`, `upper_bound`, `CONST` 三个变量记录了当前 batch 中每个样本的搜索区间和本轮权重值。

在每轮外循环结束时，算法会基于本轮内循环的攻击结果对搜索区间进行更新。前面讨论过这个更新过程，不过实际代码实现中的更新规则略有不同。具体如下：

``` python title="l2_attack.py" linenums="223" hl_lines="3 6 12"
            # adjust the constant as needed
            for e in range(batch_size):
                if compare(bestscore[e], np.argmax(batchlab[e])) and bestscore[e] != -1:
                    # success, divide const by two
                    upper_bound[e] = min(upper_bound[e],CONST[e])
                    if upper_bound[e] < 1e9:
                        CONST[e] = (lower_bound[e] + upper_bound[e])/2
                else:
                    # failure, either multiply by 10 if no solution found yet
                    #          or do binary search with the known upper bound
                    lower_bound[e] = max(lower_bound[e],CONST[e])
                    if upper_bound[e] < 1e9:
                        CONST[e] = (lower_bound[e] + upper_bound[e])/2
                    else:
                        CONST[e] *= 10
```

先看 225 行，`bestscore` 记录了模型对每个解的预测结果，`batchlab` 记录了每个样本的标签，`#!python compare(bestscore[e], np.argmax(batchlab[e]))` 比较将模型的预测结果与真值标签进行比较，判断是否攻击成功。这里额外判断 `#!python bestscore[e] != -1` 的原因是，如果梯度下降中每一步都没有攻击成功，则有 `#!python bestscore[e]` 的值为 -1，这会导致 `#!python compre()` 误判攻击成功。

??? note "225 行可以使用更简单的条件"

    实际上 225 行只需要判断 `#!python bestscore[e] != -1` 就足够了。因为如果条件成立，就说明本轮有进入过 217 行的更新（内循环中只有这一处更新），而每次进入时必然有 `#!python l2 < bestl2[e] and compare(sc, np.argmax(batchlab[e]))` 成立，因此 `#!python bestscore[e] != -1` 成立时，225 行的中左半边的条件一定也成立。
    
    **补充**：记命题 $A$ 为 `#!python compare(bestscore[e], np.argmax(batchlab[e]))`, 命题 $B$ 为 `#!python bestscore[e] != -1`，依据上面的分析，我们有 $A \rightarrow B$ 因此有 $A \land B \equiv A$。也即 225 行使用的条件与 `#!python bestscore[e] != -1` 等价。
    
    ``` python title="l2_attack.py" linenums="213" hl_lines="5"
                    # adjust the best result found so far
                    for e,(l2,sc,ii) in enumerate(zip(l2s,scores,nimg)):
                        if l2 < bestl2[e] and compare(sc, np.argmax(batchlab[e])):
                            bestl2[e] = l2
                            bestscore[e] = np.argmax(sc)
                        if l2 < o_bestl2[e] and compare(sc, np.argmax(batchlab[e])):
                            o_bestl2[e] = l2
                            o_bestscore[e] = np.argmax(sc)
                            o_bestattack[e] = ii
    ```


读者如果有兴趣的话，不妨带着上面的认识把下面带注释版本的 `#! attack_batch()` 过一遍（建议复制到自己的 IDE 中查看），应该会有更深刻的理解。

??? note "attack_batch 代码(带注释)"

    **TODO: 加上注释** 
    
    ```python title="l2_attack.py"
    
    def attack_batch(self, imgs, labs):
        """
        对一批输入图像执行 C&W L2 对抗攻击。
        
        参数:
            imgs: 原始输入数据（如图像），形状为 [batch_size, ...]，值域通常为 [0, 1] 或 [-1, 1]
            labs: 对应的真实标签（one-hot 或整数形式）
    
        返回:
            o_bestattack: 成功生成的对抗样本列表，每个样本在满足攻击成功的前提下具有最小 L2 扰动
        """
    
        def compare(x, y):
            """
            判断模型输出 x 是否满足攻击成功条件（针对目标类 y）。
            
            - 若为非目标攻击（默认）：要求预测结果 ≠ 真实标签
            - 若为目标攻击（TARGETED=True）：要求预测结果 == 目标标签
            
            此外，若启用了 CONFIDENCE（置信度 margin），则需确保预测分数差超过该阈值。
            """
            if not isinstance(x, (float, int, np.int64)):
                # x 是 logits 向量（未经过 softmax）
                x = np.copy(x)
                if self.TARGETED:
                    # 目标攻击：目标类得分需比其他所有类高至少 CONFIDENCE
                    x[y] -= self.CONFIDENCE
                else:
                    # 非目标攻击：真实类得分需比次高类低至少 CONFIDENCE
                    x[y] += self.CONFIDENCE
                x = np.argmax(x)  # 取 argmax 作为最终预测
            if self.TARGETED:
                return x == y
            else:
                return x != y
    
        batch_size = self.batch_size
    
        # === 步骤1：将输入映射到 tanh 空间 ===
        # C&W 使用 tanh 变换约束输出在合法像素/坐标范围内（如 [0,1]）
        # 公式反推：x = boxmul * tanh(w) + boxplus，其中 w 是优化变量
        imgs = np.arctanh((imgs - self.boxplus) / self.boxmul * 0.999999)
    
        # === 初始化二分搜索边界 ===
        lower_bound = np.zeros(batch_size)          # 每个样本的 c 下界
        CONST = np.ones(batch_size) * self.initial_const  # 当前轮次使用的权重 c
        upper_bound = np.ones(batch_size) * 1e10    # 每个样本的 c 上界
    
        # === 全局最优解记录（跨所有 outer_step）===
        o_bestl2 = [1e10] * batch_size              # 最小 L2 距离（全局）
        o_bestscore = [-1] * batch_size             # 对应的模型预测结果（-1 表示尚未成功）
        o_bestattack = [np.zeros(imgs[0].shape)] * batch_size  # 对抗样本本身
    
        # === 外层循环：二分搜索最优权重 c ===
        for outer_step in range(self.BINARY_SEARCH_STEPS):
            print("当前全局最优 L2 距离:", o_bestl2)
    
            # 重置优化器（如 Adam）的状态，避免历史梯度干扰
            self.sess.run(self.init)
    
            # 获取当前 batch 数据（支持动态 batch）
            batch = imgs[:batch_size]
            batchlab = labs[:batch_size]
    
            # === 局部最优解记录（仅限当前 c 值下的 inner loop）===
            bestl2 = [1e10] * batch_size
            bestscore = [-1] * batch_size
    
            # 特殊处理：最后一轮使用上界 c 再尝试一次（提高成功率）
            if self.repeat and outer_step == self.BINARY_SEARCH_STEPS - 1:
                CONST = upper_bound
    
            # 将当前 batch、标签、c 值注入计算图
            self.sess.run(self.setup, {
                self.assign_timg: batch,
                self.assign_tlab: batchlab,
                self.assign_const: CONST
            })
    
            prev_loss = np.inf  # 用于早停判断
    
            # === 内层循环：固定 c，用梯度下降优化对抗样本 ===
            for iteration in range(self.MAX_ITERATIONS):
                # 执行一次优化步，返回：
                # - train: 优化操作（如 Adam 更新）
                # - loss: 总损失 c*L_adv + L_dist
                # - l2s: 当前扰动的 L2 范数
                # - scores: 模型 logits 输出
                # - nimg: 当前对抗样本（已从 tanh 空间映射回原始空间）
                _, total_loss, l2s, scores, nimg = self.sess.run([
                    self.train, self.loss, self.l2dist, self.output, self.newimg
                ])
    
                # === 安全检查：确保输入的是 logits（而非 softmax 概率）===
                if (np.all(scores >= -0.0001) and np.all(scores <= 1.0001) and
                    np.allclose(np.sum(scores, axis=1), 1.0, atol=1e-3)):
                    if not self.I_KNOW_WHAT_I_AM_DOING_AND_WANT_TO_OVERRIDE_THE_PRESOFTMAX_CHECK:
                        raise Exception(
                            "模型输出应为 softmax 前的 logits！\n"
                            "当前输出看起来是概率分布（post-softmax）。若确定要这样，请设置：\n"
                            "attack.I_KNOW_WHAT_I_AM_DOING_AND_WANT_TO_OVERRIDE_THE_PRESOFTMAX_CHECK = True"
                        )
    
                # 每 10% 的迭代打印一次损失（调试用）
                if iteration % (self.MAX_ITERATIONS // 10) == 0:
                    loss1, loss2 = self.sess.run([self.loss1, self.loss2])  # L_adv 和 L_dist
                    print(f"迭代 {iteration}: 总损失={total_loss:.4f}, 对抗损失={loss1:.4f}, 距离损失={loss2:.4f}")
    
                # === 早停机制：若损失不再显著下降，则提前终止 ===
                if self.ABORT_EARLY and iteration % (self.MAX_ITERATIONS // 10) == 0:
                    if total_loss > prev_loss * 0.9999:
                        break
                    prev_loss = total_loss
    
                # === 更新局部与全局最优解 ===
                for e, (l2, sc, ii) in enumerate(zip(l2s, scores, nimg)):
                    current_label = np.argmax(batchlab[e])
                    is_success = compare(sc, current_label)
    
                    # 更新局部最优（当前 c 下的最佳）
                    if l2 < bestl2[e] and is_success:
                        bestl2[e] = l2
                        bestscore[e] = np.argmax(sc)
    
                    # 更新全局最优（所有 c 中的最佳）
                    if l2 < o_bestl2[e] and is_success:
                        o_bestl2[e] = l2
                        o_bestscore[e] = np.argmax(sc)
                        o_bestattack[e] = ii
    
            # === 根据本轮攻击结果更新二分搜索区间 ===
            for e in range(batch_size):
                # 判断本轮是否成功（注意：bestscore[e] == -1 表示从未成功）
                attack_succeeded = (bestscore[e] != -1)
    
                if attack_succeeded:
                    # 攻击成功：尝试减小 c（追求更小扰动）
                    upper_bound[e] = min(upper_bound[e], CONST[e])
                    if upper_bound[e] < 1e9:
                        # 已有有效上界，进行标准二分
                        CONST[e] = (lower_bound[e] + upper_bound[e]) / 2
                    # 否则保持 CONST 不变（但上界已更新）
                else:
                    # 攻击失败：需增大 c（放宽扰动限制）
                    lower_bound[e] = max(lower_bound[e], CONST[e])
                    if upper_bound[e] < 1e9:
                        # 已知上界，进行二分
                        CONST[e] = (lower_bound[e] + upper_bound[e]) / 2
                    else:
                        # 尚未找到成功案例，指数级放大 c
                        CONST[e] *= 10
    
        # === 返回最终结果 ===
        o_bestl2 = np.array(o_bestl2)
        return o_bestattack
    ```
## 总结
C&W Attack 作为一种经典的对抗攻击方法，通过将对抗样本的生成问题建模为双目标优化问题，并引入超参数搜索技术，有效地平衡了对抗性和不可感知性之间的关系。本文从算法设计的角度详细剖析了 C&W Attack 的设计思想和代码实现，旨在帮助读者清晰理解其核心思想和实现细节。后续我将从算法框架的角度分析 C&W Attack 在点云对抗领域的应用，并介绍一些常见的改进。
