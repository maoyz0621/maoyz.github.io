# guava

## Stopwatch

`com.google.common.base.Stopwatch`

<img src="\image\Stopwatch.png" alt="image-20211226204332322" style="zoom: 67%;" />

- start  启动
- stop  停止
- reset  重置
- elapsed  按照特定格式输出经历的时间

```java
Stopwatch stopWatch = Stopwatch.createStarted();

TimeUnit.MILLISECONDS.sleep(500);

// elapsed 用特定的格式返回这个stopwatch经过的时间
log.info("first = {}", stopWatch.elapsed(TimeUnit.MILLISECONDS));

// 停止计时
log.info("this work tasks {}", stopWatch.stop());

// 重新开始
stopWatch.start();
TimeUnit.MILLISECONDS.sleep(600);
log.info("second = {}", stopWatch.elapsed(TimeUnit.MILLISECONDS));

// 重置
stopWatch.reset().start();
TimeUnit.MILLISECONDS.sleep(700);
log.info("third = {}", stopWatch.elapsed(TimeUnit.MILLISECONDS));

log.info("end = {}", stopWatch.stop());
```

## EventBus





## RateLimiter

## BloomFilter

布隆过滤器