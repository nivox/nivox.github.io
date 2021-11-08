---
title: "At Least Once Processing in Akka Streams"
description: ""
summary: ""
keywords: ["akka", "akka-streams", "at-least-once"]
tags: ["akka", "akka-streams"]
slug: "akka-stream-at-least-once-processing"
date: 2021-11-13T20:00:00Z
draft: true
---

When we talk about streaming application we are, in essence, talking about a system which read a continuous stream of data one message at a time, performs some computations on such data and eventually produce an output on an external system.

This *simplistic* description quicky becomes more complicated as soon as we start considering latency, scalability and failure resilience requirements.
Focusing on the latter, we need to think how to handle failures (harware crash, power outage, network disruption) and how to guarantee that our recovery strategy is consistent from the point of view of the computation.

In order to solve this problem a common strategy is the following:
- incoming data is saved in a durable storage with sequential aceess
- each message read from the durable storage is associated to an offset
- the offset is persisted to indicate that the message has been processed
- upon startup the application fetches the last persisted offset from storage and uses it to locate the position of the next message to be read

This scheme is promising and offers different guarantees based on when the offset is persited. If we persist the offset immediately after having read the message, we achieve **at-most-once** semantic. If instead we wait until after the message has been completely processed (as well as any related side effect, like updating the state in an external database) then we get **at-least-once** semantic.

These names are self-explanatory enough but lets go over them nonetheless:
- **at-most-once**: the application never tries to recover failed messages: this implies that any side effect related to the message processing either occur once or never at all
- **at-least-once**:  the application tries to recover failed messages until it succede in completely process them: this implies that any side effect we perform while processing the message might happen multiple times.

Both these semantics seem sub-optimal. We would not be happy if our bank adopted either of them: using the former we could have our salary never be deposited in our account, while using the latter we could have see multiple identical payment statement for our latest gadget!

When it comes to our hard earned money the only semantic we are happy with is **exactly-once**. Unfortunately implementing a system which is able to guarantee this semantic is possible only under strict conditions and becomes impossible as soon as you deviate from them. Fortunately we can get the same guarantees without the inherent limitations using a strategy called **effectively-once**. This strategy is based on an **at-least-once** semantic with either:
- message deduplication
- idempotent operation

The idea is that if we are able to recognise that we already have performed a certain operation for an incoming message we can safely ignore it. On the other end if our operation is idempotent we can skip the check and just perform it as many times as we receive the same message, knowning that the end result will be the same.

In the rest of this article we will concentrate on how to achieve **at-least-once** semantic in an Akka Stream application by using Kafka as our durable storage.

# 