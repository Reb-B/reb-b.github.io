---
title: "Money saving tips for your AWS bill"
date: 2022-12-22 00:00:00 +0200
image: /assets/img/posts/placeholder.svg
description: |
  Monitoring and reducing your cloud costs doesn't have to be complex. With a few tips and tricks you can get your team(s) insight into their spending and set up good practices that ensure your cloud spend is kept under control.
categories: [AI & ML]
tags: [aws, cost-optimization, cloud, finops, aws-budgets]
---

![piggy bank](/assets/img/posts/money-saving-tips-for-your-aws-bill/save-money.jpg)

A commonly touted benefit of migrating to the cloud is that you can save money by taking advantage of on-demand pricing and skip the months of planning required to set up in a traditional data centre. But this flexibility comes at a cost and you might be looking at your bill month on month, wondering whether it was worth it.

Here we want to share with you some AWS features and services we've used with customers to reduce their cloud costs.

## Raising Cost Awareness

Costs often build up when there is a lack of awareness of which resources are active in an account (this is one reason we always recommend defining your [infrastructure as code](https://docs.aws.amazon.com/whitepapers/latest/introduction-devops-aws/infrastructure-as-code.html)). Some costs are obvious and stable, like the price of an EC2 on-demand instance but dynamic costs such as transferring data between AWS regions, can add up.

### Budget Alerts

Budget Alerts can be a great way to point your teams to rising costs in an account. With [AWS Budgets ](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) you can set a fixed, or planned spend, and receive an alert via email when the cost in an account has breached this amount, or you are close to maxing out your planned budget. Auto-adjusting Budgets are another option that look at your past spending over a specified period of time and alert when spikes in spending are identified.

Budgets are easily automated. We recently used [AWS Stepfunctions](https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html) configured via [CDK](https://docs.aws.amazon.com/cdk/v2/guide/home.html), and the [AWS Budgets API](https://docs.aws.amazon.com/aws-cost-management/latest/APIReference/API_Operations_AWS_Budgets.html), to automate rolling out of Autoadjusting Budgets and email alerts, for existing and newly created accounts within an organization.

### Cost Explorer

So you've been alerted that your costs are increasing, but now what? A good first step is to check out the [AWS Cost Explorer Service](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html) in the AWS console of the relevant account. Here you can get an overview of your spending for a period of time that you select.

Typically, we would look at the past few months and group spend by service to see if there are any clear outliers. Maybe someone ( 😬 ) forgot that big Sagemaker Instance they were testing out? Now you can find the owner and get it shut down.

![cost-explorer-by-service](/assets/img/posts/money-saving-tips-for-your-aws-bill/cost-explorer-by-service.png)

Alternatively, once you have identified a cost spike in a service, you can select that service and use the usage-type filter to determine which feature of the service has led to your cost increase and decide 1) if the usage is necessary and 2) whether the price of that usage can be reduced. Maybe you are storing a large amount of data in S3 but you rarely retrieve it, or you are transferring an unnecessary amount of data via the internet. Cost Explorer can pinpoint these issues. More info on filtering data in Cost Explorer can be found [here](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-filtering.html).

## Easy Wins for Common Services

AWS has a multitude of services with their own pricing and cost saving features. Once you have identified an issue with your AWS service usage in the Cost Explorer, it is worth looking into the pricing in more detail. In the sections below are some easy ways to save money on the most popular AWS services.

## Compute Savings

The flexibility offered by on-demand compute pricing is great for start ups and businesses with highly fluctuating workloads. But larger businesses, with steady workloads, migrating to the cloud, don't always benefit from this pricing model. Here are where AWS's compute savings options come into play.

### Savings Plans

If you can commit to an expected usage per hour for a 1 or 3 year period and pay for it upfront, then you can save up to 66% on you bill with [Compute Savings Plans](https://aws.amazon.com/savingsplans/compute-pricing/). You can also choose to pay partially or after the term is up for smaller discounts.

**Compute savings plan**

- save on Lambda, Fargate and EC2 compute costs
- applies even if you change workload region and/or EC2 Instance family

If you are only using EC2 and know which region and instance family are required for your workload, you can save up to 72% with an EC2 specific savings plan

**EC2 savings plans**

- save on EC2 compute costs only
- larger savings rate than compute savings plan
- instance family and region are fixed
- can change EC2 Instance size within an instance family and OS

### Reserved Instances

[Reserved Instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-reserved-instances.html) offer a billing discount which is applied to On-Demand Instances in your account.

- save up to 72% compared to On-Demand pricing
- instance family, region and OS are fixed
- EC2 family is fixed (unless you choose Convertible Instances for a smaller discount of up to 66%)
- the Reserved Instance payment model can be applied to additional services such as RDS and OpenSearch.

### Rightsizing Recommendations in AWS Cost Explorer

Overprovisioned EC2 instances often create avoidable costs that can be quickly and easily reduced. AWS offers ways to identify and adjust these resources.

![right-sizing-recommendations](/assets/img/posts/money-saving-tips-for-your-aws-bill/right-sizing-recommendations-blurred.png)

Right-Sizing Recommendation in Cost Explorer use Cloudwatch metrics to analyse the utilisation of EC2 Instances over the last 14 days with regard to CPU, memory and network utilisation as well as disk use. Based on this information, EC2 usage is placed into the following categories:

- idle: the maximum CPU utilisation was <= 1% in the last 14 days. It is therefore recommended to terminate the instance.
- underutilized: the maximum CPU utilisation in the last few days was more than 1%, but there is potential for savings by changing the instance type.
- unsuitable instance type: A more suitable instance is recommended here.

Please note: The recommendations only refer to the observed workloads of the last 14 days.

You can also check out [Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/latest/ug/viewing-dashboard.html), which is another right-sizing tool from AWS. It has some advanced features, including the ability to identify EC2 workloads that can be upgraded to newer, more cost-efficient instance types, with the smallest amount of migration effort. It can also analyse other services for usage efficiency, such as ECS and EBS volumes.

### Spot instances

[Spot Instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html) offer users unused EC2 capacity at a heavily discounted price (up to 90% discount compared to On-Demand).

The reason the discount is high, is that AWS can reclaim the use of the Spot Instances on which you run your workload - this causes an interruption. This can happen when demand for instances is high, leaving AWS without the capacity to offer the discounted Spot Instance price.

That means that Spot Instances are great for non-critical workloads, such as:

- Batch applications
- Test environments
- Data processing
- Training ML models

## Storage Savings

Another common source of rising costs is from increasing storage demands in S3. This is a challenge with a fix that is easy to implement.

We've managed to save customers more than $12,000 per month on their S3 bill by transferring data to more appropriate storage in AWS 😄

### S3 - Object classes

[Amazon S3 Object Storage](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) offers several so-called [Object Storage Classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html), which offer discounts that scale according to the access performance that they can provide. Which storage class to use depends on your workload.

![s3-object-classes](/assets/img/posts/money-saving-tips-for-your-aws-bill/s3-object-classes.png)

If you very rarely access your data, then ideally you would choose an object class that has a low storage cost, which according to AWS's tiering system, corresponds to a higher retrieval cost. Since you rarely retrieve your data, the retrieval costs will be negligible in comparison to what you save month on month by reducing the price of storage.

### S3 - Lifecycle Rules

If your retrieval pattern changes according to the age of the data, and that pattern is consistent, you can implement a [Lifecycle Rule](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html) that moves or deletes the data after a specified amount of time. For example, if you know that you only access your data in the first month that it arrives in your S3 Bucket and afterwards won't access it again, it could be scheduled for deletion.

Not sure what data usage pattern you have? Check out [Storage Class Analysis](https://docs.aws.amazon.com/AmazonS3/latest/userguide/analytics-storage-class.html). It can monitor your use pattern for a month and suggest a storage class on the basis of that use pattern.

### S3 - Intelligent Tiering

If your retrieval pattern changes over time, you can go one step further than a Lifecycle Rule and leave it to [S3 Intelligent Tiering](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intelligent-tiering.html) to decide automatically in which storage class your data should be moved to. In fact, unless your retrieval pattern is highly consistent, this is an approach we would recommend over a Lifecycle Rule in many cases. Intelligent Tiering provides three default Object Storage Classes which are essentially the same as the standard S3 Object Storage Classes in terms of price and retrieval behaviour. After 30 days without a retrieval, Intelligent Tiering will automatically move your object to the next cheapest Object Storage Class and will move it again after the next 30 days and so on.

Note: Once on, you cannot simply "switch off" Intelligent Tiering. You will need to copy your data to S3 Standard Class.

## Need More Cost saving advice?

These are just a few of the services available in AWS and your architecture and workload determine how best to save money.
