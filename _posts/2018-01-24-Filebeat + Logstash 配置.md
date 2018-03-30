---
layout:     post
title:      "Filebeat + Logstash 配置"
date:       2018-01-24
author:     "pggsnap"
tags:
    - Filebeat
    - Logstash
---

# Filebeat + Logstash 配置

### Filebeat

- 配置文件, 将日志发送到 Logstash。

    ```
    filebeat.prospectors:
    - type: log
      paths:
        - /home/logs/proxyapi/proxyapi.log
      tags: ["proxyapi"]
    #  fields:    # 如果filebeat直接发送到elasticsearch，可以为不同的日志设置不同的自定义字段；但是，如果发送到logstash，则绝对不应该在输入中设置字段。因为在logstash中，输入会生成事件，而事件都有属性(字段)，所以在输入块中没有字段，因为此时还不存在。
    #    service: proxyapi

    output.logstash:
      hosts: ["192.168.99.13:5044"]
    ```

- 启动

    ```
    systemctl start filebeat
    ```

    `proxyapi.log` 日志内容格式如下:

    ```
    2018-01-15 14:52:38, 610:INFO https-jsse-nio-443-exec-9 (LogFilter.java:37) - ip: 10.192.1.189, url: /proxyapiservlet, funcno: 333104, cost: 31ms
    2018-01-15 14:52:39, 355:INFO https-jsse-nio-443-exec-8 (ProxyApiService.java:70) - 请求账号: weixin, 功能号: 337451, 入参: {op_entrust_way=7, op_station=op_station, password=*, op_branch_no=52, branch_no=52, funcno=337451, fund_account=52000023, client_id=52000023}
    2018-01-15 14:52:39, 363:INFO https-jsse-nio-443-exec-8 (HsService.java:93) - 返回: 337451, 结果集: {"errorno":0,"errorinfo":"","datasets":{"dataset0":[]},"datasetname":"dataset0"}
    2018-01-15 14:52:39, 364:INFO https-jsse-nio-443-exec-8 (LogFilter.java:37) - ip: 10.192.1.189, url: /proxyapiservlet, funcno: 337451, cost: 9ms
    2018-01-15 14:52:47, 120:INFO https-jsse-nio-443-exec-4 (ProxyApiService.java:70) - 请求账号: weixin, 功能号: 8907408, 入参: {init_date=20180115, branch_no=, funcno=8907408, begin_id=59, js=1000, client_id=}
    2018-01-15 14:52:47, 123:ERROR https-jsse-nio-443-exec-4 (HsService.java:67) - dataset0 result: return_code=0, error_no=116
    2018-01-15 14:52:47, 123:INFO https-jsse-nio-443-exec-4 (HsService.java:93) - 返回: 8907408, 结果集: {"errorno":116,"errorinfo":"路由不能转发[转发错误],发送者信息 \u003d [GLSC_FWZX_UF20_TEST#0] 插件ID \u003d [router] 节点编号 \u003d [0] 子系统编号 \u003d [0]","datasets":{},"datasetname":"dataset0"}
    2018-01-15 14:52:47, 123:INFO https-jsse-nio-443-exec-4 (LogFilter.java:37) - ip: 10.197.0.85, url: /proxyapiservlet, funcno: 8907408, cost: 4ms
    ```

- 其他

    Filebeat 会将已经成功发送的文件位置记录下来，保存在 `/var/lib/filebeat/registry` 文件中。如果需要重新发送，只需要删除该文件并重启服务即可。

### Logstash

- 配置文件, 过滤日志后发送到 Elasticsearch。

    ```
    input {
      beats {
        port => "5044"
      }
    }
    filter {
      if "proxyapi" in [tags] {
        mutate {
          #add_field => { "service" => "proxyapi" }
          #通过filebeat作为input，默认会打beats_input_codec_plain_applied这个tag，可以通过replace来重置tags的值
          replace => { "tags" => "proxyapi" }
        }
        grok {
          match => [
            "message", "(?m)(?<date>%{YEAR}-%{MONTHNUM}-%{MONTHDAY} %{TIME}).+?请求账号:\s(?<appId>%{USER}).+?功能号:\s(?<funcno>\d+)",
            "message", "(?m)(?<date>%{YEAR}-%{MONTHNUM}-%{MONTHDAY} %{TIME}).+?返回:\s%{NUMBER:funcno}",
            "message", "(?m)%{WORD}"
          ]
        }
      }
    }
    output {
      elasticsearch {
        hosts => ["192.168.99.13:9200"]
      }
    }
    ```

- 启动

    ```
    /usr/share/logstash/bin/logstash -f first-pipeline.conf --config.reload.automatic
    ```

    或者

    ```
    systemctl start logstash
    ```

    启动之前可以先校验配置文件是否合法，

    ```
    /usr/share/logstash/bin/logstash -f first-pipeline.conf --config.test_and_exit
    ```

### 过滤插件

- grok

    内置了很多规则，可以在以下目录中找到: `/usr/share/logstash/vendor/bundle/jruby/2.3.0/gems/logstash-patterns-core-4.1.2/patterns`
