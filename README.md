# Build a Native Java image for an Apache Beam data pipeline

Environment
```
# Java 17 + GraalVM 22.3.x
openjdk version "17.0.5" 2022-10-18
OpenJDK Runtime Environment GraalVM CE 22.3.0 (build 17.0.5+8-jvmci-22.3-b08)
OpenJDK 64-Bit Server VM GraalVM CE 22.3.0 (build 17.0.5+8-jvmci-22.3-b08, mixed mode, sharing)
```

Build the JIT app image
1. ```./mvnw clean package``` 

2. Run the pipeline from the CLI
```
java -agentlib:native-image-agent=config-output-dir=target/native-image -cp target/word-count-beam-bundled-0.1.jar  org.apache.beam.examples.WordCount --output=counts --inputFile=pom.xml

# Params: 
Main class: org.apache.beam.examples.WordCount
Input: pom.xml (count the words in this file)
Output: word + count
```

3. copy the `JSON files` from `target/native-image` folder to `src\main\resources\META-INF\services

4. build the JIT app image with the META-INF configuration ```./mvnw clean package``` 

5. build the native image
```
 native-image  -cp target/word-count-beam-bundled-0.1.jar org.apache.beam.examples.WordCount
 ```

 6. start the native image 
 ```
 ./org.apache.beam.examples.wordcount --output=counts --inputFile=pom.xml
 ```

 ## Challenge - to resolve - proxy errors
 An error occcurs:
 ```Exception in thread "main" com.oracle.svm.core.jdk.UnsupportedFeatureError: Proxy class defined by interfaces [interface org.apache.beam.runners.direct.DirectTestOptions, interface org.apache.beam.runners.portability.testing.TestUniversalRunner$Options, interface org.apache.beam.runners.direct.DirectOptions, interface org.apache.beam.sdk.extensions.gcp.options.GcpOptions, interface org.apache.beam.runners.portability.testing.TestPortablePipelineOptions] not found. Generating proxy classes at runtime is not supported. Proxy classes need to be defined at image build time by specifying the list of interfaces that they implement. To define proxy classes use -H:DynamicProxyConfigurationFiles=<comma-separated-config-files> and -H:DynamicProxyConfigurationResources=<comma-separated-config-resources> options.
	at org.graalvm.nativeimage.builder/com.oracle.svm.core.util.VMError.unsupportedFeature(VMError.java:89)
	at org.graalvm.nativeimage.builder/com.oracle.svm.core.reflect.proxy.DynamicProxySupport.getProxyClass(DynamicProxySupport.java:171)
	at java.base@17.0.5/java.lang.reflect.Proxy.getProxyConstructor(Proxy.java:47)
	at java.base@17.0.5/java.lang.reflect.Proxy.getProxyClass(Proxy.java:398)
	at org.apache.beam.sdk.options.PipelineOptionsFactory$Cache.validateWellFormed(PipelineOptionsFactory.java:2165)
	at org.apache.beam.sdk.options.PipelineOptionsFactory$Cache.validateWellFormed(PipelineOptionsFactory.java:2108)
	at org.apache.beam.sdk.options.PipelineOptionsFactory$Cache.register(PipelineOptionsFactory.java:2103)
	at org.apache.beam.sdk.options.PipelineOptionsFactory$Cache.initializeRegistry(PipelineOptionsFactory.java:2091)
	at org.apache.beam.sdk.options.PipelineOptionsFactory$Cache.<init>(PipelineOptionsFactory.java:2083)
	at org.apache.beam.sdk.options.PipelineOptionsFactory$Cache.<init>(PipelineOptionsFactory.java:2047)
	at org.apache.beam.sdk.options.PipelineOptionsFactory.resetCache(PipelineOptionsFactory.java:581)
	at org.apache.beam.sdk.options.PipelineOptionsFactory.<clinit>(PipelineOptionsFactory.java:547)
	at org.apache.beam.examples.WordCount.main(WordCount.java:205)
```

Add the missing proxy to the /META-INF/services/proxy-config.json
```json
{
    "interfaces": ["org.apache.beam.runners.direct.DirectTestOptions","org.apache.beam.runners.portability.testing.TestUniversalRunner$Options","org.apache.beam.runners.direct.DirectOptions","org.apache.beam.sdk.extensions.gcp.options.GcpOptions","org.apache.beam.runners.portability.testing.TestPortablePipelineOptions"]
}
```

Rebuild the JIT image and native image
```./mvnw clean package``` 

```
 native-image  -cp target/word-count-beam-bundled-0.1.jar org.apache.beam.examples.WordCount
 ```

## Repeated - the next failure occurs --> how to identify the missing proxies, to be resolved
```
./org.apache.beam.examples.wordcount --output=counts --inputFile=pom.xml
Exception in thread "main" com.oracle.svm.core.jdk.UnsupportedFeatureError: Proxy class defined by interfaces [interface org.apache.beam.runners.direct.DirectTestOptions, interface org.apache.beam.runners.portability.testing.TestPortablePipelineOptions, interface org.apache.beam.runners.portability.testing.TestUniversalRunner$Options, interface org.apache.beam.runners.direct.DirectOptions] not found. Generating proxy classes at runtime is not supported. Proxy classes need to be defined at image build time by specifying the list of interfaces that they implement. To define proxy classes use -H:DynamicProxyConfigurationFiles=<comma-separated-config-files> and -H:DynamicProxyConfigurationResources=<comma-separated-config-resources> options.
	at org.graalvm.nativeimage.builder/com.oracle.svm.core.util.VMError.unsupportedFeature(VMError.java:89)
	at org.graalvm.nativeimage.builder/com.oracle.svm.core.reflect.proxy.DynamicProxySupport.getProxyClass(DynamicProxySupport.java:171)
	at java.base@17.0.5/java.lang.reflect.Proxy.getProxyConstructor(Proxy.java:47)
	at java.base@17.0.5/java.lang.reflect.Proxy.getProxyClass(Proxy.java:398)
	at org.apache.beam.sdk.options.PipelineOptionsFactory$Cache.validateWellFormed(PipelineOptionsFactory.java:2165)
	at org.apache.beam.sdk.options.PipelineOptionsFactory$Cache.validateWellFormed(PipelineOptionsFactory.java:2108)
	at org.apache.beam.sdk.options.PipelineOptionsFactory$Cache.register(PipelineOptionsFactory.java:2103)
	at org.apache.beam.sdk.options.PipelineOptionsFactory$Cache.initializeRegistry(PipelineOptionsFactory.java:2091)
	at org.apache.beam.sdk.options.PipelineOptionsFactory$Cache.<init>(PipelineOptionsFactory.java:2083)
	at org.apache.beam.sdk.options.PipelineOptionsFactory$Cache.<init>(PipelineOptionsFactory.java:2047)
	at org.apache.beam.sdk.options.PipelineOptionsFactory.resetCache(PipelineOptionsFactory.java:581)
	at org.apache.beam.sdk.options.PipelineOptionsFactory.<clinit>(PipelineOptionsFactory.java:547)
	at org.apache.beam.examples.WordCount.main(WordCount.java:205)
```
