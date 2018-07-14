# 使用JMH实现性能测试

## 前言

如果我们开发一个很实用的功能，会期望能让更多人用，以发挥他的价值。但是事实上，能不能被使用者接受，最终往往因为一些功能之外的原因决定，比如稳定性，可用性，性能等。所以需要在这些非功能方面做到足够好，才能被大家所接受。

要做到足够好，就需要知道它目前表现得好不好。只有知道它能表现得什么样，才能知道它好不好，才能知道该怎样做好，才可能做到足够好。要知道表现得什么样，我们需要一套测试的方法。只有测出来的数据才能说明一切。这些非功能的指标很多，相应的测试方法也有很多。这里我们只聊性能的测试。

## 性能测试

性能指执行任务的能力（速度），一般可以用如下两个指标来表示。

* 吞吐量（throughput）表示任务处理进度的速度
* 响应时间（response time）表示单个任务处理的速度

性能测试一般就是要得出这两个参数的信息。所以性能测试方法大多就是启多个进程和线程同时循环执行任务，测量各种压力下的这两个方面的值。所以一个性能测试的工作包括：

* 编写压测逻辑：调用被测方法执行任务
* 编写起压逻辑：起进程和线程执行压测方法
* 编写控制逻辑：何时开始运行，何时开始采集数据，何时停止
* 编写数据收集逻辑：收集压测过程中`吞吐量`和`响应时间`这些信息
* 编写报告生成逻辑：生成一个结论性的报表

这些工作中，除了`编写压测逻辑`对于每个压测任务不同，其余的工作几乎都是相同的，可以复用。`JMH`就是实现了这样一些通用的逻辑，让使用者不再为这些东西花费心思，可以非常简单地编写压测程序，将心思都放在压测逻辑的编写、压测结果的分析和被测功能的优化上。但是`JMH`是java的性能测试工具，对于其它语言，应该看看那个语言类似工具。

## 使用JMH编写简单的性能测试程序

### 快速上手

一般首先使用JMH所提供的archetype创建压测工程，archetype如下

```xml
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-java-benchmark-archetype</artifactId>
    <version>1.21</version>
</dependency>
```

然后就可以在`src/main/java`目录下编写压测程序了。压测程序很简单，只需要在一个`public`成员方法上加上`@Benchmark`注解，那么这个方法就是被测试的目标方法。比如我们可以写一个判断一个数(`1234`)是不是整数的方法。

```java
public class NumberTestBenchmark {

    public boolean isNumber(String value) {
        try {
            Integer.parseInt(value);
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    // 这就是一个被测试的目标方法
    @Benchmark
    public boolean isNumber() {
        return isNumber("1234");
    }

}
```

