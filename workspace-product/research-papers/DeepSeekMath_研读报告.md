# DeepSeekMath 研读报告

## 核心信息

| 项目 | 内容 |
|---|---|
| **论文标题** | DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models |
| **作者团队** | DeepSeek-AI（联合清华大学、北京大学） |
| **发布平台** | arXiv:2402.03300v3（2024年4月27日） |
| **开源地址** | https://github.com/deepseek-ai/DeepSeek-Math |

---

## Part A: 深度专业学术解析

### 结构化摘要 (Structured Abstract)

| 维度 | 内容 |
|---|---|
| **背景/目标** | 开源大语言模型在数学推理能力上远落后于闭源模型（如GPT-4、 Gemini-Ultra），本研究旨在构建一个接近闭源模型性能的开源数学专用模型 |
| **方法** | （1）从Common Crawl中通过fastText分类器构建120B token的高质量数学预训练语料DeepSeekMath Corpus；（2）基于DeepSeek-Coder-Base-v1.5 7B继续预训练；（3）引入Group Relative Policy Optimization（GRPO）强化学习算法 |
| **结果** | DeepSeekMath 7B在MATH基准上达到51.7%（无外部工具），接近Gemini-Ultra和GPT-4；使用64样本自一致性后提升至60.9% |
| **结论** | 公开可获取的Common Crawl数据蕴含大量数学知识；小模型通过高质量数据可以达到与超大模型相当的数学推理能力；代码预训练有助于数学推理能力提升 |

---

### 1. 引言 (Introduction)

#### 1.1 研究背景与核心问题

大语言模型（LLM）在数学推理领域引发了显著变革，在定量推理基准（如MATH）和几何推理基准（如Trink等人2024年的研究）中均取得了重大进展。然而，GPT-4和Gemini-Ultra等前沿模型并不开源，当前可获取的开源模型在性能上存在显著差距。

**核心研究问题**：如何构建一个开源数学专用模型，使其性能接近闭源前沿模型？

#### 1.2 文献综述与研究缺口

现有工作包括：
- **Minerva系列**（Lewkowycz等，2022a）：在PaLM基础上继续训练数学文本，540B版本在MATH上达33.6%
- **Llemma**（Azerbayev等，2023）：在Proof-Pile-2上训练，34B版本在MATH上达25.3%
- **OpenWebMath**（Paster等，2023）：从Common Crawl过滤出136亿token数学网页

**研究缺口**：
1. 现有数学语料规模有限（最大仅约50B token）
2. 现有语料以英文为主，中文数学能力不足
3. PPO等强化学习算法需要大量计算资源（需要额外的价值网络）

#### 1.3 研究目标与核心贡献

本研究提出DeepSeekMath 7B，包含两个核心贡献：

**贡献一：大规模数学预训练**
- 构建DeepSeekMath Corpus：120B token（约为Minerva使用数学网页的7倍，OpenWebMath的9倍）
- 提出迭代式fastText数据选择pipeline
- 发现代码预训练对数学推理有正向迁移作用

**贡献二：强化学习探索**
- 提出Group Relative Policy Optimization（GRPO）：无需价值网络，显著降低训练资源
- 提供统一范式理解RFT、DPO、PPO、GRPO等方法
- 仅使用GSM8K和MATH的CoT数据，RL阶段即可带来显著提升

---

### 2. 研究设计与方法

#### 2.1 预训练数据构建：DeepSeekMath Corpus

<p align="center">
<img src="https://i.img402.dev/iw5bbdizn4.jpg" width="600"/>
</p>
<p align="center">图1：DeepSeekMath Corpus迭代式构建pipeline（来源：原论文 [Figure 2]）</p>

**数据收集流程**：

1. **种子语料**：以OpenWebMath作为初始种子
2. **fastText分类器训练**：使用500K正例（来自OpenWebMath）和500K负例（来自Common Crawl）
3. **第一轮召回**：从40B HTML网页中召回数学相关内容
4. **迭代优化**：通过分析已召回数据的域名分布，识别数学相关域名（如mathoverflow.net），人工标注URL路径，扩充种子语料
5. **重复迭代**：经过4轮迭代，最终获得35.5M数学网页，共120B token

**关键参数**：向量维度256，学习率0.1，n-gram最大长度3，最小词频3，训练3个epoch

