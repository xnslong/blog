# 性能优化——概述

看了一篇关于性能优化的[博客](https://www.cnblogs.com/xybaby/p/9055734.html)，它介绍了性能优化的一般性原则，以及需求实现生命周期的各个阶段对性能的影响，这一段介绍得比较好。尽管在各个阶段我们都可能尽量使用最优设计来避免性能问题，但是最后系统实现完成的时候，它还是难免有各种性能问题。这时候我们还是需要发现性能问题并解决掉，也就是在实战阶段来解决性能问题。

# 性能优化概述

## 建立性能指标

要解决性能问题，我们需要知道系统当前有没有性能问题，性能问题有多严重；优化之后系统的性能得到了多大的改善。为此需要建立一系列的指标来指示系统的表现。

## 初始压测

当指标确定之后，我们就可以来一遍初始测试，看系统在没有任何优化的情况下性能表现如何。这个初始表现，一方面是优化必要性的依据，因为如果一个系统的初始表现已经很好了，那么对它再进一步的优化就很难再有什么收益了。如果一个系统的初始表现很糟糕，那么对它进行优化将是一个收益很大的事情。另一方面这个初始表现也是指征优化效果的一个重要依据，将优化之后的效果与初始效果对比，就是优化带来的提升。

## 性能优化

至此，一个优化正式开始之前需要做的工作就都已经完成了，接下来就是实打实的去解决系统瓶颈问题提升性能。这个过程中需要用一系统的手段去发现当前的瓶颈在什么地方，对性能影响多大。然后再决定使用什么方法去解决瓶颈问题，或者规避瓶颈问题。优化过程是一个持续的过程，这个过程中，解决一个瓶颈，就会发现一个新的瓶颈；就像是把天底下个子最高的那个人干掉了，那个原来第二高的人就会成为新的个子最高的人。但是这个过程也不是无止境的，因为当解决掉一个性能瓶颈的时候，系统的性能都会获得一定的提升，这个提升都能通过最初建立的指标来衡量。当系统的性能提升到一定程度之后，就会达到一个“够用”的水平，此时性能优化也就接近尾声了。

## 优化结项

当性能优化到了我们期望的水平，性能优化就可以宣告结束了。但是这个结束我们不能草草结束，那样会有种虎头蛇尾的感觉。最主要的是，如果优化的成果没有以一种合适的方式留存下来，优化所解决的问题，在后续的系统演进中还有可能被重新加了回来（被其它同伴，或者甚至是自己），这样性能优化的效果就大打折扣了。性能优化最好以一份性能优化报告来结项。大家通过阅读这份报告，就都能知道这次优化遇到了哪些问题，分别会给系统性能带来多大的影响，有什么样的解决的路子。有了这样的印象之后，以后再系统研发的过程中，大家就能有意识地去规避这些性能陷阱了。这样这一次性能优化的效果就充分达到了。

# 如何建立性能指标

一个系统性能如何，在于我们对它的需求什么。比如一个需要扛用户请求的系统，我们在意的是它处理请求的吞吐量、响应时间、错误比例等；比如一个消息系统，我们关心的是消息的吞吐量与传递延迟。这样我们可以用相应的这些指标来作为性能指标。总体来说，不论我们关心的是什么方面的指标，一般来说这些指标都可以归为这3类：

* 吞吐量
* 响应时间
* 错误率

## 外围观测

这样的一些指标的采集，一般是在相应的事件外围进行的，即拦截外部对我们感兴趣的方法或者行为的调用，在这一层进行一些统计的工作。工作过程如下：

```plantuml
[-> profiler: perform work
activate profiler

profiler -> profiler: record start
activate profiler
deactivate profiler

profiler -> target: perform work
activate target
profiler <-- target: work finished
deactivate target

profiler -> profiler: handle result
activate profiler
deactivate profiler

[<-- profiler: finish
deactivate profiler
```

根据关心的指标发生的位置，这个数据采集器也可以按置在不同的地方。比如我们关心一个接口的的性能，那么这个采集器既可以放在起压器侧，也可以放在被压程序侧。两者的差别在于，从起压器到被压程序之间的这一段开销是否被采集。这需要视情况而定。

```plantuml
component "起压程序" {
    [起压器]
    [profiler]
}

component "被压程序" {
    [被压接口]
}

[起压器]->[profiler]
[profiler]->[被压接口]
```

```plantuml
component "起压程序" {
    [起压器]
}

component "被压程序" {
    [profiler]
    [被压接口]    
}

[起压器]->[profiler]
[profiler]->[被压接口]    
```

这一层拦截统计的功能，很适合使用一个装饰器模式来实现，或者对于熟悉使用AOP的人，也可以使用aspect的方式来实现，这样对于调用者与被调用者都将是无感知的。

## 轻量profiler

这一层拦截统计的功能需要达的要求是足够轻量，使得增加这一层不会对系统的表现产生明显的影响。拦截器是否足够轻量，会对系统产生多大的影响，可以单独压测拦截器来确定（比如使用[JMH][]来测试）。profiler的逻辑很简单，一般都很轻量。可以根据需要自己实现一个简单的profiler，也可以找一个现有的profiler，一般来说都可以直接使用。

[JMH]: tools/java/JMH-benchmarking.md

# 如何发现性能瓶颈

## 资源竞争问题

一般来说：吞吐量，响应时间和错误率之间有着一些奇妙的关系。

较为笼统抽象地说，系统资源总量是固定的（比如每秒钟能执行多少次CPU计算）。如果平均单个操作消耗的系统资源更多（比如需要更多次数的运算），那么操作的吞吐量就会下降，单个操作响应时间一定程度的变长。同时由于每个操作都需要消耗更多资源，那么当压力足够大的时候，就可能一部分操作占着大量的资源，另一部分操作得不到需要的资源，此时就会发生等待（排队），等待又会使得响应时间变得更长。同时如果长时间得不到资源，系统还可能会报错误；得不到资源的概率越大，错误率也就越高。

性能瓶颈点，就是在系统争的最激烈的资源，系统中的代码就会大量使用这类资源或者停在这类资源的等待上。就像一个拥堵的十字路口成为了交通的瓶颈点的时候，我们就会发现有大量车辆都在等待这个资源。这种在大量运行或者等待的代码逻辑，就是热点代码逻辑。找到了它们也就能知道系统的瓶颈点是啥了。从统计的角度来看，当资源已经到达瓶颈时，我们从当前系统中正在运行的代码路径（线程栈）取一个样本，所取出来的样本中最高比例的路径也应该在这个热点代码路径上。

## 代码逻辑问题

另一类耗时高的性能问题则与资源竞争无关，即使没有任何其它进程或者线程与它抢资源，它也会很慢，比如下面这类代码逻辑。

```
for (int i = 0; i < 10000; i++) {
    runCostlyProcess();
}
```

从一个操作执行的生命周期来看，如果在它的所有逻辑中，有一些慢的逻辑导致整体变慢了，那么它里面一定有一些导致它慢的执行路径，使得当前线程大量的时间花费在该路径上。从统计的角度来看，假设有大量同类线程都在执行着，如果线程中某个代码路径需要消耗大量的时间，那么任何一个时刻也就会有大量的线程正在执行这一个代码路径。比如有一个线程会执行`A`和`B`两部份逻辑，并且花费在`B`上的时间是`A`上的两倍，那么当有多个线程同时执行的时候，对这些线程进行抽样，会发现样本中的在执行路径`B`的线程数接近执行路径`A`的线程数的2倍。

```plantuml
concise "Thread-1" as t1
concise "Thread-2" as t2
concise "Thread-3" as t3

@t1
0 is A
1 is B
3 is A
4 is B
6 is A
7 is B
9 is A

@t2
1 is A
2 is B
4 is A
5 is B
7 is A
8 is B

@t3
2 is A
3 is B
5 is A
6 is B
8 is A
9 is B

@4.5
t1 -> t2
t2 -> t3

@6
t1 -> t2
t2 -> t3
```

## 抽样统计的方法

对于这两类问题，我们都可以用抽样的方法来知道有性能瓶颈的代码路径是什么。当前的运行线程取样的工具有很多，比如Linux平台上有[`perf`](https://perf.wiki.kernel.org/index.php/Main_Page)，java的[`jstack`](https://docs.oracle.com/javase/7/docs/technotes/tools/share/jstack.html), [`JFR`](https://docs.oracle.com/javacomponents/jmc-5-4/jfr-runtime-guide/about.htm#JFRUH170)。取出了线程运行路径的样本中会有很多线程的数据，每个线程的路径一般都是一个调用链，比如下面是java的一个线程栈，表示采集到了一个线程名是`pool-25-thread-1`的线程，当前处于`TIMED_WAITING`状态，线程栈顶是在执行一个`sun.misc.Unsafe.park`方法。

```
"pool-25-thread-1" #301 prio=5 os_prio=0 tid=0x00007fa9f4057000 nid=0x151 waiting on condition [0x00007fa91bd71000]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
        at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1093)
        at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:809)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)
```

### 火焰图

我们所采集到的热点线程会有大量地都在执行同一条路径（在接近栈顶的地方或许会有分叉），最大量的线程都在执行的路径，就是热点路径。对于采集到的线程数据，从中找出热点路径，一个最直观的方法就是使用[火焰图](https://github.com/brendangregg/FlameGraph)。

![](http://www.brendangregg.com/FlameGraphs/cpu-bash-flamegraph.svg)

上图就是`perf`命令采集到的某个线程样本生成的火焰图。

火焰图中纵向一列表示数据中存在的一个执行路径，这一列中每一行表示一个线程栈结点（正在执行的方法），上下相邻的两个结点的关系是下一个结点对应的方法在执行过程中调用了上一个结点对应的方法。同一个下面的结点的情况上方可能会出现多个结点，这横向相邻的多个结点是下方结点方法中调出去的几个不同的方法调用。

火焰图横向每个结点的宽度表示这一个当前路径在样本总体中的数量（比例）。所以热点路径都是图中很宽路径。如果在火焰图中看到一个宽而平的部分，那么这一部份一定是出现比例高的，换言之是热点路径。它们很可能就是系统的瓶颈所在。

其实火焰图就是个对路径进行合并统计并图行化展示的一个工具。如果熟悉之后，我们统计的路径可以不拘泥现有的使用姿势，比如把`调用者`在下`被调用者`在上的姿势改成`调用者`与`被调用者`位置调换一下，我们会发现新的视角。因为如果性能问题是因为某一个逻辑造成的，那么从任何路径上来的调用，栈顶最多的都将是这一个逻辑。在火焰图中把栈顶和栈底的位置调换，底部最宽的结点可能就是导致热点路径的原因。

## 逻辑分析优化

### 干了没用的活

热点路径之所以是热点，有一类情况是它干了很多本可以没用的活。我们可以顺着热点路径查看每一层的逻辑，确认是否有这类问题。如果是这类问题，那么想办法把不用干的逻辑去掉，就能减小很多执行开销，对性能的提升是很明显的。

比如我最近的一个压测，发现一个热点路径中最耗时的方法其实是不需要做的(`L1`处)，因为`receiver`其实是一个代码，没有提供被调用的方法，而是提供了一个`invokeMethod`方法（`L2`处调用的方法）来处理方法调用。火焰图中可以看到`call`方法结点上方有两个子结点分别调用`L1`和`L2`，对应的宽度比为`L1 : L2 = 12 : 1`，所以这里有`12/13`的处理开销都是不需要的，应该消除掉。这类干了不用干的活导致开销大的情况，一般在使用了一些通用组件，并且没有做特殊配置和优化的时候容易出现。

```groovy
public final Object call(Object receiver, Object[] args) throws Throwable {
    if (checkCall(receiver)) {
        try {
            try {
L1:             return metaClass.invokeMethod(receiver, name, args); // <-- hot path
            } catch (MissingMethodException e) {
                if (e instanceof MissingMethodExecutionFailed) {
                    throw (MissingMethodException)e.getCause();
                } else if (receiver.getClass() == e.getType() && e.getMethod().equals(name)) {
L2:                 return ((GroovyObject)receiver).invokeMethod(name, args); // <-- expected path
                } else {
                    throw e;
                }
            }
        } catch (GroovyRuntimeException gre) {
            throw ScriptBytecodeAdapter.unwrap(gre);
        }
    } else {
      return CallSiteArray.defaultCall(this, receiver, args);
    }
}
```

### 可以高效替换

另一类情况是热点方法可能使用了一个有性能问题的特殊的实现。

比如这是另一个例子，我使用了一个倒置的火焰图，发现在大量方法停留在了一个groovy枚举类的某个方法上(如下`FooEnum.find(int): FooEnum`)。虽然当时有点没想明白为什么枚举类的静态方法很是瓶颈，但是还是使用[JMH][]来压了一下这个方法，因为如果这个方法真是瓶颈，那么单独压测性能表现也应该会不好（压测过程中很多性能猜想都可以用[JMH][]来验证）。压测结果显示性能表现确实不好，单线程每秒只能200万次调用，CPU有`2.7GHz`的情况下，对它的期望值应该是每秒千万次的调用，甚至上亿次调用。于是把它的代码翻译成最普通的java代码，发现性能上来了，达到了每秒1.2亿次调用。将改之后的逻辑放在原来的系统中重新压一次，发现这个性能瓶颈消失了，性能有了提升。

> 其实这里慢的原因是因为使用了groovy的方法调用，就凭空增加了很多方法查找的工作。它其实也是干了没用的活（无效的方法查找与绑定）。

```groovy
enum FooEnum {
    A(1), B(2), C(3)
    
    private final int code
    FooEnum(int code) {this.code = code}
    
    static FooEnum find(int code) {
        FooEnum.enumConstants.find { it.code == code }
    }
}
```

分析火焰图，能看到一些我们所不理解的瓶颈。只是如果纯考虑优化，有时候我们在不理解它为什么是瓶颈的情况下也能达到较好的优化效果。就像前一类干了没用的活的问题，我们在不理解那些没用的活为什么慢的情况下，就可以通过把那些活去掉。这一步我们在不知道它为什么慢的情况下，甚至可以直接替换成更高效的实现来达到优化目的。查明瓶颈点为什么慢，是优化的一个重要手段；但是有时候我们明确它确实慢了就可以做出优化决策了。

用这种态度优化的一个最常见的场景就是`缓存`，因为在很多情况下很多人并不是很清楚相应的`后备存储`为什么慢，但是当它们慢的时候使用`缓存`就可以很好的达到性能优化的目的了。

### 高等待操作

还有一类情况是有效操作本身消耗不高，但是它花费了很多时间在等待上。这类情况的火焰图会发现它们的线程大多处于等待状态。对于这类情况，我们可以做的优化是根据代码逻辑，把等待的粒度调小，尽量提升并发度；另一方面减少等待的次数，从而降低总耗时。

一个典型的例子就是`Collections.synchronizedMap(Map): Map`方法所返回的map，它会让所有请求都变成顺序化的请求，即任何时候只能有一个线程在访问，其它线程都要等着。对于这种情况，我们可以考虑作用`ConcurrentHashMap`来替换它，使用分段锁来减小竞争粒度。

另一个典型的例子是远程调用，远程调用都是耗时比较高的，减小远程调用次数，在一次调用中尽可能返回足够的信息，这样能有效提升性能。

# 性能优化报告的要求

TBD