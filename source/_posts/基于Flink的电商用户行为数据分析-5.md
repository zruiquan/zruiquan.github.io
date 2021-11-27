---
title: 基于Flink的电商用户行为数据分析（5）
date: 2021-11-28 00:39:31
tags:
- Flink
categories:
- Flink
---

## 基于Flink的电商用户行为数据分析（5）

### 恶意登录检测

#### 模块创建和数据准备

继续在UserBehaviorAnalysis下新建一个 maven module作为子项目，命名为LoginFailDetect。在这个子模块中，我们将会用到flink的CEP库来实现事件流的模式匹配，所以需要在pom文件中引入CEP的相关依赖：

```xml
<dependency>
	<groupId>org.apache.flink</groupId>
	<artifactId>flink-cep-scala_${scala.binary.version}</artifactId>
	<version>${flink.version}</version>
</dependency>
```

#### 代码实现

对于网站而言，用户登录并不是频繁的业务操作。如果一个用户短时间内频繁登录失败，就有可能是出现了程序的恶意攻击，比如密码暴力破解。因此我们考虑，应该对用户的登录失败动作进行统计，具体来说，如果同一用户（可以是不同 IP） 在 2 秒之内连续两次登录失败，就认为存在恶意登录的风险，输出相关的信息进行报警提示。这是电商网站、也是几乎所有网站风控的基本一环。

#### 状态编程

由于同样引入了时间，我们可以想到，最简单的方法其实与之前的热门统计类似，只需要按照用户 ID 分流，然后遇到登录失败的事件时将其保存在 ListState 中，然后设置一个定时器，2 秒后触发。定时器触发时检查状态中的登录失败事件个数，如果大于等于 2，那么就输出报警信息。

在 src/main/java 下创建 LoginFail 类，定义 POJO 类 LoginEvent，这是输入的登录事件流。登录数据本应该从 UserBehavior 日志里提取，由于 UserBehavior.csv 中没有做相关埋点，我们从另一个文件 LoginLog.csv 中读取登录数据。

代码如下：