此时就可以来执行了这个benchmark了。简单的执行方式就是安装JMH插件，然后像执行单元测试那样执行benchmark。（后续将详细介绍[benchmark启动方式](#benchmark启动方式)）执行结果如下：


```
# JMH version: 1.19
# VM version: JDK 1.8.0_131, VM 25.131-b11
# VM invoker: /Library/Java/JavaVirtualMachines/jdk1.8.0_131.jdk/Contents/Home/jre/bin/java
# VM options: -Dfile.encoding=UTF-8
# Warmup: 20 iterations, 1 s each
# Measurement: 20 iterations, 1 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Throughput, ops/time
# Benchmark: benchmark.demo.NumberTestBenchmark.isNumber

# Run progress: 0.00% complete, ETA 00:00:40
# Fork: 1 of 1
# Warmup Iteration   1: 62970693.345 ops/s
# Warmup Iteration   2: 93324937.645 ops/s
# Warmup Iteration   3: 105334487.871 ops/s
# Warmup Iteration   4: 102237892.363 ops/s
# Warmup Iteration   5: 104664621.717 ops/s
# Warmup Iteration   6: 104931613.208 ops/s
# Warmup Iteration   7: 104509623.222 ops/s
# Warmup Iteration   8: 104715106.648 ops/s
# Warmup Iteration   9: 103632707.411 ops/s
# Warmup Iteration  10: 101949112.063 ops/s
# Warmup Iteration  11: 98902292.905 ops/s
# Warmup Iteration  12: 101377811.592 ops/s
# Warmup Iteration  13: 101479726.188 ops/s
# Warmup Iteration  14: 100134341.643 ops/s
# Warmup Iteration  15: 100348767.127 ops/s
# Warmup Iteration  16: 100711223.068 ops/s
# Warmup Iteration  17: 83685861.940 ops/s
# Warmup Iteration  18: 95675372.260 ops/s
# Warmup Iteration  19: 98437784.006 ops/s
# Warmup Iteration  20: 100731674.069 ops/s
Iteration   1: 100901550.381 ops/s
Iteration   2: 100799647.361 ops/s
Iteration   3: 102025437.593 ops/s
Iteration   4: 100232830.105 ops/s
Iteration   5: 101527005.670 ops/s
Iteration   6: 102005446.245 ops/s
Iteration   7: 99333047.435 ops/s
Iteration   8: 99756746.049 ops/s
Iteration   9: 100589892.099 ops/s
Iteration  10: 100632598.021 ops/s
Iteration  11: 102487874.209 ops/s
Iteration  12: 102538183.096 ops/s
Iteration  13: 95965921.238 ops/s
Iteration  14: 102804822.281 ops/s
Iteration  15: 103465675.891 ops/s
Iteration  16: 104221280.605 ops/s
Iteration  17: 103730011.157 ops/s
Iteration  18: 103779527.007 ops/s
Iteration  19: 102530124.572 ops/s
Iteration  20: 104076161.974 ops/s


Result "benchmark.demo.NumberTestBenchmark.isNumber":
  101670189.150 ±(99.9%) 1728020.035 ops/s [Average]
  (min, avg, max) = (95965921.238, 101670189.150, 104221280.605), stdev = 1989990.442
  CI (99.9%): [99942169.114, 103398209.185] (assumes normal distribution)


# Run complete. Total time: 00:00:41

Benchmark                      Mode  Cnt          Score         Error  Units
NumberTestBenchmark.isNumber  thrpt   20  101670189.150 ± 1728020.035  ops/s
```

我们能看到：

* 在正式开始执行之前，JMH会打印出当前的环境信息已经执行的配置信息。
* 然后逐个打出预热的信息，这个过程中，我们一般能看到方法的执行速度提升再到稳定的过程
* 预热完成之后，我们就能看到它的出的各次测试的信息，包含着当前这一次测试所采集的信息
* 一次测试完成之后，我们会看到该次测试的一个结果汇总，即`均值`、`最大值`、`最小值`、`标准偏差`、`99.9%置信区间`这些信息。
* 最后再打印一个格式良好的报告，说明本次测试的结果。

至次，就是一个简单压测的完整过程。我们可以看到，我们只需要做非常少量的工作即可完成压测，这对于压测任务简单化有非常大的帮助。

### benchmark方法参数

但是大多数时候，一个压测任务都不是像上面所展示的那样简单，比如我们需要测一个HTTP接口的性能，那么我们需要做的工作有

1. 创建一个合适的`HTTP client`对象
2. 使用这个`client`发HTTP请求，测执行请求的吞吐量和响应时间

这个时候`HTTP Client`的创建与初始化操作就不适合放在`Benchmark`方法中（不然这个压测方法就会变成测试服务端能承受的`建连`与`断连`的速度）。而且`client`有一些初始化和销毁的方法，它们都需要在特定的时间执行。对于这一点，JMH提供了一种参数传递client的解决方式。

可以定义一个类`HttpClientProvider`如下

```java
@State(Scope.BENCHMARK)
public static class HttpClientProvider {
    HttpClient client;
    
    @Setup(Level.Trial)
    public void init() {
        client = HttpClient.custom().maxConnections(100).build();
    }
    
    @TearDown(Level.Trial)
    public void destroy() {
        client.close();
    }
}
```



并写benchmark方法

```java
@Benchmark
public void httpApi(HttpClientProvider provider) {
    GetMethod get = new GetMethod("http://example.com/foo/bar");
    HttpResponse resp = provider.client.execute(get);
    EntityUtils.consume(resp.getEntity());
    resp.close();
}
```

此时，JMH会在所有的测试执行之前先初始化一个`HttpClientProvider`实例，并调用它的`init()`方法进行初始化，然后一直用这个实例调用benchmark方法进行压测，所有压测都完成之后，再调用`destroy()`方法销毁client。

benchmark方法支持的参数有两类，JMH内部自定类型的参数（比如`Blackhole`），和注有`@State`注解类型的参数。



## 性能比较

## benchmark启动方式

执行的方法有3种：

1. 编写main方法启动执行，如下：

    ```java
    public static void main(String[] args) throws Exception {
        Options options = new OptionsBuilder()
                .include(NumberTestBenchmark.class.getName())
                .build();
                
        new Runner(options).run();
    }
    ```
2. 安装JMH插件，此时在IDE中就可以像执行单元测试那样执行这个Benchmark。
3. 先用maven打包，然后用`java -jar benchmarks.jar`的方式执行这些测试用例。

