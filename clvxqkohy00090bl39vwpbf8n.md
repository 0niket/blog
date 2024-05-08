---
title: "What the RAG"
datePublished: Wed May 08 2024 11:28:37 GMT+0000 (Coordinated Universal Time)
cuid: clvxqkohy00090bl39vwpbf8n
slug: what-the-rag
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/6nuz52vsbWc/upload/4a5308ec7df10f59dd27d4c081822b55.jpeg
tags: llm, rag

---

Retrieval Augmented Generation is particularly useful when you need Language Model to produce answers based on documents other than the ones that Language Model was trained on.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715166622517/d450cf38-296f-4a15-affa-53a7727d9667.png align="center")

Building RAG application can be divided into two main parts

1. Indexing documents
    
2. Retrieval and Generation
    

## Indexing

Indexing is a prerequisite for the retrieval and generation step. At the indexing step, large documents are split into chunks, embedded using vector embedding, and stored in a vector store.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715166650481/06c263eb-664a-4993-a25e-2cf6be5c3271.png align="center")

Let’s step back. Why do we need a vector store? What is vector embedding?

Vector store stores data in numerical format. A string like **“foo bar”** would be converted to numerical format like **\[0.04, -1.2, 0.10\]**. This conversion operation is known as vector embedding. Once data is in vector format, vector operations can be performed to do a similarity search to get the required documents. Various operations could be performed to get the similarity search results like KNN, ANN, and Cosine similarity.

Before embedding to vector format, it is important to do splitting & chunking of documents. By splitting large documents into smaller ones, it is easier to provide context to LMs within the context boundary. While chunking some overlap with the previous text maintains the context. Ultimately chunks will be embedded and indexed in vector store.

Let’s look at the following example of indexing blog post in vector store.

```python
loader = WebBaseLoader(
    web_paths=("<https://lilianweng.github.io/posts/2023-06-23-agent/>",),
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer(
            class_=("post-content", "post-title", "post-header")
        )
    ),
)
docs = loader.load()

text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
splits = text_splitter.split_documents(docs)

vectorstore = Chroma.from_documents(documents=splits, embedding=OpenAIEmbeddings())
```

`WebBaseLoader` - Language Model based applications often require loading data from databases, files, or web. `WebBaseLoader` converts the HTML content into text format to form a document.

`bs4` - BeautifulSoup is HTML parser used for scraping information from web pages. In the above example, we are only interested in scraping information from the web page where div has classes `post-content`, `post-title`, `post-header`.

`RecursiveCharacterTextSplitter` - Language Models have limitations on the context window. The loaded document has over `42K` characters. This is large enough to fit into the context window of many LMs and also to bring attention to the right part of the document. To overcome this issue, the document is split and chunked to form smaller documents. In above example, the chunk size is 1000 characters with an overlap of 200 characters. Overlap helps LMs get the surrounding context from the chunked document. As a result, we have `66` chunked documents.

`OpenAIEmbeddings` - Converts text documents to vector embeddings as explained above. Important point to note here is that there are several vector embedding models. We have chosen OpenAI’s embedding model. This means we have to use the same model for embedding questions so that the similarity search works correctly.

`Chroma` - Open source vector database.

## Retrieval and Generation

Once documents are indexed, we can start asking questions to get the relevant documents back with the help of similarity search. Relevant documents are passed as context along with the question in prompt to LLM.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715166774689/9e38cf74-c2dc-4b15-977f-b8f8b2caa77e.png align="center")

This is how the prompt template is designed to fill in question & retrieved documents.

```plaintext
You are an assistant for question-answering tasks. Use the following pieces
of retrieved context to answer the question. If you don't know the answer,
just say that you don't know. Use three sentences maximum and keep the answer
concise.

Question: {question} 

Context: {context} 

Answer:
```

Let’s look at the code example:

```python
llm = ChatOpenAI(model="gpt-3.5-turbo-0125")

# Retrieve and generate using the relevant snippets of the blog.
retriever = vectorstore.as_retriever()
prompt = hub.pull("rlm/rag-prompt")

def format_docs(docs):
    return "\\n\\n".join(doc.page_content for doc in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
rag_chain.invoke("What is Task Decomposition?")
```

`vectorstore.as_retriever()` creates a retriever object to fetch relevant documents from the vector store.

