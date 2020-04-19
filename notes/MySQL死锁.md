# Spring Mysql 死锁

```text
    并发情况下，定时任务出现 update 死锁问题
    分析后 update 未走索引，引发了全表锁
```
