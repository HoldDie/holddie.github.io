---
title: ElasticSearch-ELK（十二）
author: HoldDie
tags: [ElasticSearch,搜索,聚合分析]
top: false
date: 2018-10-05 19:43:41
categories: ElasticSearch
---



利剑虽强却斩不断流水，微风虽弱却能平息海浪。 ——子房

## ELK 组合

ELK 主要用途是：大型分布式系统的日志集中分析

1、在生产环境中出现问题，你应该如何定位问题？

2、大型的分布式系统中如出出现问题，如何定位问题？

一个完整的集中式日志系统，需要包含特点：

- 收集–能够收集多种来源的日志数据
- 传输—能够稳定的吧日志数据传输到中央系统
- 转换—能够对手机的日志数据进行转换处理
- 存储–如何存储日志数据
- 分析–可以支持UI分析
- 告警–能够提供错误报告，监控机制

FileBeat 数据收割

直接指定日志产生的目录，会自动匹配文件，记性收割机的与之匹配，然后进行收集。

Linux下安装完成之后，数据目录是什么

启动时读取的配置文件是 filebeat.yml 中配置