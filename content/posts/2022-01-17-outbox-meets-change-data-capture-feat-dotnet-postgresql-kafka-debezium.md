---
author: João Antunes
date: 2022-01-17 17:30:00+00:00
layout: post
title: "[Video] Outbox meets change data capture (feat. .NET, PostgreSQL, Kafka and Debezium)"
summary: "Quick video with an interesting approach to implement the publisher part of the outbox pattern. Using change data capture (often referred to as CDC), we hook up something to the database transaction log, forwarding incoming entries to the outbox table. In this example, we'll make use of Debezium, hooked up into a PostgreSQL database, forwarding messages to Kafka."
images:
- '/images/2022/01/17/outbox-meets-change-data-capture-feat-dotnet-postgresql-kafka-debezium.png'
categories:
- csharp
- dotnet
tags:
- outbox
- 'change data capture'
- postgresql
- kafka
- debezium
- video
slug: outbox-meets-change-data-capture-feat-dotnet-postgresql-kafka-debezium
---

Hey folks!

Quick video with an interesting approach to implement the publisher part of the outbox pattern.

Using change data capture (often referred to as CDC), we hook up something to the database transaction log, forwarding incoming entries to the outbox table.

In this example, we'll make use of Debezium, hooked up into a PostgreSQL database, forwarding messages to Kafka.

{{< yt WcmLvoxs9ps >}}

Here come a bunch of related links:

- [GitHub repository](https://github.com/joaofbantunes/DebeziumOutboxSample)
- [Intro to the transactional outbox pattern](https://blog.codingmilitia.com/2020/04/13/aspnet-040-from-zero-to-overkill-event-driven-integration-transactional-outbox-pattern/)
- [Debezium](https://debezium.io/)
- [Debezium PostgreSQL connector](https://debezium.io/documentation/reference/stable/connectors/postgresql.html)
- [Debezium outbox event router](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html)

Thanks for stopping by, cyaz! 👋