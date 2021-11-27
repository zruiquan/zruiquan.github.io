---
title: 基于Flink的电商用户行为数据分析（2）
date: 2021-09-09 10:08:35
tags:
- Flink
categories:
- Flink
---

## 基于Flink的电商用户行为数据分析（2）

## 实时热门商品统计

### 创建Maven项目

* 打开IDEA，创建一个maven项目，命名为UserBehaviorAnalysis。由于包含了多个模块，我们可以以UserBehaviorAnalysis作为父项目，并在其下建一个名为HotItemsAnalysis的子项目，用于实时统计热门top N商品
* 在UserBehaviorAnalysis下新建一个maven module作为子项目，命名为HotItemsAnalysis
* 父项目只是为了规范化项目结构，方便依赖管理，本身是不需要代码实现的，所以UserBehaviorAnalysis下的src 文件夹可以删掉



```xml
<properties>
    <flink.version>1.10.1</flink.version>
    <scala.binary.version>2.12</scala.binary.version>
    <kafka.version>2.2.0</kafka.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-java</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-streaming-java_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka_${scala.binary.version}</artifactId>
        <version>${kafka.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-connector-kafka_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
    </dependency>
</dependencies>
<build>
    <plugins>
        <plugin>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
                <encoding>UTF-8</encoding>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### 数据准备

将数据文件UserBehavior.csv复制到资源文件目录src/main/resources 下，我们将从这里读取数据。至此，我们的准备工作都已完成，接下来可以写代码了

### 实时热门商品实现

我们将实现一个"实时热门商品"的需求，可以将"实时热门商品"翻译成程序员更好理解的需求：每隔5分钟输出最近一小时内点击量最多的前N个商品。将这个需求进行分解我们大概要做这么几件事情：

* 抽取出业务时间戳，告诉Flink框架基于业务时间做窗口
* 过滤出点击行为数据
* 按一小时的窗口大小,每5分钟统计一次,做滑动窗口聚合( Sliding Window)
* 按每个窗口聚合，输出每个窗口中点击量前N名的商品



#### 程序主体

在 src/main/java/beans 下定义 POJOs： UserBehavior 和 ItemViewCount。 创建类  HotItems， 在 main 方 法中创建 StreamExecutionEnvironment 并做配 置，然后从 UserBehavior.csv 文件中读取数据，并包装成 UserBehavior 类型。代码如下：  

```java
public class HotItems {
    public static void main(String[] args) throws Exception{
        // 1. 创建环境
        StreamExecutionEnvironment env =
        StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        // 2. 读取数据，创建 DataStream
        DataStream<String> inputStream =
        env.readTextFile("..\\UserBehavior.csv");
        // 3. 转换为 POJO，并分配时间戳和 watermark
        DataStream<UserBehavior> dataStream = inputStream
            .map( line -> {
                String[] fields = line.split(",");
                return new UserBehavior(new Long(fields[0]), new
                Long(fields[1]), new Integer(fields[2]), fields[3], new Long(fields[4]));
            } )
            .assignTimestampsAndWatermarks(new AscendingTimestampExtractor<UserBehavior>() {
                @Override
                public long extractAscendingTimestamp(UserBehavior element) {
                	return element.getTimestamp() * 1000L;
                }
         	});
        env.execute("hot items");
    }
}
```

这里注意， 我们需要统计业务时间上的每小时的点击量，所以要基于 EventTime来处理。那么如果让 Flink 按照我们想要的业务时间来处理呢？这里主要有两件事情要做。

第一件是告诉 Flink 我们现在按照 EventTime 模式进行处理， Flink 默认使用ProcessingTime 处理，所以我们要显式设置如下：  

```java
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```

第二件事情是指定如何获得业务时间，以及生成 Watermark。 Watermark 是用来追踪业务事件的概念，可以理解成 EventTime 世界中的时钟，用来指示当前处理到什么时刻的数据了。由于我们的数据源的数据已经经过整理，没有乱序，即事件的时间戳是单调递增的，所以可以将每条数据的业务时间就当做 Watermark。这里我们用 AscendingTimestampExtractor 来实现时间戳的抽取和 Watermark 的生成。  

```java
.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<UserBehavior>() {
	@Override
	public long extractAscendingTimestamp(UserBehavior element) {
		return element.getTimestamp() * 1000L;
	}
});
```

这样我们就得到了一个带有时间标记的数据流了，后面就能做一些窗口的操作。  

#### 过滤出点击事件

在开始窗口操作之前，先回顾下需求“每隔 5 分钟输出过去一小时内点击量最多的前 N 个商品”。由于原始数据中存在点击、购买、收藏、喜欢各种行为的数据，但是我们只需要统计点击量，所以先使用 filter 将点击行为数据过滤出来。  

```java
.filter( data -> "pv".equals(data.getBehavior()) )
```

#### 设置滑动窗口，统计点击量  

由于要每隔 5 分钟统计一次最近一小时每个商品的点击量，所以窗口大小是一小时，每隔 5 分钟滑动一次。即分别要统计[09:00, 10:00), [09:05, 10:05), [09:10,10:10)…等窗口的商品点击量。是一个常见的滑动窗口需求（ Sliding Window）。  

```java
.keyBy("itemId")
.timeWindow(Time.minutes(60), Time.minutes(5))
.aggregate(new CountAgg(), new WindowResultFunction());
```

我们使用.keyBy("itemId")对商品进行分组，使用.timeWindow(Time size, Time slide)对 每 个 商 品 做 滑 动 窗 口 （ 1 小 时 窗 口 ， 5 分 钟 滑 动 一 次 ） 。 然 后 我 们 使用 .aggregate(AggregateFunction af, WindowFunction wf) 做增量的聚合操作，它能使用 AggregateFunction 提 前 聚 合 掉 数 据 ， 减 少 state 的 存 储 压 力 。 较之 .apply(WindowFunction wf) 会将窗口中的数据都存储下来，最后一起计算要高效地多。这里的 CountAgg 实现了 AggregateFunction 接口，功能是统计窗口中的条数，即遇到一条数据就加一。  

```java
// 自定义预聚合函数类，每来一个数据就 count 加 1
public static class ItemCountAgg implements AggregateFunction<UserBehavior, Long, Long>{
    @Override
    public Long createAccumulator() {
    	return 0L;
	}
	@Override
	public Long add(UserBehavior value, Long accumulator) {
    	return accumulator + 1;
    }
    @Override
    public Long getResult(Long accumulator) {
    	return accumulator;
    }
    @Override
    public Long merge(Long a, Long b) {
    	return a + b;
    }
}
```

聚 合 操 作 .aggregate(AggregateFunction af, WindowFunction wf) 的 第 二 个 参 数WindowFunction 将每个 key 每个窗口聚合后的结果带上其他信息进行输出。我们这里 实 现 的 WindowResultFunction 将 < 主 键 商 品 ID ， 窗 口 ， 点 击 量 > 封 装 成 了ItemViewCount 进行输出。  

代码如下:

```java
// 自定义窗口函数，结合窗口信息，输出当前 count 结果
public static class WindowCountResult implements WindowFunction<Long,ItemViewCount, Tuple, TimeWindow>{
    @Override
    public void apply(Tuple tuple, TimeWindow window, Iterable<Long> input,
        Collector<ItemViewCount> out) throws Exception {
        Long itemId = tuple.getField(0);
        Long windowEnd = window.getEnd();
        Long count = input.iterator().next();
        out.collect( new ItemViewCount(itemId, windowEnd, count) );
    }
}
```

现在我们就得到了每个商品在每个窗口的点击量的数据流 。



#### 计算最热门 Top N 商品  

为了统计每个窗口下最热门的商品，我们需要再次按窗口进行分组，这里根据ItemViewCount 中的 windowEnd 进行 keyBy()操作。然后使用 ProcessFunction 实现一个自定义的 TopN 函数 TopNHotItems 来计算点击量排名前 3 名的商品，并将排名结果格式化成字符串，便于后续输出。  

```java
.keyBy("windowEnd")
.process(new TopNHotItems(3)); // 求点击量前 3 名的商品
```

ProcessFunction 是 Flink 提供的一个 low-level API，用于实现更高级的功能。它主要提供了定时器 timer 的功能（支持 EventTime 或 ProcessingTime）。本案例中我们将利用 timer 来判断何时收齐了某个 window 下所有商品的点击量数据。由于Watermark 的 进 度 是 全 局 的 ， 在 processElement 方 法 中 ， 每 当 收 到 一 条 数 据ItemViewCount，我们就注册一个 windowEnd+1 的定时器（ Flink 框架会自动忽略同一时间的重复注册）。 windowEnd+1 的定时器被触发时，意味着收到了 windowEnd+1的 Watermark，即收齐了该 windowEnd 下的所有商品窗口统计值。我们在 onTimer()中处理将收集的所有商品及点击量进行排序，选出 TopN，并将排名信息格式化成字符串后进行输出。

这里我们还使用了 ListState<ItemViewCount>来存储收到的每条 ItemViewCount消息，保证在发生故障时，状态数据的不丢失和一致性。 ListState 是 Flink 提供的类似 Java List 接口的 State API，它集成了框架的 checkpoint 机制，自动做到了exactly-once 的语义保证。  

```java
// 求某个窗口中前 N 名的热门点击商品， key 为窗口时间戳，输出为 TopN 的结果字符串
public static class TopNHotItems extends KeyedProcessFunction<Tuple,ItemViewCount, String>{
    private Integer topSize;
    