```java
package com.test.loginfail_detect;
import com.test.loginfail_detect.beans.LoginEvent;
import com.test.loginfail_detect.beans.LoginFailWarning;
import org.apache.flink.api.common.state.ListState;
import org.apache.flink.api.common.state.ListStateDescriptor;
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.shaded.guava18.com.google.common.collect.Lists;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.Collector;
import java.net.URL;
import java.util.ArrayList;
import java.util.Iterator;

public class LoginFail {
    public static void main(String[] args) throws Exception{
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        // 1. 从文件中读取数据
        URL resource = LoginFail.class.getResource("/LoginLog.csv");
        DataStream<LoginEvent> loginEventStream = env.readTextFile(resource.getPath())
                .map(line -> {
                    String[] fields = line.split(",");
                    return new LoginEvent(new Long(fields[0]), fields[1], fields[2], new Long(fields[3]));
                })
                .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<LoginEvent>											(Time.seconds(3)) {
                    @Override
                    public long extractTimestamp(LoginEvent element) {
                        return element.getTimestamp() * 1000L;
                    }
                });

        // 自定义处理函数检测连续登录失败事件
        SingleOutputStreamOperator<LoginFailWarning> warningStream = loginEventStream
                .keyBy(LoginEvent::getUserId)
                .process(new LoginFailDetectWarning(2));

        warningStream.print();

        env.execute("login fail detect job");
    }

    // 实现自定义KeyedProcessFunction
    public static class LoginFailDetectWarning0 extends KeyedProcessFunction<Long, LoginEvent, 											LoginFailWarning>{
        // 定义属性，最大连续登录失败次数
        private Integer maxFailTimes;

        public LoginFailDetectWarning0(Integer maxFailTimes) {
            this.maxFailTimes = maxFailTimes;
        }

        // 定义状态：保存2秒内所有的登录失败事件
        ListState<LoginEvent> loginFailEventListState;
        // 定义状态：保存注册的定时器时间戳
        ValueState<Long> timerTsState;

        @Override
        public void open(Configuration parameters) throws Exception {
            loginFailEventListState = getRuntimeContext().getListState(new ListStateDescriptor<LoginEvent>							("login-fail-list", LoginEvent.class));
            timerTsState = getRuntimeContext().getState(new ValueStateDescriptor<Long>("timer-ts", 											Long.class));
        }

        @Override
        public void processElement(LoginEvent value, Context ctx, Collector<LoginFailWarning> out) throws 					Exception {
            // 判断当前登录事件类型
            if( "fail".equals(value.getLoginState()) ){
                // 1. 如果是失败事件，添加到列表状态中
                loginFailEventListState.add(value);
                // 如果没有定时器，注册一个2秒之后的定时器
                if( timerTsState.value() == null ){
                    Long ts = (value.getTimestamp() + 2) * 1000L;
                    ctx.timerService().registerEventTimeTimer(ts);
                    timerTsState.update(ts);
                }
            } else {
                // 2. 如果是登录成功，删除定时器，清空状态，重新开始
                if( timerTsState.value() != null )
                    ctx.timerService().deleteEventTimeTimer(timerTsState.value());
                loginFailEventListState.clear();
                timerTsState.clear();
            }
        }

        @Override
        public void onTimer(long timestamp, OnTimerContext ctx, Collector<LoginFailWarning> out) throws 						Exception {
            // 定时器触发，说明2秒内没有登录成功来，判断ListState中失败的个数
            ArrayList<LoginEvent> loginFailEvents = Lists.newArrayList(loginFailEventListState.get());
            Integer failTimes = loginFailEvents.size();

            if( failTimes >= maxFailTimes ){
                // 如果超出设定的最大失败次数，输出报警
                out.collect( new LoginFailWarning(ctx.getCurrentKey(),
                        loginFailEvents.get(0).getTimestamp(),
                        loginFailEvents.get(failTimes - 1).getTimestamp(),
                        "login fail in 2s for " + failTimes + " times") );
            }

            // 清空状态
            loginFailEventListState.clear();
            timerTsState.clear();
        }
    }

    // 实现自定义KeyedProcessFunction
    public static class LoginFailDetectWarning extends KeyedProcessFunction<Long, LoginEvent, 									LoginFailWarning> {
        // 定义属性，最大连续登录失败次数
        private Integer maxFailTimes;

        public LoginFailDetectWarning(Integer maxFailTimes) {
            this.maxFailTimes = maxFailTimes;
        }

        // 定义状态：保存2秒内所有的登录失败事件
        ListState<LoginEvent> loginFailEventListState;

        @Override
        public void open(Configuration parameters) throws Exception {
            loginFailEventListState = getRuntimeContext().getListState(new ListStateDescriptor<LoginEvent>					("login-fail-list", LoginEvent.class));
        }

        // 以登录事件作为判断报警的触发条件，不再注册定时器
        @Override
        public void processElement(LoginEvent value, Context ctx, Collector<LoginFailWarning> out) throws 					Exception {
            // 判断当前事件登录状态
            if( "fail".equals(value.getLoginState()) ){
                // 1. 如果是登录失败，获取状态中之前的登录失败事件，继续判断是否已有失败事件
                Iterator<LoginEvent> iterator = loginFailEventListState.get().iterator();
                if( iterator.hasNext() ){
                    // 1.1 如果已经有登录失败事件，继续判断时间戳是否在2秒之内
                    // 获取已有的登录失败事件
                    LoginEvent firstFailEvent = iterator.next();
                    if( value.getTimestamp() - firstFailEvent.getTimestamp() <= 2 ){
                        // 1.1.1 如果在2秒之内，输出报警
                        out.collect( new LoginFailWarning(value.getUserId(), firstFailEvent.getTimestamp(), 												value.getTimestamp(), "login fail 2 times in 2s") );
                    }

                    // 不管报不报警，这次都已处理完毕，直接更新状态
                    loginFailEventListState.clear();
                    loginFailEventListState.add(value);
                } else {
                    // 1.2 如果没有登录失败，直接将当前事件存入ListState
                    loginFailEventListState.add(value);
                }
            } else {
                // 2. 如果是登录成功，直接清空状态
                loginFailEventListState.clear();
            }
        }
    }
}
```

