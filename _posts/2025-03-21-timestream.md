---
title: "An Overview and Lessons learned from Storing and Analysing IoT Data in AWS Timestream"
date: 2025-03-21 00:00:00 +0200
image: /assets/img/posts/timestream/timestream-splash.jpeg
description: |
   In this post, we share our experience with AWS Timestream—its strengths, limitations, and key lessons learned from real-world implementation.
categories: [Data]
tags: [aws, timestream, iot, time-series, databases]
---

![timestream](/assets/img/posts/timestream/timestream.jpg)

# Overview

[AWS Timestream](https://docs.aws.amazon.com/timestream/latest/developerguide/what-is-timestream.html) is Amazon's purpose-built time-series database. It's been widely available for a while now (since 2020), and we recently had a fitting use case for it. As you may have seen, we've been working on a project that involves collecting and storing timeseries data from IoT devices, and we needed a database that could handle this kind of data at scale. Since this is exactly the point of Timestream, we decided to use the opportunity to get to know the service a bit better, and share what we've learned along the way.

## What is AWS Timestream?

AWS Timestream is a fully managed database service optimized for time-series data. It supports data ingestion, querying, and [tiered storage](https://docs.aws.amazon.com/timestream/latest/developerguide/storage.html) options to balance performance and cost. A Timestream database consists of the underlying database itself, and then the data within it is organized in tables. In terms of how the data is phyically stored, the two tiers offered are a short term memory store, which provides rapid query response times, and a magnetic memory store that is slower for retrievals but costs less than the short term store. The retention periods for each store type can be set by the user. Timestream is designed to integrate well with other AWS services, such as [Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html), [Kinesis](https://docs.aws.amazon.com/streams/latest/dev/introduction.html), and [CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html) and it scales automatically to handle high volumes of data. It supports millions of writes per second and is optimised for large datasets and data compression, with nanosecond granularity.

## When should you use Timestream?

If you are unsure if you need a time-series database, you can ask yourself a few questions to help you decide:

- Does my data use a timestamp?
- Is the timestamp a meaningful aspect of my data?
- Am I interested in trends over time?
- Do I need to store a large amount of data?
- Do I have very frequent writes/events?

If you are answering yes to these questions, you might want to consider using a time-series database like Timestream.

Typical use cases for Timestream would include:

- IoT applications
- DevOps monitoring
- Industrial telemetry
- Financial data analysis

# Our experience

## Setup

Our use case for Timestream was straighforward: the idea was to store data from a large number of IoT devices and display it in an app. It was also necessary to be able to query older data for analytics purposes. The project was built using CDK and TypeScript. At the time, the CDK library for Timestream only provided [L1 constructs](https://docs.aws.amazon.com/cdk/v2/guide/constructs.html), but the minimal configuration required made this a non-issue. The database and tables were set up with just a few lines of code.

### Data types

The [data format](https://docs.aws.amazon.com/timestream/latest/developerguide/data-modeling.html) in Timestream is pretty simple. Records in timestream can consist of `dimensions` and `measures`. A dimension is any attribute that can be used to filter or group data, such as a device ID, location or sensor type.
Measures meanwhile are the actual data points, such as temperature, humidity, or pressure. Each record has a timestamp, and you can have multiple dimensions and measures in a single record. The following measures are supported:

- BIGINT
- DOUBLE
- VARCHAR
- TIMESTAMP
- MULTI (more on this in the next section …)

### Single vs. Multimeasure values

As you can see there is a 'multi' measure data type. This is because there are two options for sending data to AWS Timestream; they can either be sent as single or multimeasure values.

A single measure value can essentially just be a single data point with a timestamp, dimension, and a measure name. This is useful for sending data from a single sensor. An example would be sending the temperature of a room at a given time. In our case, we needed to use multimeasure values, which allow you to send several data points associated with a single timestamp in one request. This is for situations in which a device might, for example, send both temperature and humidity data of a room at the same time point.

Ultimately, your data will determine what kind of data type you need to use, but here are a couple of things to keep in mind:

Single measures offer simplicity:

- Each record is a single data point
- Easier to handle changes in data structure
- IoT specific advantage: data can be forwarded directly to Timestream using only an [IoT Rule](https://docs.aws.amazon.com/iot/latest/developerguide/iot-rules.html)

However, sending each record separately as a single data point has disadvantages:

- More data points to store -> increased cost
- More data points to query -> increased query time (if aggregating data)

Multimeasure records offer efficiency

- Fewer records to store -> reduced cost
- Fewer records to query -> reduced query time

Nonetheless, multimeasure records can introduce complexity:

- Complex data structure -> harder to manage schema changes
- IoT specific disadvantage: data must be processed via a Lambda function before forwarding to Timestream

As we discovered during our project, where single measure values can be forwarded from IoT Core directly to Timestream using an IoT Rule, multimeasure values need to be processed first. We used a Lambda function to do this, which was a minor inconvenience but if your data is not coming via IoT Core you may need to use the SDK anyway. Here's an example of how to write records to Timestream using the boto3 SDK Timestream write client:

```python
import boto3
import time

timestream = boto3.client('timestream-write')

records = [
    {
        'Time': str(int(time.time() * 1000)),
        'Dimensions': [
            {'Name': 'device_id', 'Value': 'device123'},
            {'Name': 'location', 'Value': 'Room 101'},
            {'Name': 'sensor_type', 'Value': 'temperature'}
        ],
        'MeasureName': 'temperature',
        'MeasureValue': '23.5',
        'MeasureValueType': 'DOUBLE'
    },
    {
        'Time': str(int(time.time() * 1000)),
        'Dimensions': [
            {'Name': 'device_id', 'Value': 'device123'},
            {'Name': 'location', 'Value': 'Room 101'},
            {'Name': 'sensor_type', 'Value': 'humidity'}
        ],
        'MeasureName': 'humidity',
        'MeasureValue': '45.3',
        'MeasureValueType': 'DOUBLE'
    }
]

response = timestream.write_records(
    DatabaseName='your_database_name',
    TableName='sensor_data',
    Records=records
)
```

And an additional note regarding schema changes: Timestream allows for the addition of new dimensions and measures but no updates to datatypes or deletion of dimensions or measures from a multimeasure value table with existing data. So if you need to drastically change the schema of your data, you will need to create a new table and migrate everything. This can be an annoyance.

### If you've never used timestream before ...

Timestream documentation leans heavily into the idea of use cases involving IoT analytics, but if you are not well versed in the capabilities of the service, you might be persuaded that you can do more 'real-time' analytics with Timestream than you actually can.
From experience, and discussion with experts at AWS, Timestream is great for storing and querying time-series data, but it is not a full-fledged analytics tool. It is not designed for delivering results of complex queries or data transformations in real-time, but rather, for example, simple aggregations, like getting the max temperature measured by a device over a period of time, or counting breaches of a set threshold in recent data. If you need to do complex analytics on your data, you will need to consider extending your solution with other AWS services like [Kinesis](https://docs.aws.amazon.com/streams/latest/dev/introduction.html) and [Glue](https://docs.aws.amazon.com/glue/latest/dg/what-is-glue.html).

Query times also depend on whether you are taking data from the short-lived in memory store or the longer term magnetic memory store (essentially an S3 bucket under the hood). For a simple query, retrieving and displaying recent data can be accomplished within milliseconds,
but querying older data can take more than a second. This is not a problem if you are only interested in recent data but if you need to query older data often or combine older data with newer values, you might need to consider storing or accessing your data in a different way.

We needed to query data very often and with a very low response time, so when performance fell short of expectations, we settled for caching our newest data point for each measure in a dynamodb table to avoid querying Timestream too often. This was a good compromise for us and did not involve much additional work but bear in mind that Timestream is rather optimised for very frequent writes and querying time related trends, not for querying individual data points.

These aren't drawbacks of timestream, but rather limitations of the service that you should be aware of when considering using it and that tripped us up in places, on first use.

## Summing it up

In short, we found Timestream easy to set up and integrate with our existing infrastructure, and it was able to handle the large amount of data we were sending to it. We did run into some inconveniences while working with the service, such as the need to process multimeasure values before sending them to Timestream, and optimising our infrastructure to query the data in the most performant way, but essentially this led to a better understanding of timestream and its place in our infrastructure.
Hopefully this post makes it a little clearer to what extent Timestream can be useful for your use case, and what you might need to consider when using it.
