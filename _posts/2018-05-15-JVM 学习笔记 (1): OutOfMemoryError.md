---
layout:     post
title:      "JVM 学习笔记 (1): OutOfMemoryError"
date:       2018-05-15
author:     "pggsnap"
tags:
    - JVM
    - VisualVM
    - OutOfMemoryError
---

## OutOfMemoryError

```
private static int INSERT_BATCH = 1000000;

@Transactional
public long insertUUIDBatch(int num) {
    long start = System.currentTimeMillis();
    for (int k = 0; k < Math.ceil((double)num/INSERT_BATCH); k++) {
        // 生成 INSERT_BATCH 个 uuid，并保存到 arrayList
        List<String> array = new ArrayList<>();
        for (int i = 0; i < Math.min(num - k * INSERT_BATCH, INSERT_BATCH); i++) {
            array.add(UUID.randomUUID().toString().replace("-", ""));
        }
        // 通过 insert into table values('$1'),('$2')... 批量插入记录
        uuidDao.insertBatch(array);
    }
    long end = System.currentTimeMillis();
    return end - start;
}
```

MySQL 5.7 默认的 max_allowed_packet=4M，即每次允许的包大小为 4M，如果发送的请求或者返回的结果数据量比较大，会报以下错误：
`Error updating database.  Cause: com.mysql.jdbc.PacketTooBigException: Packet for query is too large (37000033 > 4194304). You can change this value on the server by setting the max_allowed_packet' variable.`

解决方法是更改 max_allowed_packet 的值，并且重启数据库；或者控制传输的包不大于 4M。

通过 `java -jar jvm-0.0.1-SNAPSHOT.jar -Xms256m -Xmx256m` 运行程序，并发 2 次执行 insertUUIDBatch(1000000) 报错 `java.lang.OutOfMemoryError: Java heap space at  java.util.Arrays.copyOf(Arrays.java:3332)~[na:1.8.0_162]`。

ArrayList 有个初始容量为 10，看错误应该是当数组容量满并且继续新增元素时，没能申请到足够的堆空间。

## 通过 VisualVM 分析堆空间

执行一次 insertUUIDBatch(1000000)，通过 jmap 命令或者 VisualVM 执行 heap dump，解析出的结果如下：
![](/blog_img/jvm-oom-1.jpg)

char[] 占用了 171M 内存，占比 50.9%，说明整个堆内存差不多使用了 340M。

占用堆内存最多的几个分别是：char[] 171M 左右，byte[] 100M 左右，String 29M。

点开 char[] 查看详情：
![](/blog_img/jvm-oom-2.jpg)

char[]#866027 是批量插入记录的 insert 语句，`insert into table values('$uuid1'),('$uuid2')`，一共 100 万个 uuid；
左右括号 + 单引号 + 32 位 uuid + 逗号，占用内存差不多 100 万 * (2 + 2 + 32 + 1) * 2B /每个字符 = 74MB。
生成 100 万个 uuid，占用内存差不多 100 万 * (24(数组对象头) + 32 * 2) = 88MB。
加上其他 char[] 9M 左右，一共是 171MB。

100 万个 uuid，字符串本身所占内存：100 万* (16(对象头) + 8(指向 char[] 的引用) + 4(String 对象内部定义了 int hash)) = 28MB。
加上其他 String，一共占用 29M 符合预期。

点开 byte[] 查看详情：
![](/blog_img/jvm-oom-3.jpg)

发现主要占用内存的是 byte[]#2822, 2821, 2825，其中 2822，2825 是 com.mysql.jdbc.Buffer，而 2821 则被 PreparedStatement 对象引用。查看 PreparedStatement 类的结构:
```
public class PreparedStatement extends StatementImpl implements java.sql.PreparedStatement {
    protected int parameterCount;
    protected MysqlParameterMetadata parameterMetaData;
    private InputStream[] parameterStreams = null;
    private byte[][] parameterValues = (byte[][])null;
    protected int[] parameterTypes = null;
}
```
其中的 parameterValues 字段用来保存需要执行的语句。之前 char[]#866027 也保存过该语句，占用 74MB 内存；这里采用 byte[][] 类型保存，通过 utf8 编码，数字、字母和常用符号都只需要 1 个字节即可表示，所以只需要占用 37MB 即可。

关于对象占用的内存问题，参考文章 [一个 Java 对象到底占用多大内存？](http://www.cnblogs.com/zhanjindong/p/3757767.html)。

单次执行 insertUUIDBatch(1000000) 方法，需要的 char[] 内存就需要 162MB；设置 -Xmx256m 的情况下，并发执行该方法出现 OOM 错误就可以预测了。


## 改进办法

既然主要原因在于 arrayList 以及基于 arrayList 生成的 sql 语句占用内存太大，那么首先应该减小 arrayList 的值，比如设置为最大允许 1 万条记录；另外也可以增加 -Xmx 的值；arrayList 初始化时指定容量，避免因容量不够时的数组拷贝操作等。

代码改进后如下：
```
private static int INSERT_BATCH = 10000;

@Transactional
public long insertUUIDBatch(int num) {
    long start = System.currentTimeMillis();
    for (int k = 0; k < Math.ceil((double)num/INSERT_BATCH); k++) {
        // 生成 INSERT_BATCH 个 uuid，并保存到 arrayList
        int arrayInitialCapacity = Math.min(num - k * INSERT_BATCH, INSERT_BATCH);
        List<String> array = new ArrayList<>(arrayInitialCapacity);
        for (int i = 0; i < arrayInitialCapacity; i++) {
            array.add(UUID.randomUUID().toString().replace("-", ""));
        }
        // 通过 insert into table values('$1'),('$2')... 批量插入记录
        uuidDao.insertBatch(array);
    }
    long end = System.currentTimeMillis();
    return end - start;
}
```

通过 `java -jar jvm-0.0.1-SNAPSHOT.jar -Xms256m -Xmx256m` 运行程序，使用 jmeter 测试：1s 内启动 10 个线程，循环 2 次，执行 insertUUIDBatch(1000000) 结果如下：
![](/blog_img/jvm-oom-4.jpg)

## 为什么 VisualVM 显示的堆内存超出了设置的 Xmx 值？
