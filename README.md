本文是对近期llm的入门学习总结，也是想帮助其他刚接触llm的新手玩家，快速入门。本文只涉及模型处理流程中的算法和模型架构，不涉及模型的应用。

# 1 需要学习的零件
- tokenization，使用Tokenizer将连续的文本序列转换为离散的token
- embedding，本质是一个查找表。首先将词汇表映射到一个'd'维的特征空间，然后通过token对应的index来查找'd'维词向量，每个维代表一个词特征
- position encoding，模型需要词的位置信息，就像人类阅读文本，需要获取上下文关系，因此需要一种编码把token的位置信息传递给模型
- transformer，把'embedding'和'position encoding'以某种方式整合后，作为transformer的input，经过一些列的'Layer Norm','Self-Attention','MLP'和'Softmax'...后为词汇表中的每个词生成一个分数(该词是下一个输出token的概率)。这些分数有一个特殊的名称：'logits'，可以通过直接取最大值和采样的方法输出最终的token

# 2 分词器Tokenizer
![Tokenize](img/Tokenizer.jpg)
使用模型前，需要使用tokenizer处理输入的文本序列，输出是一系列的token序列。tokenizer总体上做三件事情：
- 分词，tokenizer将字符串分为一些sub-word token字符串，再将token 字符串映射到index，并保留该映射关系的mapping。从字符串到index的encode过程和从index到字符串的decode过程都需要该mapping
- 扩展词汇表
- 识别并处理特殊token  

这里着重介绍'分词'。
## 2.1 word-based按word拆分句子的古典分词器
![word_based](img/word_based.png)
将一个word作为最小单元，也就是根据空格或者标点、语法等分词，局限性：  
- 对于未在词表中出现的词out of vocabulary，OOV,模型将无法处理，未知符号标记为[UNK]
- 词表中的低频/稀疏词在模型训练无法得到训练 
- 很多语言难以用空格进行分词

## 2.2 char-based按单个字符拆分句子的方法
![char_based](img/char_based.png)
更为极端的分词方法，直接把序列分成一个一个的字母和特殊符号。虽然能解决OOV问题，也避免了大词汇量问题，但缺点明显：  
- 粒度太细 
- 训练花费成本太高，由于字符数量太小，为每个字符学习嵌入向量的时候，每个向量就容纳了太多的语义在内，学习起来非常困难

## 2.3 subword-based改进的分词器方法
![char_based](img/subword.png)
基于子词的分词方法（Subword Tokenization），遵循“尽量不分解常用词，将不常用词分解为常用的子词”的原则。例如"unfriendly"在英文中是un+副词的组合，表否定的意思，可以分解成un”+"friendly"。"friendly"又可以分为名词+ly的形式。通过这种方法，词汇量大小不会特别大，也能学到词的关系，在能缓解oov问题同时还尽可以将结果中token数目降到最低。  

主流的subword算法有：
- Byte Pair Encoding, BPE，被GPT族的模型广泛使用。核心思想在于将最常出现的子词对合并，直到词汇表达到预定的大小时停止。 
- WordPiece，一种字词粒度的tokenize算法，很多transformer模型都使用，如BERT等。原理和BPE非常相似，不同之处在于它做合并时，并不是直接找最高频的组合，而是找能够最大化训练数据似然的来合并，即它每次合并的两个字符A和B，应该具有最大的P(AB)/P(A)P(B)值。
- Unigram，与WordPiece一样，Unigram Language Model(ULM)同样使用语言模型来挑选子词。不同之处在于，BPE和WordPiece算法的词表大小都是从小到大变化，属于增量法。而ULM则是减量法,即先初始化一个大词表，根据评估准则不断丢弃词表，直到满足限定条件。ULM算法考虑了句子的不同分词可能，因而能够输出带概率的多个子词分段。 
- SentencePiece，谷歌推出的子词开源工具包，其中集成了BPE、ULM子词算法。除此之外，SentencePiece还能支持字符和词级别的分词。SentencePiece主要为多语言模型设计，做了以下两个主要的转化： 1.以unicode方式编码字符，将所有的输入(英文、中文等不同语言)都转化为unicode字符，解决了多语言编码方式不同的问题;2. 将空格编码为'_'，这也是为了能处理多语言问题，比如英文解码时有空格，而中文没有。

## 2.4 Byte-Pair Encoding: Subword-based tokenization algorithm
由于比较流行的模型架构都使用了BPE分词法，比如从GPT-2开始一直到GPT-4，OpenAI一直采用BPE分词法，llama架构的模型也是使用BPE分词法。所以这里举一个例子来详细介绍"最先进的 NLP 模型使用的基于子词的标记化算法 - 字节对编码 (BPE)"。  