	public TopNHotItems(Integer topSize) {
		this.topSize = topSize;
	}
    
	// 定义状态，所有 ItemViewCount 的 List
	ListState<ItemViewCount> itemViewCountListState;
    
    @Override
    public void open(Configuration parameters) throws Exception {
    	itemViewCountListState = getRuntimeContext().getListState(new
   	 	ListStateDescriptor<ItemViewCount>("item-count-list", ItemViewCount.class));
    }
    
	@Override
    public void processElement(ItemViewCount value, Context ctx,Collector<String> out) throws Exception {
    	itemViewCountListState.add(value);
    	ctx.timerService().registerEventTimeTimer(value.getWindowEnd() +1L);
    }
    
    @Override
    public void onTimer(long timestamp, OnTimerContext ctx,Collector<String> out) throws Exception {
    	ArrayList<ItemViewCount> itemViewCounts =
    	Lists.newArrayList(itemViewCountListState.get().iterator());
    	// 排序
    	itemViewCounts.sort(new Comparator<ItemViewCount>() {
    		@Override
    		public int compare(ItemViewCount o1, ItemViewCount o2) {
    		return o2.getCount().intValue() - o1.getCount().intValue();
    	}
    });
        
	// 将排名信息格式化成 String
    StringBuilder result = new StringBuilder();
    result.append("====================================\n");
    result.append("窗口结束时间: ").append(new Timestamp(timestamp -
    1)).append("\n");
    for( int i = 0; i < topSize; i++ ){
    ItemViewCount currentItemViewCount = itemViewCounts.get(i);
    result.append("No").append(i+1).append(":")
    .append(" 商品 ID=")
    .append(currentItemViewCount.getItemId())
    .append(" 浏览量=")
    .append(currentItemViewCount.getCount())
    .append("\n");
    }
    result.append("====================================\n\n");
    // 控制输出频率，模拟实时滚动结果
    Thread.sleep(1000);
    out.collect(result.toString());
}
```

至此整个程序代码全部完成，我们直接运行 main 函数，就可以在控制台看到不断输出的各个时间点统计出的热门商品。  

#### 完整代码

```java
package com.zruiquan.hotitems_analysis;

import com.zruiquan.beans.ItemViewCount;
import com.zruiquan.beans.UserBehavior;
import org.apache.commons.compress.utils.Lists;
import org.apache.flink.api.common.functions.AggregateFunction;
import org.apache.flink.api.common.state.ListState;
import org.apache.flink.api.common.state.ListStateDescriptor;
import org.apache.flink.api.java.tuple.Tuple;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.streaming.api.functions.timestamps.AscendingTimestampExtractor;
import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.util.Collector;

import java.sql.Timestamp;
import java.util.ArrayList;
import java.util.Comparator;

/**
 * @author ruirui
 * @version 1.0.0
 * @create 2021/9/9 10:49
 */
public class HotItems
{
    public static void main(String[] args) throws Exception
    {
        // 创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        //读取数据，创建dataStream
        DataStream<String> inputStream = env.readTextFile("D:\\learn\\UserBehaviorAnalysis\\HotItemsAnalysis\\src\\main\\resources\\UserBehavior.csv");

        //转换pojo,分配时间戳watermark(增序)
        DataStream<UserBehavior> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new UserBehavior(new Long(fields[0]), new Long(fields[1]), new Integer(fields[2]), fields[3], new Long(fields[4]));
        })
                .assignTimestampsAndWatermarks(new AscendingTimestampExtractor<UserBehavior>()
                {
                    @Override
                    public long extractAscendingTimestamp(UserBehavior element)
                    {
                        return element.getTimestamp() * 1000L;
                    }
                });

