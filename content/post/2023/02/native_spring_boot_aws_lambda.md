+++
author = "Vladlen Gladis"
title = "AWS Lambda using Spring Boot 3 and GraalVM"
date = "2023-02-06"
description = "How to build an AWS Lambda using Spring Boot 3 with GraalVM and custom runtime."
featured = true
toc = true
draft = true

keywords = [
    "springboot",
    "springboot3",
    "graalvm",
    "aws",
    "serverless"
]
tags = [
    "springboot",
    "graalvm",
    "aws",
    "serverless"
]
categories = [
    "Cloud"
]
thumbnail = "/images/post1_native_aws_lambda/thumbnail.png"
+++

In this blog post, we'll dive into the world of serverless computing and show how to create an AWS Lambda function with custom runtime using **Spring Boot 3** and **GraalVM**. We'll be using Github Actions for building and testing our code, and deploying it to AWS using the Serverless Application Model (SAM).

By the end of this tutorial, you'll have a solid understanding of how to create a scalable and efficient serverless application using modern technology. So let's get started!"

<!--more-->

## Foreword
This blog post will guide though steps how to build a production ready native AWS Lambda function that will receive batch messages from an SQS and write the to DynamoDB and will explain all the deails during the way.

{{% notice info "In order to DIY you will need" %}}
* Basic understaging of Spring and Spring Boot
* Java 17
* Docker
* AWS SAM ([Installation guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html))
* GraalVM (for local testing [Tutorial how install it locally](/tutorial/2023/manage_multiple_jdk/)) *optional* (you may need it to build a native image locally)
* AWS Account *optional* (you may need it if want to deploy lambda to AWS)
* Github Repository *optional* (you may need it if want to automate deployments to AWS using Github Actions workflows)
{{% /notice %}}

## What is a native image, GraalVM, Spring Boot AOT transformation and how do they connect ?
GraalVM is a universal VM developed by Oracle JDK created by Oracle which is designed to accelerate the execution of applications written in Java and other JVM languages, that support `JVM Runtime Mode`, `Java on Truffle` and `Native Image`, for our use-case we are interested in the latest feature.

The GraalVM [native image](https://www.graalvm.org/22.0/reference-manual/native-image/) is a component of GraalVM that ahead-of-time (AOT) compiles Java code into a standalone executable, which can be executed without a JVM. This allows for faster startup times and lower memory footprint compared to traditional JVM-based applications. Native-Image is particularly useful for serverless and cloud-native applications, where lower startup times and lower memory usage can help reduce resource utilization, improve scalability and reduce overall costs.

> Note: the artifact produced by the GraalVM is platform dependent, that means you won't be able to run it on a platform with different architure or OS.

There are some minor inconveniences of using GraalVM and assembiling standalone executables:
* Longer build time (depending on machine and application size it can be from ~2 minute to ~10 minutes)
* GraalVM is not directly aware of dynamic elements of your code and must be hinted about reflection, dynamic proxies, any resources, and serialization. (Spring Boot plugin will try to do most of the job for you, but sometimes you have to provide hints on your own)
* GraalVM is a relatively new technology, and there may be limited tooling and resources available to help developers debug and troubleshoot issues.

If you are not scaried of these inconveniences let's move forward, and try to understand what is Spring Ahead-of-Time Processing and how it simplifies our interaction with native-image assamblation.

<!--TODO explain AOT START-->
Taking in consideration the last point and the fact that we are using Spring, it would be extremly hard to build a native executable binary for an appliaction that uses Spring Boot, since Spring havely relyes on reflection, Dynamic Proxies, etc.

But thanks to Spring Ahead-of-Time Processing that converts all of these to proper java classes with pre-processed classes.
<!--TODO explain AOT END-->


## What is SAM and why do we need it ?
The AWS SAM (Serverless Application Model) is a framework for building serverless applications within AWS, that defacto is a subset of [CloudFormation](https://docs.aws.amazon.com/cloudformation/index.html) that allows you to define and configure all necessary resources for the lambda, i.e. :
* Detailed resources configuration like, timeout, memory, ephemeral storage, concurrency)
* Triggers of the lambda like HTTP endpints, SQS message, SNS notification, S3 events, etc.
* AWS permissions for the lambda by referencing existing IAM Policies, IAM Role and Connectors.
* The runtime in which your code will run (i.e. java11, python, nodejs, custom runtime, etc.)
* The deployment overhead is removed since all action on the resource such as create/update/delete are handled by SAM and we are just reponsible to update our code/configiraton and trigger the deployment.

>Note: A really nice feature of SAM is that we can emulate AWS Lambda environment locally for debugging and testing purposes by using `sam local invoke`.


#### Custom runtime explained
At the moment AWS Lambda supports 7 different runtimes: 
1. Java
2. Python
3. Node.js
4. .NET
5. Go
6. Ruby
7. Custom Runtime

> Note: The Custom Runtime allows you to implement an AWS Lambda runtime in any programming language by following a simple specification.

Since the final artifact produced by the GraalVM native-image is a binary executable file that is not in the list of supported runtimes, we'll have to build our own custom runtime (actually we don't have to since it's implemented as part of of [spring-cloud-function-adapter-aws](https://github.com/spring-cloud/spring-cloud-function/tree/main/spring-cloud-function-adapters/spring-cloud-function-adapter-aws), but let's try to understand how it works, since it can be helpful to understand how your program runs).


<!--TODO add diagram and explain-->

## Creating AWS Lambda with Spring boot
### Code part (todo rename)
### SAM Part
### deployment part
### testing part
## It's time for automation (CI/CD workflows)


## Conclusion

## Refence links
* [Spring Native Docs](https://docs.spring.io/spring-boot/docs/current/reference/html/native-image.html)
* [AWS SAM documentation](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html)
* [link1]()