---
layout: post
title: Memcached学习笔记
date: 2017-04-26 
tag: 算法
---

what why how

### 什么是Memcached
Memcached是一个免费开源、高性能、分布式内存键值对缓存系统。

#### 结构

{% mermaid %}
graph LR
         subgraph one
         a1
         a2
         a3
         end
         subgraph three
         subgraph two
         b1==>b2
         end
         c1==>b1
         end
         b2==>a2
{% endmermaid %}

#### 设计理念
- 键值对存储
存储项由key, expiration time, optional flags和raw data组成。存储数据的服务器并不关心raw data的内容和格式

- 



