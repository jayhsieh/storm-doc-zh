---
title: Windowing Support in Core Storm
layout: documentation
documentation: true
---

Storm core 支持处理落在窗口内的一组元组。窗口操作指定了一下两个参数

    1.窗口的长度 - 窗口的长度或持续时间

    2.滑动间隔 - 窗口滑动的时间间隔

## 滑动窗口

元组被分组在窗口和每个滑动间隔窗口中。 一个元组可以属于多个窗口。

例如一个持续时间长度为 10 秒和滑动间隔 5 秒的滑动窗口。
```
........| e1 e2 | e3 e4 e5 e6 | e7 e8 e9 |...
-5      0       5            10          15   -> time
|<------- w1 -->|
        |<---------- w2 ----->|
                |<-------------- w3 ---->|
```
窗口每5秒进行一次评估，第一个窗口中的某些元组与第二个窗口重叠。

注意：窗口第一次滑动在 t = 5s，并且将包含在前 5 秒钟内收到的事件。
## Tumbling Window

元组根据时间或数量被分组在一个窗口中。任何元组只属于其中一个窗口。

例如一个持续时间长度为 5s 的 tumbling window。

```
| e1 e2 | e3 e4 e5 e6 | e7 e8 e9 |...
0       5             10         15    -> time
   w1         w2            w3
```

窗口每五秒进行一次评估，并且没有窗口重叠。

Storm 支持指定窗口长度和滑动间隔作为元组数的计数或持续时间。
bolt 接口 `IWindowedBolt` 需要由窗口支持的bolts来实现。

```java
public interface IWindowedBolt extends IComponent {
    void prepare(Map stormConf, TopologyContext context, OutputCollector collector);
    /**
     * Process tuples falling within the window and optionally emit 
     * new tuples based on the tuples in the input window.
     */
    void execute(TupleWindow inputWindow);
    void cleanup();
}
```

每次窗口激活时，都会调用 `execute` 方法。TupleWindow 的参数允许访问窗口中的当前元组，过期的元组以及自上次窗口计算后添加的新元组，这对于高效的窗口计算将是有用的。

需要窗口支持的 Bolts 一般会扩展为 `BaseWindowedBolt`，它有用来指定窗口长度和滑动间隔的apis.

例如

```java
public class SlidingWindowBolt extends BaseWindowedBolt {
	private OutputCollector collector;
	
    @Override
    public void prepare(Map stormConf, TopologyContext context, OutputCollector collector) {
    	this.collector = collector;
    }
	
    @Override
    public void execute(TupleWindow inputWindow) {
	  for(Tuple tuple: inputWindow.get()) {
	    // do the windowing computation
		...
	  }
	  // emit the results
	  collector.emit(new Values(computedValue));
    }
}

public static void main(String[] args) {
    TopologyBuilder builder = new TopologyBuilder();
     builder.setSpout("spout", new RandomSentenceSpout(), 1);
     builder.setBolt("slidingwindowbolt", 
                     new SlidingWindowBolt().withWindow(new Count(30), new Count(10)),
                     1).shuffleGrouping("spout");
    Config conf = new Config();
    conf.setDebug(true);
    conf.setNumWorkers(1);

    StormSubmitter.submitTopologyWithProgressBar(args[0], conf, builder.createTopology());
	
}
```

支持以下窗口配置

```java
withWindow(Count windowLength, Count slidingInterval)
基于元组计数的滑动窗口，在多个tuples进行 `slidingInterval`滑动之后。

withWindow(Count windowLength)
基于元组计数的窗口，它与每个传入的元组一起滑动。

withWindow(Count windowLength, Duration slidingInterval)
基于元组计数的滑动窗口，在`slidingInterval`持续时间滑动之后。

withWindow(Duration windowLength, Duration slidingInterval)
基于持续时间的滑动窗口，在`slidingInterval`持续时间滑动之后。

withWindow(Duration windowLength)
基于持续时间的窗口，它与每个传入的元组一起滑动。

withWindow(Duration windowLength, Count slidingInterval)
基于时间的滑动窗口配置在`slidingInterval`多个元组之后滑动。

withTumblingWindow(BaseWindowedBolt.Count count)
计数的tumbling窗口在指定的元组数之后tumbles.

withTumblingWindow(BaseWindowedBolt.Duration duration)
基于持续时间的tumbling窗口在指定的持续时间后tumbles。

```

## 元组时间戳和乱序元组
默认情况下，在窗口中追踪的时间戳是 bolt 处理元组的时间。窗口计算是根据正在处理的时间戳进行的。 Storm 支持基于源生成的时间戳的追踪窗口。

```java
/**
* Specify a field in the tuple that represents the timestamp as a long value. If this
* field is not present in the incoming tuple, an {@link IllegalArgumentException} will be thrown.
*
* @param fieldName the name of the field that contains the timestamp
*/
public BaseWindowedBolt withTimestampField(String fieldName)
```
上述`fieldName`的值将从传入的元组中查找并考虑进行窗口计算。如果该元组中不存在该字段，将抛出异常。或者，[TimestampExtractor](../storm-core/src/jvm/org/apache/storm/windowing/TimestampExtractor.java)可以用于从元组导出时间戳值（例如，从元组中的嵌套字段提取时间戳）。