实际上，BPE 是一种简单形式的数据压缩算法，把序列中最常见的一对儿连续数据字节替换为该序列数据中不存在的字节。举个简单例子：
- 假设数据 'aaabdaaabac' 需要编码（压缩）。因为字节pair 'aa' 最常出现，将其替换为 Z，因为 Z 现有的数据中不存在。现在有了 ZabdZabac，其中 Z = aa。
![char_based](img/bpe_e_1.jpg)
- 下一个最常出现字节pair是 ab，因此将其替换为 Y。现在有 ZYdZYac，其中 Z = aa 且 Y = ab。
![char_based](img/bpe_e_2.jpg)
- 剩下的唯一字节pair是 ac，但当前序列中仅有一个ac，因此不会对其进行编码。但此时可以使用递归字节对编码将 ZY 编码为 X。数据现在已转换为 XdXac，其中 X = ZY、Y = ab、Z = aa。此时无法进一步压缩，因为没有字节对出现多次。
![char_based](img/bpe_e_3.jpg)
- 相反，通过按相反顺序执行替换就可解压缩数据。  

以上是基本的BPE算法的流程，NLP 中使用了这种算法的变体，具体例子如下：  

### Step 1. 准备数据
- 假设现在有一个语料库，其中包含单词——old、old、finest 和 lowest，假设这些单词在语料库中出现的频率如下：  
```{“old”: 7, “older”: 3, “finest”: 9, “lowest”: 4}```
- 每个单词的末尾添加一个特殊的结束标记“<\/w>”：  
```{“old</w>”: 7, “older</w>”: 3, “finest</w>”: 9, “lowest</w>”: 4}```  
每个单词末尾添加“<\/w>”标记来标识单词边界，以便算法知道每个单词的结束位置。这有助于算法查看每个字符并找到频率最高的字符配对

### Step 2. 开始迭代
![char_based](img/bpe_1.webp)  
首先，将每个单词拆分为字符并计算它们的出现次数。初始标记将是所有字符和“<\/w>”token。  

BPE 算法的下一步是寻找最频繁的字节对(在这里，将字符视为与字节相同。这是英语的情况，在其他语言中可能会有所不同)，合并它们，并一次又一次地执行相同的迭代，直到达到token数量限制或迭代限制  

所以现在，对于这个例子，将合并最常见的'字符对'以形成一个token，并将该token添加到token列表中，并重新计算每个token的频率。这意味着频率计数将在每个合并步骤后发生变化。然后继续执行此合并步骤，直到达到迭代次数或达到token列表的数量限制。

- Iteration 1: 从除<\/w>特殊token外的第二个最常出现的token “e” 开始，带有字符“e”的最常出现字符对是<e, s>，分别在“finest”和“lowest”中出现9次和4次，共13次。因此，将它们合并形成一个新的token “es”，并记下其频率为 13。同时，把token “e”和“s”的出现次数减去 13。更行后的token列表如下：  
![char_based](img/bpe_2.webp)

- Iteration 2: 合并token “es”和“t”。因为，字符对<es, t>在语料库中出现了 13 次(最高)。因此，产生一个频率为 13 的新token “est”，同时把“es”和“t”的频率减去 13。更行后的token列表如下：  
![char_based](img/bpe_3.webp)

- Iteration 3: 合并字符对<est, <\/w>>，因为“est<\/w>”在预料库中出现了13次(最高)。因此，产生一个频率为 13 的新token “est<\/w>”，同时把“est”和“<\/w>”的频率减去 13。更行后的token列表如下：  
![char_based](img/bpe_4.webp)
合并停止标记“<\/w>”非常重要。这有助于算法理解“estimate”和“highest”等词之间的区别。这两个词都有一个共同点“est”，但highest以“est”结尾，estimate以“est”开头。因此，像“est”和“est<\/w>”这样的token将以不同的方式处理。如果算法看到标记“est<\/w>”，它将知道它是单词“high”的token，而不是单词“estate”的token。

- Iteration 4: 现在，语料库中的字符对<l, o>出现次数最多，old中7次，older中3次，共10次。同理，更新后的token列表如下：  
![char_based](img/bpe_5.webp)


- Iteration 5: 然后合并出现10次的字符对<lo, d>，更新后的token列表如下：  
![char_based](img/bpe_6.webp)

### Step 3. 停止迭代
5次迭代后，语料库中剩余没合并的字符集为：```{“</w>”: 7, “er</w>”: 3, “fin”: 9, “low”: 4}```，同时从token列表中删去频率为0的token后，现在的token列表为：  
![char_based](img/bpe_7.webp)