**去污染处理**：过滤掉与GSM8K、MATH、CMATH、AGIEval等基准匹配的10-gram字符串

#### 2.2 模型训练设置

**基座模型初始化**：DeepSeek-Coder-Base-v1.5 7B

**训练数据分布**（500B token总量）：
- 56%：DeepSeekMath Corpus
- 4%：AlgebraicStack（数学代码）
- 10%：arXiv论文
- 20%：GitHub代码
- 10%：中英文自然语言数据

**训练配置**：峰值学习率4.2e-4，batch size 10M tokens，4K上下文长度

#### 2.3 强化学习：GRPO算法

<p align="center">
<img src="https://i.img402.dev/vua7do4guk.png" width="900"/>
</p>
<p align="center">图2：PPO与GRPO对比（来源：原论文 [Figure 4]）</p>

**PPO的核心问题**：
- 需要同时训练策略网络和价值网络，带来大量内存和计算开销
- LLM场景下通常只在最后一个token有奖励信号，价值函数训练困难

**GRPO的创新**：
- 舍弃价值网络，改为使用同一问题采样的多个输出之平均奖励作为baseline
- 对于每个问题q，从旧策略π_old采样G个输出{o₁, o₂, ..., o_G}
- 通过组内相对奖励计算优势函数

**GRPO目标函数**：

$$J_{GRPO}(\theta) = \mathbb{E}\left[q \sim P(Q), \{o_i\}_{i=1}^G \sim \pi_{\theta_{old}}(O|q)\right] \frac{1}{G} \sum_{i=1}^G \frac{1}{|o_i|} \sum_{t=1}^{|o_i|} \min\left[ \frac{\pi_\theta(o_{i,t}|q, o_{i,<t})}{\pi_{\theta_{old}}(o_{i,t}|q, o_{i,<t})} \hat{A}_{i,t}, \text{clip}\left(\frac{\pi_\theta(o_{i,t}|q, o_{i,<t})}{\pi_{\theta_{old}}(o_{i,t}|q, o_{i,<t})}, 1-\varepsilon, 1+\varepsilon\right) \hat{A}_{i,t}\right] - \beta D_{KL}(\pi_\theta || \pi_{ref})$$

**核心优势**：无需训练价值网络，显著降低强化学习的资源门槛

---

### 3. 结果与发现

#### 3.1 预训练语料质量验证

| 语料 | 规模 | GSM8K | MATH | MMLU-STEM | CMATH |
|---|---|---|---|---|---|
| 无数学训练 | - | 2.9% | 3.0% | 19.5% | 12.3% |
| MathPile | 8.9B | 2.7% | 3.3% | 15.7% | 0.0% |
| OpenWebMath | 13.6B | 11.5% | 8.9% | 29.6% | 16.8% |
| Proof-Pile-2 | 51.9B | 14.3% | 11.2% | 29.2% | 5.1% |
| **DeepSeekMath** | **120.2B** | **23.8%** | **13.6%** | **33.1%** | **41.5%** |

<p align="center">表1：不同数学语料训练的DeepSeek-LLM 1.3B性能对比（来源：原论文 [Table 1]）</p>

**关键发现**：
- DeepSeekMath Corpus在所有8个基准上均显著领先
- 学习曲线更陡峭，持续提升能力更强
- 中文数学能力显著提升（CMATH从最高的19.9%提升至41.5%）

#### 3.2 DeepSeekMath-Base 7B 主实验结果

<p align="center">
<img src="https://i.img402.dev/h9vhvv8rud.png" width="900"/>
</p>
<p align="center">图3：开源模型在MATH基准上的Top1准确率对比（来源：原论文 [Figure 1]）</p>

**数学问题求解（Chain-of-Thought）**：

| 模型 | 参数量 | GSM8K | MATH | OCW Courses | SAT |
|---|---|---|---|---|---|
| Minerva 540B（闭源） | 540B | 58.8% | 33.6% | 17.6% | - |
| Mistral 7B | 7B | 40.3% | 14.3% | 9.2% | 71.9% |
| Llemma 34B | 34B | 54.0% | 25.3% | 10.3% | 71.9% |
| **DeepSeekMath-Base** | **7B** | **64.2%** | **36.2%** | **15.4%** | **84.4%** |

