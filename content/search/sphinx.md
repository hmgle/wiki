---
title: "Sphinx (search engine) 使用笔记"
layout: page
date: 2016-11-19 19:00
---

[TOC]

# Sphinx 简介

[Sphinx](http://sphinxsearch.com/) 是一个轻量级的全文检索引擎。

## 部分同类开源产品

- [Elasticsearch](https://www.elastic.co/products/elasticsearch) 基于 Lucene 的开源搜索引擎，支持 RESTful API。
- [Redis-Search](https://github.com/huacnlee/redis-search) 基于 Redis 的高效搜索组件。
- [Searchdaimon ES](http://www.searchdaimon.com/) 一个针对公司数据和网站的搜索引擎。

另外，Redis 的不稳定版已经支持[加载外部模块](http://antirez.com/news/106)，[RediSearch](https://github.com/RedisLabsModules/RediSearch) 就是一个全文检索的模块。以 Redis 社区的活跃程度，此类基于 Redis 模块方式的检索引擎应该会越来越成熟。

查看更多[同类产品](https://en.wikipedia.org/wiki/Full-text_search)。

## 基本原理

Sphinx 包含几个可执行程序：wordbreaker，indexer，searchd 等。

- wordbreaker 分词程序。

- indexer 从数据源读取输入，分词，建立索引数据。

- searchd 为客户端提供检索服务，解析请求，读取索引数据，返回结果给客户端。

## 部分功能

- 支持的数据源：
  * MySQL, PostgreSQL, MS QL...
  * xmlpipe, xmlpipe2: 相比 xmlpipe，xmlpipe2 支持直接在 xml 内指定数据的 field 和 attr 等配置功能。
  * tsvpipe, csvpipe...

- 支持实时增加索引(RT 实时索引)，目前 RT 实时索引仅支持 SphinxQL 协议操作，不支持 API 方式。

## 中文支持

- [Coreseek](http://www.coreseek.cn/)
- [sphinx-for-chinese](http://sphinxsearchcn.github.io/) 注意：这里的 sphinx-for-chinese 是基于 sphinx 2.2.1 版的，如果需要 Golang 客户端操作 SphinxQL 语句，根据 Go 的 [mysql](https://github.com/go-sql-driver/mysql) 说明，至少要求 Sphinx(2.2.3) 以上版本。我尝试了下，的确不能操作成功，改用这个[较新的版本](https://github.com/eric1688/sphinx)后才解决 Go 客户端操作 SphinxQL 的问题。

## API 协议

Sphinx 源码已经提供了 Python，Ruby，PHP，Java，C 这几种语言的 API 包了，如需实现其他语言的 API 包，就要分析 Sphinx API 协议了。具体可以通过 Python 的客户端 `sphinx-x.x.x/api/sphinxapi.py` 分析。下面分析一个 wireshark 抓取的 Sphinx 请求包:

### 请求

发出一个检索 "hello" 的请求，返回数量限制(limit)为 20。

DATA 内容:

```
00:00:00:00:00:00:00:14:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:
00:00:00:06:68:65:6c:6c:6f:20:00:00:00:02:00:00:00:64:00:00:00:01:00:00:
00:01:2a:00:00:00:01:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:
00:00:00:00:00:00:00:00:00:00:00:00:00:03:e8:00:00:00:0b:40:67:72:6f:75:
70:20:64:65:73:63:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:
00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:
00:01:2a
```

对应的结构：

```
offset:	        00:00:00:00:
limit:	        00:00:00:14:
mode:	        00:00:00:00:
ranker:	        00:00:00:00:
sortMode:	00:00:00:00:
sortByLen:	00:00:00:00:
queryLen:	00:00:00:06:
query:	        68:65:6c:6c:6f:20: (hello )
weightsLen:	00:00:00:02:
weights:	00:00:00:64:
weights:	00:00:00:01:
indexLen:	00:00:00:01:
index:	        2a: (*)
id64 range marker:	00:00:00:01:
minID:	        00:00:00:00:00:00:00:00:
maxID:	        00:00:00:00:00:00:00:00:
filters len:	00:00:00:00:
groupFunc:	00:00:00:00:
groupBy len:	00:00:00:00:
maxMatches:	00:00:03:e8: (1000)
groupSort len:	00:00:00:0b:
groupSort:	40:67:72:6f:75:70:20:64:65:73:63: ("@group desc")
cutoff:	        00:00:00:00:
retryCount:	00:00:00:00:
retryDelay:	00:00:00:00:
groupDistinct len:	00:00:00:00:
anchor point flag:	00:00:00:00:
indexWeights len:	00:00:00:00:
maxQueryTime:	00:00:00:00:
fieldWeights:	00:00:00:00:
comment len:	00:00:00:00:
overrides len:	00:00:00:00:
select len:	00:00:00:01:
select:		2a: ("*")
```

