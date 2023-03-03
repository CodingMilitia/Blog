---
author: João Antunes
date: 2022-07-27 19:45:00+01:00
layout: post
title: "[Video] Outbox meets change data capture - hooking into the Write-Ahead Log (feat. .NET, PostgreSQL & Kafka)"
summary: "Another video on using change data capture (often referred to as CDC), to hook up into the database transaction log, forwarding incoming entries to the outbox table."
images:
- '/images/2022/07/27/outbox-meets-change-data-capture-hooking-into-the-write-ahead-log.png'
categories:
- csharp
- dotnet
tags:
- outbox
- 'change data capture'
- postgresql
- kafka
- video
slug: outbox-meets-change-data-capture-hooking-into-the-write-ahead-log
---

Hey folks! 👋

📢📽️ New one sent right into the tubes!

Another video on using change data capture (often referred to as CDC), to hook up into the database transaction log, forwarding incoming entries to the outbox table.

Previously, we used Debezium for this, but I was left wondering, what if I want to implement something similar with .NET? Turns out it's not super hard, with the help of Npgsql, which provides us with APIs to hook into PostgreSQL Write Ahead Log, so we can read the incoming outbox messages and forward them to Kafka.

{{< youtube 4rnSzEd9jPI >}}

Here are a couple of related links:

- [Sample implementation](https://github.com/joaofbantunes/PostgresChangeDataCaptureOutboxSample)
- [Intro to the transactional outbox pattern](https://youtu.be/suKSJ5DvynA)
- [Outbox meets change data capture (feat. .NET, PostgreSQL, Kafka and Debezium)](https://youtu.be/WcmLvoxs9ps)
- [PostgreSQL Write-Ahead Logging (WAL)](https://www.postgresql.org/docs/current/wal-intro.html)
- [Npgsql Logical and Physical Replication](https://www.npgsql.org/doc/replication.html)

Thanks for stopping by, cyaz! 👋
