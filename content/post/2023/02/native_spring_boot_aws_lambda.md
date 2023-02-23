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

## What is a Native Image, GraalVM, Spring Boot AOT processing, and How Do They Connect?
GraalVM is a universal VM developed by Oracle JDK that is designed to accelerate the execution of applications written in Java and other JVM languages, supporting `JVM Runtime Mode`, `Java on Truffle` and `Native Image`, with the latter being the focus of this blog post.

### GraalVM
The GraalVM [native image](https://www.graalvm.org/22.0/reference-manual/native-image/) is a component of GraalVM that ahead-of-time (AOT) compiles Java code into a standalone executable, which can be executed without a JVM. This allows for faster startup times and lower memory footprint compared to traditional JVM-based applications. Native-Image is particularly useful for serverless and cloud-native applications, where lower startup times and lower memory usage can help reduce resource utilization, improve scalability, and reduce overall costs.

> **Note:** The artifact produced by the GraalVM is platform-dependent, meaning you won't be able to run it on a platform with a different architecture or OS.

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


To create a custom runtime, you simply need to adhere to a straightforward specification that outlines how the runtime should interact with AWS Lambda. By doing so, you gain the ability to construct your own runtime using your preferred programming language, which grants you a more flexible approach to constructing serverless applications.


1. The uploaded artifact should include a script called `bootstrap`, which serves as the entry point for our AWS Lambda function. This file will be executed to crete the runtime.

**Bootstrap script example**:
```bash
#!/bin/sh
cd $LAMBDA_TASK_ROOT
./application
```

2. When the application starts up, it needs to perform several initialization steps to ensure it is ready to handle incoming requests efficiently:
   1. Load its configuration from environment variables : `_HANDLER`, `LAMBDA_TASK_ROOT`, `AWS_LAMBDA_RUNTIME_API`). More information on how to do this can be found in the [documentation](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html#runtimes-custom-build:~:text=Initialization%20tasks-,Retrieve%20settings,-%E2%80%93%20Read%20environment%20variables).
   2. Initialize all heavy resources such as JDBC connections,SDK clients, etc. which can be reused across multiple invocations. This helps to avoid the overhead of repeatedly creating and destroying these resources with each request.
   3. Create a global error handler that calls the Lambda's API and immediately exits (This is an essential step to ensure that any unhandled errors are reported to the Lambda's API, allowing the appropriate measures to be taken).
3. Once the initialization tasks are completed, the custom AWS Lambda runtime enters a loop to process incoming events. The following steps are executed during each iteration of the loop:
   1. The runtime tries to retrieve an incoming event from the **Next Invocation** API. *If there is no response, the runtime exits.*
   2. The function handler is invoked. *If the invocation fails, the runtime calls the **Invocation error** API to report the failure.*
   3. The runtime returns a response by calling the **Invocation response** API.
   4. After completing the invocation, the runtime cleans up all the resources used in the current iteration. *This is important to prevent any resource leaks and ensure that the runtime is ready to handle the next incoming event.*

**To visualize the flow, we can use this flow diagram.**
![AWS Custom runtime Flow Diagram::polaroid](/images/post1_native_aws_lambda/flow_diagram_custom_runtime.png)

## Building an AWS Lambda with Spring Boot

