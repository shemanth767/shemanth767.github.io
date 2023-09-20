---
title: Replacing Google Drive with NextCloud - Part 2
date: 2022-09-18 10:00:00 +0530
categories: [Server]
tags: []
toc: true
published: true
mermaid: true
---



## Overview

I've finally settled on a framework that works for me reliably. It has a lot of moving parts, but AWS made it
so simple to manage and synchronize them.

![](/assets/img/nextcloud-2-post/ec2-nc.drawio.svg)

## Description

### EC2 Instance

EC2 Instance is the t3a.small server that I'm renting to run NextCloud on. It contains 2 vCPUs and 2GB of Memory.
I think T3 is the best tool for my use case here. In the words of [Amazon](https://aws.amazon.com/ec2/instance-types/t3/)
itself,
> T3 instances are designed for applications with moderate CPU usage that experience temporary spikes in use.

#### 1 -- NextCloud

This is the heart of everything. It's the main NextCloud application.

#### 2 -- MySQL DB

NextCloud maintains a local disk DB that keeps track of the metadata for files updated i.e., the name of each file, when it
was uploaded, directory path etc. Optionally, this DB can also store the thumbnails of images so that users can only
get the whole image when needed.

#### 3 -- Redis Cache

To serve frequently accessed files quickly, NextCloud maintains a cache in Redis.

#### 4 -- Apache Server

The final piece of the core NextCloud infrastructure is an apache server for the NextCloud Web application and APIs. This
allows seamless connectivity from NextCloud's Android app!

#### 5 -- Static IP

AWS gives one free static IP for every *running* EC2 instance. You get charged hourly if you have the static IP, but the EC2
instance is shut down. This becomes important in the cost analysis :/

### Data Layer

#### 6 -- S3 Storage Bucket

NextCloud is configured to store all files in an S3 Bucket. Accesses from this bucket are relatively slow when compared
to disk fetches, so Redis comes in handy here. This bucket can grow infinitely (not really), so scalability FTW.

#### 7 -- S3 Access Logs Bucket

This is an optional element where access logs to the main bucket are stored.

#### 8 -- Offline Backups

This is a optional and manual backup I've configured to sync data from the bucket onto my local disk. Can't be too cautious.

### Other Infra

#### 9 -- Lambda Functions

EC2 Instances are charged by the hour. So, if I can schedule the hours in which my instance should be up, I can
significantly cut down costs. This is done by the lambda functions,

* StartNCInstance - Starts this particular instance
* StopNCInstance - Stops this particular instance

#### 10 -- EventBridge

Lambda Functions cannot run themselves and need an external trigger. EventBridge lets me trigger the configured lambda
functions using a cron trigger. Following are the triggers I have configured,

* **StartNCInstance** - `30 2 * * ? *` - Triggers the corresponding Lambda function at 02:30 UTC daily (8AM IST).
* **StopNCInstance** - `30 12 * * ? *` - Triggers the corresponding Lambda function at 12:30 UTC daily (6PM IST).

This setup keeps my instance up for 10 hours a day.

## Cost Analysis

### Breakdown

#### EC2 Instance

**Monthly Hours:** 30 Days * 10 Hours per day = 300 Hours  
**Cost per Hour:** 0.0123$  

**Actual Monthly Cost:** 3.69$ 

#### EBS Storage

The disk that EC2 uses is an Elastic Block Storage which is also charged. However, it's free for the first 12 months! 

**Size:** 16GB  
**Cost per GB-Month:** 0.114$  
**Estimated Monthly Cost:** 1.82$  

**Actual Monthly Cost:** 0$ (Free-tier!) 

#### Elastic IP Address

As mentioned before, the static IP Address is free whenever it's attached to an active EC2 Instance. So, only the hours
when the IP is not attached to a running instance is billed.

**Monthly Hours:** 30 Days * 14 Hours per day = 420 Hours  
**Cost per Hour:** 0.005$  

**Actual Monthly Cost:** 2.1$ 

#### S3 Storage

5GB Storage per month is free for the first twelve months. 

**Average Monthly Storage:** 10GB  
**Cost per GB-Month:** 0.025$  
**Estimated Monthly Cost:** 0.25$  

**Actual Monthly Cost** 0.125$

#### S3 Get Requests

I was surprised to learn that S3 bills for the number of requests you make to the storage. The free tier subsidies 20,000
GET requests per month.

**Average GET Requests:** 6,000  
**Cost per 1,000 GET Requests** 0.0004$  
**Estimated Monthly Cost:** 0.0024$  

**Actual Monthly Cost:** 0$

#### S3 PUT, COPY, and POST Requests

The free tier subsidies 2,000 PUT requests per month.

**Average Requests:** 3,000  
**Cost per 1,000 Requests** 0.005$  
**Estimated Monthly Cost:** 0.015$  

**Actual Monthly Cost:** 0.005$

#### Lambda

AWS gives out 3.2 Million seconds of compute time and 1 Million free requests per month. This is ALWAYS FREE :)

**Requests per month:** (2 per day) * (30 days) = 60 Requests  
**Compute seconds per month**: (60 Requests) * (600 ms average) = 36 Seconds  
**Estimated Monthly Cost:** Don't bother with this. It's less than a cent.  

**Actual Monthly Cost:** 0$ (Free-tier!)

#### Event Bridge

It's free.

### Total Costs

| Service | Estimated Cost  | Actual Cost  |
|---|---|---|
| EC2 | 3.69 | 3.69 |
| EBS | 1.82 | 0 |
| Elastic IP | 2.1 | 2.1 |
| S3 | 0.25 | 0.125 |
| S3 GET | 0.0024 | 0 |
| S3 POST | 0.015 | 0.005 |
| Lambda | 0 | 0 |
| EventBridge | 0 | 0 |

Total Estimated Monthly Cost: 7.87$  
Total Actual Monthly Cost: **5.92$**

Google One gives 100GB of Storage at Rs. 120 per month. My setup is 4 times more expensive.  

I can never ever compete with Google's economies of scale!