#### CEP编程

上一节我们通过对状态编程的改进，去掉了定时器，在 process function 中做了更多的逻辑处理，实现了最初的需求。不过这种方法里有很多的条件判断，而我们目前仅仅实现的是检测“连续 2 次登录失败”，这是最简单的情形。如果需要检测更多次，内部逻辑显然会变得非常复杂。那有什么方式可以方便地实现呢？很幸运，flink 为我们提供了 CEP（Complex Event Processing，复杂事件处理）库，用于在流中筛选符合某种复杂模式的事件。接下来我们就基于 CEP 来完成这个模块的实现。

在 src/main/java 下继续创建 LoginFailWithCep 类，POJO 类 LoginEvent 由于在LoginFail.java 已经定义，我们在同一个模块中就不需要再定义了。

代码如下：

```java
package com.test.loginfail_detect;
import com.test.loginfail_detect.beans.LoginEvent;
import com.test.loginfail_detect.beans.LoginFailWarning;
import org.apache.flink.cep.CEP;
import org.apache.flink.cep.PatternSelectFunction;
import org.apache.flink.cep.PatternStream;
import org.apache.flink.cep.pattern.Pattern;
import org.apache.flink.cep.pattern.conditions.SimpleCondition;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import java.net.URL;
import java.util.List;
import java.util.Map;

public class LoginFailWithCep {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        // 1. 从文件中读取数据
        URL resource = LoginFailWithCep.class.getResource("/LoginLog.csv");
        DataStream<LoginEvent> loginEventStream = env.readTextFile(resource.getPath())
                .map(line -> {
                    String[] fields = line.split(",");
                    return new LoginEvent(new Long(fields[0]), fields[1], fields[2], new Long(fields[3]));
                })
                .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<LoginEvent>											(Time.seconds(3)) {
                    @Override
                    public long extractTimestamp(LoginEvent element) {
                        return element.getTimestamp() * 1000L;
                    }
                });

        // 1. 定义一个匹配模式
        // firstFail -> secondFail, within 2s
        Pattern<LoginEvent, LoginEvent> loginFailPattern = Pattern
                .<LoginEvent>begin("firstFail").where(new SimpleCondition<LoginEvent>() {
            @Override
            public boolean filter(LoginEvent value) throws Exception {
                return "fail".equals(value.getLoginState());
            }
        })
                .next("secondFail").where(new SimpleCondition<LoginEvent>() {
                @Override
                public boolean filter(LoginEvent value) throws Exception {
                  return "fail".equals(value.getLoginState());
                }
              })
                .next("thirdFail").where(new SimpleCondition<LoginEvent>() {
                @Override
                public boolean filter(LoginEvent value) throws Exception {
                  return "fail".equals(value.getLoginState());
                }
              })
                .within(Time.seconds(3));

        // 2. 将匹配模式应用到数据流上，得到一个pattern stream
        PatternStream<LoginEvent> patternStream = CEP.pattern(loginEventStream.keyBy(LoginEvent::getUserId), 				loginFailPattern);

        // 3. 检出符合匹配条件的复杂事件，进行转换处理，得到报警信息
        SingleOutputStreamOperator<LoginFailWarning> warningStream = patternStream.select(new 											LoginFailMatchDetectWarning());

        warningStream.print();

        env.execute("login fail detect with cep job");
    }

    // 实现自定义的PatternSelectFunction
    public static class LoginFailMatchDetectWarning implements PatternSelectFunction<LoginEvent, 								LoginFailWarning>{
        @Override
        public LoginFailWarning select(Map<String, List<LoginEvent>> pattern) throws Exception {
            LoginEvent firstFailEvent = pattern.get("firstFail").iterator().next();
            LoginEvent lastFailEvent = pattern.get("thirdFail").get(0);
            return new LoginFailWarning(firstFailEvent.getUserId(), firstFailEvent.getTimestamp(), 														 lastFailEvent.getTimestamp(), "login fail 2 times");
        }
    }
}
```

