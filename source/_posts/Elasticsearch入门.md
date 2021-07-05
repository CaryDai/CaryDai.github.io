---
title: Elasticsearch入门
date: 2021-07-02 20:26:02
tags: Elasticsearch
categories: 搜索引擎
description: 这篇开始学习Elasticsearch
---

Elasticsearch 是一个开源的搜索引擎，建立在一个全文搜索引擎库 [Apache Lucene™](https://lucene.apache.org/core/) 基础之上。Elasticsearch 将所有的功能打包成一个单独的服务，这样你可以通过程序与它提供的简单的 RESTful API 进行通信，可以使用自己喜欢的编程语言充当 web 客户端，甚至可以使用命令行（去充当这个客户端）。

Elasticsearch 是 *面向文档* 的，意味着它存储整个对象或*文档*。Elasticsearch 不仅存储文档，而且*索引* 每个文档的内容，使之可以被检索。在 Elasticsearch 中，我们对文档进行索引、检索、排序和过滤——而不是对行列数据。这是一种完全不同的思考数据的方式，也是 Elasticsearch 能支持复杂全文检索的原因。Elasticsearch 使用 JavaScript Object Notation（或者 [*JSON*](http://en.wikipedia.org/wiki/Json)）作为文档的序列化格式。

下面这个 JSON 文档代表了一个 user 对象：

```js
{
    "email":      "john@smith.com",
    "first_name": "John",
    "last_name":  "Smith",
    "info": {
        "bio":         "Eco-warrior and defender of the weak",
        "age":         25,
        "interests": [ "dolphins", "whales" ]
    },
    "join_date": "2014/05/01"
}
```

虽然原始的 `user` 对象很复杂，但这个对象的结构和含义在 JSON 版本中都得到了体现和保留。在 Elasticsearch 中将对象转化为 JSON 后构建索引要比在一个扁平的表结构中要简单的多。

## 安装并运行Elasticsearch

1. 使用brew安装

2. 启动es：

```bash
cd /usr/local/Cellar/elasticsearch/7.10.2
./bin/elasticsearch
```

3. 测试是否启动成功，可以另起一个终端，输入：

```bash
curl 'http://localhost:9200/?pretty'
daiqianjie@daiqianjiedeMacBook-Pro ~ % curl 'http://localhost:9200/?pretty'
{
  "name" : "daiqianjiedeMacBook-Pro.local",
  "cluster_name" : "elasticsearch_brew",
  "cluster_uuid" : "BoA4Hjt1T8eL_LeSMuGHCA",
  "version" : {
    "number" : "7.10.2-SNAPSHOT",
    "build_flavor" : "oss",
    "build_type" : "tar",
    "build_hash" : "unknown",
    "build_date" : "2021-01-16T01:41:27.115673Z",
    "build_snapshot" : true,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

## 与Elasticsearch交互

Java可以使用内置的节点客户端（Node client）或传输客户端（Transport client）与es进行交互，默认端口为*9300*。

所有其他语言可以使用 RESTful API 通过端口 *9200* 和 Elasticsearch 进行通信。一个 Elasticsearch 请求和任何 HTTP 请求一样由若干相同的部件组成：

```bash
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```

被<>标记的部分：

| VERB         | 适当的 HTTP *方法* 或 *谓词* : GET、 POST、 PUT、 HEAD 或者 DELETE。 |
| ------------ | ------------------------------------------------------------ |
| PROTOCOL     | http 或者 https（如果你在 Elasticsearch 前面有一个 https 代理） |
| HOST         | Elasticsearch 集群中任意节点的主机名，或者用 localhost 代表本地机器上的节点。 |
| PORT         | 运行 Elasticsearch HTTP 服务的端口号，默认是 9200 。         |
| PATH         | API 的终端路径（例如 _count 将返回集群中文档数量）。Path 可能包含多个组件，例如：_cluster/stats 和 _nodes/stats/jvm 。 |
| QUERY_STRING | 任意可选的查询字符串参数 (例如 ?pretty 将格式化地输出 JSON 返回值，使其更容易阅读) |
| BODY         | 一个 JSON 格式的请求体 (如果请求需要的话)                    |

例如，计算集群中文档的数量，我们可以用这个：

```bash
curl -XGET 'http://localhost:9200/_count?pretty' -H 'Content-Type: application/json' -d '
{
    "query": {
        "match_all": {}
    }
}
'
{
  "count" : 1000,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  }
}
```

> 这里加`-H 'Content-Type: application/json'`的原因是：Starting from Elasticsearch 6.0, all REST requests that include a body must also provide the correct content-type for that body.

## 基本概念

### 集群（Cluster）

集群是一个或多个节点(服务器)的集合。集群中的节点一起存储数据，对外提供搜索功能。集群由一个唯一的名称标识，该名称默认是“elasticsearch”。集群名称很重要，节点都是通过集群名称加入集群。

集群不要重名，取名一般要有明确意义，否则会引起混乱。例如，开发、测试和生产集群的名称可以使用logging-dev、logging-test和logging-prod。

集群节点数不受限制，可以只有一个节点。

### 节点（Node）

节点是一个服务器，属于某个集群。节点存储数据，参与集群的索引和搜索功能。与集群一样，节点也是通过名称来标识的。默认情况下，启动时会分配给节点一个UUID（全局惟一标识符）作为名称。如有需要，可以给节点取名，通常取名时应考虑能方便识别和管理。

默认情况下，节点加入名为elasticsearch的集群，通过设置节点的集群名，可加入指定集群。

### 索引（Index）

索引是具有某种特征的文档集合，相当于一本书的目录。例如，可以为客户数据建立索引，为订单数据建立另一个索引。索引由名称标识(必须全部为小写)，可以使用该名称，对索引中的文档进行建立索引、搜索、更新和删除等操作。

一个集群中，索引数量不受限制。

### 文档（Document）

es中的文档可以理解为用json表示的对象

### 分片与副本（Shards&Replicas）

分布式集群的概念。

索引可能存储大量数据，数据量可能超过单个节点的硬件限制。例如，一个索引包含10亿个文档，将占用1TB的磁盘空间，单个节点的磁盘放不下。

Elasticsearch提供了索引分片功能。创建索引时，可以定义所需的分片数量。每个分片本身都是一个功能齐全，独立的“索引”，可以托管在集群中的任何节点上。

## 操作

下面借助一个例子进行相关概念的学习。

### 索引员工文档

第一个业务需求是存储员工数据。这将会以*员工文档* 的形式存储：一个文档代表一个员工。存储数据到 Elasticsearch 的行为叫做*索引* ，但在索引一个文档之前，需要确定将文档存储在哪里。

<u>一个 Elasticsearch 集群可以包含多个*索引* ，相应的每个索引可以包含多个*类型*。 这些不同的类型存储着多个*文档* ，每个文档又有多个*属性*。</u>

> **Index Versus Index Versus Index**
>
> 你也许已经注意到 *索引* 这个词在 Elasticsearch 语境中有多种含义， 这里有必要做一些说明：
>
> 索引（名词）：
>
> 如前所述，一个 *索引* 类似于传统关系数据库中的一个 *数据库* ，是一个存储关系型文档的地方。 *索引* (*index*) 的复数词为 *indices* 或 *indexes* 。
>
> 索引（动词）：
>
> *索引一个文档* 就是存储一个文档到一个 *索引* （名词）中以便被检索和查询。这非常类似于 SQL 语句中的 `INSERT` 关键词，除了文档已存在时，新文档会替换旧文档情况之外。
>
> 倒排索引：
>
> 关系型数据库通过增加一个 *索引* 比如一个 B树（B-tree）索引 到指定的列上，以便提升数据检索速度。Elasticsearch 和 Lucene 使用了一个叫做 *倒排索引* 的结构来达到相同的目的。
>
> \+ 默认的，一个文档中的每一个属性都是 *被索引* 的（有一个倒排索引）和可搜索的。一个没有倒排索引的属性是不能被搜索到的。我们将在 [倒排索引](https://www.elastic.co/guide/cn/elasticsearch/guide/current/inverted-index.html) 讨论倒排索引的更多细节。

对于员工目录，我们将做如下操作：

- 每个员工索引一个文档，文档包含该员工的所有信息。
- 每个文档都将是 `employee` *类型* 。
- 该类型位于 *索引* `megacorp` 内。
- 该索引保存在我们的 Elasticsearch 集群中。

我们可以通过一条命令完成所有这些动作：

```bash
PUT /megacorp/employee/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}