`hub.pull("rlm/rag-prompt")` fetches the prompt template mentioned above from [prompt hub](https://smith.langchain.com/hub/rlm/rag-prompt?organizationId=9e705fd3-d139-5836-83e7-126db7f45f9e).

Lastly putting it all together

```python
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
```

Chain retrieves relevant documents, formats them, constructs a prompt, passes that to a model, and parses the output. The chaining is defined using Lang Chain Expression Language. It allows us to chain different LangChain components and functions.

`RunnablePassthrough` - in simple terms, it is an identity function returning question.

`StrOutputParser` parses output returned from Language Models.

Lastly, we ask the question `rag_chain.invoke("What is Task Decomposition?")` to get the answer from LLM and it is:

```plaintext
Task decomposition is a technique that breaks down complex tasks into smaller
and simpler steps. It allows models to think step by step and transform
big tasks into manageable components. Different approaches like Chain of Thought
and Tree of Thoughts help in decomposing tasks effectively.
```

## Debugging RAG applications

[LangSmith](https://www.langchain.com/langsmith) is the debugging tool for LangChain-based applications. Let’s walk through the chaining.

```python
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
```

First is the `Retriever` . It retrieves the documents that are relevant to the asked question. In this case, we have got 4 documents.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715166859605/7f7bc86a-4ef6-4480-8dab-c9d132bfb131.png align="center")

On the first line of the chain, we are creating dict of context and question.

```python
{"context": retriever | format_docs, "question": RunnablePassthrough()}
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715166887428/4d439eed-7bfd-4e53-8514-4db44679e8c7.png align="center")

On the second line, this dict is passed through the prompt template to get the prompt that needs to be passed to the LLM.

Input:

```json
{
  "context": "Fig. 1. Overview of a LLM-powered autonomous agent system.\\nComponent One: Planning#\\nA complicated task usually involves many steps. An agent needs to know what they are and plan ahead.\\nTask Decomposition#\\nChain of thought (CoT; Wei et al. 2022) has become a standard prompting technique for enhancing model performance on complex tasks. The model is instructed to “think step by step” to utilize more test-time computation to decompose hard tasks into smaller and simpler steps. CoT transforms big tasks into multiple manageable tasks and shed lights into an interpretation of the model’s thinking process.\\n\\nFig. 1. Overview of a LLM-powered autonomous agent system.\\nComponent One: Planning#\\nA complicated task usually involves many steps. An agent needs to know what they are and plan ahead.\\nTask Decomposition#\\nChain of thought (CoT; Wei et al. 2022) has become a standard prompting technique for enhancing model performance on complex tasks. The model is instructed to “think step by step” to utilize more test-time computation to decompose hard tasks into smaller and simpler steps. CoT transforms big tasks into multiple manageable tasks and shed lights into an interpretation of the model’s thinking process.\\n\\nTree of Thoughts (Yao et al. 2023) extends CoT by exploring multiple reasoning possibilities at each step. It first decomposes the problem into multiple thought steps and generates multiple thoughts per step, creating a tree structure. The search process can be BFS (breadth-first search) or DFS (depth-first search) with each state evaluated by a classifier (via a prompt) or majority vote.\\nTask decomposition can be done (1) by LLM with simple prompting like \\"Steps for XYZ.\\\\n1.\\", \\"What are the subgoals for achieving XYZ?\\", (2) by using task-specific instructions; e.g. \\"Write a story outline.\\" for writing a novel, or (3) with human inputs.\\n\\nTree of Thoughts (Yao et al. 2023) extends CoT by exploring multiple reasoning possibilities at each step. It first decomposes the problem into multiple thought steps and generates multiple thoughts per step, creating a tree structure. The search process can be BFS (breadth-first search) or DFS (depth-first search) with each state evaluated by a classifier (via a prompt) or majority vote.\\nTask decomposition can be done (1) by LLM with simple prompting like \\"Steps for XYZ.\\\\n1.\\", \\"What are the subgoals for achieving XYZ?\\", (2) by using task-specific instructions; e.g. \\"Write a story outline.\\" for writing a novel, or (3) with human inputs.",
  "question": "What is Task Decomposition?"
}
```

Output:

```json
{
  "output": {
    "messages": [
      {
        "content": "You are an assistant for question-answering tasks. Use the following pieces of retrieved context to answer the question. If you don't know the answer, just say that you don't know. Use three sentences maximum and keep the answer concise.\\nQuestion: What is Task Decomposition? \\nContext: Fig. 1. Overview of a LLM-powered autonomous agent system.\\nComponent One: Planning#\\nA complicated task usually involves many steps. An agent needs to know what they are and plan ahead.\\nTask Decomposition#\\nChain of thought (CoT; Wei et al. 2022) has become a standard prompting technique for enhancing model performance on complex tasks. The model is instructed to “think step by step” to utilize more test-time computation to decompose hard tasks into smaller and simpler steps. CoT transforms big tasks into multiple manageable tasks and shed lights into an interpretation of the model’s thinking process.\\n\\nFig. 1. Overview of a LLM-powered autonomous agent system.\\nComponent One: Planning#\\nA complicated task usually involves many steps. An agent needs to know what they are and plan ahead.\\nTask Decomposition#\\nChain of thought (CoT; Wei et al. 2022) has become a standard prompting technique for enhancing model performance on complex tasks. The model is instructed to “think step by step” to utilize more test-time computation to decompose hard tasks into smaller and simpler steps. CoT transforms big tasks into multiple manageable tasks and shed lights into an interpretation of the model’s thinking process.\\n\\nTree of Thoughts (Yao et al. 2023) extends CoT by exploring multiple reasoning possibilities at each step. It first decomposes the problem into multiple thought steps and generates multiple thoughts per step, creating a tree structure. The search process can be BFS (breadth-first search) or DFS (depth-first search) with each state evaluated by a classifier (via a prompt) or majority vote.\\nTask decomposition can be done (1) by LLM with simple prompting like \\"Steps for XYZ.\\\\n1.\\", \\"What are the subgoals for achieving XYZ?\\", (2) by using task-specific instructions; e.g. \\"Write a story outline.\\" for writing a novel, or (3) with human inputs.\\n\\nTree of Thoughts (Yao et al. 2023) extends CoT by exploring multiple reasoning possibilities at each step. It first decomposes the problem into multiple thought steps and generates multiple thoughts per step, creating a tree structure. The search process can be BFS (breadth-first search) or DFS (depth-first search) with each state evaluated by a classifier (via a prompt) or majority vote.\\nTask decomposition can be done (1) by LLM with simple prompting like \\"Steps for XYZ.\\\\n1.\\", \\"What are the subgoals for achieving XYZ?\\", (2) by using task-specific instructions; e.g. \\"Write a story outline.\\" for writing a novel, or (3) with human inputs. \\nAnswer:",
        "additional_kwargs": {},
        "response_metadata": {},
        "type": "human",
        "example": false
      }
    ]
  }
}
```

On the third line, LLM produces the output based on the prompt

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715166984315/0fa1c8fd-47e7-46f9-bd01-954d7e6981d0.png align="center")

Input:

```plaintext
You are an assistant for question-answering tasks. Use the following pieces of retrieved context to answer the question. If you don't know the answer, just say that you don't know. Use three sentences maximum and keep the answer concise.
Question: What is Task Decomposition? 
Context: Fig. 1. Overview of a LLM-powered autonomous agent system.
Component One: Planning#
A complicated task usually involves many steps. An agent needs to know what they are and plan ahead.
Task Decomposition#
Chain of thought (CoT; Wei et al. 2022) has become a standard prompting technique for enhancing model performance on complex tasks. The model is instructed to “think step by step” to utilize more test-time computation to decompose hard tasks into smaller and simpler steps. CoT transforms big tasks into multiple manageable tasks and shed lights into an interpretation of the model’s thinking process.

Fig. 1. Overview of a LLM-powered autonomous agent system.
Component One: Planning#
A complicated task usually involves many steps. An agent needs to know what they are and plan ahead.
Task Decomposition#
Chain of thought (CoT; Wei et al. 2022) has become a standard prompting technique for enhancing model performance on complex tasks. The model is instructed to “think step by step” to utilize more test-time computation to decompose hard tasks into smaller and simpler steps. CoT transforms big tasks into multiple manageable tasks and shed lights into an interpretation of the model’s thinking process.

Tree of Thoughts (Yao et al. 2023) extends CoT by exploring multiple reasoning possibilities at each step. It first decomposes the problem into multiple thought steps and generates multiple thoughts per step, creating a tree structure. The search process can be BFS (breadth-first search) or DFS (depth-first search) with each state evaluated by a classifier (via a prompt) or majority vote.
Task decomposition can be done (1) by LLM with simple prompting like "Steps for XYZ.\\n1.", "What are the subgoals for achieving XYZ?", (2) by using task-specific instructions; e.g. "Write a story outline." for writing a novel, or (3) with human inputs.

Tree of Thoughts (Yao et al. 2023) extends CoT by exploring multiple reasoning possibilities at each step. It first decomposes the problem into multiple thought steps and generates multiple thoughts per step, creating a tree structure. The search process can be BFS (breadth-first search) or DFS (depth-first search) with each state evaluated by a classifier (via a prompt) or majority vote.
Task decomposition can be done (1) by LLM with simple prompting like "Steps for XYZ.\\n1.", "What are the subgoals for achieving XYZ?", (2) by using task-specific instructions; e.g. "Write a story outline." for writing a novel, or (3) with human inputs. 
Answer:
```

Output:

```plaintext
Task decomposition is a technique that breaks down complex tasks into smaller
and simpler steps. It allows models to think step by step and transform
big tasks into manageable components. Different approaches like Chain of Thought
and Tree of Thoughts help in decomposing tasks effectively.
```

## References

Jupyter Notebook: [https://github.com/0niket/RAG\_from\_scratch/blob/main/rag\_overview.ipynb](https://github.com/0niket/RAG_from_scratch/blob/main/rag_overview.ipynb)