> To find the complete code check out [Github repository](https://github.com/gvart/every-note-persister), in this part we will cover only the main parts.

For the purpose of this implementation, we will focus on a common use case where AWS Lambda is triggered by a message from Amazon Simple Queue Service (SQS), performs some business logic on the message, and then persists the modified message in Amazon DynamoDB.
![Project HLD::polaroid](/images/post1_native_aws_lambda/aws_lambda_hld.png)
### Writing the code
To prepare the runtime for the AWS Lambda, we will need to add the following plugins in the project:
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
> 1. `spring-cloud-starter-function-web` provide the runtime for the function we'll be creating.
> 2. `spring-cloud-starter-function-adapter-aws` adjusts existing runtime and allows to use custom-runtime specification described earlier.


To define a lambda handler (which serves as the entry point to the application), a bean of type Consumer<T>, Function<T,R>, or Supplier<T> needs to be registered in the Spring context. 
> If there are multiple beans that implement any of these functional interfaces, the `spring.cloud.function.definition` property in the application.properties or application.yml file must be explicitly configured.

For the given use case where a Lambda function consumes an event from SQS and saves it in DynamoDB, the implementation should use the Consumer<SqsEvent> interface. Here, `SqsEvent` represents an SQS object that can contain anywhere from 1 to 10 messages.
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

The `SaveNoteRequest` implementation applied some changed on provided DTO using RequestTransformer class and then the database entry got save in DynamodbDB.
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

The classes mentioned above are quite simple and do not require any special code for the native-image. It's a typical Spring Boot application that we create on a regular basis. This code is almost ready to be deployed to AWS Lambda runtime, but before that, we need to provide some reflection hints to GraalVM. This is because the following operations use reflection:
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

The main SAM file is `template.yaml`, which defines all the resources related to the Lambda function. Using SAM, we will define the following:
* Define custom runtime via `Runtime` parameter
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

### Testing the Lambda function
To test created lambda function, I found several options that can be applied separately or combined together for different situations.

1. **Using Integration tests with LocalStack and TestContainers (JVM)**

   To test our application, we can use old good unit tests or integration tests to test some parts of the application, but since our application is not aware of the trigger (like SQS message, API Gateway request, or SNS notification), we won't be able to create a "real-integration" test, but grants to spring-cloud-function which provides an utility class which emulates APIs provided by amazon for the lambdas with custom-runtime.

   To test our application, we can use old good unit tests or integration tests. However, since our application is not aware of the trigger, such as an SQS message, an API Gateway request, or an SNS notification, we won't be able to create a complete integration test. Fortunately, we can leverage Spring Cloud Function, which provides a utility class that emulates the APIs provided by Amazon for the lambdas with custom-runtime. 

   The [AWSCustomRuntime](https://github.com/spring-cloud/spring-cloud-function/blob/main/spring-cloud-function-adapters/spring-cloud-function-adapter-aws/src/main/java/org/springframework/cloud/function/adapter/test/aws/AWSCustomRuntime.java) can be useful for creating a complete integration test with AWS custom runtime. Additionally, we will need to use LocalStack to run DynamoDB locally and test the complete execution flow. 
   > **Before listing the steps for testing with AWSCustomRuntime, it is important to note a few important things:**
   >1. We need to add `AWSCustomRuntime` to the application context.
   >2. We need to use a real `webEnvironment` since it is required by the `AWSCustomRuntime`.
   >3. We need to specify explicitly the function name (via `spring.cloud.function.definition` property) that will be used as a handler because `AWSCustomRuntime` defines additional beans that implement functional interfaces.
   >4. We need to provide a property with the key `_HANDLER`. This is necessary for the AWS adapter to realize that the CustomRuntimeEventLoop should be started.


   With all of this in mind, we can create an integration test that will send an event to the `CustomRuntime` (which will later be pulled by our function), and we expect that within 5 seconds, we will have published the event in DynamoDB.
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

       // Some setup and depedency injection

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

   Once the application is created, it can be run as a regular Spring Boot application. On startup, the Lambda handler created is exposed as an HTTP endpoint that can be invoked at  [http://localhost:8080/lambdaFunction](http://localhost:8080/lambdaFunction). The `lambdaFunction` is the bean name which is used as the entry point for our application.

   To emulate SQS message receiving, we can use the following curl call:
    ```bash
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
    ```bash
    aws sqs send-message \
    --queue-url=https://sqs.eu-west-1.amazonaws.com/<ACCOUNT-ID>/<QUEUE-NAME> \
    --message-body='{"body":"my message"}'
    ```
    > **Note:** Don't forget to replace `<ACCOUNT-ID>` and `<QUEUE-NAME>` placeholders with values which correspond to your account and created queue name.

    After that, we should check the Lambda's monitor tab to ensure that the lambda was executed successfully.
    ![Lambda Monitor screen::polaroid](/images/post1_native_aws_lambda/cloud_watch_monitor.png)
    We should also check that there is an entry in DynamoDB with the posted message.
    ![Dynamodb items::polaroid](/images/post1_native_aws_lambda/dynamodb_item.png)
    
    *I would recommend using this approach when we want to perform end-to-end (E2E) tests in a pre-production account or to perform load/performance tests within a real AWS environment, all these steps can be automated and also introduced as part of CI/CD.*


### Deploying the Lambda function

## It's time for automation (CI/CD workflows)
    


## Conclusion

## Refence links
* [Github repository](https://github.com/gvart/every-note-persister)
* [Spring Native Docs](https://docs.spring.io/spring-boot/docs/current/reference/html/native-image.html)
* [AWS SAM documentation](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html)
* [link1]()