# curl形式
curl -X PUT "localhost:9200/megacorp/employee/1?pretty" -H 'Content-Type: application/json' -d'
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
'
```

注意，路径 `/megacorp/employee/1` 包含了三部分的信息：

- **megacorp**：索引名称
- **employee**：类型名称
- **1**：特定雇员的ID

请求体 —— JSON 文档 —— 包含了这位员工的所有详细信息。继续增加更多员工：

```bash
curl -X PUT "localhost:9200/megacorp/employee/2?pretty" -H 'Content-Type: application/json' -d'
{
    "first_name" :  "Jane",
    "last_name" :   "Smith",
    "age" :         32,
    "about" :       "I like to collect rock albums",
    "interests":  [ "music" ]
}
'
curl -X PUT "localhost:9200/megacorp/employee/3?pretty" -H 'Content-Type: application/json' -d'
{
    "first_name" :  "Douglas",
    "last_name" :   "Fir",
    "age" :         35,
    "about":        "I like to build cabinets",
    "interests":  [ "forestry" ]
}
'
```

### 检索文档

目前我们已经在 Elasticsearch 中存储了一些数据， 接下来就能专注于实现应用的业务需求了。第一个需求是可以检索到单个雇员的数据。

这在 Elasticsearch 中很简单。简单地执行 一个 HTTP `GET` 请求并指定文档的地址——索引库、类型和ID。 使用这三个信息可以返回原始的 JSON 文档：

```bash
GET /megacorp/employee/1

