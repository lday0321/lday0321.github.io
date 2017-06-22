---
title: ElasticSearch总结
date: 2017-06-22 22:03:18
tags:
	- elasticsearch
---

前段时间对分布式追踪相关的实现方案进行了一些调研，了解到近期对于大数据的日志检索、分析从原来基于hadoop的实现逐渐过渡到基于es的方案上来。近期在消息审计追踪相关的项目上也尝试的使用了类似的方案。这里对es的一些了解以及常用的一些使用整理于此。

<!-- more -->

## 1. 全文索引

全文索引是指计算机搜索程序通过扫描文件中的每个单词，对每个词建立一个索引，指明该词在文章中出现的次数和位置，当用户查询时，搜索程序就根据事先建立的索引进行查找，并将查找结果反馈给用户。这个过程类似于通过字典中的搜索字表查字的过程。

### Lucene倒排索引

倒排索引原语实际应用中需要更具属性的值来查找记录。这种索引表中的每一项都包括一个属性值和具有该属性值的各记录的地址。由于不是由记录类确定属性值，而是由属性值来确定记录的位置，因而称为倒排索引(inverted index)。带有倒排索引的文件我们称为倒排索引文件，简称倒排文件(inverted file)

> **倒排索引为什么叫倒排索引？**  
> * 渣翻译的例子之一。英文原名Inverted index，大概因为 Invert 有颠倒的意思，就被翻译成了倒排。但是倒排这个名称很容易让人理解为从A-Z颠倒成Z-A。个人认为翻译成转置索引可能比较合适。一个未经处理的数据库中，一般是以文档ID作为索引，以文档内容作为记录。而Inverted index 指的是将单词或记录作为索引，将文档ID作为记录，这样便可以方便地通过单词或记录查找到其所在的文档。  
>  
> * 倒排索引对应的英文术语为inverted index，有的papers里也成为inverted files，说的都是同一种东西。倒排索引是区别于正排索引(forward index)来说的。  
> 文档是有许多的单词组成的，其中每个单词也可以在同一个文档中重复出现很多次，当然，同一个单词也可以出现在不同的文档中。  
> 正排索引(forward index)：从文档角度看其中的单词，表示每个文档（用文档ID标识）都含有哪些单词，以及每个单词出现了多少次（词频）及其出现位置（相对于文档首部的偏移量）。  
> 倒排索引(inverted index，或inverted files)：从单词角度看文档，标识每个单词分别在那些文档中出现(文档ID)，以及在各自的文档中每个单词分别出现了多少次（词频）及其出现位置（相对于该文档首部的偏移量）。  
> 简单记为：  
> 正排索引：文档 ---> 单词  
> 倒排索引：单词 ---> 文档  

倒排索引中的索引对象是文档或者文档集合中的单词等，用来存储这些单词在一个文档或者一组文档中的存储位置，是对文档或者文档集合的一种最常见的索引机制。

搜索引擎的关键步骤就是建立倒排索引，倒排索引一般表示为一个关键词，然后是他的频度（出现次数）、位置（出现在哪一篇文章或者网页中，及有关的日期，作者信息等），好比一本书的目录、标签。读者想看哪一个主题相关的章节，直接根据目录即可找到相关的页面。不必再从书的第一页到最后一页，一页一页的查找。

#### 示例

假设有两篇文章1和文章2
文章1的内容为：Tom lives in Guangzhou, I live in Guangzhou too.
文章2的内容为：He once lived in Shanghai.

##### 步骤1：取得关键词

由于lucene是基于关键词索引和查询的，首先我们要取得这两篇文章的关键词，通常我们需要如下处理措施：

1. 我们现在有的是文章内容，即一个字符串，我们先要找出字符串中的所有单词，即分词。英文单词由于用空格分隔，比较好处理。中文单词间是连在一起的需要特殊的分词处理。
2. 文章中的”in”, “once” “too”等词没有什么实际意义，中文中的“的”“是”等字通常也无具体含义，这些不代表概念的词可以过滤掉
3. 用户通常希望查“He”时能把含“he”，“HE”的文章也找出来，所以所有单词需要统一大小写。
4. 用户通常希望查“live”时能把含“lives”，“lived”的文章也找出来，所以需要把“lives”，“lived”还原成“live”
5. 文章中的标点符号通常不表示某种概念，也可以过滤掉

在lucene中以上措施由Analyzer类完成
经过上面处理后
* 文章1的所有关键词为：[tom] [live] [guangzhou] [live] [guangzhou]
* 文章2的所有关键词为：[he] [live] [shanghai]


