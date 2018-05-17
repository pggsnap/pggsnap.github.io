# 一、测试计划

测试计划是对已配置的请求集执行的具体说明，最重要组件包括：

## 1. 线程组（ThreadGroup）
![](/blog_img/jmeter-1.jpg)

1 秒内创建 10 个线程，并且每个线程执行 3 次请求；如果执行失败，则终止该线程。如果所有线程都正常，则一共会执行 30 次请求。

## 2. 创建测试请求（Sampler 采样器）

包括 HTTP 请求、TCP 请求、JDBC 请求、AJP、JMS、JSR223、SMTP 等各种请求类型。
![](/blog_img/jmeter-2.jpg)

## 3. 监听器（Listeners）

监听器提供不同的方式查看由采样器请求产生的结果，以报表、树形结构、图形结构或日志文件等形式分析结果。
![](/blog_img/jmeter-3.jpg)

图中共生成了树形、图形以及表格 3 种形式来查看结果。以 Aggregate Graph 为例：

Samples：30，一共 30 个请求；

Average：31991，请求平均耗时 32s 左右；

Median：31744，有一半请求的耗时低于 31744ms；

99%Line：34853，有 99% 请求的耗时低于 34853ms；

Error%：0，所有请求都正常响应；

Throughput：18.5/min，吞吐量为每分钟可以处理 18.5 个请求。

# 参考

[使用 JMeter 进行负载测试——终极指南](http://www.importnew.com/13876.html)

[JMeter TCP性能测试](http://www.cnblogs.com/youxin/p/8684891.html)
