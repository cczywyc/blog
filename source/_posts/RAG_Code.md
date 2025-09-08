---
title: "RAG（下）"
description: "一个建议的RAG系统的代码实现"
date: 2025-05-29
tags:
  - AI
categories:
  - AI文章合集
cover: https://img.cczywyc.com/post-cover/rag_code_cover.jpg
---

[上一篇文章](https://cczywyc.com/2025/05/25/RAG/)是关于 RAG 的一些基本介绍，这篇主要是根据上篇的流程来实现一个简易的 RAG 系统，进一步加深对 RAG 工作流程的理解。本文所列举的示例代码非常简单，完全是按照上篇文章中所划分的三个阶段来的，但是包含了 RAG 系统所有的必要步骤。



## 数据源加载与分块

```python
from typing import List

def split_into_chunks(doc_file: str) -> List[str]:
    with open(doc_file, 'r') as file:
        content = file.read()
    return [chunk for chunk in content.split("\n\n")]

chunks = split_into_chunks("doc.md")

for i, chunk in enumerate(chunks):
    print(f"[{i}] {chunk}\n")
```

我们上篇说到，分块的方式有很多种，这里随便选取一种，按照换行符来给数据源分块。

## 文本的向量化

```python
from sentence_transformers import SentenceTransformer

embedding_model = SentenceTransformer("shibing624/text2vec-base-chinese")

def embed_chunk(chunk: str) -> List[float]:
    embedding = embedding_model.encode(chunk, normalize_embeddings=True)
    return embedding.tolist()

embedding = embed_chunk("测试内容")
print(len(embedding))
print(embedding)
```

1. 代码首先从 sentence-transformers 库导入 SentenceTransformer 类，并加载一个预训练好的中文文本向量化模型 shibing624/text2vec-base-chinese。这个模型擅长将中文文本转换成能表达其语义的数字向量。

2. embed_chunk 函数接收一个文本块，使用加载的模型将其编码（encode）成一个向量。normalize_embeddings=True 参数会将向量进行归一化，这有助于后续的相似度计算。
3. 最后测试打印一下向量化，运行可以看到将测试内容转化成高维（本例是 768）向量。

下面便是调用向量化方法，将第一步加载分块的数据源全部向量化。

```python
embeddings = [embed_chunk(chunk) for chunk in chunks]

print(len(embeddings))
print(embeddings[0])
```

## 索引与存储

```python
import chromadb

chromadb_client = chromadb.EphemeralClient()
chromadb_collection = chromadb_client.get_or_create_collection(name="default")

def save_embeddings(chunks: List[str], embeddings: List[List[float]]) -> None:
    for i, (chunk, embedding) in enumerate(zip(chunks, embeddings)):
        chromadb_collection.add(
            documents=[chunk],
            embeddings=[embedding],
            ids=[str(i)]
        )

save_embeddings(chunks, embeddings)
```

1. 示例中使用的向量数据库是 chromadb
2. 为了演示方便，chromadb.EphemeralClient() 创建了一个临时的、在内存中运行的向量数据库实例。
3. get_or_create_collection 创建了一个名为 "default" 的集合（类似于数据库中的表），用于存放我们的数据。
4. save_embeddings 函数遍历文本块和它们对应的向量，并将它们成对地添加到 chromadb_collection 中。每个条目都包含三部分：
    - documents: 原始的文本内容
    - embeddings: 文本对应的向量
    - ids: 每个条目的唯一标识符，这里使用其在列表中的索引

## 检索与召回

```python
def retrieve(query: str, top_k: int) -> List[str]:
    query_embedding = embed_chunk(query)
    results = chromadb_collection.query(
        query_embeddings=[query_embedding],
        n_results=top_k
    )
    return results['documents'][0]

query = "哆啦A梦使用的3个秘密道具分别是什么？"
retrieved_chunks = retrieve(query, 5)

for i, chunk in enumerate(retrieved_chunks):
    print(f"[{i}] {chunk}\n")
```

1. retrieve 函数接收一个用户问题 query 和一个整数 top_k（代表需要检索的数量，这里是 5）。

2. 它首先将用户问题也转换成一个向量（使用与之前完全相同的模型）。
3. 然后，它使用 chromadb_collection.query 方法，用问题的向量去数据库中搜索最相似的 top_k 个向量。
4. 数据库返回最相似的条目，函数从中提取出原始的文本文档并返回。
5. 代码最后用一个具体问题 "哆啦A梦使用的 3 个秘密道具分别是什么？" 来测试检索功能，并打印出找到的前 5 个最相关的文本块。

## 重排

```python
from sentence_transformers import CrossEncoder

def rerank(query: str, retrieved_chunks: List[str], top_k: int) -> List[str]:
    cross_encoder = CrossEncoder('cross-encoder/mmarco-mMiniLMv2-L12-H384-v1')
    pairs = [(query, chunk) for chunk in retrieved_chunks]
    scores = cross_encoder.predict(pairs)

    scored_chunks = list(zip(retrieved_chunks, scores))
    scored_chunks.sort(key=lambda x: x[1], reverse=True)

    return [chunk for chunk, _ in scored_chunks][:top_k]

reranked_chunks = rerank(query, retrieved_chunks, 3)

for i, chunk in enumerate(reranked_chunks):
    print(f"[{i}] {chunk}\n")
```

1. 此处引入了一个 CrossEncoder 模型。与 SentenceTransformer（将单个句子转为向量）不同，CrossEncoder 直接比较一对文本（在这里是 (query, chunk)），并输出一个它们之间的相关性得分。

2. rerank 函数将用户问题与上一步检索到的每个文本块配对。
3. cross_encoder.predict 为每一对计算一个精确的相关性分数。
4. 代码根据这个分数从高到低对检索到的文本块进行重新排序，并返回排序后最靠前的 top_k 个。

上篇文章我们讲了一下重排的作用和目的，这里再次重述一下：重排可以显著提高 RAG 系统的质量，在召回阶段召回的结果可能会包含一些不那么相关的片段，重排序则使用一个更复杂、更精确的模型（CrossEncoder）对初步结果进行精细打磨，确保最终提供给语言模型的上下文质量最高、相关性最强。

## 生成

```python
from dotenv import load_dotenv
from google import genai

load_dotenv()
google_client = genai.Client()

def generate(query: str, chunks: List[str]) -> str:
    prompt = f"""你是一位知识助手，请根据用户的问题和下列片段生成准确的回答。

用户问题: {query}

相关片段:
{"\n\n".join(chunks)}

请基于上述内容作答，不要编造信息。"""

    print(f"{prompt}\n\n---\n")

    response = google_client.models.generate_content(
        model="gemini-2.5-flash",
        contents=prompt
    )

    return response.text

answer = generate(query, reranked_chunks)
print(answer)
```

最后一步就比较好理解了，将问题和重排后的相关片段作为上下文丢给大模型，然后调用 LLM 大模型接口生成答案，代码也比较简洁易懂，这里就不做描述。



以上就利用 python + langchain 实现了一个非常简易的 RAG 系统，主要是针对上一篇文章中三个阶段的代码实现，详细的代码运行结果和数据源文本见[本项目地址](https://github.com/cczywyc/rag-py-eg)，运行前项目跟目录下创建一个 .env 文件，将 gemini 的 api key 填进去即可，像下面这样。

```tex
GEMINI_API_KEY=xxxxxxxxxxxxx<替换成你的API_KEY>
```