##### 步骤2：建立倒排索引

有了关键词后，我们就可以建立倒排索引了。上面的对应关系是：“文章号”对“文章中所有关键词”。倒排索引把这个关系倒过来，变成：“关键词”对“拥有该关键词的所有文章号”。文章1，2经过倒排后的对应关系见表：

| 关键词 | 文章号 |
|---|---|
| guangzhou | 1 |
| he | 2 |
| i | 1 |
| live | 1, 2 |
| shanghai | 2 |
| tom | 1 |

通常仅知道关键词在哪些文章中出现还不够，我们还需要知道关键词在文章中出现次数和出现的位置，通常有两种位置:
1. 字符位置，即记录该词是文章中第几个字符（优点是关键词亮显时定位快）
2. 关键词位置，即记录该词是文章中第几个关键词（优点是节约索引空间、词组（phase）查询快），lucene 中记录的就是这种位置。

加上“出现频率”和“出现位置”信息后，我们的索引结构变为：

| 关键词 | 文章号[出现频率] | 出现位置 |
|---|---|---|
| guangzhou | 1[2] | 3，6 |
| he | 2[1] | 1 |
| i  | 1[1] | 4 |
| live | 1[2], 2[1] | 2, 5, 2 |
| shanghai | 2[1] | 3 |
| tom | 1[1] | 1 |

以live 这行为例我们说明一下该结构：live在文章1中出现了2次，文章2中出现了一次，它的出现位置为“2,5,2”这表示什么呢？我们需要结合文章号和出现频率来分析，文章1中出现了2次，那么“2,5”就表示live在文章1中出现的两个位置，文章2中出现了一次，剩下的“2”就表示live是文章2中第 2个关键字。

以上就是lucene索引结构中最核心的部分。我们注意到关键字是按字符顺序排列的（lucene没有使用B树结构），因此lucene可以用二元搜索算法快速定位关键词。

##### 实现

实现时 lucene将上面三列分别作为词典文件（Term Dictionary）、频率文件(frequencies)、位置文件 (positions)保存。其中词典文件不仅保存有每个关键词，还保留了指向频率文件和位置文件的指针，通过指针可以找到该关键字的频率信息和位置信息。

Lucene中使用了field的概念，用于表达信息所在位置（如标题中，文章中，url中），在建索引中，该field信息也记录在词典文件中，每个关键词都有一个field信息(因为每个关键字一定属于一个或多个field)。

##### 压缩算法

为了减小索引文件的大小，Lucene对索引还使用了压缩技术。首先，对词典文件中的关键词进行了压缩，关键词压缩为<前缀长度，后缀>，例如：当前词为“阿拉伯语”，上一个词为“阿拉伯”，那么“阿拉伯语”压缩为<3，语>。其次大量用到的是对数字的压缩，数字只保存与上一个值的差值（这样可以减小数字的长度，进而减少保存该数字需要的字节数）。例如当前文章号是16389（不压缩要用3个字节保存），上一文章号是16382，压缩后保存7（只用一个字节）。

##### 应用场景

下面我们可以通过对该索引的查询来解释一下为什么要建立索引。

假设要查询单词 “live”，lucene先对词典二元查找、找到该词，通过指向频率文件的指针读出所有文章号，然后返回结果。词典通常非常小，因而，整个过程的时间是毫秒级的。

而用普通的顺序匹配算法，不建索引，而是对所有文章的内容进行字符串匹配，这个过程将会相当缓慢，当文章数目很大时，时间往往是无法忍受的。


## 2. 术语