curl -X GET "localhost:9200/megacorp/employee/1?pretty"
```

> 将 HTTP 命令由 `PUT` 改为 `GET` 可以用来检索文档，同样的，可以使用 `DELETE` 命令来删除文档，以及使用 `HEAD` 指令来检查文档是否存在。如果想更新已存在的文档，只需再次 `PUT` 。

### 轻量搜索

第一个尝试的几乎是最简单的搜索了。我们使用下列请求来搜索所有雇员：

```bash
GET /megacorp/employee/_search

curl -X GET "localhost:9200/megacorp/employee/_search?pretty"
```

返回结果包括了所有三个文档，放在数组 `hits` 中。一个搜索默认返回十条结果。

接下来，尝试下搜索姓氏为 ``Smith`` 的雇员。为此，我们将使用一个 *高亮* 搜索，很容易通过命令行完成。这个方法一般涉及到一个 *查询字符串* （*query-string*） 搜索，因为我们通过一个URL参数来传递查询信息给搜索接口：

```bash
GET /megacorp/employee/_search?q=last_name:Smith

curl -X GET "localhost:9200/megacorp/employee/_search?q=last_name:Smith&pretty"
```

### 使用查询表达式搜索

Query-string 搜索通过命令非常方便地进行临时性的即席搜索 ，但它有自身的局限性（参见 [*轻量* 搜索](https://www.elastic.co/guide/cn/elasticsearch/guide/current/search-lite.html) ）。Elasticsearch 提供一个丰富灵活的查询语言叫做 *查询表达式* ， 它支持构建更加复杂和健壮的查询。

*领域特定语言* （DSL）， 使用 JSON 构造了一个请求。我们可以像这样重写之前的查询所有名为 Smith 的搜索 ：

```bash
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}

