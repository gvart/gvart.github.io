+++
author = "Vladlen Gladis"
title = "AWS Lambda using Spring Boot 3 and GraalVM"
date = "2023-02-06"
description = "How to build an AWS Lambda using Spring Boot 3 with GraalVM and custom runtime."
featured = true
toc = true
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
GraalVM is a JDK created by Oracle which is designed to accelerate the execution of applications written in Java and other JVM languages, that support `JVM Runtime Mode`, `Java on Truffle` and `Native Image`, for our use-case we are interested in the latest feature.

The GraalVM [native image](https://www.graalvm.org/22.0/reference-manual/native-image/) is an alterantive way to build and run your Java application with a smaller memory footprint and with much faster startup time, which are the key features for the application that runs inside AWS Lambda.

There are some minor downsides of using GraalVM with the
* Longer build times (depending on machine and application you are trying to build and it can be from ~2 minute to ~10 minutes), but a good think is we don't need to build a native image every time we do a change, and can simply compile or java classes for local development
* GraalVM is not directly aware of dynamic elements of your code and must be hinted about reflection, dynamic proxies, any resources, and serialization. (Spring Boot will try to do most of the job for you, but sometimes you have to provide hints on your own)


Taking in consideration the last point and the fact that we are using Spring, it would be extremly hard to build a native executable binary for an appliaction that uses Spring Boot, since Spring havely relyes on reflection, Dynamic Proxies, etc.

But thanks to Spring Ahead-of-Time Processing that converts all of these to proper java classes with pre-processed classes.




> Note: the artifact produced by the GraalVM is platform dependent, that means you won't be able to run it on a platform with different architure or OS.

## What is SAM and why do we need it ?
####
#### Custom runtime explained
explained
## Creating AWS Lambda with Spring boot
### Code part (todo rename)
### SAM Part
### deployment part
### testing part
## It's time for automation (CI/CD workflows)
sadasd


## Conclusion

## Refence links
* [Spring Native Docs](https://docs.spring.io/spring-boot/docs/current/reference/html/native-image.html)

* [link1]()