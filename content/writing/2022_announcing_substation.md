---
title: 'Announcing Substation'
author: 'Josh Liburdi'
date: 2022-10-25
description: 'Announces the release of Substation, an open-source toolkit for creating highly configurable, no maintenance, and cost-efficient data pipelines.'
tags:
   - analytics
   - logging
---

Over [on Brex's tech blog](https://medium.com/brexeng/announcing-substation-188d049d979b) I wrote a post announcing the release of [Substation](https://github.com/brexhq/substation). The original text is copied below.

---

# Introduction
We are thrilled to publicly announce the release of Substation, an open source toolkit for creating highly configurable, no maintenance, and cost-efficient data pipelines. Substation solves a problem that every security team has, but few may recognize - the need to normalize, correlate, and enrich their security event data at scale.

# Why We Built Substation
Brex's Detection and Response team needed a low-maintenance, scalable solution that allowed us to manage the data processing requirements of dozens of unique security event sources and datasets. Our goal was to build a solution that is reusable for any dataset, scales to any event volume, and is easy to deploy and maintain.

The overarching goal of Substation is to empower security and observability teams to gain control over their event logs with minimal operational overhead. Most security platforms (i.e., SIEM) provide few options for teams that want to manage data quality. At best, they may give users the ability to do simple data transformations after events are ingested into the platform. If teams want high-quality data, they typically need to invest engineering and operational resources into deploying and maintaining complex distributed systems.

Our team has used Substation in production for more than 15 months, and we feel that now is the right time to share it with the open source security community. Since every team has a different definition of "at scale," below are some numbers from our internal deployment to help explain how we use it:
- Process more than 5 terabytes (TB) of data each day, with a variable cost of $30–40 per TB.
- Sustain more than 100k events per second (EPS), with peaks of more than 200k EPS.
- Deploy new data pipelines with 100s of unique resources in minutes.
- Manage 30 data pipelines, each with unique designs and configurations.
- Spend less than 1 hour per week on operations and maintenance.

# Differentiating Benefits
Substation differentiates itself from other data pipeline solutions in several ways:
- Fully serverless - you'll never manage a server or think about sizing a cluster.
- Designed for scale - the system scales from 10s to more than 100,000s of events per second, and does it automatically with no intervention required by an engineer.
- Infrastructure and configs as code - we use Terraform, Jsonnet, and AWS AppConfig, which means you can deploy unique, reusable data pipelines in minutes.
- Cost efficient - we went the extra mile to make things affordable, including creating a compliant, minimal version of the Kinesis Producer and Client Libraries in Go.
- Extensible - we expose the Substation core as Go packages, so you can build your own custom data processing systems.

# Use Cases
Substation is two solutions in one: it's an event-driven, streaming data pipeline system and a package for building custom data processing systems.

As a data pipeline system, Substation has these features:
- Real-time event filtering and processing.
- Cross-dataset event correlation and enrichment.
- Concurrent event routing to downstream systems.
- Runs on containers, built for extensibility.
- Support for new event filters and processors.
- Support for new ingest sources and load destinations.
- Supports creation of custom applications (e.g., multi-cloud).

As a package, Substation can:
- Evaluate and filter structured and unstructured data.
- Modify data from, to, and in place as JSON.

Most importantly, the project makes no assumptions about the design of a data pipeline. At Brex, we're fans of two-phase design patterns (similar to what is described in [Google's SRE workbook](https://sre.google/workbook/data-processing/)) and use it for nearly all of our pipelines:

![image](/images/writing/2022_announcing_substation_0.png)

This design pattern supports storage of raw and processed data, and concurrently loads processed data to multiple destinations.

The project supports other use cases, too. Every data pipeline shown below is an example of what can be built today:

![image](/images/writing/2022_announcing_substation_1.png)

There are hundreds of unique design permutations, all of which are fully controlled by users through Terraform and Jsonnet configurations. Today the project offers these data ingest and load options:
- AWS API Gateway (ingest).
- AWS DynamoDB (load).
- AWS Kinesis Data Streams (ingest and load).
- AWS Kinesis Firehose (load).
- AWS S3 (ingest and load).
- AWS S3 via SNS (ingest).
- AWS SNS (ingest).
- AWS SQS (ingest and load).
- HTTP (load).
- Print to standard output (load).
- Sumo Logic (load).

# Achieving Scale, Engineers Not Invited
One of the biggest advantages of Substation is that the data pipeline system minimizes infrastructure and operational costs by automatically scaling resources in and out without intervention by an engineer. We see this every day at Brex and it's usually correlated to changes in employee, system, and user activity. Here's an example:

![image](/images/writing/2022_announcing_substation_2.png)

A normal week at Brex creates peaks and valleys of activity that our resources scale with, but sometimes unplanned things happen and the peaks get really, really tall. One day in mid-June I checked our event processing metrics dashboard and saw this:

![image](/images/writing/2022_announcing_substation_3.png)

It's difficult to see, but our event volume increased 10x over a period of a few hours and increased 4x in under one hour. The scale of this increase would wreak havoc on traditional data pipeline systems, likely causing an incident due to memory, CPU, and event loss alarms all triggering at the same time, but for us it was a nonevent and instead we were left wondering what fun we had missed out on.

# Future Work
Substation is under active development and uses semantic versioning for releases. Although we don't have a roadmap to share, we're interested in exploring these opportunities with the project in 2023:
- Data enrichment functions for both the data pipeline system and public package.
- Support for multiple cloud service providers, including cross-cloud functionality.
- Configuration verification and automated rollback in AppConfig.
- Automated recovery for failures during asynchronous data ingest (e.g., S3).
- Scalable, idempotent producers for sinking data.

We're always on the lookout for new data sources, sinks, and processing functions, so if there's something you would like to see added (or to add yourself), then check out the project on GitHub and provide feedback in our Discussions forum.

Open source projects this large aren't created in isolation, so thank you to everyone at Brex who has contributed to the project and made it possible, and thank you to all the future collaborators we haven't met yet!

*Special thanks* to the Brex project team and contributors:
- Josh Liburdi (project team lead, core architect and engineer).
- Julie Agnes Sparks (project team, documentation, code review).
- Parsa Attari (documentation, code review).
- Jake Miller (code contributions).
- Ben Morris (security review).
- Dan Gilmartin (architecture review).
- Mike Ruth (architecture review).
