---
title: "Optimising and Evaluating LLM Knowledge Retrieval: GraphRAG vs Vector RAG on AWS"
date: 2026-05-27 00:00:00 +0200
image: /assets/img/posts/vector-vs-graph-rag/graph-fireworks.png
description: |
      Have you picked a RAG architecture and never looked back? Maybe you should. We benchmarked vector, graph, and hybrid retrieval on AWS and found that how you evaluate matters as much as what you choose.
categories: [AI & ML]
tags: [aws, rag, graphrag, neptune, vector-database]
---

*Co-written with Michelangelo Markus.*

![graph structure firework](/assets/img/posts/vector-vs-graph-rag/graph-fireworks.png)

## What Led Us to GraphRAG

We recently had a customer whose data resembled a dense web of relationships and before I knew it, I was several tabs deep into a GraphRAG rabbit hole. Thankfully, I met my colleague Michelangelo on the same trip and we decided to team up for a little experiment.

## Vector Database vs Knowledge Graph: When Does It Matter?

[Retrieval-Augmented Generation (RAG)](/posts/rag-with-bedrock/) is great for providing LLMs with access to external knowledge, but it can struggle when information is not directly related to a single entity. Here is an example of what I mean: You might ask "How old is Jack Black?" to a standard vector database backed RAG system storing musician specific data and it should provide a pretty solid answer. But in some cases you might want to ask "Which band features both Jack Black and Kyle Gass?". This is more of a graph question because it requires the system to understand the relationships between entities (in this case, Jack Black and Kyle Gass) and how they connect to other entities, i.e. they are both "members of" Tenacious D. Here, "members of" is a relationship type in the graph. It's completely possible to answer this question with a vector database, but it might not be as efficient or accurate as using a graph database that is designed to encode and store relationships.

## Does GraphRAG Actually Outperform Vector Search?

This is all nice in theory but if you are going to make business decisions that require significant investment in terms of time and resources, at your company, you want to be sure that the benefits are truly there. How slow is slow? How fast is fast? Does anyone really notice if your RAG system is a little bit less accurate? How do you even measure accuracy of a RAG system? Optimising is all well and good but if the benefits are marginal, it might not be worth the effort. But maybe it's not marginal. We wanted to see if GraphRAG would blow the vector database out of the water, so we devised a little test case.

## Our Quick Test: Vector, Graph, and Hybrid RAG on AWS

