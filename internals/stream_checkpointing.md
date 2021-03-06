---
title:  "Data Streaming的容错机制"
# Top navigation
top-nav-group: internals
top-nav-pos: 4
top-nav-title: Data Streaming的容错机制
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

本文档描述了 Flink 中Data Streaming的容错机制。

* This will be replaced by the TOC
{:toc}


## 简介

Apache Flink 提供一种容错的机制来保证Data Streaming应用在恢复状态时能保证一致性。这种机制保证了即使出现故障，数据流中的每条记录只会作用于状态一次 **exactly-once**。另外 Apache Flink 还提供了降级的机制，只保证 *至少一次(at least once)* 的语义。

这个机制在整个计算的过程中，通过持续不断的给分布式数据流的状态信息打上快照来实现。相对于那些只维护较少数据量的状态的应用，打快照是非常轻量级的，可以持续地操作而并不会对性能造成多大的影响。
状态的快照数据可以根据配置，存储到不同的存储服务上(例如可以存在 master 节点，或者 HDFS上)

假设出现了程序失败的情况(原因可能是机器，网络，或软件故障)，Flink 会停止数据流。
系统会重启计算节点，并把它的状态恢复到最新备份成功的一次checkpoint，同时数据源也会被重置到当前快照状态(state snapshot)中相应的起始offset，以保证重放的数据流不会是已经经过计算并存储相关的状态(checkpointed)的数据。

