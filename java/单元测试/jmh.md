# write before

正在学习中，在实际应用中不够多，后续实际应用上再补充一些见解，进度如下

- [x] 了解基础知识
- [ ] 基本指标
- [ ] 实操项目jmh模块搭建
- [ ] jmh、jmeter场景互补

# 简介



# 示例代码

maven依赖

```xml
        <properties>
            <jmh.version>1.35</jmh.version>
        </properties>       
		<dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-core</artifactId>
            <version>${jmh.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-generator-annprocess</artifactId>
            <version>${jmh.version}</version>
            <scope>test</scope>
        </dependency>
```

```java
@BenchmarkMode(Mode.AverageTime)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 5)
@Threads(4)
@Fork(1)
@State(value = Scope.Benchmark)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class StringConnect {

    @Param(value = {"10", "50", "100"})
    private int length;

    @Benchmark
    public void testStringAdd(Blackhole blackhole) {
        String a = "";
        for (int i = 0; i < length; i++) {
            a += i;
        }
        blackhole.consume(a);
    }

    @Benchmark
    public void testStringBuilderAdd(Blackhole blackhole) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < length; i++) {
            sb.append(i);
        }
        blackhole.consume(sb.toString());
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(StringConnect.class.getSimpleName())
                .result("result.json")
                .resultFormat(ResultFormatType.JSON).build();
        new Runner(opt).run();
    }
}
```

# `JMH`基础

## `@BenchmarkMode`

1. Throughput: 整体吞吐量，每秒执行多少次调用，单位为ops/time
2. AverageTime: 每次操作的平均时间，单位为time/op
3. SampleTime: 随机抽样
4. SingleShotTime: 只运行一次

## `@Warmup`

预热设置，因为JIT的存在，所以预热之后的出来的数据更接近真实情况

1. iterations:预热的次数
2. time:每次预热的时间
3. timeUnit: 时间单位
4. batchSize: 批处理大小每次操作调用几次方法

## `@Measurement`

实际调用，参数与预热一致

## `@Threads`

每个进程中的测试线程

## `@Fork`

进程数

## `@State`

指定一个对象的作用范围

1. Scope.Benchmark: 所有测试线程共享一个示例，测试有状态实例在多线程共享下的性能
2. Scope.Group: 同一个线程在同一个group里共享实例
3. Scope.Thread:默认的State，每个测试线程分配一个实例

## `@OutputTimeUnit`

统计结果的时间单位

# 参考

- [`openjdk-github`](https://github.com/openjdk/jmh)

- [简书](https://www.jianshu.com/p/0da2988b9846)

- [知乎](https://www.zhihu.com/question/276455629/answer/1259967560)