<p align="center">表2：Base模型在英文数学基准上的对比（来源：原论文 [Table 2]）</p>

**关键数据**：
- DeepSeekMath-Base 7B在MATH上超越Minerva 540B（36.2% vs 33.6%）
- 参数量仅为Minerva 540B的1/77，但性能更强
- 在中文基准上同样表现优异（CMATH 71.7%，Gaokao-MathQA 35.3%）

**工具集成求解（Program-of-Thought）**：

| 模型 | GSM8K+Python | MATH+Python |
|---|---|---|
| Llemma 34B | 64.6% | 26.3% |
| **DeepSeekMath-Base 7B** | **66.9%** | **31.4%** |

#### 3.3 指令微调与强化学习结果

| 模型 | 尺寸 | GSM8K | MATH | MGSM-zh | CMATH |
|---|---|---|---|---|---|
| GPT-4（闭源） | - | 92.0% | 52.9% | - | 86.0% |
| Gemini Ultra（闭源） | - | 94.4% | 53.2% | - | - |
| **DeepSeekMath-Instruct** | **7B** | **82.9%** | **46.8%** | **73.2%** | **84.6%** |
| **DeepSeekMath-RL** | **7B** | **88.2%** | **51.7%** | **79.6%** | **88.8%** |

<p align="center">表3：指令微调与强化学习后的性能对比（来源：原论文 [Table 5]）</p>

**关键发现**：
- DeepSeekMath-RL在MATH上达到51.7%，首次开源模型超过50%
- 仅使用CoT格式的GSM8K和MATH数据，RL即可带来全面提升
- 自一致性投票（64样本）可进一步提升至60.9%

---

### 4. 讨论

#### 4.1 理论贡献

1. **数据规模与质量的双重重要性**：120B token规模的DeepSeekMath Corpus远超同类数据集，且质量更高（体现在更陡峭的学习曲线上）

2. **参数效率的重新审视**：DeepSeekMath-Base 7B超越Minerva 540B，证明了"小模型+高质量数据"策略的有效性，参数规模并非数学推理能力的唯一决定因素

3. **代码预训练的正向迁移**：从代码训练模型初始化比从通用LLM初始化效果更好，为"代码训练能否提升推理能力"这一长期问题提供了肯定答案

4. **GRPO的统一框架**：将RFT、DPO、PPO、GRPO统一为直接或简化RL技术的不同形态，深化了对LLM对齐技术的理论理解

#### 4.2 实践启示

1. **数据工程的重要性**：精心设计的数据选择pipeline比盲目扩大数据规模更有效
2. **多语言数学语料的价值**：包含中英文等多语言数据可提升模型在非英文基准上的表现
3. **强化学习的资源优化**：GRPO无需价值网络，为资源有限的团队提供了可行的RL方案

#### 4.3 局限性与未来方向

1. **基准污染风险**：尽管进行了去污染处理，但无法完全排除潜在污染
2. **arXiv训练的有限收益**：在论文中声称arXiv训练未带来显著提升，这与常见做法不同，值得进一步研究
3. **GRPO的理论分析**：论文未深入分析为何GRPO中的组相对优势估计比其他方法更有效

---

### 5. 结论

DeepSeekMath通过两个核心创新突破了开源模型在数学推理能力上的瓶颈：

1. **高质量大规模数学语料**：通过迭代式fastText选择pipeline从Common Crawl中构建120B token的DeepSeekMath Corpus，涵盖多语言数学内容
2. **高效的强化学习算法**：GRPO通过组相对优势估计取代价值网络，大幅降低训练资源需求

DeepSeekMath 7B在MATH基准上达到51.7%，首次将开源模型提升至接近闭源前沿模型的水平，证明了数据质量和算法效率在数学推理中的关键作用。

---

### 6. 核心参考文献

1. Lewkowycz et al. (2022a). Minerva: Solving Quantitative Reasoning Problems with Language Models. *NeurIPS 2022*.

2. Azerbayev et al. (2023). Llemma: An Open Language Model For Mathematics. *arXiv:2310.10631*.

3. Paster et al. (2023). OpenWebMath: An Open Dataset of Mathematical Web Content. *NeurIPS 2023*.

4. Schulman et al. (2017). Proximal Policy Optimization Algorithms. *arXiv:1707.06347*.

