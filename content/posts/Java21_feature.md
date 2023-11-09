---
title: "聊聊 Java21 新特性"
subtitle: ""
description: "Java21 新功能介绍"
date: 2023-09-29
toc: true
author: "cczywyc"
draft: false
tags: ["Java", "JDK 新功能"]
---

# 背景

​	2023 年 09 年 20 日，Java21 正式发布，这是最新的 LTS 版本，官方预计会对此版本提供至少 8 年的维护。这一次 JDK 的升级是否会引起 Java8 项目升级暂不清楚，但是此版本带来了诸如虚拟线程等重量级功能更新，可能会导致 Java21 版本将会成为未来几年最经典的 JDK 版本，就像曾经的 Java8 一样。

# 新功能介绍

​	此次 Java21 版本一共带来了 15 个新功能更新，具体可以见官方的[发布页面](https://openjdk.org/projects/jdk/21/)，一眼望去，其中最重磅的更新摸过于 JEP444，也就是我们所说的虚拟线程，一般我们在其他语言（比如 golang）里面叫做协程。除此之外，还有诸如分代 ZGC、Switch 模式匹配等实用功能的更新，以下是 Java21 版本所有的新功能：

|                  提案                  |                        标题                         |                             说明                             |
| :------------------------------------: | :-------------------------------------------------: | :----------------------------------------------------------: |
| [JEP430](https://openjdk.org/jeps/430) |             String Templates (Preview)              | 字符串模版。Java 语言现有字符串文本和文本块增加功能，其他常用语言均已提供相应的功能用法 |
| [JEP431](https://openjdk.org/jeps/431) |                Sequenced Collections                | 序列集合。该 JEP 提议引入”一个新的接口族，用于表示集合的概念，这些集合的元素按照预定义的序列或顺序排列，它们是作为集合的结构属性”。这一提案的动机是由于集合框架中缺乏预定义的顺序和统一的操作集。 |
| [JEP439](https://openjdk.org/jeps/439) |                  Generational ZGC                   | 分代 ZGC。通过扩展 Z 垃圾收集器（ZGC）来维护年轻对象和年老对象的独立生成，从而提高应用程序性能。这将使 ZGC 能够更频繁地收集年轻对象 -- 这些对象往往英年早逝。 |
| [JEP440](https://openjdk.org/jeps/440) |                   Record Patterns                   | 记录模式。使用记录模式增强 Java 语言，以解构记录值。可以嵌套记录模式和类型模式，以实现功能强大、声明性和可组合形式的数据导航和处理。 |
| [JEP441](https://openjdk.org/jeps/441) |             Pattern Matching for switch             | switch 增强。通过将模式匹配扩展到 switch语句，可以针对多个模式测试表达式，每个模式都有一个特定的操作，从而可以简洁、安全地表达复杂的面相数据的查询。 |
| [JEP442](https://openjdk.org/jeps/442) |    Foreign Function & Memory API (Third Preview)    | 外部函数和内存 API。Java 程序可以通过该 API 与 Java 运行时之外的代码和数据进行互操作。通过有效地调用外部函数（即 JVM 外部的代码），并通过安全地访问外部内存（即不受 JVM 管理的内存），API 使 Java 程序能够调用本机库并处理本机数据，而不会出现 JNI 的脆弱性和危险性。 |
| [JEP443](https://openjdk.org/jeps/443) |      Unnamed Patterns and Variables (Preview)       | 未命名模式和变量。未命名模式匹配记录组件而不说明组件的名称或类型，未命名变量可以初始化但不使用。两者都用下划线字符_表示。 |
| [JEP444](https://openjdk.org/jeps/444) |                   Virtual threads                   |                          虚拟线程。                          |
| [JEP445](https://openjdk.org/jeps/445) | Unnamed classed and Instance Main Methods (Preview) |                    未命名类和实例主方法。                    |
| [JEP446](https://openjdk.org/jeps/446) |               Scoped Values (Preview)               | 作用域值。引入作用域值，这些值可以在不使用方法参数的情况下安全有效地共享给方法。它们优先于线程化局部变量，尤其是在使用大量虚拟线程时。 |
| [JEP448](https://openjdk.org/jeps/448) |            Vector API (Sixth Incubator)             |              向量计算 API，该功能还在孵化阶段。              |
| [JEP449](https://openjdk.org/jeps/449) |  Deprecate the Windows 32-bit x86 Port for Removal  | 弃用 Windows 32 位 x86 移植，并打算在将来的版本中将其删除。  |
| [JEP451](https://openjdk.org/jeps/451) |  Prepare to Disallow the Dynamic Loading of Agents  |                    准备禁止动态加载代理。                    |
| [JEP452](https://openjdk.org/jeps/452) |           Key Encapsulation Methanism API           | 密钥封装机制 API。一种用于密钥封装机制的 API，这是一种实用公钥加密来保护对称密钥的加密技术。 |
| [JEP453](https://openjdk.org/jeps/453) |          Structured Concurrency (Preview)           | 结构化并发。结构化并发将在不同线程中运行的相关任务组视为单个工作单元，从而简化错误处理和消除，提高可靠性，并增强可观察性。 |

以上就是此次 Java 21 带来的全部新功能更新，本文将针对大家关注比较多的几个功能作详细介绍。

## 分代 ZGC

​	ZGC 最初是在 [JEP333](https://openjdk.org/jeps/333) 中提出的，全称叫做可扩展的低延迟垃圾收集器，最开始是在 Java11 上作为实验性的功能提供。随后在 [JEP377](https://openjdk.org/jeps/377) 中确定在 Java15 版本中正式生产环境可用。

​	ZGC (The Z Garbage Collector) 以低延迟著称，在设计之初，它的设计目标包括：

* 停顿时间不超过 10ms；
* 停顿时间不会随着堆大小，或者活跃对象的大小而增加；
* 支持 8MB ～ 4TB 级别的堆（在 Java13 版本已经最大支持 16TB）

### CMS 和 G1 回顾

​	从设计目标我们可以看出，ZGC 适用于大内存低延迟服务的内存管理和回收。在正式说 ZGC 之前，我们先来说说常用的垃圾收集器遇到的典型问题 -- GC 的停顿时间。GC 的停顿时间指垃圾回收期间 STW（Stop The World），当 STW 时，所有应用线程停止活动，等待 GC 停顿结束。

​	这里我把 CMS 和 G1 垃圾收集器加入对比，首先，CMS 新生代的 Young GC、G1 和 ZGC 都基于标记-复制算法，但是算法在不同的垃圾收集器下就表现出了巨大的性能差异。这里我以 G1 为例，分析一下 G1 垃圾收集器在混合回收阶段的耗时过程，需要说明的是 G1 的新生代回收和 CMS 的新生代回收均采用的是标记-复制算法，其过程全程 STW 这里不做讨论。G1 的混合回收过程可以分为标记阶段、清理阶段和复制阶段。

#### 标记阶段停顿分析

* 初始标记阶段：这个阶段是从 GC Root 出发标记，该阶段是 STW 的，由于 GC Root 数量不多，通过该阶段耗时非常短。
* 并发标记阶段：并发标记阶段是指从 GC Root 开始对堆中对象进行可达性分析，找出存活对象。该阶段通常耗时较长，但由于该阶段不是 STW 的，我们一般不太关心该阶段的耗时。
* 再标记阶段：重新标记那些在并发阶段发生变化的对象，该阶段是 STW 的。

#### 清理阶段停顿分析

* 清理阶段清点出有存活对象的分区和没有存活对象的分区，该阶段不会清理垃圾对象，也不会执行存活对象的复制，该阶段是 STW 的。

#### 复制阶段停顿分析

* 复制算法中的转移阶段需要分配新内存和复制对象的成员变量。转移阶段是 STW 的，其中内存分配通常耗时非常短，但对象成员变量的复制耗时有可能较长，这是因为复制耗时与存活对象数量与对象复杂度成正比，对象越复杂，复制耗时越长。

通过上面的回顾可以看出，四个 STW 过程中，初始标记因为只有标记 GC Root，耗时较短。再标记因为对象较少，耗时也较短。清理阶段因为内存分区数量少，耗时也较短。转移阶段要处理所有存活的对象，耗时会较长。因此 G1 停顿时间的瓶颈主要是转移阶段 STW，关于转移阶段不能和并发标记阶段一样并发执行呢？**主要是 G1 未能解决转移过程中准确定位对象地址的问题**。

### ZGC 原理

​	与 CMS 和 G1 类似，ZGC 也是采用标记-复制算法，不同的是，ZGC 对垃圾回收的过程做了重大改进：ZGC 在标记、转移和重定位阶段几乎都是并发的，这是 ZGC 实现停顿时间小于 10ms 的关键原因。ZGC 垃圾回收的周期如下图所示：

![](https://static001.infoq.cn/resource/image/c8/bf/c8yyb76e534124219874ed233707cabf.png)

前面说到，G1 的转移阶段是完全 STW 的，并且停顿时间随着活跃对象的增加而增加，ZGC 与之不同，整个 ZGC 只有三个阶段是 STW 的，分别是初始标记阶段、再标记阶段、初始转移阶段，再标记阶段 STW 的时间很短，最多 1ms，超过 1ms 则再次进入并发标记阶段。最后再说转移阶段，前面说到，G1 垃圾收集器，未能解决在转移阶段过程中准确定位对象地址的问题，所以它的 GC 耗时会随着存活对象的增加而增加。与 G1 不同的是，**ZGC 是通过着色指针和读屏障技术解决转移过程中准确定位对象的问题，实现了并发转移**，至于着色指针和读屏障技术下文会稍加说明。

### Java21 中的 ZGC 改进

​	回顾完了 ZGC 的大致过程和特点，最后再来说本次 Java21 版本中对 ZGC 的改进。首先在 Java21 中默认不开启分代 ZGC，要开启分代 ZGC，需要使用命令行参数：-XX:+ZGenerational

```bash
java -XX:+UseZGC -XX:+ZGenerational ...
```

需要特别说明的是，按照官方的说法，在后面的版本中，分代 ZGC 将会作为默认的选项开启，届时 -XX:+ZGenerational 将作为不使用分代 ZGC 的开关，直到最后，不使用分代 ZGC 选项将会被移除。

本次改动，分代 ZGC 将堆分成了两个逻辑代，年轻代用于最近分配的对象，老年代用于长期存在的对象。年轻代和老年代的垃圾回收都是独立进行的，因此 ZGC 可以更专注于收集收益较大的年轻代对象。和不开启分代 ZGC 一样，分代 ZGC 所有的垃圾回收工作都是和应用线程并发的执行的。由于 ZGC 读取和修改对象是和应用线程并发进行的，因此必须确保垃圾收集器和应用线程提供一致的对象视图，ZGC 通过着色指针、加载屏障和存储屏障来做到这一点的。

* 着色指针是指一种在堆中的对象指针，它与对象的内存地址联系在一起，包含了编码对象的已知状态的元数据，这个元数据描述了对象是否存活、对象的地址是否正确等信息。ZGC 始终使用 64 位对象指针，因此它可以容纳多达数 TB 堆的元数据位和对象地址。当一个对象中的字段指向另外一个对象时，ZGC 会使用着色指针来实现这种指向。
* 加载屏障是指 ZGC 注入在应用程序中的代码片段，只要应用程序读取到了引用自另外对象的一个对象字段，加载屏障就会解释存储在字段中的着色指针中的元数据，并且在应用程序使用引用对象之前采取一些措施。需要说明的是，之前不开启分代的 ZGC 也是通过着色指针和加载屏障来实现上述过程的，只不过分代 ZGC 针对上述过程做了相应优化。
* 另外一点，分代 ZGC 还是通过存储屏障来跟踪一代对象到另外一代对象的引用的。存储屏障也是 ZGC 注入到应用程序中的代码片段，只要应用程序将引用存储到对象字段中。分代 ZGC 为着色指针添加了新的元数据位，这样存储屏障就能确定正在写入的字段是否已被记录为可能包含跨代指针。着色指针使分代 ZGC 的存储屏障比传统 ZGC 的存储屏障更加高效。

增加存储屏障后，分代 ZGC 可以将标记可达对象的工作从加载屏障转移到存储屏障。也就是说，存储屏障可以使用着色指针中的元数据位来有效地确定在存储之前字段所引用的对象是否需要被标记。将标记移出加载屏障后，可以更容易地对其进行优化，这一点是很重要的，因为加载屏障的执行效率往往高于存储屏障。现在当加载屏障解释着色指针时，如果对象被重新定位，它只需要更新对象地址，并更新元数据，以表明已知地址是正确的。后续的加载屏障将解释此元数据，而不再检查对象是否已被重新定位。

分代 ZGC 在着色指针中使用不同的标记和重定位元数据位集，因此可以独立收集分代数据。

## 虚拟线程

​	虚拟线程这个词想必大家都不陌生，我们或多或少在其他语言（例如 golang 或者 python）都听说过协程的概念，这里的协程就是我要说的虚拟线程。

虚拟线程最初是 Java19 中在提案 [JEP425](https://openjdk.org/jeps/425) 中以预览功能提出，后面在 Java20 中在提案 [JEP436](https://openjdk.org/jeps/436) 中做了改进，仍然是以预览版的形式出现。终于在 Java21 版本，该特性作为正式可用的功能跟我们见面。

​	虚拟线程是一种在 JVM 中实现的轻量级的线程，也叫用户线程，它不受操作系统管理和调度，被 JVM 管理。和传统的线程相比，虚拟线程创建和销毁的速度更快，开销更小，可以被大量的创建，因此更加适合轻量的多任务场景。

![](https://belief-driven-design.com/images/2023-10-05-java-virtual-threads-scheduler.webp#center)

### 虚拟线程的几种创建方式

1. 静态构造器

```java
var virtual = Thread.ofVirtual();
virtual.name("my-virtual-thread").start(() -> {
		System.out.println("hello");
});
```

```java
Thread.startVirtualThread(() -> {
		System.out.println("hello virtual thread");
});
```

2. Executors

```java
var executor = Executors.newVirtualThreadPerTaskExecutor();
executor.submit(() -> {
		System.out.println("hello virtual thread");
});
```

3. 线程工厂

```java
ThreadFactory virtualThreadFactory = Thread.ofVirtual().name("my-virtual-thread").factory();
virtualThreadFactory.newThread(() -> {
    System.out.println("hello virtual thread");
}).start();
```

## Switch 表达式

​	最后，Java21 新特性里面，我还想说一下 switch 表达式。最近几年，几乎每个 jdk 的新版本都能看到对 switch 表达式的改进，在以往的 switch 语句中，对于 case 中的类型匹配限制还是很多的，switch 表达式在 Java14 开始趋于稳定，在 Java17 中 switch 的模式匹配首次以预览版的新式出现（见 [JEP406](https://openjdk.org/jeps/406)），在 Java18、Java19、Java20 多个版本中又进行了更新和功能完善，如今在 Java21 中以正式功能特性发布。

​	现在假设我有一个 HashMap<String, Object>，这个 map 的 value 我可能存放有多个不同类型的变量，在以前，我们需要使用 if 语句对类型进行判断，现在借助 switch 语句的模式匹配，便可以简化这种写法，具体例子看下面的代码：

```java
public class Main {
    public static void main(String[] args) {
        Map<String, Object> data = new HashMap<>();
        data.put("key", "cczywyc");
        // data.put("key", 56);
        // data.put("key", 56.12);

        // old code
        if (data.get("key") instanceof String s) {
            System.out.println("this is string type:" + s);
        } else if (data.get("key") instanceof Integer s) {
            System.out.println("this is integer type:" + s);
        } else if (data.get("key") instanceof Double s) {
            System.out.println("this is double type:" + s);
        }

        // new code
        switch (data.get("key")) {
            case String s -> System.out.println("this is string type:" + s);
            case Integer s -> System.out.println("this is integer type:" + s);
            case Double s -> System.out.println("this is double type:" + s);
            default -> System.out.println("there is no type be found");
        }
    }
}
```

相信看了上面的示例代码，便很清晰的看到新特性的用法，这里对 switch 语句的新特性不作过多介绍。

# 总结

​	毫无疑问，Java21 作为一个 LTS 版本，带来了期待已久的虚拟线程新特性，我预测这个版本在 Java 发展的历史上会是非常重要的一个版本，至于有多少公司愿意升级，还需要打一个大大的问号，至少目前在我看来，新版 JDK 的升级还需要观望，目前国内大部分公司还是清一色 JDK8，暂时可能升不太动，但是我作为一个技术人员，看到新版 JDK 这些新特性，还是异常兴奋的，尤其是虚拟线程，期待后面在 Java 并发编程领域有更多不一样的玩法，让我们拭目以待。

## Reference

Java21 Releas: https://openjdk.org/projects/jdk/21/

JEP439 Generational ZGC: https://openjdk.org/jeps/439

JEP444 Virtual Thread: https://openjdk.org/jeps/444

JEP441 Pattern Matching for switch in Java21: https://openjdk.org/jeps/441

JEP406 Pattern Matching for switch in Java17: https://openjdk.org/jeps/406

(全文完)