*注意:* 为了充分发挥这一机制的保障能力, 数据源(可能是一个消息队列或者代理)需要能够重放数据流。例如 [Apache Kafka](http://kafka.apache.org) 就有这个功能，Flink 的 Kafka connector 已经封装好了这些功能，用户可以直接使用。

*注意:* 因为Flink的checkpoint功能是通过分布式快照来实现的。我们后面可能会交替地使用 *快照* 或者 *checkpoint* 两个词。


## 检查点机制（Checkpointing）

Flink的容错机制最核心的部分就是把数据流和计算节点的状态一起生成一致性的快照。
这些快照组成一个个的检查点(checkpoint)，使得系统在失败的时候能够恢复到备份时的状态。Flink的快照生成算法在以下这篇论文中有详细描述 "[Lightweight Asynchronous Snapshots for Distributed Dataflows](http://arxiv.org/abs/1506.08603)". 这个算法是受到Chandy-Lamport算法的启发 [Chandy-Lamport algorithm](http://research.microsoft.com/en-us/um/people/lamport/pubs/chandy.pdf) 并且针对 Flink 的运行模型进行了一些简化。



### 数据栅栏（Barriers）

Flink的分布式快照算法的核心概念之一被称为 *数据栅栏*。 这些栅栏数据被注入到数据流中，和普通的数据一起组成数据流。栅栏数据不会干扰正常的数据，正常数据会按照原先的顺序流动。栅栏数据把正常的数据流切割成多个的数据块，每个数据块都会被打进一个快照中。每个栅栏数据会都会带上一个快照ID，表明该快照数据是该栅栏前面的数据流数据组成数据块。栅栏数据不会干扰正常的数据流，而且非常轻量。多个属于不同快照的栅栏可以同时出现在数据流中，这也就是说多个快照动作有可能会同时发生。

<div style="text-align: center">
  <img src="{{ site.baseurl }}/internals/fig/stream_barriers.svg" alt="Checkpoint barriers in data streams" style="width:60%; padding-top:10px; padding-bottom:10px;" />
</div>

数据栅栏在数据源端被注入到数据流当中，当snapshot n的barriers被注入后，系统会记录当前snapshot数据的位置值 *n* (用<i>S<sub>n</sub></i>表示)。例如，在 Apache Kafka 中,这个变量表示数据某个分组(partition)中最后一条数据的偏移量。这个位置值 <i>S<sub>n</sub></i> 会被报告到一个称为 *checkpoint仲裁者* 的模块去。(在Flink中，这个模块叫做 JobManager).

这些栅栏数据随着数据流动。当一个中间计算节点从它所有的输入流中收到快照点 *n* 的栅栏数据，并且计算完成后，也会发送一个*n*的栅栏数据到它所有的输出数据流中。当最后的计算节点(即 DAG 图中的终点)从它所有的输入流中收到*n* 的栅栏数据后，会发一个*n*的确认消息给checkpoint仲裁模块(即 JobManager)。当所有的终点都发出了确认消息，那么这个checkpoint就会被认为已经完成并且从配置的状态备份器(Job Manager或者其他外部存储)中删除。

当快照 *n* 已经完成后，可以确定，从源节点开始，所有<i>S<sub>n</sub></i>前面的数据都已经不再需要了，因为这些数据都已经经过了拓扑计算图中的节点处理完了。


<div style="text-align: center">
  <img src="{{ site.baseurl }}/internals/fig/stream_aligning.svg" alt="Aligning data streams at operators with multiple inputs" style="width:100%; padding-top:10px; padding-bottom:10px;" />
</div>

那些需要处理多条数据流的节点，需要对每条数据流里面的快照栅栏进行 *对齐* 。上面的表格说明了以下几点：
  - 当计算节点收到其中一条数据流中的栅栏数据 *n* 后，要先停下来等待其他数据流的栅栏 *n* 。否则有可能把某条数据流快照 *n* 的数据和另外的数据流 *n+1* 的数据混起来。
  - 先遇到栅栏*n*的数据流，要把数据保留下来。该条数据流的数据会先被放到一个输入缓存中等待。
  - 当从最后一条流中收到栅栏*n*后，该节点把之前在等待的数据都发送出去，并且发送一个自身的 *n* 栅栏数据。
  - 最后，该节点重新从输入流开始处理数据，它会先把输入缓存的数据先处理完毕，然后再处理后来的数据。


### 状态

如果计算节点包含任何类型的 *状态*，这些状态必须被作为快照的一部分。
节点的状态可能有如下几种类型：

  - *用户自定义状态*: 这个状态是在一些转换函数中生成和修改的（例如 `map()` 或者 `filter()`）。 用户自定义状态可以是函数中的一个简单的java变量, 也可以是基于key/value的状态接口 (参考 [State in Streaming Applications]({{ site.baseurl }}/apis/streaming_guide.html#stateful-computation)).
  - *系统状态*: 系统状态指的是节点计算中的一些必须的数据缓存。一个典型的例子是 *窗口缓存* , 这个是系统用来存储某个窗口内的原始数据加上聚合后的数据，一直到该窗口被触发计算了或者直接发到下游去。

当某个计算节点收到它所有输入流中的栅栏快照数据后就会马上把所有的状态打成快照，然后再插入一个栅栏数据到自身的输出流中。在这个时间节点，栅栏前所有的计算结果和状态都已经确定，所有后续的更新都不会再依赖这些数据。因为这些状态快照的数据量可能会很大，所以会被存储在一个可配置的 *状态存储后端* 中。默认的情况下存在JobManager的内存里，但是在实际的生产场景下，一般会配置为一个可靠的分布式存储（例如HDFS）。当状态都被存储完成后，节点会发送确认信息来确认checkpoint完成，并插入快照栅栏数据到它的输出流中，然后继续处理。

总的来说，快照包含以下数据：

  - 对于每条数据流，包含了被快照分割的每段数据的起始位置。
  - 对于每个计算节点，包含了指向状态数据的指针。

<div style="text-align: center">
  <img src="{{ site.baseurl }}/internals/fig/checkpointing.svg" alt="Illustration of the Checkpointing Mechanism" style="width:100%; padding-top:10px; padding-bottom:10px;" />
</div>


### Exactly Once vs. At Least Once

数据流对齐的特性可能会增加流的延时。通常情况下，延时会大概会是毫秒级别，但是我们也观察到在一些场景下，延时可能会变得非常大。对于那些所有的数据都需要在非常低延时的情况下被处理的应用（10毫秒以内）, Flink 可以取消数据流对齐这个特性。节点在输入流中遇到栅栏checkpoint的时候会打一个快照，而不会等待其他输入流的栅栏。

对齐功能被取消后，即使节点遇到某个checkpoint *n* 的栅栏数据，它也会继续处理所有的输入流，而不会再等待所有的栅栏*n*到齐。这样的情况下，节点有可能先处理某个输入的 *n+1* checkpoint的数据，后处理另外一路输入 *n* checkpoint的数据。
当该节点从错误中恢复后，部分数据有可能会被重复处理，因为他们包含在 *n* 的checkpoint快照中，并且会作为*n*checkpoint后面的部分而被重放。

*注意*: 流对齐仅出现在节点处理多输入（例如join）和多输出流（输出数据重新分组）的场景下。如果在只有并行操作的场景下（例如`map()`,`flatMap()`, `filter()`, ...等等），即使你使用的是*at least once*模式，仍然可以保证达到*exactly once*的效果。

<!--

### Asynchronous State Snapshots

Note that the above described mechanism implies that operators stop processing input records while they are storing a snapshot of their state in the *state backend*. This *synchronous* state snapshot introduces a delay every time a snapshot is taken.

It is possible to let an operator continue processing while it stores its state snapshot, effectively letting the state snapshots happen *asynchronously* in the background. To do that, the operator must be able to produce a state object that should be stored in a way such that further modifications to the operator state do not affect that state object.

After receiving the checkpoint barriers on its inputs, the operator starts the asynchronous snapshot copying of its state. It immediately emits the barrier to its outputs and continues with the regular stream processing. Once the background copy process has completed, it acknowledges the checkpoint to the checkpoint coordinator (the JobManager). The checkpoint is now only complete after all sinks received the barriers and all stateful operators acknowledged their completed backup (which may be later than the barriers reaching the sinks).

User-defined state that is used through the key/value state abstraction can be snapshotted *asynchronously*.
User functions that implement the interface {% gh_link /flink-FIXME/flink-streaming/flink-streaming-java/src/main/java/org/apache/flink/streaming/api/checkpoint/Checkpointed.java "Checkpointed" %} will be snapshotted *synchronously*, while functions that implement {% gh_link /flink-FIXME/flink-streaming/flink-streaming-java/src/main/java/org/apache/flink/streaming/api/checkpoint/CheckpointedAsynchronously.java "CheckpointedAsynchronously" %} will be snapshotted *asynchronously*. Note that for the latter, the user function must guarantee that any future modifications to its state to not affect the state object returned by the `snapshotState()` method.



### Incremental State Snapshots

For large state, taking a snapshot copy of the entire state can be costly, and may prohibit very frequent checkpoints. This problem can be solved by drawing *incremental state snapshots*.
For incremental snapshots, only the changes since the last snapshot are stored in the current snapshot. The state can then be reconstructed by taking the latest full snapshot and applying the incremental changes to the state.

-->


## 错误恢复

错误恢复在上面所说的这种机制下是非常直观的：每当遇到故障时，Flink会选择最新完成的checkpoint *k*来恢复。然后框架重放整个分布式数据流，并且给每个节点恢复之前打好的checkpoint快照 *k* 的状态。数据源被重置到<i>S<sub>k</sub></i>这个位置继续读取数据。例如对于Apache Kafka来说, 就是告知Kafka中的consumer从<i>S<sub>k</sub></i>偏移开始获取数据。

当状态数据非常大的情况下，全量打快照代价非常高，所以可以通过增量式地打快照来解决。即打快照的时候只存相对于上次快照的变化，恢复的时候先恢复最近一次的全量快照，然后增量式地恢复到最近的快照。


