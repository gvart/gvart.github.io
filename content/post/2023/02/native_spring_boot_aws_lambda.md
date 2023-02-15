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

By the end of this blog, you'll have a solid understanding of how to create a scalable and efficient serverless application using modern technology. So let's get started!"

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

## What is a native image, GraalVM, Spring Boot AOT rocessing and how do they connect ?
GraalVM is a universal VM developed by Oracle JDK that is designed to accelerate the execution of applications written in Java and other JVM languages, supporting `JVM Runtime Mode`, `Java on Truffle` and `Native Image`, with the latter being the focus of this blog post.

### GraalVM
The GraalVM [native image](https://www.graalvm.org/22.0/reference-manual/native-image/) is a component of GraalVM that ahead-of-time (AOT) compiles Java code into a standalone executable, which can be executed without a JVM. This allows for faster startup times and lower memory footprint compared to traditional JVM-based applications. Native-Image is particularly useful for serverless and cloud-native applications, where lower startup times and lower memory usage can help reduce resource utilization, improve scalability, and reduce overall costs.

> Note: The artifact produced by the GraalVM is platform-dependent, meaning you won't be able to run it on a platform with a different architecture or OS.

There are some minor inconveniences of using GraalVM and assembling standalone executables:
* Longer build time (depending on machine and application size, it can take anywhere from ~2 minutes to ~10 minutes)
* GraalVM is not directly aware of the dynamic elements of your code and must be hinted about reflection, dynamic proxies, any resources, and serialization. (The Spring Boot plugin will try to do most of the job for you, but sometimes you have to provide hints on your own.)
* GraalVM is a relatively new technology, and there may be limited tooling and resources available to help developers debug and troubleshoot issues.

If you're not scared of these inconveniences, let's move forward and try to understand what Spring Ahead-of-Time Processing is and how it simplifies our interaction with native-image assembly.


### Spring Ahead-of-Time Processing
<!--Explain AOT START-->
Most of us know that the magic hidden behind the scenes of the Spring Framework heavily relies on reflection and Dynamic Proxies. However, this can be a complete nightmare when it comes to building a native executable binary using GraalVM. Fortunately, the [Spring Native project](https://github.com/spring-projects-experimental/spring-native)(which became part of Spring Boot) pre-processes all dynamically accessed and created classes using Ahead-of-Time Processing.

To explain what AOT Processing does, let's take a look at a class from the lambda that we'll be creating:

```java
@Configuration
public class UseCaseConfig {

    @Bean
    public ProcessNoteRequest processNoteRequest(
            ReadSqsMessageBody readSqsMessageBody,
            SaveNoteRequest saveNoteRequest
    ) {
        return new ProcessNoteRequest(readSqsMessageBody, saveNoteRequest);
    }
}
```

This is a simple configuration class that defines a single bean. In a JVM runtime, Spring would create a proxy object for the `UseCaseConfig` class and attempt to retrieve a bean instance by calling the `processNoteRequest` method. This method would return either the same instance or a new one based on the bean's scope. When a bean is created, Spring uses reflection to perform dependency injection, call the `init` method, and perform other initialization tasks.

For each method annotated with the `@Bean` annotation, Spring creates a Bean Definition, which provides instructions on how the bean should be assembled. The configuration class itself also requires a Bean Definition.

To sum up, the configuration above will provide three configuration points during the runtime (:exclamation: keep in mind that all these objects are created during application startup):
1. A `BeanDefinition` for the UseCaseConfig class.
2. A `BeanDefinition` for the ProcessNoteRequest class.
3. A proxy on top of `processNoteRequest` method that will instantiate this bean.


Now my adding a `org.springframework.boot` plugin from version **3.0.0** and above, adds aditional steps to our default compilation tasks, which includes AOT processing.

After a successful build of the projects the above class will be tranformed in the following Java Configuration class(:exclamation: these classes are generated during the build time and later are used by GraalVM to build final native artifact.):

```java
public class UseCaseConfig__BeanDefinitions {
    public UseCaseConfig__BeanDefinitions() {
    }

    public static BeanDefinition getUseCaseConfigBeanDefinition() {
        Class<?> beanType = UseCaseConfig.class;
        RootBeanDefinition beanDefinition = new RootBeanDefinition(beanType);
        ConfigurationClassUtils.initializeConfigurationClass(UseCaseConfig.class);
        beanDefinition.setInstanceSupplier(UseCaseConfig..SpringCGLIB..0::new);
        return beanDefinition;
    }

    private static BeanInstanceSupplier<ProcessNoteRequest> getProcessNoteRequestInstanceSupplier() {
        return BeanInstanceSupplier.forFactoryMethod(UseCaseConfig.class, "processNoteRequest", new Class[]{ReadSqsMessageBody.class, SaveNoteRequest.class}).withGenerator((registeredBean, args) -> {
            return ((UseCaseConfig)registeredBean.getBeanFactory().getBean(UseCaseConfig.class)).processNoteRequest((ReadSqsMessageBody)args.get(0), (SaveNoteRequest)args.get(1));
        });
    }

    public static BeanDefinition getProcessNoteRequestBeanDefinition() {
        Class<?> beanType = ProcessNoteRequest.class;
        RootBeanDefinition beanDefinition = new RootBeanDefinition(beanType);
        beanDefinition.setInstanceSupplier(getProcessNoteRequestInstanceSupplier());
        return beanDefinition;
    }
}
```

As a result we can see that we have the exact 3 things we've mentioned before:
1. A `BeanDefinition` for the UseCaseConfig class: `getUseCaseConfigBeanDefinition`.
2. A `BeanDefinition` for the ProcessNoteRequest class: `getProcessNoteRequestBeanDefinition`.
3. A supplier that instantiate `ProcessNoteRequest`: `getProcessNoteRequestInstanceSupplier`.


{{% notice info "In a nutshell" %}}
Spring Ahead-of-Time Processing takes all the components that will be created and instantiated during runtime by Spring's magic and transforms them into plain Java configurations.
{{% /notice %}}


With this understanding, it's clear why Spring Ahead-of-Time Processing is so important for an application built with GraalVM. Without it, all the annotation-based configurations would need to be manually migrated to Java configurations by developers.
<!--Explain AOT END-->


## What is SAM and why do we need it ?
The AWS SAM (Serverless Application Model) is a framework for building serverless applications within AWS, and it is essentially a subset of [CloudFormation](https://docs.aws.amazon.com/cloudformation/index.html).Using SAM, you can define and configure all the necessary resources for the lambda, including:
* **Resource Configuration**: Detailed configuration of resources, such as timeout, memory, ephemeral storage, concurrency, and more.
* **Triggers**: Configuring triggers for the lambda, such as HTTP endpoints, SQS messages, SNS notifications, S3 events, etc.
* **Permissions**: Configuring AWS permissions for the lambda by referencing existing IAM policies, IAM roles, and connectors.
* **Runtime**: Specifying the runtime in which your code will run (e.g. Java 11, Python, Node.js, custom runtime, etc.).
* **Deployment**: Handling deployment automatically by SAM, thereby eliminating deployment overhead. You just need to update your code and configuration, and trigger the deployment.

>Note: SAM offers a nice feature that allows you to emulate the AWS Lambda environment locally for debugging and testing purposes using the `sam local invoke` command.

It's worth noting that SAM is based on AWS Lambda and is designed to simplify the process of building and deploying serverless applications. By using SAM, you can save time and reduce complexity, allowing you to focus on developing and refining your application's core features.

{{% notice info "In a nutshell" %}}
SAM is composed of two components. The first is a Command Line Interface (CLI) that allows you to build, create, deploy, and delete Serverless applications on AWS. The second component is a YAML template that defines all the configurations your application consists of.

For example, in the template below, an AWS function is declared that is triggered by an SQS event:
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: >
  sam-app

Globals:
  Function:
    Timeout: 20
    Runtime: java11
    MemorySize: 256
    Architectures:
      - x86_64

Resources:
  MyFunction:
    Type: AWS::Serverless::Function

    Properties:
      CodeUri: .
      Handler: com.gvart.AppHandler.java
      Events:
        SQSEvent:
          Type: SQS
          Properties:
            Queue: my.queue.arn
```
{{% /notice %}}

### Custom runtime explained
AWS Lambda currently supports seven different runtimes, which are:
1. Java
2. Python
3. Node.js
4. .NET
5. Go
6. Ruby
7. Custom Runtime

> Note: The Custom Runtime allows you to implement an AWS Lambda runtime in any programming language by following a simple specification.

While the final artifact produced by the GraalVM native-image is a binary executable file that is not in the list of supported runtimes, we can build our own custom runtime to use it with AWS Lambda. (In fact, the custom runtime is already implemented as part of [spring-cloud-function-adapter-aws](https://github.com/spring-cloud/spring-cloud-function/tree/main/spring-cloud-function-adapters/spring-cloud-function-adapter-aws), but it's helpful to understand how it works, so we'll explore it further.)


Building a custom runtime involves following a simple specification that defines how the runtime should work with AWS Lambda. This allows you to implement your own runtime in any programming language that you prefer, giving you greater flexibility in building serverless applications.

<!--TODO add diagram and explain-->

<!--TODO Explain http steps and bundle-->
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