---
title: "Grounding LLMs in reality with Amazon Bedrock"
date: 2023-12-07 00:00:00 +0200
image: /assets/img/posts/rag-with-bedrock/reading-robot.png
description: |
  A look at how we can use features of Amazon Bedrock to ground LLMs in reality and make them more reliable for real-world applications.
categories: [AI & ML]
tags: [aws, bedrock, rag, llm, hallucination]
---

![Robot reading a book](/assets/img/posts/rag-with-bedrock/reading-robot.png)

## Confabulation, Hallucination, and LLMs

In a dementia care unit, a cheerful guy smiles at a room full of students as he regales us with tales of his life. He's confident, he's funny, and he's lying. He's not trying to impress or deceive us, he's just filling in the gaps in his memory with whatever seems to make sense. This is a phenomenon known as [confabulation](https://en.wikipedia.org/wiki/Confabulation) and can occur particularly in people with [Alzheimer's](https://en.wikipedia.org/wiki/Alzheimer%27s_disease) disease and [Wernicke-Korsakoff's](https://en.wikipedia.org/wiki/Wernicke%E2%80%93Korsakoff_syndrome) syndrome. It's _also_ a problem in the behaviour of large language models (LLMs).

If you've been playing around with LLMs like [chatGPT](https://openai.com/blog/chatgpt) or [Amazon Q](https://aws.amazon.com/q/), you've probably already noticed that they can excel at producing a convincing text in response to a question, but sometimes that response is just indisputably wrong. LLMs have been known to make up facts, references, DOIs of journal articles that don't exist, and even entire scientific fields. In an LLM context, this tends to be referred to as ['hallucination'](https://en.wikipedia.org/wiki/Hallucination_(artificial_intelligence)). Hallucination is a problem for LLMs because it means that they can't be trusted to produce reliable information. If an LLM is asked by a customer about a particular product, you don't want it to inform the customer of features of the product that don't exist.

## Retrieval Augmented Generation

One way to address the problem of hallucination is to implement [retrieval augmented generation (RAG)](https://docs.aws.amazon.com/sagemaker/latest/dg/jumpstart-foundation-models-customize-rag.html). RAG models are a combination of an LLM and a knowledge base of information that can supplement the LLM's output. Think of it like an open textbook exam. To reduce the chance that an LLM makes up product features, we might ensure that it has access to a knowledge base of products and their actual features. When a user asks about a product, a RAG model architecture would compare a vector transformed user query with the existing vectors in a [vector database](https://www.pinecone.io/learn/vector-database/). The original user prompt would then be appended with relevant context from the documents within the knowledge base, that are considered semantically similar to the initial query. This augmented prompt is then sent to the LLM that can use the information in its response. In this way, a knowledge base can increase the chance that an LLM produces more reliable responses that an unaugmented model.

## Amazon Bedrock Knowledge Bases and Agents

Amazon Bedrock includes services that allow you to implement RAG. There are three Bedrock features that are key for this:

- Foundational models
- Knowledge bases
- Agents

### Foundational Models

[Bedrock foundational models](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) are pre-trained models from various providers (e.g. [Amazon](https://aws.amazon.com/bedrock/titan/), [Meta](https://ai.meta.com/llama/), [Anthropic](https://www.anthropic.com/product)), that are made available as a service by AWS, and can be accessed via API calls. Typically these models are trained on general language data, for example, a large number of wikipedia articles. Hence their _general_ knowledge.

### Knowledge Bases

A knowledge base is a collection of documents that can provide more specific information on a particular topic of choice to a model. In the context of Amazon Bedrock, there are several potential document databases that can act as the basis for a knowledge base. The first is [AWS Opensearch](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/what-is.html), which is a fully managed search engine service, that can behave as a vector database via its knn-plugin. There's a brief explanation of how that works [here](https://aws.amazon.com/blogs/big-data/amazon-opensearch-services-vector-database-capabilities-explained/).

In Bedrock it's possible to upload text data to an [S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/GetStartedWithS3.html), tell the Bedrock [knowledge base service](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html) where to find the data, and it will handle the rest, dividing the data into chunks and then placing it in a serverless Opensearch collection for you. Alternatively you can bring your own existing OpenSearch, [Redis](https://redis.com/solutions/use-cases/vector-database/), or [Pinecone](https://www.pinecone.io/) vector database, and configure the knowledge base service to use these instead.

### Bedrock Agents

[Bedrock agents](https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html) are the final piece of the puzzle. They are the interface between the user, the knowledge base, and the LLM. You can provide a Bedrock agent with instructions on its role and how it should behave with users. For example, you might specify how the agent should behave if it's not confident in its response. Maybe you want it to clearly indicate to the user that it's not sure of the answer, or for it to ask further questions to the user if specific information is missing.

We might tell our agent to behave like this:

`"You are sales agent for NewCompany, you sell tech products. Help the customer find the right NewCompany product for them. Ask the user questions about their needs and preferences until you have enough information to recommend a product. If you are unsure of the answer to a question, inform the user that you are unsure and ask them to clarify."`

Importantly, you can tell your agent _when_ it should look in the knowledge-base for information to supplement its general knowledge. For instance, you might tell it to look in the knowledge base if you know that information on a topic is not publically available or is highly specific.

Here are some sample instructions for our agent:

`"Look in the knowledge base to help know which products are available for you to recommend and which features those products have. Do not recommend the user products that are not in the knowledge base."`

With these restrictions and instructions in place, your LLM should respond with more realistic and relevant information for its users, and be less likely to hallucinate 🥳

---

**Note:** Bedrock agents have another very cool feature. They can perform actions on behalf of users such as booking appointments or ordering products, via interaction with lambda functions. This allows them to behave like a personal assistant. We won't cover this feature in this article, but you can read more about it [here](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-setup.html).

---

## How to implement RAG with Bedrock

In its simplest form you could click this together in around 10 minutes in the AWS console (CDK constructs are not yet available for Bedrock)

There are just a few prerequisites:

1. You have access to services in N. Virginia (us-east-1)
1. You have unlocked model access for the foundational models you want to use (we used Claude 2.0)
1. You have some text data for your knowledge base

Once you've satisfied those requirements, there are only a few things to configure.

1. First create an S3 bucket and place your text data inside it

1. Then create a knowledge base in the Bedrock console and point it to your S3 bucket

1. Next Sync your knowledge base with your S3 bucket, so that the latest data is available

1. Now create an Agent in the Bedrock console and configure it to use your knowledge base as a data source.

1. Make sure to tell the agent how to behave based on your use case

1. Provide clear instructions to the agent on when it should look at the knowledge base for information

1. You should be able to test your agent out in the console from this point on. Try out different prompts to see how its behaviour changes.

For debugging or tweaking your model, there is an [event tracing](https://docs.aws.amazon.com/bedrock/latest/userguide/trace-events.html) feature that allows you to see what the agent is doing at each step.

## Things to consider

RAG can be a nice way to deal with hallucinating LLMs, but it does have some drawbacks. The knowledge base needs to be carefully curated and maintained, and it needs to be updated regularly to ensure that your LLM does not have access to old irrelevant data. Size is also a consideration. It needs to be large enough to cover the topics your LLM should be knowledgeable about, but not so large that it's difficult to search. Finally, it is worth considering the privacy implications of using a knowledge base. If you're using a knowledge base that contains personal information or information that is not publically available, you need to ensure that you set up the appropriate access controls.

## Honourable mentions

Bedrock currently provides a fast and simple way to implement RAG workloads, but there are other services that can offer similar functionality with more control. Obviously, this is the classic tradeoff between control and convenience that will depend heavily on your use case. If you're looking for more control, you might want to consider the following:

[Amazon Kendra](https://aws.amazon.com/kendra/features/) is an ML powered search service from AWS that has built-in features to perform as part of a RAG workload. It can boost the relevance of your most important documents, filter returning context and has a specific retriever API that is designed to be used with RAG architectures. It also offers a confidence bucket in which it indicates how certain it is that a returned document is relevant.

The Sagemaker service also has a lot to offer. For more information on Sagemaker you can take a look at [this article here](/posts/generative-ai-on-aws/) and consider playing around with the [sagemaker jumpstart notebooks](https://docs.aws.amazon.com/sagemaker/latest/dg/studio-jumpstart.html).

Finally there are some constructs available for CDK that can help you implement RAG and other generative AI workloads. These are currently experimental but can be found [here](https://github.com/awslabs/generative-ai-cdk-constructs) if you are interested in trying them out.

## Conclusion

RAG in Bedrock is a great way to start tackling the problem of hallucination in LLMs. It's easy to implement and can be a good way to familiarise yourself with AWS's generative AI features. Of course, this isn't a magic bullet, there are several other approaches to hallucination reduction that you might want to consider. For example, you could try [fine-tuning your LLM](https://aws.amazon.com/blogs/machine-learning/domain-adaptation-fine-tuning-of-foundation-models-in-amazon-sagemaker-jumpstart-on-financial-data/) on a specific domain, or implement a more complex RAG architecture. But if you're looking for a quick and easy way to get started, RAG with Bedrock is a nice introduction.
