---
title: "Exploring AWS OpenSearch: A guide to use cases, features, and deployment"
date: 2023-09-03 00:00:00 +0200
image: /assets/img/posts/exploring-aws-opensearch/magnifying-glass.png
description: |
  Cloud powered search is a powerful tool for many businesses. In this article we explore the features of AWS OpenSearch and how it can be leveraged to build search and analytics solutions.
categories: [Data]
tags: [aws, opensearch, search, elasticsearch, serverless]
---

![magnifying glass](/assets/img/posts/exploring-aws-opensearch/magnifying-glass.png)

## What is AWS OpenSearch

[AWS OpenSearch](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/what-is.html) is a powerful search and analytics service, based on [Elasticsearch](https://www.elastic.co/elasticsearch/) and made available in the AWS cloud. It's designed to provide scalable and performant search capabilities for a broad range of applications and use cases. With OpenSearch, developers can efficiently index and search huge amounts of structured and unstructured data in real-time, enabling users to find the information they need quickly and accurately. OpenSearch then integrates with other AWS services, allowing you to design comprehensive cloud based search solutions. Example use cases for AWS OpenSearch include search functionality for websites, log data analysis, or personalized recommendations.

## OpenSearch Service vs. OpenSearch Serverless

There are two primary options for running OpenSearch on AWS: [the OpenSearch (managed) Service](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/what-is.html) and [OpenSearch Serverless Service](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless.html). These two deployment options have different use cases and features, and it's important to understand the differences between them before deciding which one is right for your application. We have experience with both deployment options and despite our enthusiasm for serverless solutions we have had cases in which the managed service was the better choice. We'll outline some of the differences below.

But first, a note on terminology. In the context of the AWS OpenSearch managed service, an OpenSearch domain refers to your OpenSearch cluster and the data stored within it. The domain has a collection of nodes that work together to provide search and analytics functionality. A node is a single instance of OpenSearch.

An OpenSearch Serverless _collection_ is a group of OpenSearch indexes that work together to support a specific workload or use case. OpenSearch collections are deployed in a serverless cluster. OpenSearch Serverless offers 3 types of collection: timeseries, search, and a new vector search option to support semantic search use cases. The type of collection you choose determines the features available to you and optimizes the underlying cluster for your workload type.

So, when we refer to a OpenSearch serverless workload, we will talk in terms of collections and when we are talking about the OpenSearch managed service we will talk in terms of domains.

### Scalability and Performance:

The first and most obvious difference between the OpenSearch managed service and the OpenSearch serverless service is how they handle scaling and performance.

The OpenSearch managed service offers tighter control over scalability and performance options than the serverless variant. You can choose the number of nodes, their specifications, and scale up or down as needed to accommodate varying workloads. Importantly, though, this is something you have to manage yourself. You have control over the configuration but you may need to invest considerable time and effort in the management of the domain and its resources. This is the typical trade off between control and convenience that we see in many AWS services that offer both a managed and serverless deployment option.

The OpenSearch serverless service meanwhile abstracts away most management complexities. It automatically scales based on demand, handling spikes and fluctuations in traffic without manual intervention. You select the capacity of your service in so called OpenSearch Capacity Units (OCUs). There are two types of OCU, search and indexing capacity units and on deployment of a collection, you select the number of each type of OCU. By default you get 1 search OCU and 1 indexing OCU in 2 availability zones, so a total of 4 OCUs are deployed. This means you are charged for 4 OCUs (I'll talk more about costs later). These units are a combination of 6 GiB of memory and corresponding virtual CPU (vCPU) and data transfer to storage in S3.

The service automatically scales up or down based on the number of capacity units you select. This is a great option for smaller workloads or startups as it allows you to focus on building your application's search functionality without worrying about cluster provisioning, scaling, or patching. As new versions are released, OpenSearch Serverless will automatically upgrade your collections to include new features, bug fixes, and performance improvements.

### Customisation:

Customisability is also worth keeping in mind. The OpenSearch managed service allows for extensive customisation. You can configure advanced index settings, add plugins, and fine-tune the domain to match your application's specific requirements. Plugins are a powerful feature of OpenSearch and can be used to add functionality to support your workload. For example, you can add plugins to support machine learning, security, or monitoring.

The availability of plugins for the OpenSearch serverless service is improving rapidly but if you are looking to migrate an OpenSearch cluster to AWS, and rely heavily on plugins, you should definitely [check their availability](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-genref.html) ahead of time. In general the serverless service currently offers less customisation compared to the OpenSearch managed service. While you can configure indices and mappings, some advanced settings might not be available. However, if you are not heavily reliant on customisable features, this should not be a problem. There are still a lot of built-in features that are available out of the box.

### Cost Structure:

The cost structure for the OpenSearch managed service is based on the type and number of instances you provision, along with storage and data transfer fees. You are billed based on the number and type of instances you provision for your domain and costs increase with higher-performing instance types. If you have a very steady workload and plan to use the service for a long period of time you can still reduce these costs by [reserving OpenSearch instances](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ri.html) AWS offers reserved instance pricing for longer-term commitments, which can provide cost savings over on-demand pricing for a one or three year period. You then pay for storage according to the EBS volume type you choose and for data transfer to and from your domain. There are options to reduce the cost of your data storage as well. For example you can place rarely or less searched data into so-called 'ultrawarm' as opposed to 'hot' nodes. In this case an ultrawarm node refers to data stored in S3 as opposed to hot node data that is stored on instance or EBS volumes, which are more rapidly accessible. You can find more information on the pricing structure [here](https://aws.amazon.com/opensearch-service/pricing/).

We have already introduced the idea of OCUs for the OpenSearch serverless service and the fact that you always have four running by default. At the time of writing, it is not possible to scale down to zero OCUs. This means that even if you have no traffic at all for a period of time, you will still be charged for 4 OCUs. Currently in the Frankfurt region indexing and search OCUs each cost $0.339 per hour. This means that if you have no traffic for a month, you will still be charged around $900 + storage costs of $0.026 per GB per month in S3. This might not sound great, but depending on the capacity you might require at peak usage of your service, it could still work out cheaper than having a large cluster without adequate capacity scaling, alongside maintenance effort costs. The serverless service additionally handles caching for you, if dealing with timeseries data. Frequently accessed data are stored in a 'hot' cache to optimise request response times. The OpenSearch managed service requires you to handle data lifecycles yourself. When working with customers, we typically use the [AWS pricing calculator](https://calculator.aws) to estimate the costs of various options and compare them to the expected usage of the service. This is a good way to determine which service is best for your use case.

For more general info on controlling AWS costs, check out our [blog article](/posts/money-saving-tips-for-your-aws-bill/) on the topic.

## Our anecdotal experience with AWS OpenSearch deployment options

We recently built sample applications in CDK using both the OpenSearch managed service and OpenSearch serverless service to get a general idea of how the configuration and ease of use differ. The applications allowed us to search for movies by title and genre via an API gateway, a lambda and opensearch. We used the `movieId` as the document id and the title and genres as the searchable fields. It's a common example used in several tutorials (for example, [here](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/search-example.html)) and we found it a good hands on approach to explore the configuration possibilities.

### Documentation

The AWS OpenSearch managed Service is a mature product with a large community of users and contributors and its history as an Elasticsearch fork means that there is a lot of existing documentation and guides available.

The serverless option is relatively new and the documentation is undergoing a lot of changes at the time of writing. This can sometimes result in some difficulties working with the service due to mistakes or incompletely documented features. Inevitably there are then also fewer guides available but there are enough to get started with.

### Features

In terms of disaster recovery for production search solutions, it's worth noting that [cross cluster replication](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/replication.html) i.e. replicating indexes, mappings, and metadata from one OpenSearch domain to another, is not currently available for serverless OpenSearch. There is also no snapshot functionality to create backups at present, but automating the backup of your data to S3 is one solution. If the requirement is high availability though, the serverless variant of OpenSearch is multi-AZ per default, a search and index OCU in one AZ and the same in a second AZ. You can take a look [here](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-overview.html#serverless-limitations) for current feature limitation of the OpenSearch serverless service. Despite these limitations, I would expect to see cross cluster replication and several other features of the OpenSearch managed service rolled out to the OpenSearch serverless service in the future.

### Ease of use

Even with occasionally shoddy documentation, configuration of the serverless service was straightforward. Unsurprisingly the managed service was also easy to set up but required more configuration and was more time consuming as a result.

## Conclusion

Both the AWS OpenSearch managed service and the AWS OpenSearch serverless service offer powerful search and analytics capabilities. Before jumping into using one or the other, you will need a clear idea of the features that are necessary for your application and your expected workload.
