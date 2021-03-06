---
title: Word Embedding 梳理
date: 2017-07-31 20:32:55
tags: [Word2Vec,  Word Embedding, One-Hot, VSM]
categories: Feeling
---


## 语言模型


### 统计语言模型

NLP 中的一个关键概念描述清楚：语言模型。语言模型包括文法语言模型和统计语言模型。一般我们指的是统计语言模型。

- Wiki: 

    A statistical language model is a probability distribution over sequences of words. Given such a sequence, say of length m, it assigns a probability to the whole sequence: $$ P(w_{1},\ldots ,w_{m})$$ 

    Having a way to estimate the relative likelihood of different phrases is useful in many natural language processing applications, especially ones that generate text as an output. 

    Language modeling is used in speech recognition, machine translation, part-of-speech tagging, parsing, handwriting recognition, information retrieval and other applications.

- 中文：

    统计语言模型： 统计语言模型把语言（词的序列）看作一个随机事件，并赋予相应的概率来描述其属于某种语言集合的可能性。给定一个词汇集合 V，对于一个由 V 中的词构成的序列 `S = (w_1, ... , w_T) ∈ Vn`，统计语言模型赋予这个序列一个概率 P(S)，来衡量 S 符合自然语言的语法和语义规则的置信度。

    用一句简单的话说，就语言模型就是计算一个句子的概率大小的这种模型。有什么意义呢？一个句子的打分概率越高，越说明他是更合乎人说出来的自然句子。 

### N-gram

常见的统计语言模型有 N 元文法模型（N-gram Model），最常见的是 unigram model、bigram model、trigram model 等等。形式化讲，统计语言模型的作用是为一个长度为 m 的字符串确定一个概率分布 P(w1; w2; …… wm)，表示其存在的可能性，其中 w1 到 wm 依次表示这段文本中的各个词。一般在实际求解过程中，通常采用下式计算其概率值：

$$P(w_1,w_2,...,w_n) = \prod_{i}P(w_i|w_1w_2...w_{i-1}) ≈ \prod_{i}P(w_i|w{i-k}...w_{i-1})$$ 

简单的例子：

$$P(w_1, w_2, w_3) = P(w_1, w_2) * P(w_3|w_1, w_2) = P(w_1) * P(w_2|w_1) * P(w_3|w_1,w_2)$$ 

同时通过这些方法均也可以保留住一定的词序信息，这样就能把一个词的上下文信息 capture 住。

### 应用

- 机器翻译：`P(high winds tonight) > P(large winds tonight)`
- 拼写纠错：`P(about fifteen minutes from) > P(about fifteenminuets from)`
- 语音识别：`P(I saw a van) >> P(eyes awe of an)`
- 音字转换：`P(你现在干什么|nixianzaiganshenme) > P(你西安在干什么|nixianzaiganshenme)`
- 自动文摘
- 问答系统
... ...

### 局限性

数据中有很多的 p(一|今天，星期), p(二|今天，星期)，数据中没有 p(一|明天，星期), p(二|明天，星期)，数据中是可以知道【今天】【明天】两个词很像，但无法判断两个词很像（判断方法是看他们有没有类似的【上下文】）。基于统计的 N-gram Language Model 没有办法【迁移】这种知识。


### 原因

Ngram 本质上是将词当做一个个孤立的原子单元（atomic unit）去处理的。这种处理方式对应到数学上的形式是一个个离散的 one-hot 向量（除了一个词典索引的下标对应的方向上是 1，其余方向上都是 0）。例如，对于一个大小为 5 的词典：{"I", "love", "nature", "luaguage", "processing"}，“nature” 对应的 one-hot 向量为：[0,0,1,0,0]。显然，one-hot 向量的维度等于词典的大小。这在动辄上万甚至百万词典的实际应用中，面临着巨大的维度灾难问题（the curse of dimensionality）。 

