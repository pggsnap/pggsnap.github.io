---
layout:     post
title:      "Filebeat + ELK 6.1.1 最简安装配置"
date:       2018-01-22
author:     "pggsnap"
tags:
    - Filebeat
    - ELK
---

# Filebeat + ELK 6.1.1 最简安装配置

环境: CentOS 7 + ELK 6.1.1

### elasticsearch安装配置
[Elasticsearch Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)

1. rpm 安装
```
rpm -ivh elasticsearch-6.1.1.rpm
```

2. 配置
`/etc/elasticsearch/elasticsearch.yml` 修改部分参数:

```
network.host: 192.168.99.13
http.port: 9200
```

3. 启动
```
systemctl start elasticsearch
```

### Kibana 安装配置
[Kibana User Guide](https://www.elastic.co/guide/en/kibana/current/index.html)

1. rpm 安装
```
rpm -ivh kibana-6.1.1-x86_64.rpm
```

2. 配置
`/etc/kibana/kibana.yml` 修改部分参数:

```
server.port: 5601
server.host: "192.168.99.13"
elasticsearch.url: "http://192.168.99.13:9200"
```

3. 启动

```
systemctl start kibana
```

### Filebeat 安装配置
[Getting Started With Filebeat](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-getting-started.html)

1. rpm 安装
```
rpm -ivh filebeat-6.1.1-x86_64.rpm
```

2. 配置
`/etc/filebeat/filebeat.yml` 参考配置, 将日志 `/var/log/*.log` 以及 `/var/log/nginx/*.log` 直接发送到 Elasticsearch。

```
filebeat.prospectors:
- type: log
  enabled: true
  paths:
    - /var/log/*.log
    - /var/log/nginx/*.log

output.elasticsearch:
  hosts: ["192.168.99.13:9200"]

setup.kibana:
  host: "192.168.99.13:5601"
```

3. 安装 Kibana dashboards

```
filebeat setup --dashboards
```

4. 启动
```
systemctl start filebeat
```

### 测试
浏览器通过 http://192.168.99.13:5601 查看日志。
