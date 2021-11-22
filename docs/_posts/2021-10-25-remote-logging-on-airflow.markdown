---
layout: post
title:  "Remote logging to S3 on Airflow"
date:   2021-05-01 16:32:02 +0530
categories: tech airflow
---

2. Why remote logging is required
  - Required if running on Kubernetes
  - Space is an issue
3. What options do we have for remote logging
  - S3
  - GS
  - Any custom hook?
4. Logging to S3 according to how you have set up Airflow
  - When Airflow is installed on K8s using the helm chart
  - When Airflow in installed on stand alone machine like Local System or Amazon Ec2 instance.

## What's the article about
This article will talk about the way you can set up your Airflow task logs to an external blob storage location like S3. This is not about the logs which the Airflow WebServer or the Scheduler generates. If you are running Airflow on Kubernetes which is installed using the Airflow Helm chart, then this article might help you set up your S3 logging.

## Why remote logging is required
_This is not related the installation so you can skip this paragraph is you are not interested in a little background_.  
If you have been running Airflow in your projects for some time you must have noticed that the task logs keep piling up as they run at quick frequencies. If you are running the Airflow instance on a stand-alone machine or an amazon Ec2 instance, then it possible for you to fill up some important space on your machine which could eventually lead to crashing of Airflow itself.  
To take another scenario as an example, you might be running Airflow instance on a Kubernetes server. In this case, Airflow does not store logs store logs on the pod where the Airflow Scheduler runs. When you user the Airflow UI to get logs of a task, it looks up the pod the task ran on and internally does a `kubectl logs pod/<pod-name-of-task>` (sort of) and gives you the logs. The problem here is that usually the pods are cleaned at a regular interval and as soon as the pod is cleaned up, the logs of the task get lost.  
To resolve this issue, Airflow has provided a functionality in which it can store the task logs at an external location. This location can give many benefits like low cost, timed backups, logs purging, etc. 