自然而然人们就想到，能否用一个连续的稠密向量去刻画一个 word 的特征呢？这样，我们不仅可以直接刻画词与词之间的相似度，还可以建立一个从向量到概率的平滑函数模型，使得相似的词向量可以映射到相近的概率空间上。这个稠密连续向量也被称为 word 的 distributed representation（分布式表示）。有点风险平摊的感觉。

## 词的表示


### One-hot

NLP 中最直观，也是到目前为止最常用的词表示方法是 One-hot Representation，这种方法把每个词表示为一个很长的向量。这个向量的维度是词表大小，其中绝大多数元素为 0，只有一个维度的值为 1，这个维度就代表了当前的词。 

举个栗子：“话筒”表示为 [0 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 …] ，“麦克”表示为 [0 0 0 0 0 0 0 0 1 0 0 0 0 0 0 0 …] ，每个词都是茫茫 0 海中的一个 1。 这种 One-hot Representation 如果采用稀疏方式存储，会是非常的简洁：也就是给每个词分配一个数字 ID。比如刚才的例子中，话筒记为 3，麦克记为 8（假设从 0 开始记）。如果要编程实现的话，用 Hash 表给每个词分配一个编号就可以了。这么简洁的表示方法配合上最大熵、SVM、CRF 等等算法已经很好地完成了 NLP 领域的各种主流任务。 

当然这种表示方法也存在一个重要的问题就是“词汇鸿沟”现象：任意两个词之间都是孤立的。光从这两个向量中看不出两个词是否有关系，哪怕是话筒和麦克这样的同义词也不能幸免于难。

### 分布式 Distributed Representation 

思想：如果两个词的上下文相同，那么两个词所表达的语义也是一样的。 

#### 向量空间模型 VSM 

事实上，这个概念在信息检索（Information Retrieval）领域早就已经被广泛地使用了。只不过，在 IR 领域里，这个概念被称为向量空间模型（Vector Space Model，以下简称 VSM ）。

VSM 是基于一种 Statistical Semantics Hypothesis：语言的统计特征隐藏着语义的信息（Statistical pattern of human word usage can be used to figure out what people mean）。 

例如，两篇具有相似词分布的文档可以被认为是有着相近的主题。这个 Hypothesis有 很多衍生版本。 

其中，比较广为人知的两个版本是 Bag of Words Hypothesis 和 Distributional Hypothesis。前者是说，一篇文档的词频（而不是词序）代表了文档的主题；后者是说，上下文环境相似的两个词有着相近的语义。后面我们会看到，word2vec算法也是基于 Distributional 的假设。

那么，VSM 是如何将稀疏离散的 one-hot 词向量映射为稠密连续的 **distributional representation** 的呢？

简单来说，基于 Bag of Words Hypothesis，我们可以构造一个 term-document 矩阵 A：矩阵的行 `A_{i,:}` 对应着词典里的一个 word；矩阵的列 `A_{:,j}` 对应着训练语料里的一篇文档；矩阵里的元素 `A_{i,j}` 代表着 word wi 在文档 Dj 中出现的次数（或频率）。那么，我们就可以提取行向量做为 word 的语义向量（不过，在实际应用中，我们更多的是用列向量做为文档的主题向量）。

类似地，我们可以基于 Distributional Hypothesis 构造一个 word-context 的矩阵。此时，矩阵的列变成了 context 里的 word，矩阵的元素也变成了一个 context 窗口里 word 的共现次数。

注意，这两类矩阵的行向量所计算的相似度有着细微的差异：term-document 矩阵会给经常出现在同一篇 document 里的两个 word 赋予更高的相似度；而 word-context 矩阵会给那些有着相同 context 的两个 word 赋予更高的相似度。后者相对于前者是一种更高阶的相似度，因此在传统的信息检索领域中得到了更加广泛的应用。

不过，这种 co-occurrence 矩阵仍然存在着数据稀疏性和维度灾难的问题。为此，人们提出了一系列对矩阵进行降维的方法（如LSI／LSA等）。这些方法大都是基于 SVD 的思想，将原始的稀疏矩阵分解为两个低秩矩阵乘积的形式。