* term，索引词。能够被索引的精确值，索引词(term)可以通过term搜索进行准确查询
* text，文本。是一段普通的非结构化文字。通常文本会被分析称一个个的索引词。
* analysis，分析。分析是将文本转换为索引词的过程，分析的结果依赖于分词器。
* index，索引。是具有相同结构的文档集合。在单个集群中，可以定义多个你想要的索引。（**索引相当于数据库中的表**）
* type，类型。类型是索引的逻辑分区。（**相当于数据库表中子表定义**，一个索引（数据库中的表）可以有多个子表）。在一般情况下，一种类型被定义为具有一组公共字段的文档。例如，让我们假设你运行一个博客平台，并把所有的数据存储在一个索引中。在这个索引中，你可以定义一种类型为“用户数据”，一种类型为“博客数据”，另外一种类型为“评论数据”
* document，文档。是存储在Elasticsearch中的一个JSON格式的字符串（**数据库表中的一行记录**）。
* mapping，映射。**相当于数据库中的表结构**，每一个索引都有一个映射，它定义了索引中每一个字段类型，以及一个索引范围内的设置。一个映射可以实现被定义，或者在第一次存储文档时候被自动识别
* field，字段。文档中包含零个或者多个字段，字段可以是一个简单的值，也可以是一个数组或者是对象的嵌套结构。**字段类似于表中的列**
* source field，来源字段。默认情况下，你的源文档江北存储在_source这个字段中，当你查询的时候也是返回这个字段。这允许你可以从搜索结构中访问原始的对象，这个对象返回一个精确的JSON字符串，这个对象不显示索引分析后的任何数据
* id，主键。是文件的唯一标识，如果在存库的时候没有提供ID，系统会自动生成一个ID，文档的“**index/type/id**”必须是唯一的