5. Guo et al. (2024). DeepSeek-Coder: Training a Large Language Model for Code. *arXiv:2401.14196*.

---

## Part B: 核心逻辑链与价值提炼

### 核心四要素

| 要素 | 内容 |
|---|---|
| **根本问题** | 开源大模型在数学推理能力上远落后于GPT-4/Gemini-Ultra等闭源模型，且强化学习训练成本高昂 |
| **切入视角** | 公开可获取的Common Crawl数据中蕴含大量高质量数学知识，通过精心设计的数据选择pipeline可以有效提取；强化学习中的价值网络可以通过组内相对奖励来替代 |
| **关键方法** | （1）迭代式fastText分类器从Common Crawl构建120B token数学语料；（2）Group Relative Policy Optimization（GRPO）无需价值网络的强化学习 |
| **核心发现** | DeepSeekMath 7B在MATH上达到51.7%，首次开源模型超过50%，接近Gemini-Ultra和GPT-4；代码预训练有助于数学推理；GRPO可显著降低RL训练资源 |

---

### 核心逻辑链

```
┌─────────────────────────────────────────────────────┐
│              根本问题 (Problem)                      │
│  开源模型数学推理能力远落后于闭源模型                  │
│  强化学习需要昂贵的价值网络                            │
└─────────────────────┬───────────────────────────────┘
                      │ 切入视角:
                      │ 1. Common Crawl有大量数学知识
                      │ 2. 组内相对奖励可替代价值网络
                      ▼
┌─────────────────────────────────────────────────────┐
│              关键方法 (Method)                        │
│  迭代式fastText数据选择 → 120B数学语料                │
│  GRPO: 无需价值网络的强化学习                         │
└─────────────────────┬───────────────────────────────┘
                      │ 验证:
                      │ 预训练+指令微调+GRPO强化学习
                      ▼
┌─────────────────────────────────────────────────────┐
│              核心发现 (Finding)                       │
│  MATH: 51.7% (无工具) → 60.9% (64样本自一致性)       │
│  超越Minerva 540B，仅用1/77参数                      │
│  首个开源模型突破50% MATH基准                        │
└─────────────────────────────────────────────────────┘
```

---

### 方法公式化

```text
GRPO = (组内采样输出 - 组内平均奖励) × PPO策略梯度 - β×KL(策略||参考策略)

DeepSeekMath = (代码预训练模型初始化 + 120B高质量数学语料) + GRPO强化学习
```

---

### 关键数据快照

| 指标 | 数值 | 意义 |
|---|---|---|
| **MATH基准** | 51.7%（Top1），60.9%（64样本自一致性） | 首个开源模型突破50%，接近Gemini-Ultra和GPT-4 |
| **参数量对比** | 7B vs 540B（Minerva） | 1/77参数实现性能超越 |
| **语料规模** | 120B token（DeepSeekMath） vs 13.6B（OpenWebMath） | 约9倍规模提升 |
| **GSM8K** | 88.2%（RL后） | 相比基座提升24pp |
| **中文CMATH** | 88.8%（RL后） | 多语言数学能力显著 |

---

### 最终双重总结

**一句话总结（核心价值）**：DeepSeekMath通过迭代式数据选择pipeline从Common Crawl构建120B高质量数学语料，并提出无需价值网络的GRPO强化学习算法，使7B参数模型在MATH基准上达到51.7%，首次将开源模型提升至接近GPT-4/Gemini-Ultra等闭源前沿模型的数学推理水平。

**一句话总结（通俗版）**：就像一个学生找到了更好的教材（Common Crawl中精选的数学内容）和更高效的学习方法（GRPO），用更少的"脑细胞"（参数）达到了尖子生的成绩——而且这个方法还开源给所有人用。

---

## 附录：图片索引

- 图1：论文整体架构/开源模型MATH准确率对比 — `fig_p1_xref155_resized.png`
- 图2：DeepSeekMath Corpus迭代式构建pipeline — `fig_p5_xref353_resized.jpg`
- 图3：PPO与GRPO对比示意图 — `fig_p7_xref380_resized.png`

> 本报告基于arXiv:2402.03300v3生成，图片均从原论文PDF中提取原始图片并上传至img402.dev图床，保留7天有效期。