#### 计数模型  Latent Semantic Analysis

- LSA: [Latent Semantic Analysis (LSA) 模型 学习笔记 - 推酷](http://www.tuicool.com/articles/ZjUJZbj)
  - 优点： 可以把原文本特征空间降维到一个低维语义空间；减轻一词多义和一义多词问题。
  - 缺点： 在SVD分解的时候，特别耗时，而且一般而言一个文本特征矩阵维数都会特别庞大，SVD此时就更加耗时；而且，LSA缺乏严谨的数理统计基础。
- pLSA: [[学习笔记]学习主题模型(Topic Model)和PLSA( probabilistic latent semantic analysis） - xiaopei的博客 - 博客频道 - CSDN.NET](http://blog.csdn.net/hxxiaopei/article/details/7617838)
- LDA: [LDA漫游指南马晨：LDA漫游指南全文阅读](https://yuedu.baidu.com/ebook/d0b441a8ccbff121dd36839a), LDA 数学八卦


#### 预测模型 Neural Probabilistic Language Models

- 目标是生成语言模型，词向量是副产品；同时得到
- 最早由百度 IDL（深度学习研究院） 的徐伟于 2000 年提出：《Can Artifical Neural Networks Learn Language Models?》提出用神经网络构建二元语言模型的方法
- 经典之作：Bengio 等人在 2001 年发表在 NIPS（Neural Information Processing Systems） 上的文章：《A Neural Probabilistic Language Model》，2003 年投到 JMLR（Journal of Machine Learning Research） 上的同名论文。 

#### 区别

- 计数法是在大型语料中统计词语及邻近的词的共现频率，然后将之为每个词映射为一个稠密的向量表示
- 预测法是直接利用词语的邻近词信息来得到预测词的词向量


## Word Embedding


鉴于 Ngram 等模型的不足，2003 年，Bengio 等人发表了一篇开创性的文章：A neural probabilistic language model。在这篇文章里，他们总结出了一套用神经网络建立统计语言模型的框架（Neural Network Language Model），并首次提出了 word embedding 的概念（虽然没有叫这个名字），从而奠定了包括 word2vec 在内后续研究 word representation learning 的基础。

NNLM模型的基本思想可以概括如下：

- 假定词表中的每一个 word 都对应着一个连续的特征向量；
- 假定一个连续平滑的概率模型，输入一段词向量的序列，可以输出这段序列的联合概率；
- 同时学习词向量的权重和概率模型里的参数。值得注意的一点是，这里的词向量也是要学习的参数。 


### 模型详解

![](http://images2015.cnblogs.com/blog/939075/201607/939075-20160719201106732-581954491.png) 

图中最下方的 $$w_{t-n+1}, …, w_{t-2}, w_{t-1}$$ 
就是前 n-1 个词。现在需要根据这已知的 n-1 个词预测下一个词 `w_t`。

C(w) 表示词 w 所对应的词向量，整个模型中使用的是一套唯一的词向量，存在矩阵 C（一个 |V| × m 的矩阵）中。其中 |V| 表示词表的大小（语料中的总词数），m 表示词向量的维度。w 到 C(w) 的转化就是从矩阵中取出一行。 

网络的第一层（输入层）是将 $$C(w_{t-n+1}), …, C(w_{t-2}), C(w_{t-1})$$ 
这 n-1 个向量首尾相接拼起来，形成一个 (n-1)m 维的向量，下面记为 x。 

网络的第二层（隐藏层）就如同普通的神经网络，直接使用 d+Hx 计算得到。d 是一个偏置项。在此之后，使用 tanh 作为激活函数。 

网络的第三层（输出层）一共有 |V| 个节点，每个节点 yi 表示 下一个词为 i 的未归一化 log 概率。最后使用 softmax 激活函数将输出值 y 归一化成概率。最终，y 的计算公式为：

$$y = b + Wx + U\tanh(d+Hx) $$

`(1×(n-1)*m, (n-1)*m×h, h×V -> 1×V)`

式子中的 U（一个 |V|× h 的矩阵）是隐藏层到输出层的参数，整个模型的多数计算集中在 U 和隐藏层的矩阵乘法中。后文的提到的 3 个工作，都有对这一环节的简化，提升计算的速度。 

式子中还有一个矩阵 W（|V|×(n-1)m），这个矩阵包含了从输入层到输出层的直连边。直连边就是从输入层直接到输出层的一个线性变换，好像也是神经网络中的一种常用技巧（没有仔细考察过）。如果不需要直连边的话，将 W 置为 0 就可以了。在最后的实验中，Bengio 发现直连边虽然不能提升模型效果，但是可以少一半的迭代次数。同时他也猜想如果没有直连边，可能可以生成更好的词向量。 

现在万事俱备，用随机梯度下降法把这个模型优化出来就可以了。需要注意的是，一般神经网络的输入层只是一个输入值，而在这里，输入层 x 也是参数（存在 C 中），也是需要优化的。优化结束之后，词向量有了，语言模型也有了。 

这样得到的语言模型自带平滑，无需传统 n-gram 模型中那些复杂的平滑算法。Bengio 在 APNews 数据集上做的对比实验也表明他的模型效果比精心设计平滑算法的普通 n-gram 算法要好 10% 到 20%。 

计算的瓶颈主要是在 softmax 层的归一化函数上（需要对词典中所有的 word 计算一遍条件概率）。

### 模型分析

然而，抛却复杂的参数空间，我们不禁要问，为什么这样一个简单的模型会取得巨大的成功呢？

仔细观察这个模型就会发现，它其实在同时解决两个问题： 
一个是统计语言模型里关注的条件概率 p(wt|context) 的计算； 
一个是向量空间模型里关注的词向量的表达。 

而这两个问题本质上并不独立： 
通过引入连续的词向量和平滑的概率模型，我们就可以在一个连续空间里对序列概率进行建模，从而从根本上缓解数据稀疏性和维度灾难的问题。 
另一方面，以条件概率 p(wt|context) 为学习目标去更新词向量的权重，具有更强的导向性，同时也与 VSM 里的 Distributional Hypothesis 不谋而合。


## Word2Vec


### NNLM 问题


- 一个问题是，同 Ngram 模型一样，NNLM 模型只能处理定长的序列。在03年的论文里，Bengio等人将模型能够一次处理的序列长度 NN 提高到了 5，虽然相比 bigram 和 trigram 已经是很大的提升，但依然缺少灵活性。 
因此，Mikolov 等人在 2010 年提出了一种 RNNLM 模型，用递归神经网络代替原始模型里的前向反馈神经网络，并将embedding 层与 RNN 里的隐藏层合并，从而解决了变长序列的问题。

- 另一个问题就比较严重了。NNLM 的训练太慢了。即便是在百万量级的数据集上，即便是借助了 40 个 CPU 进行训练，NNLM 也需要耗时数周才能给出一个稍微靠谱的解来。显然，对于现在动辄上千万甚至上亿的真实语料库，训练一个 NNLM 模型几乎是一个 impossible mission。 
这时候，还是那个 Mikolov 站了出来。他注意到，原始的 NNLM 模型的训练其实可以拆分成两个步骤：

  - 用一个简单模型训练出连续的词向量；
  - 基于词向量的表达，训练一个连续的 Ngram 神经网络模型。

而 NNLM 模型的计算瓶颈主要是在第二步。 
如果我们只是想得到 word 的连续特征向量，是不是可以对第二步里的神经网络模型进行简化呢？ 
Mikolov 是这么想的，也是这么做的。他在 2013 年一口气推出了两篇 paper，并开源了一款计算词向量的工具——至此，word2vec 横空出世。

### CBOW & Skip-Gram

对原始的 NNLM 模型做如下改造：

- 移除前向反馈神经网络中非线性的 hidden layer，直接将中间层的 embedding layer 与输出层的 softmax layer 连接；
- 忽略上下文环境的序列信息：输入的所有词向量均汇总到同一个 embedding layer；
- 将 future words 纳入上下文环境

得到的模型称之为 CBOW 模型（Continuous Bag-of-Words Model），也是 word2vec 算法的第一个模型

![](http://images2015.cnblogs.com/blog/939075/201607/939075-20160719201512435-160028706.png)

CBOW 模型等价于一个词袋模型的向量乘以一个 embedding 矩阵，从而得到一个连续的 embedding 向量。这也是 CBOW 模型名称的由来。 
CBOW 模型依然是从 context 对 target word 的预测中学习到词向量的表达。反过来，从 target word 对 context 的预测中学习到 word vector 的方法叫：Skim-Gram


![](http://images2015.cnblogs.com/blog/939075/201607/939075-20160719201532560-2134123571.png) 


Skip-Gram 模型的本质是计算输入 word 的 input vector 与目标 word 的 output vector 之间的余弦相似度，并进行 softmax 归一化。我们要学习的模型参数正是这两类词向量。


### Hierarchical Softmax


层次 Softma 的方法最早由 Bengio 在 05 年引入到语言模型中。它的基本思想是将复杂的归一化概率分解为一系列条件概率乘积的形式。其中，每一层条件概率对应一个二分类问题，可以通过一个简单的逻辑回归函数去拟合。这样，我们将对 V 个词的概率归一化问题，转化成了对 logV 个词的概率拟合问题。 

层次 Softmax 是一个很巧妙的模型。它通过构造一颗二叉树，将目标概率的计算复杂度从最初的 V 降低到了 log⁡V 的量级。不过付出的代价是人为增强了词与词之间的耦合性。例如，一个 word 出现的条件概率的变化，会影响到其路径上所有非叶节点的概率变化，间接地对其他 word 出现的条件概率带来不同程度的影响。因此，构造一颗有意义的二叉树就显得十分重要。实践证明，在实际的应用中，基于Huffman 编码的二叉树可以满足大部分应用场景的需求。 


霍夫曼编码使用变长编码表对源符号（如文件中的一个字母）进行编码，其中变长编码表是通过一种评估来源符号出现概率的方法得到的，出现概率高的字母使用较短的编码，反之出现概率低的则使用较长的编码，这便使编码之后的字符串的平均长度、期望值降低，从而达到无损压缩数据的目的。

### Negative Sampling 

负采样的思想最初来源于一种叫做 Noise-Contrastive Estimation的算法[6]，原本是为了解决那些无法归一化的概率模型的参数预估问题。与改造模型输出概率的层次 Softmax 算法不同，NCE 算法改造的是模型的似然函数。

## 应用

- 机器翻译
- 计算相似度
  - 寻找相似词
  - 信息检索
  - 推荐
- 作为 SVM/LSTM 等模型的输入
  - 中文分词
  - 命名体识别
- 句子表示
  - 情感分析
- 文档表示
  - 文档主题判别

## 参考

- 主要整理自这篇文章，作者特别厉害：[word2vec 前世今生 - 公子天 - 博客园](https://www.cnblogs.com/iloveai/p/word2vec.html)
- [DeepNLP 的表示学习・词嵌入来龙去脉](https://blog.csdn.net/scotfield_msn/article/details/69075227)
- [TensorFlow 学习笔记 3：词向量 | Jey Zhang](http://www.jeyzhang.com/tensorflow-learning-notes-3.html)
- [字词的向量表示法  |  TensorFlow Core  |  TensorFlow](https://www.tensorflow.org/tutorials/representation/word2vec)
- [[NLP] 秒懂词向量 Word2vec 的本质 - 知乎](https://zhuanlan.zhihu.com/p/26306795)