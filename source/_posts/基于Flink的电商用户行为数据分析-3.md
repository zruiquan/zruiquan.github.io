---
title: 基于Flink的电商用户行为数据分析（3）
date: 2021-09-15 10:26:30
tags:
- Flink
categories:
- Flink
---

## 基于Flink的电商用户行为数据分析（3）



### 模块创建和数据准备  

在 UserBehaviorAnalysis 下 新 建 一 个 maven module 作 为 子 项 目 ， 命 名 为NetworkFlowAnalysis。在这个子模块中，我们同样并没有引入更多的依赖，所以也不需要改动 pom 文件。将 apache 服务器的日志文件 apache.log 复制到资源文件目录 src/main/resources下，我们将从这里读取数据。当然，我们也可以仍然用 UserBehavior.csv 作为数据源，这时我们分析的就不是每一次对服务器的访问请求了，而是具体的页面浏览（ “pv”）操作。  

### 基于服务器 log 的热门页面浏览量统计 

我们现在要实现的模块是 “实时流量统计”。对于一个电商平台而言，用户登录的入口流量、不同页面的访问流量都是值得分析的重要数据，而这些数据，可以简单地从 web 服务器的日志中提取出来。

我们在这里先实现“ 热门页面浏览数”的统计，也就是读取服务器日志中的每一行 log， 统计在一段时间内用户访问每一个 url 的次数，然后排序输出显示。具体做法为： 每隔 5 秒，输出最近 10 分钟内访问量最多的前 N 个 URL。 可以看出，这个需求与之前    “实时热门商品统计”   非常类似，所以我们完全可以借鉴此前的代码。

在 src/main/java 下 创 建 NetworkFlow 类 ， 在 beans 下 定 义 POJO 类ApacheLogEvent，这是输入的日志数据流；另外还有 UrlViewCount， 这是窗口操作统计的输出数据类型。 在 main 函数中创建 StreamExecutionEnvironment 并做配置，然后从 apache.log 文件中读取数据，并包装成 ApacheLogEvent 类型。

需要注意的是，原始日志中的时间是“ dd/MM/yyyy:HH:mm:ss”的形式，需要定义一个 DateTimeFormat 将其转换为我们需要的时间戳格式：  

```java
.map( line -> {
    String[] fields = line.split(" ");
    SimpleDateFormat simpleDateFormat = new
    SimpleDateFormat("dd/MM/yyyy:HH:mm:ss");
    Long timestamp = simpleDateFormat.parse(fields[3]).getTime();
    return new ApacheLogEvent(fields[0], fields[1], timestamp, fields[5],
    fields[6]);
} )
```

完整代码如下：

