+++
author = "Vladlen Gladis"
title = "Manage multiple JDKs smart not hard"
date = "2023-02-05"
categories = ["tutorials"]
description = "Let's explore what is the benefit of using an SDK manager in jvm world"
thumbnail = "https://sdkman.io/assets//img/sdk-man-small-pattern.svg"
keywords = [
    "sdkman",
    "jdk",
    "graalvm",
    "openjdk"
]
+++

In this blog post, we'll discuss why it's crucial to use an SDK manager as a java developer and how to install and switch between JDKs in a few seconds.
<!--more-->

# Why should you an SDK manager and why SDKMAN is a crucial tool for a java developer ?

![SDK Man Logo::img-small](https://sdkman.io/assets//img/sdk-man-small-pattern.svg)

As a Java developer, it is common to work with different versions of Java:
* To try out new features in the latest JDK.
* To work on new services that run on the current LTS JDK.
* To fix bugs in legacy services that cannot be migrated from Java 8.

Switching between different JDKs and updating the current JDK involves updating the `JAVA_HOME` environment variable and `symlinks` to the Java binaries.

To simplify this process, SDKMAN comes to the rescue. SDKMAN is a lightweight software written in Bash that requires only `curl` and `zip/unzip` to install, uninstall, update, or switch your JDKs.

#### Installing SDKMAN

```bash
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
```

To verify the installation, type: (as a result version should be printed i.e. `SDKMAN 5.15.0`):
```bash
sdk version
```

[Official installation guide](https://sdkman.io/install)


#### Using SDKMAN
1. **Finding the necessary JDK**
   To discover all available distributions of JDKs, use the following command:
   ```bash
   sdk list java
   ```
   The result will be a vast list of different JDK types, starting with Java 1.7 and ending with the latest releases.
   ![sdk list java result](/images/2023/tutorial/sdk_man/sdk_list.png)

2. **Installing JDK 19 and GraalVM 22.3 with Java 19**
   * Installing JDK 19
       ```bash
       sdk install java 19.0.1-amzn
       ```

   * Installing GraalVM 22.3 with Java 19
       ```bash
       sdk install java 22.3.r19-grl
       ```

{{% notice info "Note" %}}
SDKMAN comes with auto-complete feature, that allows you to get the Java version by pressing <kbd>Tab</kbd> key
 ![installation process](/images/2023/tutorial/sdk_man/sdk_man_install_java.gif)
{{% /notice %}}

3. **Switching between Java versions**
To switch between installed JDKs, use the `default` command:
```bash
sdk default java 19.0.1-amzn
```
This will update the symlinks and `JAVA_HOME` to the `19.0.1-amzn` JDK

or
```bash
sdk default java 22.3.r19-grl
```
This will update the symlinks and `JAVA_HOME` to the `22.3.r19-grl` JDK

#### Conclusion
If you have to work with multiple JDK, **SDKMAN** is a perfect
tool that allows easily to manage JDKs, update existing JDK to the latest security patches, and switch quickly between various versions of java.
Apart from being in change of JDKs, SDKMAN can manage different build toolss like maven, gradle,ant, etc. that can be useful for someone.