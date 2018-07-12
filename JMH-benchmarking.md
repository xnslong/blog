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

然后就可以在`src/main/java`目录下编写压测程序了。压测程序很简单，只需要在一个方法上加上`@Benchmark`注解。比如我们可以写一个判断一个数(`1234`)是不是整数的方法。

```java
public class NumberTestBenchmark {

    @Benchmark
    public boolean isNumber() {
        String value = "1234";
        try {
            Integer.parseInt(value);
            return true;
        } catch (Exception e) {
            return false;
        }
    }

}
```

此时就可以来执行了这个benchmark了。执行的方法有3种：

1. 安装JMH插件，此时就可以像执行单元测试那样执行这个Benchmark。
2. 先用maven打包，然后用`java -jar target/benchmarks.jar`的方式执行这些测试用例。
3. 编写main方法启动执行，如下：

    ```java
    public static void main(String[] args) throws Exception {
        Options options = new OptionsBuilder()
                .include(NumberTestBenchmark.class.getName())
                .build();
                
        new Runner(options).run();
    }
    ```

## 性能比较
## 如何复用