```java
/**
* Specify the timestamp extractor implementation.
*
* @param timestampExtractor the {@link TimestampExtractor} implementation
*/
public BaseWindowedBolt withTimestampExtractor(TimestampExtractor timestampExtractor)
```
与时间戳字段 name/extractor 一起，可以指定一个时间滞后参数，它指示具有无序时间戳的元组的最大时间限制。

```java
/**
* Specify the maximum time lag of the tuple timestamp in milliseconds. It means that the tuple timestamps
* cannot be out of order by more than this amount.
*
* @param duration the max lag duration
*/
public BaseWindowedBolt withLag(Duration duration)
```
例如：如果滞后是5秒，并且元组`t1`到达时间戳为`06：00：05`没有元组可能会在早于`06：00：00`的元组时间戳到达。 如果一个元组在`t1`之后到达时间戳`05:59:59`，并且窗口已经移动过`t1`了，它将被视为迟到的元组。 默认情况下不处理迟到的元组，只需在INFO级别打印到工作日志文件。
```java
/**
 * Specify a stream id on which late tuples are going to be emitted. They are going to be accessible via the
 * {@link org.apache.storm.topology.WindowedBoltExecutor#LATE_TUPLE_FIELD} field.
 * It must be defined on a per-component basis, and in conjunction with the
 * {@link BaseWindowedBolt#withTimestampField}, otherwise {@link IllegalArgumentException} will be thrown.
 *
 * @param streamId the name of the stream used to emit late tuples on
 */
public BaseWindowedBolt withLateTupleStream(String streamId)

```

通过指定上述 `streamId` 来更改此行为。 在这种情况下，迟到的元组将在指定的流中发出并可通过`WindowedBoltExecutor.LATE_TUPLE_FIELD` 访问
字段。


### Watermarks
为了处理具有时间戳字段的元组，storm 根据传入的元组时间戳内部计算 watermarks。Watermark 是所有输入流中最新的元组时间戳（减去滞后）的最小值。在较高级别，watermark类似于 Flink 和 Google 的 MillWheel 用于跟踪基于事件的时间戳的概念。

定期的（默认每秒），watermark时间戳被发出，如果基于元组的时间戳被使用，这被认为是窗口计算的  clock tick（时钟勾）。可以用下面的api来改变发出 watermarks 的时间间隔。
 
```java
/**
* Specify the watermark event generation interval. For tuple based timestamps, watermark events
* are used to track the progress of time
*
* @param interval the interval at which watermark events are generated
*/
public BaseWindowedBolt withWatermarkInterval(Duration interval)
```
当接收到watermark时，将对所有时间戳记进行评估。

例如，考虑具有以下窗口参数基于元组的时间戳处理，

`Window length = 20s, sliding interval = 10s, watermark emit frequency = 1s, max lag = 5s`

```
|-----|-----|-----|-----|-----|-----|-----|
0     10    20    30    40    50    60    70
````

当前 ts = `09:00:00`

在`9:00:00`到`9:00:01`收到的元组`e1(6:00:03), e2(6:00:05), e3(6:00:07), e4(6:00:18), e5(6:00:26), e6(6:00:36)`

在time t = `09:00:01`, watermark w1 = `6:00:31`被发出,没有早于`6:00:31`的元组可以到达。

三个窗口将被评估。通过采取最早的事件时间戳（06:00:03）并基于滑动间隔（10s）计算上限来计算第一个窗口结束在 ts（06:00:10）。

1. `5:59:50 - 06:00:10` 有元组 e1, e2, e3
2. `6:00:00 - 06:00:20` 有元组 e1, e2, e3, e4
3. `6:00:10 - 06:00:30` 有元组 e4, e5

 e6未被评估，因为 watermark 时间戳`6:00:31`比元组 ts`6:00:36`更旧。

在`9:00:01`和 `9:00:02`之间，接收到的元组`e7(8:00:25), e8(8:00:26), e9(8:00:27), e10(8:00:39)`

在 time t = `09:00:02`另一个 watermark w2 = `08:00:34`被发出，没有元组比`8:00:34`更早到达。

三个窗口将被评估

1. `6:00:20 - 06:00:40` 有元组 e5, e6 (从早期批次)
2. `6:00:30 - 06:00:50` 有元组 e6 (从早期批次)
3. `8:00:10 - 08:00:30` 有元组 e7, e8, e9

e10 不被评估，因为元组 ts `8:00:39`超出了watermark time `8:00:34`.

窗口计算考虑时间间隔，并基于元组时间戳计算窗口。

## Guarantees
storm core的窗口功能目前提供一致性保证。`执行（TupleWindow inputWindow）`方法发出的值将自动锁定到 inputWindow 中的所有元组。预计下游 bolts 将确认接收的元组（即从窗口 bolt 发出的元组）以完成元组树。如果不是，元组将重播，并且重新评估窗口计算。

窗口中的元组会在过期后被自动确认，即当它们在`windowLength + slidingInterval`之后从窗口中滑落出来。请注意，配置`topology.message.timeout.secs`应该远远超过基于时间窗口的`windowLength + slidingInterval`; 否则元组将超时并重播，并可能导致重复的评估。对于基于计数的窗口，应该调整配置，使得在超时时间段内可以接收到`windowLength + slidingInterval`元组。

## 拓扑示例

示例拓扑`滑动窗口拓扑`显示了如何使用apis来计算滑动窗口总和和滚动窗口平均值。