此时，“f”、“i”和“n”的频率是 9，“o”、“l”、“w”的频率是 4，“e”、“r”的频率是 3，但只有一个单词包含这些字符，因此不会继续将它们合并。现在可以看到总token数为 11，这比初始的 12 少1(因为这是一个很小的语料库，但在实践中，大小应该会减很多)。这个包含 11 个标记的列表将作为最终的词汇表，上图中第一列即为对应token的index。

- decode容易，示例：首先根据模型的输出(token对应的index)，映射成到具体的编码序列```[“the</w>”, “high”, “est</w>”, “range</w>”, “in</w>”, “Seattle</w>”]```将被解码为``` [“the”, “highest”, “range”, “in”, “Seattle”]```
- encode难，计算成本很高。假设单词序列为 ```[“the</w>”, “highest</w>”, “range</w>”, “in</w>”, “Seattle</w>”]```。我们将迭代token列表中所有token（从最长到最短），并尝试使用这些token替换给定单词序列中的sub字符串。如果仍有一些sub字符串没被替换（对于模型在训练中没有看到的单词），将用 [UNK] token替换。最后查找token和index的mapping，得到一个数字序列，型如[8774, 1150, 55, 1],作为Embedding层的输入。  
实际会得到一个4*V的矩阵，V是词表的大小，比如51200。矩阵第一行的第8774列为1,其他全为0。矩阵的第二行的第1150列为1,其他全为0。矩阵的第三行的第55列为1,其他全为0。矩阵的第四行的第1列为1,其他全为0。

# 3 Embedding
词向量，英文名叫Word Embedding，按照字面意思，应该是词嵌入。假如一个模型的Embedding层维度是d, 那么对于token序列如[8774, 1150, 55, 1]来说，会通过“查表”的方式得到一个d×4的嵌入矩阵，来表示模型的输入，矩阵的每一列代表一个token的高维词向量。  
实际上Embedding层是以 'one hot' 为输入、中间层节点为词向量维数的全连接层！而这个全连接层的参数，就是一个“词向量表”。  
要想明白Word Embedding到底是怎么一回事，还得先了解一下'one hot'。  

## 3.1 one hot
one hot，中文可以翻译为“独热”，是最原始的用来表示字、词的方式。为了简单，本文以word为例。假如词表中一共有“I、love、you、dog、are、eat”6个word，one hot就是给这六个word分别用一个0-1编码：
```
- you : [1,0,0,0,0,0]
- love: [0,1,0,0,0,0]
- I   : [0,0,1,0,0,0]
- dog : [0,0,0,1,0,0]
- are : [0,0,0,0,1,0]
- eat : [0,0,0,0,0,1]
```
那么，如果表示“I love dog”就可以用矩阵
```
- I   : [0,0,1,0,0,0]
- love: [0,1,0,0,0,0]
- dog : [0,0,0,1,0,0]
```
可以看到，one hot的方式下，有多少个字，就得有多少维向量，假如有1万字，那么每个字向量就是1万维。但有点是这样的矩阵参与计算是十分好计算

## 3.2 Embedding层真面目
考虑以下计算：  

![char_based](img/embedding.jpg)

左边的形式表明，这是一个以2x6的 one hot 矩阵的为输入(来表示'you love')、中间层节点数为3(或者说嵌入层的维度)的全连接神经网络层。而右边，就是相当于在w矩阵中取出第一行和第二行，这就是为何说”Embedding矩阵的本质就是一个查找表“：可以通过一个one hot向量取出v×d矩阵中的第i行，v是词表大小，d是嵌入层的维度，i是该one hot向量为1的index。因此使用一个seq_len长度的序列，会得到一个seq_len×d的嵌入矩阵，该矩阵的没一行代表一个d维的词向量。

可以看到，实际上Embedding层就是以one hot为输入、中间层节点为词向量维数的全连接层。而这个全连接层的参数，就是一个“词向量表(上图中的w矩阵)”！从这个层面来看，词向量没有做任何事情！它就是一种one hot，只不过词向量是one hot的全连接层的参数而已。 

- 运算层面的，基本上就是通过研究发现，one hot型的矩阵相乘，就像是相当于查表，于是它直接用查表作为操作，而不写成矩阵再运算，这大大降低了运算量(降低了运算量不是因为词向量的出现，而是因为把one hot型的矩阵运算简化为了查表操作)。
- 思想层面的，就是它得到了这个全连接层的参数之后，直接用这个全连接层的参数作为特征，或者说，用这个全连接层的参数作为字、词的表示，从而得到了字、词向量，最后还发现了一些有趣的性质，比如向量的夹角余弦能够在某种程度上表示字、词的相似度。

# 4 position encoding