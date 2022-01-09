---
date: "2017-02-07T20:51:00Z"
title: 'Using Java Annotation Processor API for "Fun & Profit"'
slug: "using_jap_apis_for_fun_and_profit"
categories:
  - "Software Development"
tags:
  - "java"
  - "annotation processor"
---

In this blog post, we'll start by briefly explaining the purpose of annotation processors and how can we create and register one for fun (the profit will come later!). Grab a cup of coffee and let's go! &#9787;

## Annotation Processor

First of all, I'll assume you're already familiar with annotations, if that's not the case, feel free to read [Oracle's documentation](https://docs.oracle.com/javase/tutorial/java/annotations/)<sup id="a1">[[1]]({{< ref "using_jap_apis_for_fun_and_profit.md#f1" >}})</sup> on it.
Annotation Processors are essentially a mechanism designed to process annotations - I bet you did not expect this one - at compile time. The key here is _at compile time_, but we'll get there soon.

In order to register an annotation processor to be executed during compilation, certain conditions need to be met:

1. The annotation processor **must** provide a no-arg constructor
2. The code being compiled **must** contain annotations that are marked as supported by the annotation processor
   - wildcards are supported
3. The supported annotations **must** not have been claimed by another annotation processor which had previously run
4. The annotation processor **should** support the source version release (_-source_)
   - While not required it is a nice guideline to follow. If that's not the case, you'll get a warning: <small>`Supported source version 'RELEASE_<version>' from annotation processor 'your.annotation.Processor' less than -source '<source version>`</small>

For a more in-depth explanation of the above conditions, go ahead and read the contract defined by the [Processor's interface](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Processor.html)<sup id="a2">[[2]]({{< ref "using_jap_apis_for_fun_and_profit.md#f2" >}})</sup>.

The main goal of an annotation processors is to provide an extensible and dynamic way to generate _additional_ code and/or configuration files (just to mention a few examples) at compilation time.
In case you have missed, the _additional_ bits are relevant here, mostly because modifying existent code - read modifying the AST - isn't part of the API defined by JSR-259<sup id="a3">[[3]]({{< ref "using_jap_apis_for_fun_and_profit.md#f3" >}})</sup>. However, they are ways around it by relying on internal APIs instead. Otherwise, some of the libraries and frameworks that we come to love (and hate) would either be feature capped or not exist at all. We'll not cover those here.

## Creating an Annotation Processor

The API is kind enough to provide you with an abstraction<sup id="a4">[[4]]({{< ref "using_jap_apis_for_fun_and_profit.md#f4" >}})</sup> which you can extend and simply provide the configuration for your processor along with the processing code, as exemplified below.

```java
import java.util.Collections;
import java.util.Set;
import javax.annotation.processing.AbstractProcessor;
import javax.annotation.processing.RoundEnvironment;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.TypeElement;
import javax.tools.Diagnostic.Kind;

public class SmellyCodeBlockerProcessor extends AbstractProcessor {

  @Override
  public Set<String> getSupportedAnnotationTypes() {
    return Collections.singleton("*");
  }

  @Override
  public SourceVersion getSupportedSourceVersion() {
    return SourceVersion.latestSupported();
  }

  @Override
  public boolean process(
    final Set<? extends TypeElement> annotations,
    final RoundEnvironment roundEnv
  ) {
      if (!roundEnv.processingOver()) {
        processingEnv.getMessager().printMessage(Kind.ERROR,
            "Even though I am just a compiler and I don't have a nose, " +
            "I sense that your code smells. Please redo and try again.");
      }
      return true;
  }
}
```

The above processor isn't particularly helpful, at all, but I bet you'll laugh your ass off if you're able to "compile-jack" one of your buddies and observe him while he's trying to compile his code.
We'll see how we can setup your compilation procedure to use the above annotation processor, but first let's break it down:

- **AbstractProcessor**
  This is essentially the abstraction that I have mentioned earlier, it provides support for:
  - Supported source versions via the [@SupportedSourceVersion](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/SupportedSourceVersion.html)<sup id="a5">[[5]]({{< ref "using_jap_apis_for_fun_and_profit.md#f5" >}})</sup> annotation.
  - Supported annotation types via the [@SupportedAnnotationTypes](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/SupportedAnnotationTypes.html)<sup id="a6">[[6]]({{< ref "using_jap_apis_for_fun_and_profit.md#f6" >}})</sup> annotation.
  - Supported processor options via the [@SupportedOptions](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/SupportedOptions.html)<sup id="a7">[[7]]({{< ref "using_jap_apis_for_fun_and_profit.md#f7" >}})</sup> annotation.
  - Issues warnings when appropriate (e.g. no source version have been configured).
  - Holds a reference for the [ProcessingEnvironment](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/ProcessingEnvironment.html)<sup id="a8">[[8]]({{< ref "using_jap_apis_for_fun_and_profit.md#f8" >}})</sup> via the **processingEnv** variable, available after the processor initialization. This class provides a facility to create new files, report messages to the compiler (we certainly did that with the above processor!) along with other utility functions.
- **getSupportedAnnotationTypes**
  This method enables the configuration of the supported annotations by our processor. Optionally, you can configure this via annotations as well if you use choose to extend AbstractProcessor.
  You must specify the canonical name of the supported annotations. Wildcards are also supported, where a single wildcard '\*' denotes all annotations.
- **getSupportedSourceVersion**
  This method enables the configuration of the latest supported source version by the processor.
  You can also configure this via annotations if you use choose to extend AbstractProcessor, but you loose the flexibility of using [latest()](https://docs.oracle.com/javase/8/docs/api/javax/lang/model/SourceVersion.html#latest--) and
  [latestSupported()](https://docs.oracle.com/javase/8/docs/api/javax/lang/model/SourceVersion.html#latestSupported--) for the configuration.
- **process**(**annotations**, **roundEnv**)
  This is where all of the magic happens, where you implement your annotation processor code. But there are a few caveats that I want your to be aware before getting your hands dirty.
  - Annotation processing is done in [**1**..**N**] rounds: in each round both annotations being processed (annotations parameter) and annotated source elements (via roundEnv parameter) are handed to the processor.
  - [RoundEnvironment](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/RoundEnvironment.html)<sup id="a9">[[9]]({{< ref "using_jap_apis_for_fun_and_profit.md#f9" >}})</sup> contains information associated with the previous round(s).
  - The return value affects subsequent rounds. That is:
    - if **true** is returned, no subsequent processors will be asked to process the annotations being processed in that round
    - if **false** is returned, subsequent processor might be asked to process them

## Registering an Annotation Processor

There are two ways for you to specify which annotation processors shall be used:

1. Manually via the `-processor` compiler option
2. Automatically by packaging your annotation processor(s) alongside with a file named `javax.annotation.processing.Processor` under a `/META-INF/services/` directory. The contents of that files are simply the canonical names of all processors that you're packaging, one per line.

We're gonna follow option 2 for the above processor, which is imho the way to go.

Compile the "_SmellyCodeBlockerProcessor_" processor and create the _META-INF/services/javax.annotation.processing.Processor_ file as detailed below:

```shell
javac SmellyCodeBlockerProcessor.java
mkdir -p META-INF/services/
echo "SmellyCodeBlockerProcessor" > META-INF/services/javax.annotation.processing.Processor
```

We're now able to package everything into a jar file with the proper structure by issuing the following command:

```shell
jar cvf smelly-code-blocker-processor.jar SmellyCodeBlockerProcessor.class META-INF
```

You should have a jar file with the following structure:

```text
Archive:  smelly-code-blocker-processor.jar
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  02-05-2017 16:40   META-INF/
       69  02-05-2017 16:40   META-INF/MANIFEST.MF
        0  02-03-2017 14:14   META-INF/services/
       33  02-05-2017 16:38   META-INF/services/javax.annotation.processing.Processor
     1655  02-05-2017 16:39   SmellyCodeBlockerProcessor.class
```

There's really only one thing left to do, profit. Find a way to clone your awesome jar file to your buddy compile classpath - that's up to you, I cannot tell how to do it - sit back, relax and enjoy the moment. Try not to laugh too much, as you'll have some explaining to do right after &#9787;

## Final thoughts

Well, after this light introduction to the annotation processor API, we're now able to create ~~the best~~ simple annotation processors.

Creating (more complex) processors that generate arbitrary data/files/code is also quite simple through the [Filer](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Filer.html)<sup id="a10">[[10]]({{< ref "using_jap_apis_for_fun_and_profit.md#f10" >}})</sup> utility class, accessible via [processingEnv.getFiler()](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/ProcessingEnvironment.html#getFiler--).
Just a quick tip upon going on that rabbit hole, **DO NOT** generate code "by hand", use a template engine such as [Apache Velocity](http://velocity.apache.org/)<sup id="a11">[[11]]({{< ref "using_jap_apis_for_fun_and_profit.md#f11" >}})</sup> or any other that suits your needs. You can thank me later.

And that's it. See you soon!

<hr/>

<small><b id="f1">[1] </b> [https://docs.oracle.com/javase/tutorial/java/annotations/](https://docs.oracle.com/javase/tutorial/java/annotations/)</small>
<small><b id="f2">[2] </b> [https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Processor.html](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Processor.html)</small>
<small><b id="f3">[3] </b> [https://www.jcp.org/en/jsr/detail?id=269](https://www.jcp.org/en/jsr/detail?id=269)</small>
<small><b id="f4">[4] </b> [https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/AbstractProcessor.html](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/AbstractProcessor.html)</small>
<small><b id="f5">[5] </b> [https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/SupportedAnnotationTypes.html](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/SupportedAnnotationTypes.html)</small>
<small><b id="f6">[6] </b> [https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/SupportedSourceVersion.html](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/SupportedSourceVersion.html)</small>
<small><b id="f7">[7] </b> [https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/SupportedOptions.html](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/SupportedOptions.html)</small>
<small><b id="f8">[8] </b> [https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/ProcessingEnvironment.html](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/ProcessingEnvironment.html)</small>
<small><b id="f9">[9] </b> [https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/RoundEnvironment.html](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/RoundEnvironment.html)</small>
<small><b id="f10">[10] </b> [https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Filer.html](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Filer.html)</small>
<small><b id="f11">[11] </b> [http://velocity.apache.org/](http://velocity.apache.org/)</small>
