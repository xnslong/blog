# 使用JMH实现性能测试

# 前言

如果我们开发一个很实用的功能，会期望能让更多人用，以发挥他的价值。但是事实上，能不能被使用者接受，最终往往由一些功能之外的原因决定，比如稳定性，可用性，性能等。所以需要在这些非功能方面做到足够好，才能被大家所接受。要把东西做好，就需要知道它目前表现得怎么样。所以我们需要一套方法测试它目前的表现，只有测出来的数据才能说明一切。这些非功能的指标很多，相应的测试方法也有很多。这里我们就聊性能相关的测试。

性能指执行任务的能力（速度），一般可以用`吞吐量`和`影响时间`两个指标来表示。性能测试一般就是要测出这两个参数。

> * 吞吐量（throughput）表示单位时间能处理的任务数量，决定任务处理进度
> * 响应时间（response time）表示单个任务处理从提交到处理完成的耗时，影响用户体验

性能测试方法一般是启多个进程和线程同时循环执行任务，测量各种压力下的这两个参数的值。总结起来，一个性能测试的工作，包括：

* 编写压测逻辑：调用被测方法执行任务
* 编写起压逻辑：起进程和线程执行压测方法
* 编写控制逻辑：何时开始运行，何时开始采集数据，何时停止
* 编写数据收集逻辑：收集压测过程中`吞吐量`和`响应时间`这些信息
* 编写报告生成逻辑：生成一个结论性的报表
* 实际起压测试，获取报表