curl -X GET "localhost:9200/megacorp/employee/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
'
```

这个请求使用 JSON 构造，并使用了一个 `match` 查询（属于查询类型之一，后面将继续介绍）。

### 更复杂的搜索

现在尝试下更复杂的搜索。 同样搜索姓氏为 Smith 的员工，但这次我们只需要年龄大于 30 的。查询需要稍作调整，使用过滤器 *filter* ，它支持高效地执行一个结构化查询。

```bash
GET /megacorp/employee/_search
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 	# 这部分是一个 range 过滤器，它能找到年龄大于 30 的文档，其中 gt 表示_大于_(great than)。
                }
            }
        }
    }
}

curl -X GET "localhost:9200/megacorp/employee/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}
'
```

### 全文搜索

截止目前的搜索相对都很简单：单个姓名，通过年龄过滤。现在尝试下稍微高级点儿的全文搜索——一项传统数据库确实很难搞定的任务。搜索下所有喜欢攀岩（rock climbing）的员工：

```bash
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}

curl -X GET "localhost:9200/megacorp/employee/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}
'

{
  ...
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.4167401,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 1.4167401,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.4589591,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      }
    ]
  }
}
```

Elasticsearch 默认按照相关性得分排序，即每个文档跟查询的匹配程度。第一个最高得分的结果很明显：John Smith 的 `about` 属性清楚地写着 “rock climbing” 。

这是一个很好的案例，阐明了 Elasticsearch 如何 *在* 全文属性上搜索并返回相关性最强的结果。Elasticsearch中的 *相关性* 概念非常重要，也是完全区别于传统关系型数据库的一个概念，数据库中的一条记录要么匹配要么不匹配。

### 短语搜索

找出一个属性中的独立单词是没有问题的，但有时候想要精确匹配一系列单词或者_短语_ 。 比如，我们想执行这样一个查询，仅匹配同时包含 “rock” *和* “climbing” ，*并且* 二者以短语 “rock climbing” 的形式紧挨着的雇员记录。

为此对 `match` 查询稍作调整，使用一个叫做 `match_phrase` 的查询：

```bash
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}

curl -X GET "localhost:9200/megacorp/employee/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
'
```

### 高亮搜索

许多应用都倾向于在每个搜索结果中 *高亮* 部分文本片段，以便让用户知道为何该文档符合查询条件。在 Elasticsearch 中检索出高亮片段也很容易。

再次执行前面的查询，并增加一个新的 `highlight` 参数：

```bash
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}

curl -X GET "localhost:9200/megacorp/employee/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}
'
```

当执行该查询时，返回结果与之前一样，与此同时结果中还多了一个叫做 `highlight` 的部分。这个部分包含了 `about` 属性匹配的文本片段，并以 HTML 标签 `<em></em>` 封装。

### 分析

Elasticsearch 有一个功能叫聚合（aggregations），允许我们基于数据生成一些精细的分析结果。聚合与 SQL 中的 `GROUP BY` 类似但更强大。

举个例子，挖掘出员工中最受欢迎的兴趣爱好：

```bash
GET /megacorp/employee/_search
{
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests" }
    }
  }
}

curl -X GET "localhost:9200/megacorp/employee/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests" }
    }
  }
}
'
```

## 分布式特性

Elasticsearch 天生就是分布式的，并且在设计时屏蔽了分布式的复杂性。Elasticsearch 在分布式方面几乎是透明的。Elasticsearch 尽可能地屏蔽了分布式系统的复杂性。这里列举了一些在后台自动执行的操作：

- 分配文档到不同的容器 或 *分片* 中，文档可以储存在一个或多个节点中
- 按集群节点来均衡分配这些分片，从而对索引和搜索过程进行负载均衡
- 复制每个分片以支持数据冗余，从而防止硬件故障导致的数据丢失
- 将集群中任一节点的请求路由到存有相关数据的节点
- 集群扩容时无缝整合新节点，重新分配分片以便从离群节点恢复

## 参考文献

- [Elasticsearch: 权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/getting-started.html)
- [奇客谷Elasticsearch教程](https://www.qikegu.com/docs/3047)
