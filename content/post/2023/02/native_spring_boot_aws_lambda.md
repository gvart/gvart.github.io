+++
author = "Vladlen Gladis"
title = "Deep dive into AWS Lambda using Spring Boot 3 and GraalVM"
date = "2023-02-24"
description = "Building AWS Lambda with Spring Boot 3 and GraalVM Native Image: A Comprehensive Guide."
featured = true
toc = true
draft = false
keywords = [
    "AWS Lambda",
    "Spring Boot",
    "GraalVM",
    "Native Image",
    "Custom Runtime",
    "SAM",
    "Testing",
    "GitHub Action",
    "Serverless"
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
featureImage = "/images/post1_native_aws_lambda/thumbnail.png"
thumbnail = "/images/post1_native_aws_lambda/thumbnail.png"
+++

Serverless computing has become increasingly popular in recent years, and AWS Lambda is one of the most
widely used serverless platforms. Spring Boot, on the other hand, is a popular framework for 
building microservices and web applications in Java. By combining these two technologies, 
developers can create powerful and scalable serverless applications.

In this comprehensive guide, we'll dive into the details of building an AWS Lambda function with Spring Boot 3
and GraalVM Native Image. We'll explore how to use GraalVM Native Image to create a custom runtime for our
Lambda function, which can result in faster startup times and lower memory usage. Additionally, 
we'll cover how to use the AWS Serverless Application Model (SAM) to deploy our Lambda function to AWS, 
how to test our function locally, and how to automate the deployment process with GitHub Actions.

By the end of this guide, you'll have a solid understanding of how to build and deploy efficient and scalable serverless applications with Spring Boot and AWS Lambda.
<!--more-->

## Foreword
This blog post will guide you through the steps of building a production-ready native AWS Lambda function that receives batch messages from an SQS and writes them into a DynamoDB table. It will explain all the details along the way.

{{% notice info "To DIY you will need" %}}
* Basic understanding of Spring and Spring Boot
* Java 17
* Docker
* AWS SAM ([Installation guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html))
* GraalVM (for local testing [Tutorial how to install it locally](/tutorial/2023/manage_multiple_jdk/)) *optional* (you may need it to build a native image locally)
* AWS Account *optional* (you may need it if you want to deploy lambda to AWS)
* GitHub Repository *optional* (you may need it if you want to automate deployments to AWS using GitHub Actions workflows)
{{% /notice %}}

## What is a Native Image, GraalVM, Spring Boot AOT processing, and How Do They Connect?
GraalVM is a universal VM developed by Oracle JDK that is designed to accelerate the execution of applications written in Java and other JVM languages, supporting `JVM Runtime Mode`, `Java on Truffle`, and `Native Image`, with the latter being the focus of this blog post.

### GraalVM
The GraalVM [native image](https://www.graalvm.org/22.0/reference-manual/native-image/) is a component of GraalVM that ahead-of-time (AOT) compiles Java code into a standalone executable, which can be executed without a JVM. This allows for faster startup times and a lower memory footprint compared to traditional JVM-based applications. Native-Image is particularly useful for serverless and cloud-native applications, where lower startup times and lower memory usage can help reduce resource utilization, improve scalability, and reduce overall costs.

> **Note:** The artifact produced by the GraalVM is platform-dependent, meaning you won't be able to run it on a platform with a different architecture or OS.

There are some minor inconveniences of using GraalVM and assembling standalone executables:
* Longer build time (depending on machine and application size, it can take anywhere from ~2 minutes to ~10 minutes)
* GraalVM is not directly aware of the dynamic elements of your code and must be hinted about reflection, dynamic proxies, any resources, and serialization. (The Spring Boot plugin will try to do most of the job for you, but sometimes you have to provide hints on your own.)
* GraalVM is a relatively new technology, and there may be limited tooling and resources available to help developers debug and troubleshoot issues.

If you're not scared of these inconveniences, let's move forward and try to understand what Spring Ahead-of-Time Processing is and how it simplifies our interaction with native-image assembly.


### Spring Ahead-of-Time Processing
<!--Explain AOT START-->
Most of us know that the magic hidden behind the scenes of the Spring Framework heavily relies on reflection and Dynamic Proxies. However, this can be a complete nightmare when it comes to building a native executable binary using GraalVM. Fortunately, the [Spring Native project](https://github.com/spring-projects-experimental/spring-native) (which became part of Spring Boot) pre-processes all dynamically accessed and created classes using Ahead-of-Time Processing.

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

This is a simple configuration class that defines a single bean. In a JVM runtime, Spring would create a proxy object for the `UseCaseConfig` class and attempt to retrieve a bean instance by calling the `processNoteRequest` method. This method would return either the same instance or a new one based on the bean's scope. When a bean is created, Spring uses reflection to perform dependency injection, call the `init` method and perform other initialization tasks.

For each method annotated with the `@Bean` annotation, Spring creates a Bean Definition, which provides instructions on how the bean should be assembled. The configuration class itself also requires a Bean Definition.

To sum up, the configuration above will provide three configuration points during the runtime (:exclamation: keep in mind that all these objects are created during application startup):
1. A `BeanDefinition` for the `UseCaseConfig` class.
2. A `BeanDefinition` for the `ProcessNoteRequest` class.
3. A proxy on top of the `processNoteRequest` method that will instantiate this bean.


Now adding the `org.springframework.boot` plugin from version **3.0.0** and above, add additional steps to our default compilation tasks, which include AOT processing.

After a successful build of the projects the above class will be transformed in the following Java Configuration class(:exclamation: these classes are generated during the build time and later are used by GraalVM to build the final native artifact):

```java
/** Bean definitions for {@link UseCaseConfig} */
public class UseCaseConfig__BeanDefinitions {
    /** Get the bean definition for 'useCaseConfig' */
    public static BeanDefinition getUseCaseConfigBeanDefinition() {
        Class<?> beanType = UseCaseConfig.class;
        RootBeanDefinition beanDefinition = new RootBeanDefinition(beanType);
        ConfigurationClassUtils.initializeConfigurationClass(UseCaseConfig.class);
        beanDefinition.setInstanceSupplier(UseCaseConfig$$SpringCGLIB$$0::new);
        return beanDefinition;
    }

    /** Get the bean instance supplier for 'processNoteRequest'. */
    private static BeanInstanceSupplier<ProcessNoteRequest> getProcessNoteRequestInstanceSupplier() {
        return BeanInstanceSupplier.<ProcessNoteRequest>forFactoryMethod(
                        UseCaseConfig.class,
                        "processNoteRequest",
                        ReadSqsMessageBody.class,
                        SaveNoteRequest.class)
                .withGenerator(
                        (registeredBean, args) ->
                                registeredBean
                                        .getBeanFactory()
                                        .getBean(UseCaseConfig.class)
                                        .processNoteRequest(args.get(0), args.get(1)));
    }

    /** Get the bean definition for 'processNoteRequest' */
    public static BeanDefinition getProcessNoteRequestBeanDefinition() {
        Class<?> beanType = ProcessNoteRequest.class;
        RootBeanDefinition beanDefinition = new RootBeanDefinition(beanType);
        beanDefinition.setInstanceSupplier(getProcessNoteRequestInstanceSupplier());
        return beanDefinition;
    }
}
```

As a result, we can see that we have the exact 3 things we've mentioned before:
1. A `BeanDefinition` for the UseCaseConfig class: `getUseCaseConfigBeanDefinition`.
2. A `BeanDefinition` for the ProcessNoteRequest class: `getProcessNoteRequestBeanDefinition`.
3. A supplier that instantiates `ProcessNoteRequest`: `getProcessNoteRequestInstanceSupplier`.


{{% notice info "In a nutshell" %}}
Spring Ahead-of-Time Processing takes all the components that will be created and instantiated during runtime by Spring's magic and transforms them into plain Java configurations.
{{% /notice %}}


With this understanding, it's clear why Spring Ahead-of-Time Processing is so important for an application built with GraalVM. Without it, all the annotation-based configurations would need to be manually migrated to Java configurations by developers.
<!--Explain AOT END-->


## What is SAM and why do we need it?
The AWS SAM (Serverless Application Model) is a framework for building serverless applications within AWS, and it is essentially a subset of [CloudFormation](https://docs.aws.amazon.com/cloudformation/index.html). Using SAM, you can define and configure all the necessary resources for the lambda, including:
* **Resource Configuration**: Detailed configuration of resources, such as timeout, memory, ephemeral storage, concurrency, and more.
* **Triggers**: Configuring triggers for the lambda, such as HTTP endpoints, SQS messages, SNS notifications, S3 events, etc.
* **Permissions**: Configuring AWS permissions for the lambda by referencing existing IAM policies, IAM roles, and connectors.
* **Runtime**: Specifying the runtime in which your code will run (e.g. Java 11, Python, Node.js, custom runtime, etc.).
* **Deployment**: Handling deployment automatically by SAM, thereby eliminating deployment overhead. You just need to update your code and configuration, and trigger the deployment.

>**Note:** SAM offers a nice feature that allows you to emulate the AWS Lambda environment locally for debugging and testing purposes using the `sam local invoke` command.

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

> **Note:** The Custom Runtime allows you to implement an AWS Lambda runtime in any programming language by following a simple specification.

The GraalVM native-image produces a binary executable file that is not supported by default runtimes. However, it is possible to run it using custom runtime. The custom runtime is already implemented as part of [spring-cloud-function-adapter-aws](https://github.com/spring-cloud/spring-cloud-function/tree/main/spring-cloud-function-adapters/spring-cloud-function-adapter-aws), but understanding how it works can be helpful. Therefore, we will explore it further.


To create a custom runtime, you simply need to adhere to a straightforward specification that outlines how the runtime should interact with AWS Lambda. By doing so, you gain the ability to construct your runtime using your preferred programming language, which grants you a more flexible approach to constructing serverless applications.


1. The uploaded artifact should include a script called `bootstrap`, which serves as the entry point for our AWS Lambda function. This script will be executed to create the runtime.
    
   **Bootstrap script example**:
    ```shell
    #!/bin/sh
    cd $LAMBDA_TASK_ROOT
    ./application
    ```

**To visualize the flow, we can use this flow diagram (see explanation below)**
![AWS Custom runtime Flow Diagram::polaroid](/images/post1_native_aws_lambda/flow_diagram_custom_runtime.png)

2. When the application starts up, it needs to perform several initialization steps to ensure it is ready to handle incoming requests efficiently:
   1. Load its configuration from environment variables: `_HANDLER`, `LAMBDA_TASK_ROOT`, `AWS_LAMBDA_RUNTIME_API`. *More information on how to do this can be found in the [documentation](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html#runtimes-custom-build:~:text=Initialization%20tasks-,Retrieve%20settings,-%E2%80%93%20Read%20environment%20variables).*
   2. Initialize all heavy resources such as JDBC connections, SDK clients, etc. which can be reused across multiple invocations. This helps to avoid the overhead of repeatedly creating and destroying these resources with each request.
   3. Create a global error handler that calls the Lambda's API and immediately exits (This is an essential step to ensure that any unhandled errors are reported to the Lambda's API, allowing the appropriate measures to be taken).
3. Once the initialization tasks are completed, the custom AWS Lambda runtime enters a loop to process incoming events. The following steps are executed during each iteration of the loop:
   1. The runtime tries to retrieve an incoming event from the **Next Invocation** API. *If there is no response, the runtime exits.*
   2. The function handler is invoked. *If the invocation fails, the runtime calls the **Invocation error** API to report the failure.*
   3. The runtime returns a response by calling the **Invocation response** API.
   4. After completing the invocation, the runtime cleans up all the resources used in the current iteration. *This is important to prevent any resource leaks and ensure that the runtime is ready to handle the next incoming event.*


## Building an AWS Lambda with Spring Boot

> To find the complete code check out [GitHub repository](https://github.com/gvart/every-note-persister), in this section we will cover only the main parts.

For this implementation, we will focus on a common use case where AWS Lambda is triggered by a message from Amazon Simple Queue Service (SQS), performs some business logic on the message, and then persists the modified message in Amazon DynamoDB.
![Project HLD::polaroid](/images/post1_native_aws_lambda/aws_lambda_hld.png)
### Writing the code
To prepare the runtime for the AWS Lambda, we will need to add the following plugins to the project:
```kotlin
plugins {
    java
    id("org.springframework.boot") version "$springBootVersion"
    id("org.graalvm.buildtools.native") version "$nativeBuildToolsVersion"
}
```
> `org.graalvm.buildtools.native` plugin is needed to create native executable.

And dependencies:
```kotlin
dependencies {
    implementation(platform("org.springframework.cloud:spring-cloud-dependencies:$springCloudVersion"))
    implementation(platform("io.awspring.cloud:spring-cloud-aws-dependencies:$springCloudAwsVersion"))

    implementation("io.awspring.cloud:spring-cloud-aws-starter-dynamodb")
    implementation("org.springframework.cloud:spring-cloud-starter-function-web")
    implementation("org.springframework.cloud:spring-cloud-function-adapter-aws")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```
> 1. `spring-cloud-starter-function-web` provides the runtime for the function we'll be creating.
> 2. `spring-cloud-starter-function-adapter-aws` adjusts existing runtime and allows to use custom-runtime specification described earlier.


To define a lambda handler (which serves as the entry point to the application), a bean of type `Consumer<T>`, `Function<T,R>`, or `Supplier<T>` needs to be registered in the Spring context. 
> If we have multiple beans that implement any of these functional interfaces, the `spring.cloud.function.definition` property in the application.properties or application.yml file must be explicitly configured.

For the given use case where a Lambda function consumes an event from SQS and saves it in DynamoDB, the implementation should use the `Consumer<SqsEvent>` interface. Here, `SqsEvent` represents an SQS object that can contain from 1 to 10 messages.
```java
@Component
public class LambdaFunction implements Consumer<SqsEvent> {

    private static final Logger log = LoggerFactory.getLogger(LambdaFunction.class);

    private final ProcessNoteRequest processNoteRequest;

    public LambdaFunction(ProcessNoteRequest processNoteRequest) {
        this.processNoteRequest = processNoteRequest;
    }

    @Override
    public void accept(SqsEvent sqsEvent) {
        log.info("Received event: {}", sqsEvent);

        sqsEvent.records()
                .stream()
                .map(SqsMessage::body)
                .forEach(processNoteRequest::process);
    }
}
```

The `ProcessNoteRequest` implementation reads the message body and saves it in DynamoDB.
```java
public class ProcessNoteRequest {

    private final ReadSqsMessageBody readSqsMessageBody;
    private final SaveNoteRequest saveNoteRequest;

    public ProcessNoteRequest(ReadSqsMessageBody readSqsMessageBody, SaveNoteRequest saveNoteRequest) {
        this.readSqsMessageBody = readSqsMessageBody;
        this.saveNoteRequest = saveNoteRequest;
    }

    public void process(String messageBody) {
        var result = readSqsMessageBody.read(messageBody);
        saveNoteRequest.save(result);
    }
}
```

The `ReadSqsMessageBody` implementation uses Jackson ObjectMapper to read String message body and convert it to a DTO object.
```java
@Component
public class DefaultReadSqsMessageBody implements ReadSqsMessageBody {

    private final Logger log = LoggerFactory.getLogger(DefaultReadSqsMessageBody.class);

    private final ObjectMapper mapper;

    public DefaultReadSqsMessageBody(ObjectMapper mapper) {
        this.mapper = mapper;
    }

    @Override
    public PersistNoteRequest read(String body) {
        try {
            return mapper.readValue(body, PersistNoteRequest.class);
        } catch (JsonProcessingException exception) {
            log.error("Failed to read value: {}", body, exception);
            throw new RuntimeException("Failed to process json.");
        }
    }
}
```

The `SaveNoteRequest` implementation applied some changes on provided DTO using `RequestTransformer` instance, and finally object is saved in a DynamodbDB table.
```java
@Component
public class DefaultSaveNoteRequest implements SaveNoteRequest {

    private final Logger log = LoggerFactory.getLogger(DefaultSaveNoteRequest.class);

    private final DynamoDbTemplate template;
    private final RequestTransformer<PersistNoteRequest, Note> transformer;

    public DefaultSaveNoteRequest(DynamoDbTemplate template, RequestTransformer<PersistNoteRequest, Note> transformer) {
        this.template = template;
        this.transformer = transformer;
    }

    @Override
    public void save(PersistNoteRequest request) {
        log.info("Persisting {}", request);

        var dbEntry = transformer.transform(request);
        template.save(dbEntry);
    }
}
```

The classes mentioned above are quite simple and do not require any special code for the native-image. It's a typical Spring Boot application that we create regularly. This code is almost ready to be deployed to AWS Lambda runtime, but before that, we need to provide some reflection hints to GraalVM. This is because the following operations use reflection:
* `ObjectMapper` is used internally by Spring Cloud Function to create `SqsEvent` and `SqsMessage` objects.
* `ObjectMapper#readValue` uses reflection to access our DTO object `PersistNoteRequest`
* The internals of `DynamoDbTemplate#save` also use reflection to persist our message `Note`.

To provide these hints, we can put all the details in a `/META-INF/native-image/reflect-config.json` file. Alternatively, we can use an annotation provided by Spring Boot, which is `@RegisterReflectionForBinding`, to provide a type-safe way of doing this. This is the last step we need to take, and the following code accomplishes it:
```java
@Configuration
@RegisterReflectionForBinding({
        PersistNoteRequest.class,
        SqsMessage.class,
        SqsEvent.class,
        Note.class
})
public class NativeConfig { }
```

After these steps, our code is ready to be compiled into a native image that can be deployed to AWS Lambda runtime.
### Configuring SAM (Serverless Application Model)

The main SAM file is `template.yaml`, which defines all the resources related to the Lambda function.<br> Using SAM, we will define the following:
* Custom runtime via `Runtime` parameter
* An AWS Lambda function named `EveryNotePersisterFunction`
    * An SQS trigger for that function called `SQSEvent`
    * Build instructions via the `BuildMethod` parameter (this is necessary to guide SAM on how to assemble our application)
* An SQS queue named `MainSqs`
* A simple DynamoDB table with a PK named `NoteTable`

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 20
    Runtime: provided.al2 
    MemorySize: 256
    Architectures:
      - x86_64
    Tracing: Active # https://docs.aws.amazon.com/lambda/latest/dg/lambda-x-ray.html

Resources:
  EveryNotePersisterFunction:
    Type: AWS::Serverless::Function 
    Connectors: # <- This section grants permissions to Read and Write to the DynamoDB Table
      NoteTableConnector:
        Properties:
          Destination:
            Id: NoteTable
          Permissions:
            - Read
            - Write

    Properties:
      CodeUri: .
      Handler: none
      Events:
        SQSEvent: 
          Type: SQS
          Properties:
            Queue: !GetAtt MainSqs.Arn
            BatchSize: 10
            Enabled: true
    Metadata:
      BuildMethod: makefile

  MainSqs:
    Type: AWS::SQS::Queue

  NoteTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      TableName: note
      PrimaryKey:
        Name: id
        Type: String
```
To provide instructions for SAM, we need to create a **Makefile** that will clean the build directory, compile the native
executable file, create the previously mentioned `bootstrap` script and place everything in the designated **ARTIFACTS_DIR** location. 
```makefile
build-EveryNotePersisterFunction:
	rm -rf ./build
	./gradlew clean
	./gradlew nativeCompile
	echo '#!/bin/sh' > ./build/bootstrap
	echo 'set -euo pipefail' >> ./build/bootstrap
	echo './application' >> ./build/bootstrap
	chmod 777 ./build/bootstrap
	cp ./build/native/nativeCompile/application $(ARTIFACTS_DIR)
	cp ./build/bootstrap $(ARTIFACTS_DIR)
```

The final step is to create a build environment for the Lambda function. 
Since the build artifact will run in the Amazon Linux 2 operating system, we need to ensure that the artifact can be built and executed on `provided.al2`. 
To do so, all build steps will happen inside a Docker container with the base image  `public.ecr.aws/amazonlinux/amazonlinux:2`
> **Note:**  This file was generated by the SAM CLI and has been modified to make it easier to manage versions by exposing the GraalVM, Java, and Gradle versions as arguments.
```Dockerfile
FROM public.ecr.aws/amazonlinux/amazonlinux:2

RUN yum -y update \
    && yum install -y unzip tar gzip bzip2-devel ed gcc gcc-c++ gcc-gfortran \
    less libcurl-devel openssl openssl-devel readline-devel xz-devel \
    zlib-devel glibc-static libcxx libcxx-devel llvm-toolset-7 zlib-static \
    && rm -rf /var/cache/yum

# Graal VM
ARG JAVA_VERSION
ARG GRAAL_VERSION
ENV GRAAL_FOLDERNAME graalvm-ce-${JAVA_VERSION}-${GRAAL_VERSION}
ENV GRAAL_FILENAME graalvm-ce-${JAVA_VERSION}-linux-amd64-${GRAAL_VERSION}.tar.gz
RUN curl -4 -L https://github.com/graalvm/graalvm-ce-builds/releases/download/vm-${GRAAL_VERSION}/${GRAAL_FILENAME} | tar -xvz
RUN mv $GRAAL_FOLDERNAME /usr/lib/graalvm
RUN rm -rf $GRAAL_FOLDERNAME

# Gradle
ARG GRADLE_VERSION
ENV GRADLE_FOLDERNAME gradle-${GRADLE_VERSION}
ENV GRALE_FILENAME ${GRADLE_FOLDERNAME}-bin.zip
RUN curl -L https://services.gradle.org/distributions/${GRALE_FILENAME} > $GRALE_FILENAME
RUN unzip -o $GRALE_FILENAME
RUN mv $GRADLE_FOLDERNAME /usr/lib/gradle
RUN rm $GRALE_FILENAME
ENV PATH=$PATH:/usr/lib/gradle/bin

# AWS Lambda Builders
RUN amazon-linux-extras enable python3.8
RUN yum clean metadata && yum -y install python3.8
RUN curl -L get-pip.io | python3.8
RUN pip3 install aws-lambda-builders

VOLUME /project
WORKDIR /project

RUN /usr/lib/graalvm/bin/gu install native-image
RUN ln -s /usr/lib/graalvm/bin/native-image /usr/bin/native-image

ENV JAVA_HOME /usr/lib/graalvm

ENTRYPOINT ["sh"]
```

### Deploying the Lambda function
> **Note:** Before proceeding with deployment, make sure that:
> 1. Docker has at least 4GB RAM.
> 2. AWS credentials are properly configured on your local environment.
> 3. You have sufficient permissions to create such resources as CloudFormation, S3, SQS, Lambda, and DynamoDB.

1. Build the Docker image that SAM will use to assemble the native-image executable file by running the `./build-image.sh` script
   ```shell
    #!/bin/sh
    set -e
    
    JAVA_VERSION=java17
    GRAAL_VERSION=22.3.0
    GRADLE_VERSION=7.6
    
    docker build --build-arg GRADLE_VERSION=$GRADLE_VERSION \
    --build-arg JAVA_VERSION=$JAVA_VERSION \
    --build-arg GRAAL_VERSION=$GRAAL_VERSION \
    -t al2-graalvm:gradle .
    ```
2. Execute `sam build --use-container` command to build a deployable artifact. It will use the Docker container built in step 1 together with the **Makefile** created previously.
3. Now artifact can be deployed (Don't forget to replace `<AWS_REGION>` placeholder).
   ```shell
    sam deploy \
    --no-confirm-changeset \
    --no-fail-on-empty-changeset \
    --resolve-s3 \
    --region <AWS_REGION> \
    --capabilities CAPABILITY_IAM \
    --stack-name every-note-persister
   ```

### Testing the Lambda function
To test created lambda function, I found several options that can be applied separately or combined for different situations.

1. **Using Integration tests with LocalStack and TestContainers (JVM)**

   To test our application, we can use old good unit tests or integration tests. However, since our application is not aware of the trigger, such as an SQS message, an API Gateway request, or an SNS notification, we won't be able to create a complete integration test. Fortunately, we can leverage Spring Cloud Function, which provides a utility class that emulates the APIs provided by Amazon for the lambdas with custom-runtime. 

   The [AWSCustomRuntime](https://github.com/spring-cloud/spring-cloud-function/blob/main/spring-cloud-function-adapters/spring-cloud-function-adapter-aws/src/main/java/org/springframework/cloud/function/adapter/test/aws/AWSCustomRuntime.java) can be useful for creating a complete integration test with AWS custom runtime. Additionally, we will need to use LocalStack to run DynamoDB locally and test the complete execution flow. 
   > **Before listing the steps of the test with AWSCustomRuntime, it is important to note a few important things:**
   >1. We need to add `AWSCustomRuntime` to the application context.
   >2. We need to use a real `webEnvironment` since it is required by the `AWSCustomRuntime`.
   >3. We need to specify explicitly the function name (via `spring.cloud.function.definition` property) that will be used as a handler because `AWSCustomRuntime` defines additional beans that implement functional interfaces.
   >4. We need to provide a property with the key `_HANDLER`. This is necessary for the AWS adapter to realize that the CustomRuntimeEventLoop should be started.


   With all of this in mind, we can create an integration test that will send an event to the `CustomRuntime` (which will later be pulled by our function), and we expect that within 5 seconds, we will have persisted the event in DynamoDB.
   ```java
   @ExtendWith(DynamoDbExtension.class)
   @SpringBootTest(
           classes = {Bootstrap.class, AWSCustomRuntime.class},
           webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
           properties = {
                   "spring.cloud.function.definition=lambdaFunction",
                   "_HANDLER=lambdaFunction"
           })
   class LambdaFunctionTest {

       // Some setup and dependency injection

       @Test
       void Test() throws IOException {
           //Given
           var sqsEvent = Files.readString(event);

           //When message is sent to custom-runtime, and it gets consumed by handler
           customRuntime.exchange(sqsEvent);

           //Then item should be saved in database
           await().atMost(Duration.ofSeconds(5)).until(itemSavedInDatabase());

       }

       private Callable<Boolean> itemSavedInDatabase() {
           return () -> {
               var items = table.scan().items().stream().toList();
               if (items.size() == 1) {
                   var savedItem = items.get(0);
                   assertThat(savedItem.getId()).isNotNull();
                   assertThat(savedItem.getCreatedAt()).isNotNull();
                   assertThat(savedItem.getNoteBody()).isEqualTo("message body");

                   return true;
               }
               //exact one item wasn't saved
               return false;
           };
       }
   }
   ```
   *I would recommend using this approach as part of the quality check before deployment, which should be a part of the CI/CD process.*

2. **Run locally as a Spring Boot application (JVM)**

   Once the application is created, it can be run as a regular Spring Boot application. On startup, the created Lambda handler is exposed as an HTTP endpoint that can be invoked at  [http://localhost:8080/lambdaFunction](http://localhost:8080/lambdaFunction). The `lambdaFunction` is the bean name which is used as the entry point for our application.

   To emulate SQS message receiving, we can use the following `curl` call:
    ```shell
    curl --location --request POST 'http://localhost:8080/lambdaFunction' \
    --header 'Content-Type: application/json' \
    --data-raw '{
        "Records": [
            {
                "messageId": "19dd0b57-b21e-4ac1-bd88-01bbb068cb78",
                "receiptHandle": "MessageReceiptHandle",
                "body": "{\"body\":\"hello world\"}",
                "attributes": {
                    "ApproximateReceiveCount": "1",
                    "SentTimestamp": "1523232000000",
                    "SenderId": "123456789012",
                    "ApproximateFirstReceiveTimestamp": "1523232000001"
                },
                "messageAttributes": {},
                "md5OfBody": "7b270e59b47ff90a553787216d55d91d",
                "eventSource": "aws:sqs",
                "eventSourceARN": "arn:aws:sqs:eu-west-1:156445482446:TestQueue",
                "awsRegion": "eu-west-1"
            }
        ]
    }'
    ```
    *I would recommend using this approach when it comes to manual testing and debugging.*
3.  **Test on AWS (native-image)**

    To test a deployed lambda, we can send a message to SQS since that is the trigger for the created lambda function. Afterward, we can verify the DynamoDB table to ensure that the message was saved.

    Send a message to SQS using CLI:
    ```shell
    aws sqs send-message \
    --queue-url=https://sqs.eu-west-1.amazonaws.com/<ACCOUNT-ID>/<QUEUE-NAME> \
    --message-body='{"body":"my message"}'
    ```
    > **Note:** Don't forget to replace `<ACCOUNT-ID>` and `<QUEUE-NAME>` placeholders with values that correspond to your account and created queue name.

    After that, we should check the Lambda's monitor tab to ensure that the lambda was executed successfully.
    ![Lambda Monitor screen::polaroid](/images/post1_native_aws_lambda/cloud_watch_monitor.png)
    We should also check that there is an entry in DynamoDB with the posted message.
    ![Dynamodb items::polaroid](/images/post1_native_aws_lambda/dynamodb_item.png)
    
    *I would recommend using this approach when we want to perform end-to-end (E2E) tests in a pre-production account or to perform load/performance tests within a real AWS environment, all these steps can be automated and also introduced as part of CI/CD.*

## It's time for automation (GitHub CI/CD workflows)
We all like to be lazy and delegate our work to someone else, so let's delegate the building, testing, and deployment to a GitHub workflow.
>**Note:** The workflows described below are pretty simple and can be adjusted based on your requirements.

I've created two workflows.
The first one is triggered when a pull request is opened, and GitHub Actions will try to build our application, run tests, and finally build a native executable using SAM:
```yaml
on:
  pull_request:
    branches:
      - main
jobs:
  build_and_test:
    name: "Build and Test"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - uses: actions/cache@v2
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-
      - uses: actions/setup-java@v3
        with:
          distribution: 'zulu'
          java-version: '17'

      - name: "Gradle Build"
        run: ./gradlew build -x test

      - name: "Gradle Test"
        run: ./gradlew test

  sam_build:
    name: "Build using SAM"
    needs: build_and_test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: "3.8"
      - uses: aws-actions/setup-sam@v2

      - name: "build image if missing"
        run: |
          if [[ "$(docker images -q al2-graalvm:gradle 2> /dev/null)" == "" ]]; then
            echo "Image not found. Building image."
            ./build-image.sh
          else
            echo "Image found. Skipping build."
          fi
      - run: sam build --use-container
```
The second workflow gets triggered when a merge occurs on the **main** branch. It will assume an IAM role, build the native-image (this step can be improved by caching the artifact from the previous step to reduce execution time), and finally deploy our serverless application.
> **Note:** Before running this workflow, please ensure that you have completed the following steps:
> 1. Create Secrets for GitHub Actions with `AWS_REGION` and `AWS_ROLE` which have sufficient permissions to create all resources defined in` template.yml`.
> 2. [Configure the OIDC provider](https://github.com/aws-actions/configure-aws-credentials#assuming-a-role) to allow GitHub to assume the IAM role without any AWS credentials. Alternatively, you can use `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`
> 3. Verify that the role/user you are using for deployment has sufficient permissions. For testing purposes, you can use AdministratorAccess, but it's better to configure a fine-grained IAM policy. You can find more details on how to do this here: https://sst.dev/chapters/customize-the-serverless-iam-policy.html

```yaml
on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: "3.8"
      - uses: aws-actions/setup-sam@v2
      - uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.AWS_ROLE }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: "build image if missing"
        run: |
          if [[ "$(docker images -q al2-graalvm:gradle 2> /dev/null)" == "" ]]; then
            echo "Image not found. Building image."
            ./build-image.sh
          else
            echo "Image found. Skipping build."
          fi

      - run: sam build --use-container
      # Prevent prompts and failure when the stack is unchanged
      - run: |
          sam deploy \
          --no-confirm-changeset \
          --no-fail-on-empty-changeset \
          --resolve-s3 \
          --region ${{ secrets.AWS_REGION }} \
          --capabilities CAPABILITY_IAM \
          --stack-name every-note-persister
```
## Conclusion
In conclusion, building an AWS Lambda using Spring Boot 3 GraalVM native image can provide significant benefits, such as
improved performance and reduced cold start times. By leveraging custom runtime, Spring AOT, SAM, and GitHub actions, 
developers can streamline the process of deploying and testing their serverless applications.

Using code examples, diagrams, and fine-grained configurations can help developers understand the details of building and 
deploying an AWS Lambda using Spring Boot 3 GraalVM native image. Additionally, referencing relevant resources and 
documentation can provide valuable insights and best practices for developers looking to build scalable and efficient 
serverless applications on AWS Lambda.

Overall, the combination of Spring Boot 3 GraalVM native image and AWS Lambda provides a powerful platform for developing 
and deploying serverless applications. By following the steps outlined in this blog post, developers can quickly and easily
create high-performance and scalable serverless applications using modern technologies and tools.

Thank you for taking the time to read this blog post. I hope you found it informative and helpful in your journey of building 
AWS Lambdas with Spring Boot 3 GraalVM native image. 
Be sure to stay tuned for future posts, and I look forward to connecting with you again soon.

## Reference links
* [GitHub repository](https://github.com/gvart/every-note-persister)
* [Spring Native Docs](https://docs.spring.io/spring-boot/docs/current/reference/html/native-image.html)
* [AWS SAM documentation](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html)
* [Configure SAM permissions](https://sst.dev/chapters/customize-the-serverless-iam-policy.html)