这些工作中，除了`编写压测逻辑`对于每个压测任务不同，其余的工作几乎都是相同的，可以复用。[`JMH`](http://openjdk.java.net/projects/code-tools/jmh/)就是实现了这样一些通用的逻辑，让使用者不再为这些东西花费心思，可以非常简单地编写压测程序，将心思都放在压测逻辑的编写、压测结果的分析和被测代码的优化上。

# 快速编写一个JMH性能测试程序

假设我们写了一个判断字符串是否为整数的方法(如下)，现在需要用JMH来测该方法的性能。

```java
public class StringUtils {
    public static boolean isNumber(String value) {
        try {
            Integer.parseInt(value);
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

首先使用JMH所提供的archetype创建压测工程，archetype如下

```xml
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-java-benchmark-archetype</artifactId>
    <version>1.21</version>
</dependency>
```

然后就可以在所创建工程的`src/main/java`目录下编写压测程序了。压测程序很简单：

1. 新创建一个`NumberTestBenchmark`类，作为压测程序的类。
2. 该类中添加一个`public`的成员方法`testNumber`，方法里面调用一次被测试的目标方法`StringUtils.isNumber(value)`。
3. 给`testNumber`方法带上`@Benchmark`注解，标示该方法为性能测试方法，JMH就可以执行该方法统计性能了。

此时一个完整的性能测试程序就写完了。代码如下：

```java
public class NumberTestBenchmark {

    // 这就是一个被测试的目标方法
    @Benchmark
    public boolean testNumber() {
        return StringUtils.isNumber("1234");
    }

}
```

使用`mvn clean package`进行打包，打包完会生成了一个`target/benchmarks.jar`文件。用`java -jar target/benchmarks.jar`执行这个jar包，就可以真正执行性能测试了。（后续将详细介绍[benchmark启动方式](#benchmark启动方式)）执行结果如下：

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
# Benchmark: benchmark.demo.NumberTestBenchmark.testNumber

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


Result "benchmark.demo.NumberTestBenchmark.testNumber":
  101670189.150 ±(99.9%) 1728020.035 ops/s [Average]
  (min, avg, max) = (95965921.238, 101670189.150, 104221280.605), stdev = 1989990.442
  CI (99.9%): [99942169.114, 103398209.185] (assumes normal distribution)


# Run complete. Total time: 00:00:41

Benchmark                        Mode  Cnt          Score         Error  Units
NumberTestBenchmark.testNumber  thrpt   20  101670189.150 ± 1728020.035  ops/s
```

通过这个执行结果，我们能看到：

* 在启动之后，正式开始执行之前，JMH会打印出当前的环境信息已经执行的配置信息。
* 然后可以看到开始执行benchmark的过程分为两个阶段，`预热`和`正式测试`。而每个阶段又都分成若干个测试迭代（按照前面配置信息中所打的，每秒一次迭代），一次迭代中会循环调用很多次`@Benchmark`方法。
    * 先是预热过程，逐个打出各次迭代的信息。这个过程中，我们一般能看到方法的执行速度一路提升到一个稳定值的过程。
    * 预热完成后，接着就开始正式执行，这时它所采集的信息将会用于最后结果的统计。
* 测试完成之后，我们会看到该次测试的一个结果汇总，包含`均值`、`最大值`、`最小值`、`标准偏差`、`99.9%置信区间`这些信息。
* 最后再打印一个格式良好的报告，说明本次测试的结果。

这就是一个简单压测的完整过程，我们可以看到，我们只需要做非常少量的工作即可完成压测，这对于压测任务简单化有非常大的帮助。

# 高级特性

## 耗时统计

JMH除了可以统计`吞吐量`外，还可以统计`响应时间`。相应的方法是，在benchmark方法上增加`@BenchmarkMode`注解，说明要统计哪些量（即benchmark的模式）。benchmark的模式一共有4种：

* Throughput/thrpt
* AverageTime/avgt
* SampleTime/sample
* SingleShotTime/ss

一个benchmark方法可以支持多个模式（即可以统计多个量），默认只统计`Throughput`。比如前面例子中测`isNumber`方法性能的benchmark，也可以测该方法的响应时间，相应代码如下：

```java
public class NumberTestBenchmark {

    // 这就是一个被测试的目标方法
    @Benchmark
    @BenchmarkMode(Mode.SampleTime)
    public boolean testNumber() {
        return StringUtils.isNumber("1234");
    }

}
```

结果如下：


```
# JMH version: 1.19
# VM version: JDK 1.8.0_131, VM 25.131-b11
# VM invoker: /Library/Java/JavaVirtualMachines/jdk1.8.0_131.jdk/Contents/Home/jre/bin/java
# VM options: -Dfile.encoding=UTF-8
# Warmup: 10 iterations, 1 s each
# Measurement: 10 iterations, 1 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Sampling time
# Benchmark: benchmark.demo.NumberTestBenchmark.testNumber

# Run progress: 0.00% complete, ETA 00:00:20
# Fork: 1 of 1
# Warmup Iteration   1: ≈ 10⁻⁷ s/op
# Warmup Iteration   2: ≈ 10⁻⁶ s/op
# Warmup Iteration   3: ≈ 10⁻⁷ s/op
# Warmup Iteration   4: ≈ 10⁻⁷ s/op
# Warmup Iteration   5: ≈ 10⁻⁷ s/op
# Warmup Iteration   6: ≈ 10⁻⁷ s/op
# Warmup Iteration   7: ≈ 10⁻⁷ s/op
# Warmup Iteration   8: ≈ 10⁻⁷ s/op
# Warmup Iteration   9: ≈ 10⁻⁷ s/op
# Warmup Iteration  10: ≈ 10⁻⁷ s/op
Iteration   1: ≈ 10⁻⁷ s/op
                 testNumber·p0.00:   ≈ 10⁻⁷ s/op
                 testNumber·p0.50:   ≈ 10⁻⁷ s/op
                 testNumber·p0.90:   ≈ 10⁻⁷ s/op
                 testNumber·p0.95:   ≈ 10⁻⁷ s/op
                 testNumber·p0.99:   ≈ 10⁻⁷ s/op
                 testNumber·p0.999:  ≈ 10⁻⁶ s/op
                 testNumber·p0.9999: ≈ 10⁻⁵ s/op
                 testNumber·p1.00:   ≈ 10⁻⁴ s/op

Iteration   2: ≈ 10⁻⁷ s/op
                 testNumber·p0.00:   ≈ 10⁻⁷ s/op
                 testNumber·p0.50:   ≈ 10⁻⁷ s/op
                 testNumber·p0.90:   ≈ 10⁻⁷ s/op
                 testNumber·p0.95:   ≈ 10⁻⁷ s/op
                 testNumber·p0.99:   ≈ 10⁻⁷ s/op
                 testNumber·p0.999:  ≈ 10⁻⁶ s/op
                 testNumber·p0.9999: ≈ 10⁻⁵ s/op
                 testNumber·p1.00:   ≈ 10⁻⁴ s/op

Iteration   3: ≈ 10⁻⁷ s/op
                 testNumber·p0.00:   ≈ 10⁻⁷ s/op
                 testNumber·p0.50:   ≈ 10⁻⁷ s/op
                 testNumber·p0.90:   ≈ 10⁻⁷ s/op
                 testNumber·p0.95:   ≈ 10⁻⁷ s/op
                 testNumber·p0.99:   ≈ 10⁻⁶ s/op
                 testNumber·p0.999:  ≈ 10⁻⁵ s/op
                 testNumber·p0.9999: ≈ 10⁻⁴ s/op
                 testNumber·p1.00:   ≈ 10⁻⁴ s/op

Iteration   4: ≈ 10⁻⁷ s/op
                 testNumber·p0.00:   ≈ 10⁻⁹ s/op
                 testNumber·p0.50:   ≈ 10⁻⁷ s/op
                 testNumber·p0.90:   ≈ 10⁻⁷ s/op
                 testNumber·p0.95:   ≈ 10⁻⁷ s/op
                 testNumber·p0.99:   ≈ 10⁻⁷ s/op
                 testNumber·p0.999:  ≈ 10⁻⁶ s/op
                 testNumber·p0.9999: ≈ 10⁻⁴ s/op
                 testNumber·p1.00:   0.003 s/op

Iteration   5: ≈ 10⁻⁷ s/op
                 testNumber·p0.00:   ≈ 10⁻⁷ s/op
                 testNumber·p0.50:   ≈ 10⁻⁷ s/op
                 testNumber·p0.90:   ≈ 10⁻⁷ s/op
                 testNumber·p0.95:   ≈ 10⁻⁷ s/op
                 testNumber·p0.99:   ≈ 10⁻⁷ s/op
                 testNumber·p0.999:  ≈ 10⁻⁵ s/op
                 testNumber·p0.9999: ≈ 10⁻⁴ s/op
                 testNumber·p1.00:   ≈ 10⁻⁴ s/op

Iteration   6: ≈ 10⁻⁷ s/op
                 testNumber·p0.00:   ≈ 10⁻⁸ s/op
                 testNumber·p0.50:   ≈ 10⁻⁷ s/op
                 testNumber·p0.90:   ≈ 10⁻⁷ s/op
                 testNumber·p0.95:   ≈ 10⁻⁷ s/op
                 testNumber·p0.99:   ≈ 10⁻⁷ s/op
                 testNumber·p0.999:  ≈ 10⁻⁵ s/op
                 testNumber·p0.9999: ≈ 10⁻⁴ s/op
                 testNumber·p1.00:   ≈ 10⁻⁴ s/op

Iteration   7: ≈ 10⁻⁷ s/op
                 testNumber·p0.00:   ≈ 10⁻⁷ s/op
                 testNumber·p0.50:   ≈ 10⁻⁷ s/op
                 testNumber·p0.90:   ≈ 10⁻⁷ s/op
                 testNumber·p0.95:   ≈ 10⁻⁷ s/op
                 testNumber·p0.99:   ≈ 10⁻⁷ s/op
                 testNumber·p0.999:  ≈ 10⁻⁵ s/op
                 testNumber·p0.9999: ≈ 10⁻⁴ s/op
                 testNumber·p1.00:   ≈ 10⁻⁴ s/op

Iteration   8: ≈ 10⁻⁷ s/op
                 testNumber·p0.00:   ≈ 10⁻⁸ s/op
                 testNumber·p0.50:   ≈ 10⁻⁷ s/op
                 testNumber·p0.90:   ≈ 10⁻⁷ s/op
                 testNumber·p0.95:   ≈ 10⁻⁷ s/op
                 testNumber·p0.99:   ≈ 10⁻⁷ s/op
                 testNumber·p0.999:  ≈ 10⁻⁶ s/op
                 testNumber·p0.9999: ≈ 10⁻⁴ s/op
                 testNumber·p1.00:   ≈ 10⁻⁴ s/op

Iteration   9: ≈ 10⁻⁷ s/op
                 testNumber·p0.00:   ≈ 10⁻⁷ s/op
                 testNumber·p0.50:   ≈ 10⁻⁷ s/op
                 testNumber·p0.90:   ≈ 10⁻⁷ s/op
                 testNumber·p0.95:   ≈ 10⁻⁷ s/op
                 testNumber·p0.99:   ≈ 10⁻⁷ s/op
                 testNumber·p0.999:  ≈ 10⁻⁶ s/op
                 testNumber·p0.9999: ≈ 10⁻⁴ s/op
                 testNumber·p1.00:   ≈ 10⁻⁴ s/op

Iteration  10: ≈ 10⁻⁷ s/op
                 testNumber·p0.00:   ≈ 10⁻⁷ s/op
                 testNumber·p0.50:   ≈ 10⁻⁷ s/op
                 testNumber·p0.90:   ≈ 10⁻⁷ s/op
                 testNumber·p0.95:   ≈ 10⁻⁷ s/op
                 testNumber·p0.99:   ≈ 10⁻⁷ s/op
                 testNumber·p0.999:  ≈ 10⁻⁶ s/op
                 testNumber·p0.9999: ≈ 10⁻⁵ s/op
                 testNumber·p1.00:   ≈ 10⁻⁴ s/op



Result "benchmark.demo.NumberTestBenchmark.testNumber":
  N = 227732
  mean =     ≈ 10⁻⁷ ±(99.9%) 0.001 s/op

  Histogram, s/op:
    [0.000, 0.000) = 227730 
    [0.000, 0.001) = 1 
    [0.001, 0.001) = 0 
    [0.001, 0.001) = 0 
    [0.001, 0.001) = 0 
    [0.001, 0.002) = 0 
    [0.002, 0.002) = 0 
    [0.002, 0.002) = 0 
    [0.002, 0.002) = 0 
    [0.002, 0.003) = 0 
    [0.003, 0.003) = 1 

  Percentiles, s/op:
      p(0.0000) =     ≈ 10⁻⁹ s/op
     p(50.0000) =     ≈ 10⁻⁷ s/op
     p(90.0000) =     ≈ 10⁻⁷ s/op
     p(95.0000) =     ≈ 10⁻⁷ s/op
     p(99.0000) =     ≈ 10⁻⁷ s/op
     p(99.9000) =     ≈ 10⁻⁵ s/op
     p(99.9900) =     ≈ 10⁻⁴ s/op
     p(99.9990) =     ≈ 10⁻⁴ s/op
     p(99.9999) =      0.003 s/op
    p(100.0000) =      0.003 s/op


# Run complete. Total time: 00:00:21

Benchmark                                            Mode     Cnt   Score    Error  Units
NumberTestBenchmark.testNumber                     sample  227732  ≈ 10⁻⁷            s/op
NumberTestBenchmark.testNumber:testNumber·p0.00    sample          ≈ 10⁻⁹            s/op
NumberTestBenchmark.testNumber:testNumber·p0.50    sample          ≈ 10⁻⁷            s/op
NumberTestBenchmark.testNumber:testNumber·p0.90    sample          ≈ 10⁻⁷            s/op
NumberTestBenchmark.testNumber:testNumber·p0.95    sample          ≈ 10⁻⁷            s/op
NumberTestBenchmark.testNumber:testNumber·p0.99    sample          ≈ 10⁻⁷            s/op
NumberTestBenchmark.testNumber:testNumber·p0.999   sample          ≈ 10⁻⁵            s/op
NumberTestBenchmark.testNumber:testNumber·p0.9999  sample          ≈ 10⁻⁴            s/op
NumberTestBenchmark.testNumber:testNumber·p1.00    sample           0.003            s/op
```

这个benchmark执行过程与吞吐量的很像，但是每次迭代统计的量不一样，最终结果中它还统计了一个耗时的分布情况。

## benchmark方法自定义参数

前面我们看到的JMH的benchmark方法是无参的，其实它是可以支持参数的。测试的很多场景都非常需要这个功能的，比如我们测试过程中的每次迭代中的所有benchmark方法的执行需要使用同一个对象，这时这个对象就需要在测试方法执行前统一初始化，再用参数的方式给测试方法，让它的每次执行都使用同一个对象。

比如我们需要测试`AtomicInteger`增加计数方法的性能，就是需要在同一个对象上不断执行`incrementAndGet()`方法，测试它的性能。如下一段JMH压测代码就可以很好满足这个需求。


```java
@State(Scope.Benchmark)
public static class AtomicIntegerProvider {
    AtomicInteger integer = new AtomicInteger();
}
    
@Benchmark
public void incr(AtomicIntegerProvider provider) {
    provider.integer.incrementAndGet();
}
```

### 带有`@State`注解的自定义类型

JMH的benchmark方法可以传递参数，但是它要求它的类型必需定义了`@State(Scope.XXXX)`注解类型，该注解注明它的作用范围。这样JMH就知道需要什么时候创建一个的实例，时benchmark方法的各次调用怎么共享这个实例。`@State`中的`Scope`参数有3个值

* `Scope.Benchmark`它是为整个benchmark只创建一个实例，然后所有线程执行benchmark方法时，都使用同一个实例做为参数。比如我们需要压测多线程共同操作`AtomicInteger`的性能，这时我们就会使用`Scope.Benchmark`，代码如下：
    
    ```java
    // 此时JMH只会创建一个AtomicIntegerProvider实例，
    // 并且所有线程的benchmark方法调用都使用这一个实例作为参数。
    @State(Scope.Benchmark)
    public static class AtomicIntegerProvider {
        AtomicInteger integer = new AtomicInteger();
    }
    
    @Benchmark
    public void multiThreadIncrAtomicInteger(AtomicIntegerProvider provider) {
        provider.integer.incrementAndGet();
    }
    ```
    
* `Scope.Thread`它会为每个线程创建一个实例。比如当我们需要测一测`AtomicInteger`在没有竞争的时候的性能，这时我们就会用`Scope.Thread`为每个线程创建一个`AtomicInteger`，以达到没有竞争的效果。代码如下：
    
    ```java
    // 此时JMH会为每个线程创建一个AtomicIntegerProvider实例
    @State(Scope.Thread)
    public static class AtomicIntegerProvider {
        AtomicInteger integer = new AtomicInteger();
    }
    
    @Benchmark
    public void singleThreadIncrAtomicInteger(AtomicIntegerProvider provider) {
        provider.integer.incrementAndGet();
    }
    ```
    
    有些代码多线程同时访问有并发问题，会导致结果的不正确，测他们就需要使用`Scope.Thread`，比如`DateFormat`，`Random`, `StringBuilder`这些类。
    
* `Scope.Group`

> 如果需要传递给benchmark方法的参数类型并没有定义`@State`注解（比如之前的例子中测试方法需要的是`AtomicInteger`类型的参数，但是`AtomicInteger`类型上并没有定义`@State`注解）。这种情况下我们可以定义一些`XxxProvider`的类型，通过该类型来引用我们期望的目标类型，此时我们可以很简单的往这类`XxxProvider`类型上添加`@State`注解。

### 自定义类的`@Setup`与`@TearDown`方法注解

带有`@State`注解的类型有时候需要做一些初始化和销毁的工作。比如测试MySQL性能时，需要建立MySQL连接、关闭MySQL连接。这时我可以把这些初始化和销毁的逻辑分别定义成这个类的成员方法，并在方法上面添加`@Setup`和`@TearDown`注解来实现，这样JMH就会自动在测试前后去调用带这些注解的方法，从而实现初始化和销毁的效果。它们的功能与`JUnit`中的`Before`和`After`很像，会在测试任务执行前后执行。比如刚才的MySQL，就可以使用如下代码：

```java
@State(Scope.Thread)
public class MysqlConnectionProvider {
    Connection conn;

    // 这个初始化方法带有@Setup注解，会在执行测试之前调用
    @Setup(Level.Trial)
    public void init() throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.jdbc.Driver");
        conn = DriverManager.getConnection("jdbc:mysql://localhost");
    }

    // 这个销毁方法带有 @TearDown 注解，会在测试方法执行完成之后调用
    @TearDown(Level.Trial)
    public void destroy() throws SQLException {
        conn.close();
    }
}
```

这时就可以写如下一段MySQL性能测试代码。此时，JMH会在所有的测试执行之前先初始化一个`MysqlConnectionProvider`实例，并调用`MysqlConnectionProvider.init()`方法进行初始化，然后用这个实例调用benchmark方法进行压测，所有压测都完成之后，再调用`MysqlConnectionProvider.destroy()`方法来关闭MySQL连接。

```java
public class MySQLBenchmark {

    @Benchmark
    public void test(MysqlConnectionProvider provider) throws SQLException {
        try (final PreparedStatement statement = provider.conn.prepareStatement("SELECT * FROM test WHERE id = ?")) {
            statement.setInt(1, 1);
            try (final ResultSet resultSet = statement.executeQuery()) {
                if (resultSet.first()) {
                    resultSet.getString("value");
                }
            }
        }
    }
}
```

按之前描述，JMH的性能测试分为`预热`和`测试`两个阶段；每个阶段又由多次`迭代`而成；每次迭代会多次`调用`benchmark方法，进行统计：所以测试过程中有不同粒度的概念`测试`、`迭代`和`调用`。我们可以为每个粒度设置初始化和清理方法，相应的只需要为`@Setup`和`@TearDown`注解指定`Level`参数即可。

* `Level.Trial`会在整个测试开始前调用setup，结束时调用teardown方法
* `Level.Iteration`会在每个iteration开始前调用setup方法，结束时调用teardown方法
* `Level.Invocation`会在benchmark方法每次调用前执行setup方法，调用结束时执行teardown方法

> `Level.Invocation`要慎用，因为它对测试结果的准确性可能会影响很大。

```plantuml
participant "init thread" as init
create "XxxProvider" as provider
init -> provider
init -> provider: setupForTrial
activate provider
init <-- provider
deactivate provider

==start to test: iteration 1==
init -> provider: setupForIteration
activate provider #DarkSalmon
init <-- provider
deactivate provider

participant "Benchmark Thread" as t1
t1 -> provider: conn.query
...all threads run benchmark with the same provider instance...
t1 -> provider: conn.query

"destroy thread" as destroyer -> provider: teardownForIteration
activate provider #DarkSalmon
destroyer <-- provider
deactivate provider
== iteration 2 ==
...
== iteration 10 ==
init -> provider: setupForIteration
activate provider #DarkSalmon
init <-- provider
deactivate provider

t1 -> provider: conn.query
...all threads run benchmark with the same provider instance...
t1 -> provider: conn.query

destroyer -> provider: teardownForIteration
activate provider #DarkSalmon
destroyer <-- provider
deactivate provider
==test finished, tear down==
destroyer -> provider: teardownForTrial
activate provider
destroyer <-- provider
deactivate provider
```

每个注解的方法也都可以出现多次，`@Setup`与`@TearDown`没有配对的关系。

### JMH参数

一般的benchmark测试的方式是将benchmark打成jar包，然后在一个无干扰的环境下用命令行启动测试。但是有些信息在测试代码打成jar包前是不知道的，或者是不便写在代码中，执行时需要临时设置或者改变的。这些信息可以通过在执行的时候用参数传进去。比如我们要压测MySQL，那么MySQL的服务器地址，用户名密码这些信息，就可以在运行时传进去。相应的代码中只需要在`@State`注解的类中加上`@Param`注解的字段即可。

```java
@State(Scope.Thread)
public class MysqlConnectionProvider {

    // 这时我们可以在执行命令设置 mysqlServer 参数来指定该字段的值。
    // 但是 @Param 注解还是需要定义一个默认值，如果我们没有在命令参数中指定值，那么它将用默认值
    @Param("localhost")
    String mysqlServer;
    @Param("root")
    String mysqlUser;
    @Param("password")
    String mysqlPassword;
    
    Connection conn;
    
    @Setup(Level.Trial)
    public void init() throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.jdbc.Driver");
        conn = DriverManager.getConnection("jdbc:mysql://" + mysqlServer, mysqlUser, mysqlPassword);
    }
    
    @TearDown(Level.Trial)
    public void destroy() throws SQLException  {
        conn.close();
    }
}
```

它执行的时候结果如下

```
Benchmark              (mysqlPassword)   (mysqlServer)  (mysqlUser)    Mode       Cnt      Score     Error  Units
MySQLBenchmark.test           password       localhost         root   thrpt       200  56414.767 ± 893.774  ops/s
```

我们可以看到报告中多了几列括号`()`括起来的列，它们就是这里上面代码中设置的参数，列名就是带`@Param`注解的字段名。执行`benchmarks.jar`的时候，可以通过`-p param=value1,value2,...`的方式来指定参数，比如`-p mysqlServer=localhost:3306 -p mysqlUser=test_user`。
详细情况见[benchmark启动方式](#benchmark启动方式)。

## 性能比较

掌握了以上的内容，我们就可以用`JMH`来做很多性能测试了，包括测一些简单的工具包的性能测试，到复杂的服务或者系统的性能测试。一般一个测试都能得到一些绝对的数据，比如方法的OPS是多少，方法的耗时是多少。但是面对一个绝对数据，其实我们并不能确定性能好不好（如果同类能有它`10`倍的性能，那么他的性能就不算好；如果同类的性能最高只能达到它的`1/10`，那么它的性能就是非常好了），只有比较了，才能确定性能是不是真好。

比如我们我们知道正则表达式有一个编译的过程，我们可以测试编译的性能。但是即使测出出来，我们也不知道它会给正则匹配带来多少性能开销。这时我们就可以设置一些对照实验来测量。

* 对照组：预编译 + 每次 (仅匹配)
* 裸实验组：每次 (仅编译)
* 复合实验组：每次 (编译+匹配)

```java
public static final String IP_PATTERN = "[0-9]{1,3}(\\.[0-9]{1,3}){3}";

@State(Scope.Benchmark)
public static class PatternProvider {
    Pattern pattern;

    @Setup(Level.Trial)
    public void init() {
        pattern = Pattern.compile(IP_PATTERN);
    }
}

@Benchmark
public void preCompiled(PatternProvider provider) {
    provider.pattern.matcher("1.2.3.4").matches();
}

@Benchmark
public void matchDirectly() {
    Pattern.matches(IP_PATTERN, "1.2.3.4");
}

@Benchmark
public void rawCompile(Blackhole bh) {
    final Pattern pattern = Pattern.compile(IP_PATTERN);
    bh.consume(pattern);
}
```

它执行将会得到如下结果

```
Benchmark                      Mode  Cnt        Score         Error  Units
RegExBenchmark.matchDirectly  thrpt    5  1407931.513 ± 1288502.867  ops/s
RegExBenchmark.preCompiled    thrpt    5  5777861.034 ±  139378.979  ops/s
RegExBenchmark.rawCompile     thrpt    5  2416370.706 ±   36911.292  ops/s
```

我们可以看到`匹配`的OPS是`编译`的`2.4`倍；正则匹配时，`预编译匹配`的性能是`临时编译匹配`的`4.0`倍，这样编译正则表达式的性能如何，对于正则匹配的影响有多大，就一目了然了。所以在正常的业务中，我们一般就会选择预编译正则表达式。

有比较才有优劣，比较结果一般对于我们选择架构的设计，代码的写法等方面，具有非常重要的指导意义。所以我们要经常地做一些性能比较，来做各方面的取舍。对于比较的工作，我们需要做的就是针对使用方式，设计合理的对照实验。

## benchmark启动方式

JMH中有一个`Runner`类，Benchmark的执行是由`Runner.run()`方法启动的。所以理论上只要能启动`Runner.run()`方法，我们就能启动benchmark。一般有3种方式

### 写main启动测试

直接编写main方法启动执行，如下，这种方式适合在编写测试代码时使用

```java
public static void main(String[] args) throws Exception {
   Options options = new OptionsBuilder()
           .include(NumberTestBenchmark.class.getName())
           .build();
           
   new Runner(options).run();
}
```

### archetype创建的工程，直接打包运行

[快速开始](#快速编写一个JMH性能测试程序)中介绍了用JMH提供的archetype创建工程的方式，这种方式创建的工程是可以直接打包成一个可以执行的JAR包的(benchmarks.jar)，它打成的jar的启动类是`org.openjdk.jmh.Main`。这个主类提供了非常丰富的参数，让我们可以非常清晰的查看benchmark信息，非常自由的控制benchmark参数，功能很强大，所以benchmark一般直接使用这个主类（官方archetype生成的工程就会把这个类设置成主类）。

参数格式是
    
```
java -jar benchmarks.jar match [options]
```
    
其中的`match`是正则表达式，它会把所有的`benchmark`名字都用这里指定的正责表达式去匹配，如果能查找到匹配的子串，那么相应的benchmark就会被执行。这个功能特别合适在包含了很多benchmark的jar包中执行某一个特定的测试。
    
它的参数可以分为3类
    
* 帮助类：
   * `-h`，该参数可以打印所有的选项信息，非常有用。下面列具了一些常用的选项，其它选项都可以用该参数查看。
* 信息查看类
   * `-l`查看所有测试用例
       
       ```
       Benchmarks: 
       benchmark.demo.RegExBenchmark.matchDirectly
       benchmark.demo.RegExBenchmark.preCompiled
       benchmark.demo.MySQLBenchmark.test
       ```
   
   * `-lp`查看所有测试用例及它们所需要的参数
   
       ```
       Benchmarks: 
       benchmark.demo.RegExBenchmark.matchDirectly
       benchmark.demo.RegExBenchmark.preCompiled
       benchmark.demo.MySQLBenchmark.test
         param "mysqlServer" = {localhost}
         param "mysqlPassword" = {password}
         param "mysqlUser" = {root}
       ```

   * `-lprof`查看可用的profile列表
   * `-lrf`查看可用的结果格式列表
* 参数控制类
   * `-f`启动的子进程数
   * `-t`启动的线程数
   * `-i`测试的迭代次数
   * `-wi`预热的迭代次数
   * `-p`设置参数
   * `-o`输出到指定文件，而非标准输出
   * `-rff`测试结果写到指定文件，而非标准输出
   * `-rf`测试结果使用指定的格式，而非默认格式
    
    
> 将测试程序打成jar包，然后用`java -jar benchmarks.jar`启动，这种启动方式是最常见的。这是因为IDE等程序可能会干扰测试结果的准确性，这种启动方式则基本不受IDE影响。

### JMH插件启动

安装JMH插件，此时在IDE中就可以像执行单元测试那样执行这个Benchmark，在性能测试程序开发的时候特别有用。它实际的实现机制也是启动主类`org.openjdk.jmh.Main`，可用的参数与第`2`种方式相同。