![](http://og43lpuu1.bkt.clouddn.com/elasticsearch_summary/001_es_index_terminology.png)


## 3. 操作API

elasticsearch提供了丰富的操作方法（API），使得用户可以很方便的对elasticsearch进行管理，以及通过elasticsearch查询，汇总各类统计结果。完整的详细说明可参见[Elasticsearch Reference][1]，这里我主要总结了几类基础的，我比较常用的操作API

* 索引操作相关
  * 创建索引
  * 获取索引
  * 删除索引
  * 重建索引
  * 创建索引模板
  * 查看索引模板
  * 删除索引模板
* 查询操作相关
  * 获取索引上的文档总数
  * (不)包含某个字段的文档
  * AND查询
  * Time Range查询
* 聚合操作相关
  * 按某个字段x聚合，获取x总数count(x)



因为elasticsearch提供的都是HTTP RESTful API接口，因此很容易通过http工具对各接口进行验证，我在使用时，一般使用2类工具进行验证：

1. kibana的DevTools
  通过kebana的DevTools，可以很方便的根据需要书写http msg body，同时，kibana还可以给出关键字提示，执行命令，返回结果就显示在右侧的结果栏中
  ![](http://og43lpuu1.bkt.clouddn.com/elasticsearch_summary/002_kibana_dev_tool.png)
  
2. curl命令
  强大的curl命令，可以用于构建各类http请求，并获得反馈结果。某些msgbody内容较大，较复杂的命令，我会使用curl，指定本地文件的形式来执行。例如，创建post索引，索引的映射类型定义在post.json中：`curl -XPOST 'http://192.168.149.150:9200/posts' -d @posts.json`


### 索引操作相关

#### 创建索引

创建索引，相当于创建数据库中的一张表，在创建索引时，我们往往会给出该索引的映射，方便索引文档的构建。假设我们需要创建一个索引：myIndex，在这个索引上，我们计划会创建两类文档：doc1， doc2。此时，我们会定义一个映射配置文件`my_index_map.json`，这个文件定义了doc1，doc2应该怎么被索引：
```
{
    "mappings":{
    
        "doc1" : {
            "properties" : {
                "log_timestamp" : {
                    "type" : "date",
                    "store": true,
                    "format" : "MM/dd/yy HH:mm:ss z"
                },
                "pub_appid" : {
                    "store": true,
                    "type" : "integer"
                },
                "remote_ip" : {
                    "store": true,
                    "type" : "text"
                },
                "real_ip" : {
                    "store": true,
                    "type" : "text"
                },
                "topic" : {
                    "store": true,
                    "type" : "keyword"
                },
                "partition" : {
                    "store": true,
                    "type" : "integer"
                },
                "offset" : {
                    "store": true,
                    "type" : "long"
                },
                "async_flag" : {
                    "store": true,
                    "type" : "boolean"
                }
            }
        },

        "doc2" : {
            "properties" : {
                "log_timestamp" : {
                    "type" : "date",
                    "store": true,
                    "format" : "MM/dd/yy HH:mm:ss z"
                },
                "sub_appid" : {
                    "store": true,
                    "type" : "integer"
                },
                "sub_group" : {
                    "store": true,
                    "type" : "keyword"
                },
                "remote_ip" : {
                    "store": true,
                    "type" : "text"
                },
                "real_ip" : {
                    "store": true,
                    "type" : "text"
                },
                "topic" : {
                    "store": true,
                    "type" : "keyword"
                },
                "partition" : {
                    "store": true,
                    "type" : "integer"
                },
                "offset" : {
                    "store": true,
                    "type" : "long"
                },
                "dack_flag" : {
                    "store": true,
                    "type" : "boolean"
                }
            }
        }        
        
    }
}
```
在这个映射文件中，我们定义doc1, doc2的字段映射关系。doc1中有哪些字段：log_timestamp, pub_appid, ...，同时，给这些字段定义了类型：date, integer, keyword...等，基于这个映射文件，我们可以创建我们需要的索引：myIndex:
`
curl -XPOST 'http://192.168.149.150:9200/myIndex' -d @my_index_map.json`

#### 获取索引

通过GET命令，则可以方便的获取到你的目标索引的定义信息：
`curl -XGET 'http://192.168.149.150:9200/myIndex?pretty'`

#### 删除索引

通过DELETE命令，则是删除目标索引：
`curl -XDELETE "http://192.168.149.150:9200/myIndex/"`

#### 重建索引

有些时候，我们会希望调整索引的映射配置，在调整映射配置之后，有时候是需要对索引进行重建的。重建索引指令主要用于将一个索引中的数据“搬到”另外一个索引中去。详细信息可参考：[Reindex API][2]的说明

```
curl -XPOST '192.168.149.150:9200/_reindex?pretty' -d'
{
  "source": {
    "index": "myIndex"
  },
  "dest": {
    "index": "myIndex_new"
  }
}
'
```

当索引中的数据量很大时，重建索引需要一定的时间才能完成，通过下述命令可以查看索引重建的状态

`curl -XGET '192.168.149.150:9200/_tasks?detailed=true&actions=*reindex&pretty'`

#### 创建索引模板

随着时间的推移，如果所有数据都放在同一个索引中，会导致索引变的越来越大，无法控制。为方便对索引内容进行管理，我们往往会以天为单位，每天建一个索引，这样，则可以滚动创建一批索引，例如：myIndex_2017.06.21, myIndex_2017.06.22, myIndex_2017.06.23, myIndex_2017.06.24, ...，当整个myIndex_*索引簇过大时，我们可以清理掉若干天的索引，保留最近7天的索引。如何滚动保留最近7天的数据，可参考elasticsearch工具curator的说明：[Curator Reference][3]

为达到按天建立索引的目标，我们需要创建一个索引模板，**告知elasticsearch，以某种规律命名的索引，都基于这个索引模板来设置映射。**

与创建索引类似，在创建索引模板时，我们也会给出一个索引模板的定义文件：my_index_map_template.json:

```
{
    "template": "my_index_*",
    "mappings":{
    
        "doc1" : {
            "properties" : {
                "log_timestamp" : {
                    "type" : "date",
                    "store": true,
                    "format" : "MM/dd/yy HH:mm:ss z"
                },
                "pub_appid" : {
                    "store": true,
                    "type" : "integer"
                },
                "remote_ip" : {
                    "store": true,
                    "type" : "text"
                },
                "real_ip" : {
                    "store": true,
                    "type" : "text"
                },
                "topic" : {
                    "store": true,
                    "type" : "keyword"
                },
                "partition" : {
                    "store": true,
                    "type" : "integer"
                },
                "offset" : {
                    "store": true,
                    "type" : "long"
                },
                "async_flag" : {
                    "store": true,
                    "type" : "boolean"
                }
            }
        },

        "doc2" : {
            "properties" : {
                "log_timestamp" : {
                    "type" : "date",
                    "store": true,
                    "format" : "MM/dd/yy HH:mm:ss z"
                },
                "sub_appid" : {
                    "store": true,
                    "type" : "integer"
                },
                "sub_group" : {
                    "store": true,
                    "type" : "keyword"
                },
                "remote_ip" : {
                    "store": true,
                    "type" : "text"
                },
                "real_ip" : {
                    "store": true,
                    "type" : "text"
                },
                "topic" : {
                    "store": true,
                    "type" : "keyword"
                },
                "partition" : {
                    "store": true,
                    "type" : "integer"
                },
                "offset" : {
                    "store": true,
                    "type" : "long"
                },
                "dack_flag" : {
                    "store": true,
                    "type" : "boolean"
                }
            }
        }        
        
    }
}
```

这个索引模板中的mapping部分，和表中的索引映射文件一致，需要注意的是，索引模板多了一个字段：`"template": "my_index_*"`，这个字段就是告诉elasticsearch，如果有人想要建立以`my_index_`开头的索引，则根据这个索引模板定义个mapping进行创建。

创建索引模板，我们使用命令：
`curl -XPUT 'http://192.168.149.150:9200/_template/my_index_per_day' -d @my_index_map_template.json`

#### 查看索引模板

查看索引模板的定义，则使用：
`curl -XGET 'http://192.168.149.150:9200/_template/my_index_per_day?pretty'`

#### 删除索引模板

需要删除索引模板时，使用：
`curl -XDELETE '192.168.149.150:9200/_template/my_index_per_day?pretty'`

### 查询操作相关

#### 获取索引上的文档总数

当需要获取一个索引上有多少数据时，需要使用match_all匹配，es会返回totalCount

```
GET my_index_*/_search
{
  "from":0,
  "size":0,
  "query": {
    "match_all": {}
  }
}
```

设置size:0，告诉es不用返回任何一条结果。我们只需要知道totalCount信息

#### (不)包含某个字段的文档

通过exists查询，可以获取那些包含某个字段值的记录

```
curl -XGET 'localhost:9200/_search?pretty' -H 'Content-Type: application/json' -d'
{
    "query": {
        "exists" : { "field" : "user" }
    }
}
'
```

反过来，exists查询结合bool复合查询（must_not事件类型）可以搜索不包含某字段的文档记录

```
curl -XGET 'localhost:9200/_search?pretty' -H 'Content-Type: application/json' -d'
{
    "query": {
        "bool": {
            "must_not": {
                "exists": {
                    "field": "user"
                }
            }
        }
    }
}
'
```

#### AND查询

and查询则是找出同时满足某几个条件的数据，同样可以基于bool复合查询来完成：
```
GET my_index_2017.05.05/doc1/_search
{
  "from": 0,
  "size": 5,
  "sort": [
    {
      "offset": "asc"
    }
  ],
  "query": {
    "bool" : {
      "must" : [
        {
          "term" : {"topic": "55.userpuid.v1"}
        }, 
        { 
          "term" :{"partition": 0}
        }
      ]
    }
  }
}
```
注意到，上面我们还是用了sort字段，这里规定了es按哪个字段进行排序并返回结果。



#### Time Range查询

对date类型的字段进行time range查询，可以返回某一时间区间内的结果：

```
GET my_index_2017.05.08/doc1/_search
{
  "from": 0,
  "size": 10,
  "sort" : [
        { "log_timestamp" : "asc" }
    ],
   "_source": ["topic","partition","offset", "log_timestamp"],
  "query": {
    "bool" : {
      "must" : [
        {
          "term" : {"topic": "51.cda-store.v1"}
        },
        {
          "range": {
            "log_timestamp": {
              "gte": "05/08/17 10:45:00 CST",
              "lte": "05/08/17 10:55:00 CST",
              "format": "MM/dd/yy HH:mm:ss z"
            }
          }
        }
      ]
    }
  }
}
```

`"gte": "05/08/17 10:45:00 CST",`中`gte`表示要求时间点大于或等于:"05/08/17 10:45:00 CST", 同理，`"lte": "05/08/17 10:55:00 CST"`表示要小于等于"05/08/17 10:55:00 CST"

### 聚合操作相关

#### 按某个字段x聚合，获取x总数count(x)

聚合操作会按照指定的聚合指标对数据进行聚合统计，详细的聚合接口说明可参考[Aggregations章节][4]的说明

```
GET my_index_*/doc2/_search
{
  "from": 0,
  "size": 0,
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "topic": "201.OrderStatusDelayMsg.v1"
          }
        },
        {
          "term": {
            "partition": 0
          }
        }
      ]
    }
  },
  "aggs": {
    "cg": {
      "terms": {
        "field": "sub_group",
        "size": "10000"
      }
    }
  }
}
```

上面定义了一个名为cg的聚合，该聚合的类型是terms(桶聚合的一种），这个聚合是按"sub_group"这个指标进行的，实际上，上面查询的是topic=201.OrderStatusDelayMsg.v1&partition=0的记录，按sub_group进行统计，看每个sub_group分别有多少数据量。这里需要注意到cg聚合中还有一个**字段"size"**，之所以将其设置成10000，是希望es在每个shard上都将所有数据进行聚合，获取到精确数量，而不是仅仅将topN的数据做聚合，size较少时，汇总的数据不精确。具体可参见对size字段的描述:[Document counts are approximate][5]

[1]: https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
[2]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html
[3]: https://www.elastic.co/guide/en/elasticsearch/client/curator/current/index.html
[4]: https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html
[5]: https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html#_size