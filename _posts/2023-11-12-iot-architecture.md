---
title: "Making a scalable and modular IOT architecture"
date: 2023-11-13T15:34:30-04:00
categories:
    - blog
tags:
    - AWS
    - IOT
---

# Making a scalable and modular IOT architecture

## The problem
As the company was buying smart vehicles from multiple different vendors we quickly hit the problem of trying to integrate multiple different iot communication standards into a single seamless system.
I'll try to do a quick breakdown of the micro-service architecture that I chose for this job.

The micro services are deployed on aws Lambdas due to limited devops resources at the time with the SQS used for message passing between them. This provided some benefits such as separate scaling for different IOTs, but a unified solution where the message queues are in-memory instead of in the cloud would provide you with a much better performance for the same architecture.



## Architecture

We start with a set of functions I called the listeners, with every specific IOT type having its own function. These functions convert native IOT messages and communications into one of 2 standard message types "STATUS" or "ALERT" and pass those messages to the alert and status lambdas.

Status lambda also can send messages to the alert lambda if some generic issues, not specific to any one IOT implementation happen.


We could technically package the message format marshalers for commands and status updates together and then treat them as plugins to the system. but often there are specific differences between implementations other than just the message format.
