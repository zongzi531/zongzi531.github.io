---
title: Sentence Transformers 使用介绍
date: 2024-07-05 09:17:30
categories: "NLP"
comments: true
featured_image: nlp.png
tags:
- embeddings
- tutorial
- framework
---

<!-- no node -->

<!-- more -->

> 官网 [SBERT.net](https://www.sbert.net/index.html)

有关于 Sentence Transformers 是什么，其官网都有介绍，那本篇博客将简单介绍此框架该如何使用。

Sentence Transformers 框架支持直接使用模型，也就是 **Inference** 。

另外可以使用此框架对模型进行微调（**fine-tuning**），从而训练（**Training**）属于自己的模型，当然我目前还没操作过。

安装教程请参照官网，这里接直接进入使用阶段。

```python
# 引用框架及工具
from sentence_transformers import SentenceTransformer, util

# 设置一些语句用于匹配
sentences = ["That is a happy dog", "That is a very happy person", "Today is a sunny day"]

# 加载模型，这里使用的是官方模型，首次运行程序会从官网下载
model = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')

# 这是我想进行匹配的语句，从上面设置的语句中寻找最相似的
question = "That is a happy person"
# 先将我想匹配的语句解析成向量数据
question_embeddings = model.encode(question)

# 可以尝试打印出来看一下
# print(question_embeddings)
# 可以打印该向量数据的维度
# print(question_embeddings.shape)

print("question is: {}".format(question))

for sentence in sentences:
  # 对上面设置的语句们，解析成向量数据
  embeddings = model.encode(sentence)
  # 使用官方提供的数据计算两个向量的相似度
  sym = round(float(util.pytorch_cos_sim(question_embeddings, embeddings).detach().numpy()[0][0]),5)
  print("{} -> {}".format(sentence, sym))
```

运行这段 Python 脚本可以看到如下输出结果：

```bash
question is: That is a happy person
That is a happy dog -> 0.69458
That is a very happy person -> 0.94292
Today is a sunny day -> 0.25688
```

看效果还不错吧，这时候是不是想试试看中文。我们可以使用官网 [Multilingual Models](https://www.sbert.net/docs/sentence_transformer/pretrained_models.html#multilingual-models) 下的官方模型使用。

对中文都有所支持，我们调整一下 `sentences` 和 `question` 的内容，我这边使用的是 `distiluse-base-multilingual-cased-v1` 模型来运行，我们来看一下运行结果：

```bash
question is: 我喜欢吃苹果
我喜欢吃香蕉 -> 0.79271
我最喜欢苹果 -> 0.91433
今天买了一个苹果 -> 0.61951
我拥有一个苹果 -> 0.71284
儿子喜欢吃苹果 -> 0.73278
```

看到这个输出值，让人有点感觉“舒适”，在这个结果上，我们调整一下问题，在问题上加一个 `不`：

```bash
question is: 我不喜欢吃苹果
我喜欢吃香蕉 -> 0.69195
我最喜欢苹果 -> 0.82793
今天买了一个苹果 -> 0.58365
我拥有一个苹果 -> 0.64693
儿子喜欢吃苹果 -> 0.674
```

所以要想提升这个准确度，可能就是需要我们自己对模型进的微调了吧……

当然其实也可以借助大语言模型对整个工作流进行优化，以约束这里匹配出来的结果。

官网也有一个示例集 [examples](https://github.com/UKPLab/sentence-transformers/blob/master/examples) ，可以在里面学习不少。

到这里，其实已经对 Sentence Transformers 框架在 Python 下的使用已经有了一个简单的认识，至于怎么结合自己的工作流，就要看自己的业务场景了。