```java
package com.test.networkflow_analysis;
import akka.protobuf.ByteString;
import com.atguigu.networkflow_analysis.beans.ApacheLogEvent;
import com.atguigu.networkflow_analysis.beans.PageViewCount;
import org.apache.flink.api.common.functions.AggregateFunction;
import org.apache.flink.api.common.state.ListState;
import org.apache.flink.api.common.state.ListStateDescriptor;
import org.apache.flink.api.common.state.MapState;
import org.apache.flink.api.common.state.MapStateDescriptor;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.shaded.guava18.com.google.common.collect.Lists;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.util.Collector;
import org.apache.flink.util.OutputTag;
import java.net.URL;
import java.sql.Timestamp;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.Map;
import java.util.regex.Pattern;

public class HotPages {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        env.setParallelism(1);

        // 读取文件，转换成POJO
//        URL resource = HotPages.class.getResource("/apache.log");
//        DataStream<String> inputStream = env.readTextFile(resource.getPath());
        DataStream<String> inputStream = env.socketTextStream("localhost", 7777);

        DataStream<ApacheLogEvent> dataStream = inputStream
                .map(line -> {
                    String[] fields = line.split(" ");
                    SimpleDateFormat simpleDateFormat = new SimpleDateFormat("dd/MM/yyyy:HH:mm:ss");
                    Long timestamp = simpleDateFormat.parse(fields[3]).getTime();
                    return new ApacheLogEvent(fields[0], fields[1], timestamp, fields[5], fields[6]);
                })
                .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<ApacheLogEvent>(Time.seconds(1)) {
                    @Override
                    public long extractTimestamp(ApacheLogEvent element) {
                        return element.getTimestamp();
                    }
                });

        dataStream.print("data");

        // 分组开窗聚合

        // 定义一个侧输出流标签
        OutputTag<ApacheLogEvent> lateTag = new OutputTag<ApacheLogEvent>("late"){};

        SingleOutputStreamOperator<PageViewCount> windowAggStream = dataStream
                .filter(data -> "GET".equals(data.getMethod()))    // 过滤get请求
                .filter(data -> {
                    String regex = "^((?!\\.(css|js|png|ico)$).)*$";
                    return Pattern.matches(regex, data.getUrl());
                })
                .keyBy(ApacheLogEvent::getUrl)    // 按照url分组
                .timeWindow(Time.minutes(10), Time.seconds(5))
                .allowedLateness(Time.minutes(1))
                .sideOutputLateData(lateTag)
                .aggregate(new PageCountAgg(), new PageCountResult());

        windowAggStream.print("agg");
        windowAggStream.getSideOutput(lateTag).print("late");

        // 收集同一窗口count数据，排序输出
        DataStream<String> resultStream = windowAggStream
                .keyBy(PageViewCount::getWindowEnd)
                .process(new TopNHotPages(3));

        resultStream.print();

        env.execute("hot pages job");
    }

    // 自定义预聚合函数
    public static class PageCountAgg implements AggregateFunction<ApacheLogEvent, Long, Long> {
        @Override
        public Long createAccumulator() {
            return 0L;
        }

        @Override
        public Long add(ApacheLogEvent value, Long accumulator) {
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

    // 实现自定义的窗口函数
    public static class PageCountResult implements WindowFunction<Long, PageViewCount, String, TimeWindow> {
        @Override
        public void apply(String url, TimeWindow window, Iterable<Long> input, Collector<PageViewCount> out) throws Exception {
            out.collect(new PageViewCount(url, window.getEnd(), input.iterator().next()));
        }
    }

    // 实现自定义的处理函数
    public static class TopNHotPages extends KeyedProcessFunction<Long, PageViewCount, String> {
        private Integer topSize;

        public TopNHotPages(Integer topSize) {
            this.topSize = topSize;
        }

        // 定义状态，保存当前所有PageViewCount到Map中
//        ListState<PageViewCount> pageViewCountListState;
        MapState<String, Long> pageViewCountMapState;

        @Override
        public void open(Configuration parameters) throws Exception {
//            pageViewCountListState = getRuntimeContext().getListState(new ListStateDescriptor<PageViewCount>("page-count-list", PageViewCount.class));
            pageViewCountMapState = getRuntimeContext().getMapState(new MapStateDescriptor<String, Long>("page-count-map", String.class, Long.class));
        }

        @Override
        public void processElement(PageViewCount value, Context ctx, Collector<String> out) throws Exception {
//          pageViewCountListState.add(value);
            pageViewCountMapState.put(value.getUrl(), value.getCount());
            ctx.timerService().registerEventTimeTimer(value.getWindowEnd() + 1);
            // 注册一个1分钟之后的定时器，用来清空状态
            ctx.timerService().registerEventTimeTimer(value.getWindowEnd() + 60 * 1000L);
        }

        @Override
        public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception {
            // 先判断是否到了窗口关闭清理时间，如果是，直接清空状态返回
            if( timestamp == ctx.getCurrentKey() + 60 * 1000L ){
                pageViewCountMapState.clear();
                return;
            }

            ArrayList<Map.Entry<String, Long>> pageViewCounts = Lists.newArrayList(pageViewCountMapState.entries());

            pageViewCounts.sort(new Comparator<Map.Entry<String, Long>>() {
                @Override
                public int compare(Map.Entry<String, Long> o1, Map.Entry<String, Long> o2) {
                    if(o1.getValue() > o2.getValue())
                        return -1;
                    else if(o1.getValue() < o2.getValue())
                        return 1;
                    else
                        return 0;
                }
            });

            // 格式化成String输出
            StringBuilder resultBuilder = new StringBuilder();
            resultBuilder.append("===================================\n");
            resultBuilder.append("窗口结束时间：").append(new Timestamp(timestamp - 1)).append("\n");

            // 遍历列表，取top n输出
            for (int i = 0; i < Math.min(topSize, pageViewCounts.size()); i++) {
                Map.Entry<String, Long> currentItemViewCount = pageViewCounts.get(i);
                resultBuilder.append("NO ").append(i + 1).append(":")
                        .append(" 页面URL = ").append(currentItemViewCount.getKey())
                        .append(" 浏览量 = ").append(currentItemViewCount.getValue())
                        .append("\n");
            }
            resultBuilder.append("===============================\n\n");

            // 控制输出频率
            Thread.sleep(1000L);

            out.collect(resultBuilder.toString());

			// pageViewCountListState.clear();
        }
    }
}
```

