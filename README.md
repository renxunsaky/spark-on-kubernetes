# Spark on Kubernetes - from **Zero** to **Hero**
With Hadoop, a lot of companies have succeeded to handle huge amount of data and create added value for their business.
With Spark, data analysing becomes easier and faster. Nowadays, more and more enterprises are trying to migrate also on
Cloud based services for several reasons which we will not present here.

Based on Cloud, HDFS of Hadoop plays a much less important role in the ecosystem or whole architecture, because data are
stored on Cloud provided deep storage services, such as S3 on AWS, ADLS on Azure and cloud storage on GCP.

So people are trying to bring Spark out of the ecosystem of Hadoop by using other resource managers, such as Apache Mesos
or Kubernetes.

In this project, we will try to make Spark work on Kubernetes cluster which is highly searched recently.

Doesn't like other blogs or projects on Github who prepare a all-in-one script, I will try to do it manually step by step
in order to show you how to realize it and help you to understand how it works !

We will try to work it out by the following chapters:

- [Chapter 1: Simple Spark on Kubernetes on local PC with default settings](Chapter1)
- [Chapter 2: Simple Spark on Kubernetes on local PC with some advanced settings](Chapter2)
- [Chapter 3: Spark on Kubernetes on local PC with Airflow](Chapter3)
- [Chapter 4: Spark on Kubernetes on AWS EKS with some advanced settings](Chapter4)
- [Chapter 5: Spark on Kubernetes on AWS EKS with Airflow](Chapter5)
- [Chapter 6: Spark on Kubernetes with industrialized configuration(automation, security etc.)](Chapter6)

For each one, I will work on both Spark v2.4.5 and Spark v3.0.0
I suppose you understand the basics of Kubernetes, Docker and Spark
    
     