We pit three different ways of fetching information against each other to see which produces the best answers. We had a standard vector database approach using [S3 Vectors](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors.html) as our knowledgebase, a graph database approach using the [serverless variant of AWS Neptune](https://docs.aws.amazon.com/neptune/latest/userguide/neptune-serverless.html) to store entities and relationships, and a hybrid approach combining both. We sourced the data itself and the test data from the [HotpotQA dataset on HuggingFace](https://huggingface.co/datasets/hotpotqa/hotpot_qa), which had 200 questions (166 bridge questions and 34 comparison questions, so the comparison numbers come with bigger error bars). We ran these questions as queries through all three approaches and checked to see if the paragraphs that were relevant for the question were retrieved from the databases (relevant paragraphs are pre-tagged in the dataset). A completely successful retrieval would be marked as 1 if all relevant paragraphs were retrieved and 0 otherwise. We also had an LLM judge to rate the quality of the answers on a scale of 1 to 5 and checked whether the correct answer string actually appeared in the final LLM response (exact match), both of which turned out to be less useful than the retrieval performance metric.

Excuse the brief lab coat interlude, but for the record, we expected the graph database to outperform the vector database in terms of retrieval, especially on questions that required reasoning across multiple entities (these were called 'bridge questions' and were tagged as such). We also figured that vector and graph backed retrieval would be fairly similar for other standard single entity fact-like questions (which were also tagged). Since the hybrid approach had access to both types of information, we expected it to perform the best overall, and so it should. If you're paying for requests and data for both databases, you want to be sure you're getting the best of both worlds.

**Note:** obviously if you run a hybrid approach, you're paying for both databases and the extra complexity to handle the infrastructure, though admittedly the complexity is pretty low, since they are serverless/managed services.

**Another Note:** whatever results we get out of this can depend on a lot of factors. The final message of this post isn't "oh look, obviously graphRAG will always be better for this type of question". What we're saying is "evaluating your database and LLM retrieval performance is important, here are some ways you can do that".

## What We Got Wrong Setting Up AWS Neptune

### Ingesting Data into AWS Neptune: The Bulk Loading Approach

We had some fun initially ingesting the relationship data into Neptune. Using the API to create the graph was leading to a lot of timeouts and errors. We made some gradual improvements but we eventually gave up and went for the bulk loading approach. This went something like this:

- We extracted paragraphs from the hotpotqa dataset
- Classified entities and a predefined set of relationship types via LLM.
- Converted the output into edge and node CSV files
- Uploaded the files to S3
- Used the [Neptune bulk loader](https://docs.aws.amazon.com/neptune/latest/userguide/bulk-load.html) to ingest the data from S3 into Neptune

### Formatting Node and Edge CSV Files for Neptune

The most difficult part of the above process was ensuring we got the entities and relationships correctly formatted for the graph. Typically you wouldn't use an LLM to extract this information, but rather a type of model that is trained specifically to complete this task. Since we were using an LLM, we had to ensure we structured the output consistently to produce a valid CSV file. This involved prompting the LLM carefully and iterating on the output until it met our requirements. Once it did, the bulk loading process was pretty smooth and much faster than trying to use the API directly. This makes sense since this is what the bulk loader is designed for.

## Benchmark Results: Vector vs Graph vs Hybrid RAG

After a little stats wizardry to confirm the differences weren't just noise (linear and logistic regressions, for the curious), here's what we found for retrieval performance. The percentages below are full recall — the proportion of questions where the system successfully retrieved *all* of the relevant paragraphs needed to answer it:

- On bridge questions: Graph (69.9%) ≈ Hybrid (71.1%) > Vector (52.4%)
- On comparison questions: Vector (97.1%) ≈ Hybrid (97.1%) > Graph (91.2%)

Graph and hybrid both outperformed vector on the bridge questions, and were roughly on a par with each other. On comparison questions, vector and hybrid were essentially identical, with graph slightly behind but not dramatically so.

So if your queries are mostly relationship-rich, graph or hybrid both look like reasonable choices. If they're mostly fact-based, vector holds its own and costs less.

End of the story right?

Well not exactly...

First, the LLM judge ratings and exact match metrics showed only marginal differences, hybrid came out slightly ahead, but the gaps were small enough that retrieval performance remained the more meaningful signal. We were rather interested in the retrieval metric since this is a direct measure of the ability to fetch relevant information from the database for a question, but it's clear that optimising the LLM performance is important to ensure that the retrieved information is being used effectively to generate accurate and relevant responses. Takeaway, you should be evaluating both your retrieval performance and your LLM performance to get a complete picture of how well your RAG system is working for the end user.

Second, the LLM related metrics are affected by typical LLM issues, the hotpotqa dataset is wikipedia based and the LLM has likely seen the same or similar data during training, so it might be able to answer some of the questions without needing to rely on the retrieved information. We tried to mitigate this via prompt engineering to encourage the LLM to use the retrieved information, and cite it, but it would be easier to test with proprietary information that the LLM is unlikely to have seen during training.

## When to Use Vector Search, GraphRAG, or Hybrid

**Stick with a vector database if:**
- Your queries are mostly fact-based lookups about a single entity
- You're early-stage and want lower cost and operational simplicity
- Your current retrieval accuracy is already acceptable

**Consider a graph database (AWS Neptune or similar) if:**
- Your queries are relationship-rich — connecting multiple entities, navigating dependencies, reasoning across relationships
- Your domain data is naturally graph-shaped: org charts, product dependencies, compliance rules, supply chains
- In our test, graph performed on a par with hybrid for bridge questions, so it may be worth trying before jumping straight to hybrid

**Go hybrid if:**
- You have a genuine mix of fact-based and relationship-rich queries
- You want the safest bet across both query types, and you're comfortable with the added cost of running two managed services

**A note on cost:** [S3 Vectors](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors.html) is genuinely pay-per-use (~$0.06/GB/month, ~$2.50 per million queries), so it's cheap at low volumes. [Neptune Serverless](https://docs.aws.amazon.com/neptune/latest/userguide/neptune-serverless.html) bills per NCU-second with a minimum of 1 NCU, so there's a floor cost even when idle. Hybrid means paying both. If cost is a constraint and your queries are mostly fact-based, S3 Vectors is the cheaper default. If your data is relationship-rich, the retrieval gains from Neptune might end up justifying the floor cost, but measure first on your own queries before committing. Use the [AWS Pricing Calculator](https://calculator.aws/) to model your specific workload.

In all cases: measure the performance. If you don't have an evaluation set for your queries, it's all guesswork.

## How to Start Evaluating Your RAG System

Don't choose your RAG architecture by instinct, measure it. Build a small evaluation set from your actual query distribution, benchmark retrieval performance, and let the data drive the decision.
