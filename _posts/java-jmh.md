---
title: 'Java：JMH与基准测试'
date: 2021-03-02T00:00:00+08:00
lastmod: 2021-03-02T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["java","jmh","基准测试"]
categories: ["技术"]
lightgallery: true
---

# 介绍
JMH是一套Java基准测试工具，用于对Java执行进行基准测试以及生成测试报告。平时应用于Java一些基础Api或者一些工具类这种离开网络因素的纯系统测试。

# 使用方式
maven引入：
```xml
<dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-core</artifactId>
            <version>1.21</version>
</dependency>
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <version>1.21</version>
    <scope>provided</scope>
</dependency>
```
代码示例:
```java
@BenchmarkMode(Mode.Throughput)
@Warmup(iterations = 3)
@Measurement(iterations = 20, time = 5, timeUnit = TimeUnit.SECONDS)
@Threads(1)
@Fork(2)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
public class BigLoopBenchmark {

    @Benchmark
    public void testBigLoopOutSide() {
        long k = 0;
        for (int j = 0; j < 1000000000; j++) {
            for (int i = 0; i < 100; i++) {
                k++;
            }
        }
        use(k);
    }

    @Benchmark
    public void testBigLoopInSide() {
        long k = 0;
        for (int i = 0; i < 100; i++) {
            for (int j = 0; j < 1000000000; j++) {
                k++;
            }
        }
        use(k);
    }
    void use(long k){

    }
    public static void main(String[] args) throws RunnerException {
        Options options = new OptionsBuilder()
                .include(BigLoopBenchmark.class.getSimpleName())
                .output("D:/Benchmark.log")
                .build();
        new Runner(options).run();
    }
}
```

## Api说明：
### @BenchmarkMode(Mode.Throughput)基准测试模式
测试的模式，其中包括：
#### Throughput吞吐量
即每秒该方法可以执行多少次,结果为ops/time
#### AverageTime平均时间
即平均每次执行需要花费多少秒，结果为time/ops
#### SampleTime随机取样
结果是取样的执行的花费时间，比如99%请求,结果实例：
```java
  Histogram, ms/op:
    [0.000, 0.005) = 361794 
    [0.005, 0.010) = 12 
    [0.010, 0.015) = 12 
    [0.015, 0.020) = 2 
    [0.020, 0.025) = 3 
    [0.025, 0.030) = 1 
    [0.030, 0.035) = 1 
    [0.035, 0.040) = 1 
    [0.040, 0.045) = 0 
    [0.045, 0.050) = 0 
    [0.050, 0.055) = 0 
    [0.055, 0.060) = 0 
    [0.060, 0.065) = 0 
    [0.065, 0.070) = 0 
    [0.070, 0.075) = 0 
    [0.075, 0.080) = 0 
    [0.080, 0.085) = 1 
```
#### SingleShotTime单次执行
结果是单次执行的统计，基本上用来测试冷启动

#### All所有测试

### @Warmup(iterations = 3)预热次数
配置测试执行前该段代码预先执行的次数，即预热次数，因为Java有JIT，所以预热执行可以先触发JIT，大大提升执行速度，并且可以模拟实际执行速度。统计更准确
### @Measurement(iterations = 20, time = 5, timeUnit = TimeUnit.SECONDS)执行配置
主要就是执行多少次操作，执行几秒，时间单位
### @Threads(1)
基准测试使用的线程数
### @Fork(2)
基准测试fork出测试的进程数
### @OutputTimeUnit(TimeUnit.MILLISECONDS)
输出报告时间的单位