        // 分组开窗聚合，得到每个窗口内各个商品的count值
        DataStream<ItemViewCount> windowAggStream = dataStream
                .filter(data -> "pv".equals(data.getBehavior())) // 过滤pv行为
                .keyBy("itemId")
                .timeWindow(Time.hours(1), Time.minutes(5))
                .aggregate(new ItemCountAgg(), new WindowItemCountResult());

        //收集同一窗口的所有商品count数据，排序输出top n
        DataStream<String> resultStream = windowAggStream
                .keyBy("windowEnd") // 按照窗口分组
                .process(new TopNHotItems(5)); // 用自定义处理函数排序取前5

        resultStream.print();

        env.execute("hot item analysis");
    }

    //实现自定义增量聚合函数
    public static class ItemCountAgg implements AggregateFunction<UserBehavior, Long, Long>
    {

        @Override
        public Long createAccumulator()
        {
            return 0L;
        }

        @Override
        public Long add(UserBehavior userBehavior, Long accumulator)
        {
            return accumulator + 1;
        }

        @Override
        public Long getResult(Long accumulator)
        {
            return accumulator;
        }

        @Override
        public Long merge(Long a, Long b)
        {
            return a + b;
        }
    }

    //自定义全窗口函数
    public static class WindowItemCountResult implements WindowFunction<Long, ItemViewCount, Tuple, TimeWindow>
    {
        @Override
        public void apply(Tuple tuple, TimeWindow window, Iterable<Long> input, Collector<ItemViewCount> out) throws Exception
        {
            Long itemId = tuple.getField(0);
            Long windowEnd = window.getEnd();
            Long count = input.iterator().next();
            out.collect(new ItemViewCount(itemId, windowEnd, count));
        }
    }

    //实现自定义keyedProcessFunction
    public static class TopNHotItems extends KeyedProcessFunction<Tuple, ItemViewCount, String>
    {
        private Integer topSize;

        public TopNHotItems(Integer topSize)
        {
            this.topSize = topSize;
        }

        //定义列表状态，保存当前窗口内所有输出的ItemViewCount
        ListState<ItemViewCount> itemViewCountListState;

        @Override
        public void open(Configuration parameters) throws Exception
        {
            itemViewCountListState = getRuntimeContext().getListState(new ListStateDescriptor<ItemViewCount>("item-view-count-list", ItemViewCount.class));
        }

        @Override
        public void processElement(ItemViewCount value, Context ctx, Collector<String> collector) throws Exception
        {
            //每来一条数据存入list中，并存入定时器
            itemViewCountListState.add(value);
            ctx.timerService().registerEventTimeTimer(value.getWindowEnd() + 1);
        }

        @Override
        public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception
        {
            //定时器触发，当前已收集到所有数据，排序输出
            ArrayList<ItemViewCount> itemViewCounts = Lists.newArrayList(itemViewCountListState.get().iterator());
            itemViewCounts.sort(new Comparator<ItemViewCount>()
            {
                @Override
                public int compare(ItemViewCount o1, ItemViewCount o2)
                {
                    return o2.getCount().intValue() - o1.getCount().intValue();
                }
            });

            //将排名信息格式化成String ，方便打印输出
            StringBuilder resultBuilder = new StringBuilder();
            resultBuilder.append("==============================\n");
            resultBuilder.append("窗口结束时间：").append(new Timestamp(timestamp - 1)).append("\n");

            for (int i = 0; i < Math.min(topSize, itemViewCounts.size()); i++)
            {
                ItemViewCount itemViewCount = itemViewCounts.get(i);
                resultBuilder.append("NO ").append(i + 1).append(":")
                        .append(" 商品ID= ").append(itemViewCount.getItemId())
                        .append(" 热门度= ").append(itemViewCount.getCount())
                        .append("\n");
            }
            resultBuilder.append("===============================\n\n");

            //控制输出频率
            Thread.sleep(1000L);
            out.collect(resultBuilder.toString());
        }
    }

}

```

#### 更换Kafka数据源

实际生产环境中， 我们的数据流往往是从 Kafka 获取到的。如果要让代码更贴近生产实际，我们只需将 source 更换为 Kafka 即可：  

```java
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
properties.setProperty("group.id", "consumer-group");
properties.setProperty("key.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
properties.setProperty("value.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
properties.setProperty("auto.offset.reset", "latest");
DataStream<String> inputStream = env.addSource(new FlinkKafkaConsumer<String>("hotitems",new SimpleStringSchema(), properties));
```

