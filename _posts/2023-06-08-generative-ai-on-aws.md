---
title: "Generative AI: An overview of the landscape and what AWS has to offer"
date: 2023-06-08 00:00:00 +0200
image: /assets/img/posts/placeholder.svg
description: |
  The buzz surrounding generative AI is louder than ever. What is it? Why is it a big deal and what services does AWS offer to help build generative AI solutions?
categories: [AI & ML]
tags: [aws, generative-ai, sagemaker, bedrock, llm]
---

![Brain](/assets/img/posts/generative-ai-on-aws/generative-ai-on-aws.png)

## What is generative AI?

If you've ever used [Github Copilot](https://github.com/features/copilot) to generate code for you, then it's safe to say you've used generative AI. In essence, AI is considered generative when it can create something new (e.g. text, images or sounds), that resembles the information it has been exposed to during its training. In contrast, AI services that would not be considered 'generative' are primarily designed to analyze and interpret existing data, but do not actually create anything new. This distinction seems quite clear cut, but it's likely that the lines between generative and non-generative AI will become increasingly blurred in the future, as one is used to enhance the other.

## Why is generative AI a big deal?

It's likely you've already heard of [ChatGPT](https://openai.com/blog/chatgpt), the generative [large language model (LLM)](https://www.boost.ai/blog/llms-large-language-models) from OpenAI that can write convincing articles, poems and code. Or [DALL-E](https://openai.com/dall-e-2), the model that can create images in a diverse range of styles from text description prompts. These are just two examples of the many generative AI models that are causing a stir lately. So why is this such a big deal? Well, it looks like generative AI has the potential to transform the way we work, play and live.

Right now I'm using Github Copilot and ChatGPT at least to some extent to write this article, I use DALL-E and [Midjourney](https://docs.midjourney.com/) to create character images for RPGs, and I've already used ChatGPT to plan my next holiday, including attractions to visit, activities and how many days to spend in each area. I'm not alone in this, and if generative AI becomes more accessible, it's likely that we'll see it featured more and more in our day-to-day lives.

With this kind of power at our fingertips, the Tech industry is jumping to integrate generative AI into new and existing products. If you're looking to do the same, then you'll be pleased to know that AWS has services to help you get started.

## What services does AWS offer to help build generative AI solutions?

### Amazon SageMaker

[Amazon SageMaker](https://aws.amazon.com/sagemaker/) is a fully managed service that provides developers and data scientist with the ability to build, train, and deploy machine learning (ML) models quickly. Importantly it also provides a number of built-in algorithms, frameworks and services, that can be used to build generative AI solutions.

### Specialised instance types

[Amazon EC2 Trainium](https://aws.amazon.com/machine-learning/trainium/) instances have a custom chip designed to accelerate machine learning training workloads in the cloud. At the time of writing, Trainium Trn1 instances can deliver the most teraflops of any AWS instance, and are cost effective too, with the most teraflops per dollar of any AWS instance. These make them a good option for training LLMS or other large models.

Meanwhile, [Amazon EC2 Inf2 instances](https://aws.amazon.com/ec2/instance-types/inf2/?nc1=h_ls) are optimized to deploy complex models. They allow you to deploy models with hundreds of billions of parameters, across multiple accelerators on Inf2 instances, which is exactly what you need to deploy Generative AI models at scale.

### Deep Learning Containers

Another option offered by AWS are [Deep Learning Containers](https://aws.amazon.com/machine-learning/containers/). These are Docker images pre-installed with deep learning frameworks and tools. They can be used to train and deploy models on services such as Amazon SageMaker, [Amazon Elastic Kubernetes Service (Amazon EKS)](https://aws.amazon.com/eks/), [Amazon Elastic Container Service (Amazon ECS)](https://aws.amazon.com/ecs/?nc1=h_ls), and Amazon Elastic Compute Cloud (Amazon EC2) instances. They are available for a range of frameworks, including TensorFlow, PyTorch, MXNet, Chainer, and Gluon.

Maybe you have also heard of [Hugging Face](https://huggingface.co/), the data science platform that provides tools for users to build, train and deploy ML models based on open source (OS) code and technologies. [AWS and Hugging face have partnered](https://huggingface.co/blog/aws-partnership) to provide a range of Deep Learning Containers, which can be used to train and deploy their models on AWS.

### Pre-trained models with Sagemaker Jumpstart

[Sagemaker Jumpstart](https://docs.aws.amazon.com/sagemaker/latest/dg/studio-jumpstart.html) provides a range of pre-trained models that can be used to build generative AI solutions. These can be trained and adapted to your needs using Sagemaker Autopilot, or can be used as-is.

More information on training LLMs using sagemaker can be found [here](https://aws.amazon.com/blogs/machine-learning/training-large-language-models-on-amazon-sagemaker-best-practices/)

### AWS Marketplace

If that's not enough, there is a huge variety of related services available on the [AWS Marketplace](https://aws.amazon.com/marketplace). These include, other models or managed services and tools, such as vector databases, that can be used to help build generative AI solutions on AWS. These can be great to bridge the gap if you are looking for a solution that is not yet offered by AWS or are looking for a specific feature but would otherwise like to build your solution in the cloud.

## What does AWS have in the pipeline?

### Amazon Titan

Amazon plan to offer their [own suite of Generative AI services](https://aws.amazon.com/bedrock/titan/) in the future. These will include: Titan Vision, similar to DALL-E, their own LLM, Titan Text, as an alternative to ChatGPT, and a service that can translate text inputs into numerical representations (embeddings). This will allow comparison of embeddings, which can produce more relevant and contextual responses than word matching in the context of search and personalization.

### Amazon Bedrock

[Amazon Bedrock](https://aws.amazon.com/bedrock/) is a service that will allow you to build and deploy generative AI models at scale. They will offer a range of their own pre-trained models that can be used as-is or adapted to your needs. In addition they will provide API access to existing models from other providers, for example, jurassic-2, Stable Diffusion.

## What next?

If you are interested in learning more, here's a list of resources that you might find useful, including an overview of a topic that is very important to consider when building generative AI solutions: bias.

- [Generative AI: It's not all chatGPT](https://www.forbes.com/sites/forbestechcouncil/2023/04/24/generative-ai-its-not-all-chatgpt/)
- [Prompt Injection](https://simonwillison.net/2023/Apr/14/worst-that-can-happen/)
- [Generative AI bias](https://www.insider.com/chatgpt-is-like-many-other-ai-models-rife-with-bias-2023-1)