### 基于埋点日志数据的网络流量统计  

我们发现，从 web 服务器 log 中得到的 url，往往更多的是请求某个资源地址（ /*.js、 /*.css），如果要针对页面进行统计往往还需要进行过滤。而在实际电商应用中，相比每个单独页面的访问量，我们可能更加关心整个电商网站的网络流量。这个指标，除了合并之前每个页面的统计结果之外，还可以通过统计埋点日志数据
中的 “pv” 行为来得到。  

#### 网站总浏览量（PV）的统计  

衡量网站流量一个最简单的指标，就是网站的页面浏览量（ Page View， PV）。用户每次打开一个页面便记录 1 次 PV，多次打开同一页面则浏览量累计。一般来说，PV 与来访者的数量成正比，但是 PV 并不直接决定页面的真实来访者数量，如同一个来访者通过不断的刷新页面，也可以制造出非常高的 PV。

我们知道，用户浏览页面时，会从浏览器向网络服务器发出一个请求（ Request），网络服务器接到这个请求后，会将该请求对应的一个网页（ Page）发送给浏览器，从而产生了一个 PV。 所以我们的统计方法，可以是从 web 服务器的日志中去提取对应的页面访问然后统计，就向上一节中的做法一样；也可以直接从埋点日志中提取用户发来的页面请求，从而统计出总浏览量。

所以，接下来我们用 UserBehavior.csv 作为数据源，实现一个网站总浏览量的统计。 我们可以设置滚动时间窗口，实时统计每小时内的网站 PV。在 src/main/java 下创建 PageView.java 文件， 具体代码如下：  

```java
package com.test.networkflow_analysis;
import com.test.networkflow_analysis.beans.PageViewCount;
import com.test.networkflow_analysis.beans.UserBehavior;
import org.apache.flink.api.common.functions.AggregateFunction;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.java.tuple.Tuple;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.streaming.api.functions.timestamps.AscendingTimestampExtractor;
import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.util.Collector;
import java.net.URL;
import java.util.Random;

public class PageView {
    public static void main(String[] args) throws Exception{
        // 1. 创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(4);
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        // 2. 读取数据，创建DataStream
        URL resource = PageView.class.getResource("/UserBehavior.csv");
        DataStream<String> inputStream = env.readTextFile(resource.getPath());

        // 3. 转换为POJO，分配时间戳和watermark
        DataStream<UserBehavior> dataStream = inputStream
                .map(line -> {
                    String[] fields = line.split(",");
                    return new UserBehavior(new Long(fields[0]), new Long(fields[1]), new Integer(fields[2]), fields[3], new 						Long(fields[4]));
                })
                .assignTimestampsAndWatermarks(new AscendingTimestampExtractor<UserBehavior>() {
                    @Override
                    public long extractAscendingTimestamp(UserBehavior element) {
                        return element.getTimestamp() * 1000L;
                    }
                });

        // 4. 分组开窗聚合，得到每个窗口内各个商品的count值
        SingleOutputStreamOperator<Tuple2<String, Long>> pvResultStream0 =
                dataStream
                .filter(data -> "pv".equals(data.getBehavior()))    // 过滤pv行为
                .map(new MapFunction<UserBehavior, Tuple2<String, Long>>() {
                    @Override
                    public Tuple2<String, Long> map(UserBehavior value) throws Exception {
                        return new Tuple2<>("pv", 1L);
                    }
                })
                .keyBy(0)    // 按商品ID分组
                .timeWindow(Time.hours(1))    // 开1小时滚动窗口
                .sum(1);

        //  并行任务改进，设计随机key，解决数据倾斜问题
        SingleOutputStreamOperator<PageViewCount> pvStream = dataStream.filter(data -> "pv".equals(data.getBehavior()))
                .map(new MapFunction<UserBehavior, Tuple2<Integer, Long>>() {
                    @Override
                    public Tuple2<Integer, Long> map(UserBehavior value) throws Exception {
                        Random random = new Random();
                        return new Tuple2<>(random.nextInt(10), 1L);
                    }
                })
                .keyBy(data -> data.f0)
                .timeWindow(Time.hours(1))
                .aggregate(new PvCountAgg(), new PvCountResult());

        // 将各分区数据汇总起来
        DataStream<PageViewCount> pvResultStream = pvStream
                .keyBy(PageViewCount::getWindowEnd)
                .process(new TotalPvCount());
        //      .sum("count");

        pvResultStream.print();

        env.execute("pv count job");
    }

    // 实现自定义预聚合函数
    public static class PvCountAgg implements AggregateFunction<Tuple2<Integer, Long>, Long, Long>{
        @Override
        public Long createAccumulator() {
            return 0L;
        }

        @Override
        public Long add(Tuple2<Integer, Long> value, Long accumulator) {
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

    // 实现自定义窗口
    public static class PvCountResult implements WindowFunction<Long, PageViewCount, Integer, TimeWindow>{
        @Override
        public void apply(Integer integer, TimeWindow window, Iterable<Long> input, Collector<PageViewCount> out) throws Exception {
            out.collect( new PageViewCount(integer.toString(), window.getEnd(), input.iterator().next()) );
        }
    }

    // 实现自定义处理函数，把相同窗口分组统计的count值叠加
    public static class TotalPvCount extends KeyedProcessFunction<Long, PageViewCount, PageViewCount>{
        // 定义状态，保存当前的总count值
        ValueState<Long> totalCountState;

        @Override
        public void open(Configuration parameters) throws Exception {
            totalCountState = getRuntimeContext().getState(new ValueStateDescriptor<Long>("total-count", Long.class, 0L));
        }

        @Override
        public void processElement(PageViewCount value, Context ctx, Collector<PageViewCount> out) throws Exception {
            totalCountState.update( totalCountState.value() + value.getCount() );
            ctx.timerService().registerEventTimeTimer(value.getWindowEnd() + 1);
        }

        @Override
        public void onTimer(long timestamp, OnTimerContext ctx, Collector<PageViewCount> out) throws Exception {
            // 定时器触发，所有分组count值都到齐，直接输出当前的总count数量
            Long totalCount = totalCountState.value();
            out.collect(new PageViewCount("pv", ctx.getCurrentKey(), totalCount));
            // 清空状态
            totalCountState.clear();
        }
    }
}
```

#### 网站独立访客数（UV）的统计  

在上节的例子中，我们统计的是所有用户对页面的所有浏览行为，也就是说，同一用户的浏览行为会被重复统计。而在实际应用中，我们往往还会关注，在一段
时间内到底有多少不同的用户访问了网站。
另外一个统计流量的重要指标是网站的独立访客数（ Unique Visitor， UV）。 UV指的是一段时间（比如一小时）内访问网站的总人数， 1 天内同一访客的多次访问只记录为一个访客。通过 IP 和 cookie 一般是判断 UV 值的两种方式。 当客户端第一次访问某个网站服务器的时候，网站服务器会给这个客户端的电脑发出一个 Cookie，通常放在这个客户端电脑的 C 盘当中。在这个 Cookie 中会分配一个独一无二的编号，这其中会记录一些访问服务器的信息，如访问时间，访问了哪些页面等等。当你下次再访问这个服务器的时候， 服务器就可以直接从你的电脑中找到上一次放进去的Cookie 文件，并且对其进行一些更新，但那个独一无二的编号是不会变的。

当然，对于 UserBehavior 数据源来说，我们直接可以根据 userId 来区分不同的
用户。

在 src/main/java 下创建 UniqueVisitor.java 文件，具体代码如下：  

```java
package com.test.networkflow_analysis;
import com.test.networkflow_analysis.beans.PageViewCount;
import com.test.networkflow_analysis.beans.UserBehavior;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.AscendingTimestampExtractor;
import org.apache.flink.streaming.api.functions.windowing.AllWindowFunction;
import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.util.Collector;
import java.net.URL;
import java.util.HashSet;

public class UniqueVisitor {
    public static void main(String[] args) throws Exception {
        // 1. 创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        // 2. 读取数据，创建DataStream
        URL resource = UniqueVisitor.class.getResource("/UserBehavior.csv");
        DataStream<String> inputStream = env.readTextFile(resource.getPath());

        // 3. 转换为POJO，分配时间戳和watermark
        DataStream<UserBehavior> dataStream = inputStream
                .map(line -> {
                    String[] fields = line.split(",");
                    return new UserBehavior(new Long(fields[0]), new Long(fields[1]), new Integer(fields[2]), fields[3], new 						Long(fields[4]));
                })
                .assignTimestampsAndWatermarks(new AscendingTimestampExtractor<UserBehavior>() {
                    @Override
                    public long extractAscendingTimestamp(UserBehavior element) {
                        return element.getTimestamp() * 1000L;
                    }
                });

        // 开窗统计uv值
        SingleOutputStreamOperator<PageViewCount> uvStream = dataStream.filter(data -> "pv".equals(data.getBehavior()))
                .timeWindowAll(Time.hours(1))
                .apply(new UvCountResult());

        uvStream.print();

        env.execute("uv count job");
    }

    // 实现自定义全窗口函数
    public static class UvCountResult implements AllWindowFunction<UserBehavior, PageViewCount, TimeWindow>{
        @Override
        public void apply(TimeWindow window, Iterable<UserBehavior> values, Collector<PageViewCount> out) throws Exception {
            // 定义一个Set结构，保存窗口中的所有userId，自动去重
            HashSet<Long> uidSet = new HashSet<>();
            for (UserBehavior ub: values)
                uidSet.add(ub.getUserId());
            out.collect( new PageViewCount("uv", window.getEnd(), (long)uidSet.size()) );
        }
    }
}
```

#### 使用布隆过滤器的UV统计

在上节的例子中，我们把所有数据的 userId 都存在了窗口计算的状态里，在窗口收集数据的过程中，状态会不断增大。一般情况下，只要不超出内存的承受范围，这种做法也没什么问题；但如果我们遇到的数据量很大呢？

把所有数据暂存放到内存里，显然不是一个好注意。我们会想到，可以利用 redis这种内存级 k-v 数据库，为我们做一个缓存。但如果我们遇到的情况非常极端，数据大到惊人呢？比如上亿级的用户，要去重计算 UV。

如果放到 redis 中，亿级的用户 id（每个 20 字节左右的话）可能需要几 G 甚至几十 G 的空间来存储。当然放到 redis 中，用集群进行扩展也不是不可以，但明显
代价太大了。

一个更好的想法是，其实我们不需要完整地存储用户 ID 的信息，只要知道他在不在就行了。所以其实我们可以进行压缩处理，用一位（ bit）就可以表示一个用户
的状态。这个思想的具体实现就是布隆过滤器（ Bloom Filter）。

本质上布隆过滤器是一种数据结构，比较巧妙的概率型数据结构（ probabilisticdata structure），特点是高效地插入和查询，可以用来告诉你 “某样东西一定不存
在或者可能存在”。

它本身是一个很长的二进制向量，既然是二进制的向量，那么显而易见的，存放的不是 0，就是 1。 相比于传统的 List、 Set、 Map 等数据结构，它更高效、占用
空间更少，但是缺点是其返回的结果是概率性的，而不是确切的。
我们的目标就是，利用某种方法（一般是 Hash 函数）把每个数据，对应到一个位图的某一位上去；如果数据存在，那一位就是 1，不存在则为 0。    

注意这里我们用到了 redis 连接存取数据，所以需要加入 redis 客户端的依赖  ：

```xml
<dependencies>
	<dependency>
		<groupId>redis.clients</groupId>
		<artifactId>jedis</artifactId>
		<version>2.8.1</version>
	</dependency>
</dependencies>
```

在 src/main/java 下创建 UniqueVisitor.java 文件，具体代码如下：  

```java
package com.test.networkflow_analysis;
import com.test.networkflow_analysis.beans.PageViewCount;
import com.test.networkflow_analysis.beans.UserBehavior;
import kafka.server.DynamicConfig;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.AscendingTimestampExtractor;
import org.apache.flink.streaming.api.functions.windowing.ProcessAllWindowFunction;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.triggers.Trigger;
import org.apache.flink.streaming.api.windowing.triggers.TriggerResult;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.util.Collector;
import redis.clients.jedis.Jedis;
import java.net.URL;

public class UvWithBloomFilter { 
    public static void main(String[] args) throws Exception {
        // 1. 创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        // 2. 读取数据，创建DataStream
        URL resource = UniqueVisitor.class.getResource("/UserBehavior.csv");
        DataStream<String> inputStream = env.readTextFile(resource.getPath());

        // 3. 转换为POJO，分配时间戳和watermark
        DataStream<UserBehavior> dataStream = inputStream
                .map(line -> {
                    String[] fields = line.split(",");
                    return new UserBehavior(new Long(fields[0]), new Long(fields[1]), new Integer(fields[2]), fields[3], new 						Long(fields[4]));
                })
                .assignTimestampsAndWatermarks(new AscendingTimestampExtractor<UserBehavior>() {
                    @Override
                    public long extractAscendingTimestamp(UserBehavior element) {
                        return element.getTimestamp() * 1000L;
                    }
                });

        // 开窗统计uv值
        SingleOutputStreamOperator<PageViewCount> uvStream = dataStream
                .filter(data -> "pv".equals(data.getBehavior()))
                .timeWindowAll(Time.hours(1))
                .trigger( new MyTrigger() )
                .process( new UvCountResultWithBloomFliter() );

        uvStream.print();

        env.execute("uv count with bloom filter job");
    }

    // 自定义触发器
    public static class MyTrigger extends Trigger<UserBehavior, TimeWindow>{
        @Override
        public TriggerResult onElement(UserBehavior element, long timestamp, TimeWindow window, TriggerContext ctx) throws 		         	 Exception {
            // 每一条数据来到，直接触发窗口计算，并且直接清空窗口
            return TriggerResult.FIRE_AND_PURGE;
        }

        @Override
        public TriggerResult onProcessingTime(long time, TimeWindow window, TriggerContext ctx) throws Exception {
            return TriggerResult.CONTINUE;
        }

        @Override
        public TriggerResult onEventTime(long time, TimeWindow window, TriggerContext ctx) throws Exception {
            return TriggerResult.CONTINUE;
        }

        @Override
        public void clear(TimeWindow window, TriggerContext ctx) throws Exception {
        }
    }

    // 自定义一个布隆过滤器
    public static class MyBloomFilter {
        // 定义位图的大小，一般需要定义为2的整次幂
        private Integer cap;

        public MyBloomFilter(Integer cap) {
            this.cap = cap;
        }
        // 实现一个hash函数
        public Long hashCode( String value, Integer seed ){
            Long result = 0L;
            for( int i = 0; i < value.length(); i++ ){
                result = result * seed + value.charAt(i);
            }
            return result & (cap - 1);
        }
    }

    // 实现自定义的处理函数
    public static class UvCountResultWithBloomFliter extends ProcessAllWindowFunction<UserBehavior, PageViewCount, TimeWindow>{
        // 定义jedis连接和布隆过滤器
        Jedis jedis;
        MyBloomFilter myBloomFilter;

        @Override
        public void open(Configuration parameters) throws Exception {
            jedis = new Jedis("localhost", 6379);
            myBloomFilter = new MyBloomFilter(1<<29);    // 要处理1亿个数据，用64MB大小的位图
        }

        @Override
        public void process(Context context, Iterable<UserBehavior> elements, Collector<PageViewCount> out) throws Exception {
            // 将位图和窗口count值全部存入redis，用windowEnd作为key
            Long windowEnd = context.window().getEnd();
            String bitmapKey = windowEnd.toString();
            // 把count值存成一张hash表
            String countHashName = "uv_count";
            String countKey = windowEnd.toString();

            // 1. 取当前的userId
            Long userId = elements.iterator().next().getUserId();

            // 2. 计算位图中的offset
            Long offset = myBloomFilter.hashCode(userId.toString(), 61);

            // 3. 用redis的getbit命令，判断对应位置的值
            Boolean isExist = jedis.getbit(bitmapKey, offset);

            if( !isExist ){
                // 如果不存在，对应位图位置置1
                jedis.setbit(bitmapKey, offset, true);

                // 更新redis中保存的count值
                Long uvCount = 0L;    // 初始count值
                String uvCountString = jedis.hget(countHashName, countKey);
                if( uvCountString != null && !"".equals(uvCountString) )
                    uvCount = Long.valueOf(uvCountString);
                jedis.hset(countHashName, countKey, String.valueOf(uvCount + 1));

                out.collect(new PageViewCount("uv", windowEnd, uvCount + 1));
            }
        }

        @Override
        public void close() throws Exception {
            jedis.close();
        }
    }
}
```

