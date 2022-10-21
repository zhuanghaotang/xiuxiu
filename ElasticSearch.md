# ElasticSearch

ElasticSearch是一个实时的分布式搜索和分析引擎，能够支持PB级的结构化和非结构化的数据处理。

ElasticSearch支持横向扩展，当容量不足时，可以通过不断的增加节点来进行扩容。

ElasticSearch提供了简单易用的RESTFul API。

**关于ElasticSearch的版本**

ElasticSearch6.0版本开始支持SQL访问、跨集群复制、索引的生命周期管理等功能。

ElasticSearch7.0版本开始废除了索引下的类型、开放security功能、同时索引下的主分片默认从5改成1（避免Over Sharding）



## 1.ElasticSearch的概念模型

### 1.1 文档

ElasticSearch是面向文档进行存储的，一个文档就类似数据库中的一条记录，同时文档使用JSON来进行序列化。

#### 文档的元数据

```
_index:文档对应的索引
_type:文档对应的类型
_id:文档的id
_score:文档的分数(文档与搜索的匹配程度)
_source:文档的原始报文
_version:文档的版本号,每次对文档进行新增、更新和删除操作时版本号都会+1.
_seq_no:文档的序列号,ElasticSearch能够保证后插入的文档的序列号比前插入的文档的序列号要大(序列号从0开始)
```

文档id是文档在索引中的唯一标识，在7.x版本中，文档的类型固定是_doc。

### 1.2 索引

索引是文档的集合，是一个命名空间的概念。

每个索引都有一个Mapping和Setting，Mapping用于定义文档中字段的名称与类型同时为文档中的字段进行相关的配置，而Setting则为索引进行相关的配置。

在7.X版本之前，一个索引下可以有多个类型，但在7.X版本之后一个索引下只能有一个类型，同时名称固定为_doc。



## 2.ElasticSearch的物理模型

### 2.1 节点

一个节点就是一个ElasticSearch进程，每个节点启动时都会分配一个UUID。

#### Master-Eligible节点

每个节点默认都是一个有资格成为Master的节点，可以通过node.master配置来进行修改。

#### Master节点

每个节点都保存了集群中各个节点的状态信息、索引的Mapping和Setting以及索引分片的路由信息等，但是只有Master节点可以进行修改，然后再同步给其他的节点。

当集群启动时，会从有资格成为Master节点的列表中选取一个作为Master节点，其他节点将会与Master节点进行连接以加入到集群当中。

<img src="https://i.loli.net/2020/07/25/adAKyUPTpwRWOcH.png" align="left" style="zoom:45%"/>

#### Data节点

Data节点用于存储分片，每个节点默认都是一个Data节点，可以通过node.data配置来进行修改。

当集群的存储容量不足时，可以通过往集群中添加Data节点来进行扩容。

#### Coordinating节点

Coordinating节点负责接收客户端的请求，同时将请求分发给对应的节点进行处理，然后对各个节点的返回进行汇总，最后再返回给客户端。

<img src="https://i.loli.net/2020/07/25/g2ajHDo1ASdKsyv.png" align="left" style="zoom:50%"/>

比如Client向协调节点发起一个索引的搜索请求，假如这个索引包含两个主分片P0和P1，同时这两个主分片分别位于不同的Data节点当中，那么协调节点将会把请求分发给这两个Data节点进行处理，然后对这两个节点的返回进行汇总最后再返回给Client。

每个节点都是一个Coordinating节点。

#### Ingest节点

Ingest节点用于数据的预处理，支持 pipeline操作，可以使用Ingest节点对数据进行过滤、转换等操作，类似于logstash中的filter。

每个节点默认都是一个Ingest节点，可以通过node.ingest: false来禁止。

#### Hot&Warm节点

热节点就是机器配置比较高的节点，适用于存储热数据。

冷节点就是机器配置没那么高的节点，适用于存储比较旧的数据。

**每一个节点都应该只承担一个角色，比如Master-eligible节点、Data节点。**



### 2.2 分片

文档是存储在分片当中的，分片包括主分片以及副本分片两种类型。

#### 主分片（Primary Shard）

创建索引时需要指定主分片的数量，主分片的数量一旦创建后就不能够进行修改。

在7.x版本后每个索引默认只创建一个主分片。

#### 副本分片（Replica Shard）

副本分片是主分片的备份，每个主分片默认都有一个副本分片，副本分片的数量可以动态的进行调整。

当主分片所在的节点宕机时，副本分片将会切换为主分片对外提供服务。

<img src="https://i.loli.net/2020/07/25/ctoZMvrpuiqD5GP.png" align="left" style="zoom:45%">

一个Data节点中不允许同时存在主分片及其副本分片。

#### 关于主分片的数量

如果主分片的数量设置得过少，会导致后续无法通过新增节点来实现水平扩展，同时如果单个分片中的数据量过多，会导致数据在重分配时会相当耗时。

如果主分片的数量设置得过多，会影响搜索结果的相关性打分以及统计结果的准确性，同时如果单个节点上包含过多的分片，会导致资源浪费以及性能下降。



### 2.3 集群

Elasticsearch集群是由一组具有相同cluster.name的节点组成。

集群一共有三种状态，分别为Green、Yellow、Red。

Green表示集群属于健康状态，所有的主分片以及副本分片都在正常运行。

Yellow表示集群中存在主分片没有副本分片的情况。

Red表示集群中有主分片没有正常运行，此时集群仍然能够对外提供服务，但是会有数据丢失的情况。



## 3.文档的基本操作

使用ElasticSearch提供的RESTFul API来进行操作。

```
#CURL -X请求方法 '请求地址' -d '请求体'
CURL -XGET 'http://localhost:9200' -d '{DSL}'
```

### 3.1 Index操作

索引一个文档，如果文档已存在则删除后再创建（需要指定文档的id），如果索引不存在则自动创建。

```json
PUT /index/_doc/id
{
	"field":"value"
}
```

### 3.2 Create操作

创建一个文档，如果文档已存在则报错（需要指定文档的id），如果索引不存在则自动创建。

```json
PUT /index/_create/id
{
	"field":"value"
}
```

创建一个文档（自动生成文档的id），如果索引不存在则自动创建。

```json
POST /index/_doc
{
	"field":"value"
}
```

### 3.3 Get操作

根据文档的id查找文档

```json
GET /index/_doc/id
```

### 3.4 Update操作

更新文档，如果Field相同则更新否则新增，如果文档不存在则报错。

```json
POST /index/_update/id
{
	"doc":{
		"field":"value"
	}
}
```

ElasticSearch不支持更新操作，每次更新都会创建一个新的文档。

### 3.5 Delete操作

根据文档的id删除文档 

```json
DELETE /user/_doc/id
```

### 3.6 Bulk操作

Bulk操作支持在一个请求中执行多个ES操作，Bulk操作支持Index、Create、Update、Delete四种类型的操作（支持不同的索引）

某个操作失败不会影响其他的操作执行。

```json
POST /_bulk
{"index":{"_index":"index","_id":"id"}}
{"field":"value","field":"value"}
{"create":{"_index":"index"}}
{"field":"value","field":"value"}
{"update":{"_index":"index","_id":"id"}}
{"doc":{"field":"value","field":"value"}}
{"delete":{"_index":"index","_id":"id"}}
```

### 3.7 MGet操作

批量根据文档的id查找文档（支持不同的索引）

```json
GET /_mget
{
  "docs":[
    {"_index":"index","_id":"id"},
    {"_index":"index","_id":"id"}
  ]
}
```



## 4.Search API

Search API用来进行搜索，使用_search表示这是一个搜索的请求。

有两种方式去使用Search API，一种是URL Search，通在URL中指定查询的参数，另外一种是Request Body，使用基于JSON的DSL查询语言。

```
#在集群的所有索引上进行搜索
GET /_search

#在某个索引上进行搜索
GET /index/_search

#在指定的多个索引上进行搜索
GET /index1,index2/_search

#在以index开头的索引上进行搜索(通配符)
GET /index*/_search
```

### 4.1 URL Search

```
#使用q参数表示查询参数
http://localhost:9200/user/_search?q=username:zht
```

### 4.2 DSL Search

```
http://localhost:9200/user/_search
{
	"query":{
		"term":{
			"username":{
				"value":"zht"
			}
		}
	}
}
```

### 4.3 响应报文解析

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 18,
      "relation" : "eq"
    },
    "max_score" : 0.34632954,
    "hits" : [
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "35Ec2XEBqSYcLhDOnWZG",
        "_score" : 0.34632954,
        "_source" : {
          "username" : "zht",
          "password" : "123456",
          "birthday" : "1996-08-02"
        }
      }
    ]
  }
}

```

```
took:执行此处搜索的总耗时（ms）
time_out:查询是否超时
_shards:分片信息
_shards.total:此次搜索使用到的分片数
_shard.successful:成功的分片数
_shard.skipped:跳过的分片数
_shard.failed:失败的分片数(如果查询需要使用的分片发生故障，那么不会中断整个搜索请求，只不过是会发生数据丢失)
hits:命中的文档总览
hits.total:命中的文档数
hits.max_score:命中的文档中分数的最大值
hits.hits:命中的文档集合，按照文档的分数进行倒序排序，同时只返回前10条
hits.hits._index:文档的索引
hits.hits._type:文档的类型
hits.hits._id:文档的id
hits.hits._score:文档与搜索的匹配程度，分数越高表示越匹配，最大值为1
hits.hits._source:文档的原始内容
```



## 5.ElasticSearch的DSL

### 5.1 Size

由于默认只返回命中的前10个文档，因此可以使用size属性指定返回的文档数。

```json
GET /goods/_search
{
	"size":50
}
```

### 5.2 分页

可以使用from和size属性指定从第几个文档开始返回size个文档。

```json
GET /goods/_search
{
	"from":10,
	"size":20
}
```

### 5.3 排序

由于默认按照文档的分数就行倒序排序，因此可以通过sort属性进行需改。

```json
GET /goods/_search
{
  "sort": [
    {
      "date": {
        "order": "desc"
      }
    }
  ]
}
```

如果字段的类型是text或keyword，则会按照字符串的字典排序。

### 5.4 指定查询的字段

由于响应体中的\_source会包含文档的所有字段，如果文档中包含大量的字段有可能会造成IO浪费，因此可以通过在查询时使用_source属性指定查询的字段。

```json
GET /goods/_search
{
  "_source": ["goods_title","goods_desc"]
}
```

### 5.5 全文搜索

使用match表示这是一个全文搜索请求，只要字段中出现过指定的单词即可，单词之间使用OR的关系，如果想使用AND的关系，即同时出现过，则可以使用operator属性就行修改。

```json
GET /goods/_search
{
  "query": {
    "match": {
      "goods_title": "耐克 nick"
    }
  }
}
```

```json
GET /goods/_search
{
	"query":{
		"match":{
			"goods_title":{
				"query":"nick airforce",
				"operator":"AND"
			}
		}
	}
}
```

### 5.6 全文短语搜索

使用match_phrase表示这是一个全文短语搜索请求，只要字段中出现过指定的短语即可。

可以使用slop属性设置短语中的单词之间最多可以有多少个无关的单词。

```json
GET /goods/_search
{
  "query": {
    "match_phrase": {
      "goods_desc": "2020 新款"
    }
  }
}
```

```json
GET /goods/_search
{
  "query": {
    "match_phrase": {
      "goods_desc": {
        "query": "2020 新款",
        "slop": 2
      }
    }
  }
}
```



## 6.Mapping

Mapping用于定义文档中字段的名称与类型，同时为索引中的字段进行相关的配置。

```
#查看索引的mapping
GET /index/_mapping
```

### 6.1 ElasticSearch中的数据类型

ElasticSearch中的数据类型分为简单类型、复杂类型、特殊类型三种类型。

##### 简单类型

text、keyword、short、integer、long、float、double、date、boolean

ElasticSearch没有专门提供数组类型，任何一个简单类型都可以包含多个类型相同的值。

##### 复杂类型

object（JSON对象）、nested（JSON对象数组）

##### 特殊类型

geo_point、geo_shape等

### 6.2 定义索引的mapping

```
PUT /username
{
  "mappings": {
    "properties": {
      "username":{
        "type": "text"
      },"password":{
        "type": "text"
      },"registryTime":{
        "type":"date",
        "format": "yyyy-MM-dd HH:mm"
      }
    }
  }
}
```

一旦为字段设置了类型那么就不能够进行修改，除非进行Reindex API操作重新建立索引。

### 6.3 Mapping中的相关属性


| 配置项        | 作用                                                         | 可选值                                                       |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| index         | 控制字段是否使用倒排索引                                     | true（默认）：字段使用倒排索引<br/>false：字段不使用倒排索引，可以节省很多的磁盘空间，但字段不能够被搜索。 |
| index_options | 控制倒排索引记录的内容                                       | docs：倒排索引项中只记录doc_id<br />freqs：倒排索引项中只记录doc_id与term_frequency<br />positions：倒排索引项中只记录doc_id、term_frequency、position<br />offsets：倒排索引项中会记录doc_id、term_frequency、position、 offset<br />text类型默认为positions，其他类型统一为docs。 |
| null_value    | 使keyword类型的字段的null值能够按null_value指定的值进行搜索<br />（如果字段的值为null，默认是无法按null来进行搜索的，但是可以通过exist进行搜索，exists要求必须存在该字段同时该字段的值不能为null） | "NULL"、"null"，不能为null                                   |
| copy_to       | 将字段的值拷贝到目标字段，该字段可以用来进行搜索（match），但是不会出现在文档的_source中（可以将多个字段copy_to到同一个目标字段） | copy_to:"newfield"                                           |
| analyzer      | 修改字段使用的分词器                                         | standard、simple、stop 、whitespace、keyword、patter、Language、第三方分词器 |

### 6.4 修改索引的Mapping

```
PUT /username/_mapping
{
  "properties": {
    "username": {
      "type": "text", 
      "index": true, 
      "index_options": "positions", 
      "copy_to": "fullValue"
    }, 
    "password": {
      "type": "text", 
      "index": true, 
      "index_options": "positions", 
      "copy_to": "fullValue"
    }
  }
}
```

### 6.5 Dynamic Mapping

当为索引新增一个字段时，如果有开启Dynamic Mapping功能（默认开启），那么将会触发Dynamic Mapping机制，会动态的根据文档中的信息推断出字段的类型。

Dynamic Mapping使用户不需要手动去定义索引的Mapping，但Dynamic Mapping有时候推算的类型不一定准确。

#### JSON类型与ElasticSearch类型的映射

| JSON类型   | ElasticSearch类型                                            |
| ---------- | ------------------------------------------------------------ |
| JSON对象   | 映射成object类型                                             |
| JSON数组   | 由第一个非空的元素的类型决定                                 |
| 字符串类型 | 如果是日期字符串，则映射成date类型。<br/>如果是数值字符串，则映射成float或者long类型（该功能默认关闭）<br/>如果都不是，则映射成Text类型，并且增加keyword子字段。 |
| 数值类型   | 如果是整数，则映射long类型<br/>如果是浮点数，则映射成float类型。 |
| 布尔类型   | 映射成Boolean类型。                                          |
| NULL类型   | 忽略                                                         |

#### 设置索引的Dynamic

Dynamic可以设置成true、false、strict。

当Dynamic为true时，一旦有新增的字段，那么会自动推断出字段的类型，并且更新mapping。

当Dynamic为false时，一旦有新增的字段，那么不会更新mapping，因此该字段不能够被搜索，但是会出现在文档的source中。

当Dynamic为strict时，一旦有新增的字段，那么不会更新mapping，同时文档将会写入失败。

```
PUT /index/_mapping
{
	"dynamic":true/false/strict
}
```



## 7.倒排索引

倒排索引记录了字段的单词与文档之间的关系。

倒排索引由单词词典（Term Dictionary）和倒排列表（Posting List）两部分组成。

默认情况下文档中的每个字段都会建立一个倒排索引，可以在索引的mapping中将字段的index属性设置为false来禁止使用倒排索引，好处是可以节省大量的磁盘空间，但是字段不能够被搜索。

### 7.1 单词词典

单词词典记录了字段中出现过的所有单词，单词词典记录的单词与分词器有关。

比如book索引下存在3个文档，每个文档的title分别为Mastering ElasticSearch、ElasticSearch Server、ElasticSearch Essentials，使用默认的分词器，那么单词词典中记录了Mastering、ElasticSearch、Server、Essentials这几个单词。

```json
GET /book/_doc/1
{
  "title":"Mastering ElasticSearch"
}
GET /book/_doc/2
{
  "title":"ElasticSearch Server"
}
GET /book/_doc/3
{
  "title":"ElasticSearch Essentials"
}
```

### 7.2 倒排列表

倒排列表记录了字段中出现过的所有单词与文档之间的关系。

倒排列表中的每一个倒排索引项都包含了文档的id、单词在该字段上出现的次数（tf）、单词在该字段上出现的位置（position）、单词在该字段上的偏移量（offset）

<img src="https://i.loli.net/2020/07/26/MWUyLJIQADvp731.png" align="left" style="zoom:90%"/>

单词与倒排索引项之间是一对多的关系。



## 8.分词器

分词器即按照一定的规则将一个文本分割成一系列的单词，不同的分词器使用不同的规则进行分词。

一个Analyzer由CharacterFilters、Tokenizer和TokenFilters三部分组成（CharacterFilter和TokenFilter可以有0到多个）

<img src="https://i.loli.net/2020/07/05/vjXVRNS9cPpW5n4.png" align="left" style="zoom:50%">

CharacterFilters负责对原始文本进行一些处理，比如去除HTML标签等。

Tokenizer负责按照一定的规则对文本进行分词。

TokenFilters负责对分割后的单词进行一些处理，比如转换成小写、删除停用词、增加同义词等。

### 8.1 ElasticSearch中提供的分词器

```
Standard Analyzer:按照词进行分割，并转换成小写（默认的分词器）

Simple Analyzer:按照非字母字符进行分割，非字母的字符将会被删除，并转换成小写。

Stop Analyzer:按照词进行分割，并转换成小写，同时过滤掉停用词。

Whitespace Analyzer:按照空格进行分割。

Keyword Analyzer:不分词，直接将输入当作输出。

Patter Analyzer:使用正则表达式进行分割，默认为\W+，即按照非字符进行分割。

Language Analyzer:ElasticSearch提供了30多种常见语言的分词器，比如english、chinese。

Customer Analyzer:用户自定义的分词器。
```

在自然语言处理中，停用词是指一些不包含什么信息的词以及特别高频的词，比如the，to，the，a，an，and等。

### 8.2 使用Analyzer API去查看这些分词器如何进行分词

直接指定某个analyzer进行测试

```json
GET /_analyze
{
	"analyzer":"standard",
	"text":"Elasticsearch is the distributed search and analytics engine at the heart of the Elastic Stack. Logstash and Beats facilitate collecting, aggregating, and enriching your data and storing it in Elasticsearch."
}
```

查看索引中的某个字段如何进行分词

```json
POST /index/_analyze
{
	"field":"desc",
	"text":"Elasticsearch is the distributed search and analytics engine at the heart of the Elastic Stack. Logstash and Beats facilitate collecting, aggregating, and enriching your data and storing it in Elasticsearch."
}
```

### 8.3 关于中文分词器

ElasticSearch提供的分词器会将中文分割成一个一个的字，不利于搜索，因此需要使用一些对中文更加友好的分词器。

比如会将"清华大学"分隔成"清"、"华"、"大"、"学"四个字，因此在搜索"清"时，会出来很多与清华大学不相干的文档。

常见的中文分词器有analysis-icu、ik、thulac，需要通过安装对应的插件来进行使用。

ik_smart分词器会将"清华大学"整个分为一个词，而ik_max_word分词器会将清华大学分隔成"清华大学"、"清华"、"大学"。

### 8.4 设置索引使用的分词器

```JSON
PUT /user
{
  "settings":{
    "index":{
      "analysis.analyzer.default.type":"ik_max_word" // 让索引的所有字段默认都使用ik_max_word分词器（全局）
    }
  },"mappings":{
    "properties": {
      "username":{ 
        "type": "text",
        "analyzer": "standard" // 让username字段使用standard分词器（局部）
      }
    }
  }
}
```

### 8.5 自定义分词器

ElasticSearch中提供了很多分词器以及CharacterFilter、Tokenizer、TokenFilter，用户可以对CharacterFilter、Tokenizer、TokenFilter进行组合来形成新的分词器。

**ElasticSearch提供的CharacterFilter**

```
html_strip:去除HTML标签
mapping:字符串替换
pattern_replace:通过正则表达式替换
```

**ElasticSearch提供的Tokenizer**

```
whitespace:按照空格
standard:按照词
uax_url_email:按邮箱格式
pattern:按正则
keyword:不分词
path_hierarchy:按文件路径
```

**ElasticSearch提供的TokenFilter**

```
lowercase:转小写
stop:删除停用词
synonym:添加近义词
```

使用Analyzer API去查看CharacterFilter、Tokenizer、TokenFilter的执行结果。

```json
POST /_analyze
{
  "char_filter": ["html_strip"],
  "tokenizer": "keyword",
  "text": "<b>hello world</b>"
}

{
  "tokens" : [
    {
      "token" : "hello world",
      "start_offset" : 3,
      "end_offset" : 18,
      "type" : "word",
      "position" : 0
    }
  ]
}
```

```json
POST /_analyze
{
  "char_filter": [
      {
        "type":"mapping",
        "mappings": ["- => _"] //将文本中的'-'替换成'_'，然后再按单词进行分隔
      }
  ],
  "tokenizer": "standard",
  "text": "123-456 789-101112"
}

{
  "tokens" : [
    {
      "token" : "123_456",
      "start_offset" : 0,
      "end_offset" : 7,
      "type" : "<NUM>",
      "position" : 0
    },
    {
      "token" : "789_101112",
      "start_offset" : 8,
      "end_offset" : 18,
      "type" : "<NUM>",
      "position" : 1
    }
  ]
}
```

```json
POST /_analyze
{
  "char_filter": [
    {
      "type":"pattern_replace",
      "pattern":"http://(.*)", 
      "replacement":"$1" // 剔除协议
    }
  ],
  "tokenizer": "standard",
  "text":"http://www.baidu.com"
}

{
  "tokens" : [
    {
      "token" : "www.baidu.com",
      "start_offset" : 0,
      "end_offset" : 20,
      "type" : "<ALPHANUM>",
      "position" : 0
    }
  ]
}
```

```json
POST /_analyze
{
  "tokenizer": "path_hierarchy",
  "text":"/User/zhuanghaotang/software/elasticsearch7.6.1"
}

{
  "tokens" : [
    {
      "token" : "/User",
      "start_offset" : 0,
      "end_offset" : 5,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "/User/zhuanghaotang",
      "start_offset" : 0,
      "end_offset" : 19,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "/User/zhuanghaotang/software",
      "start_offset" : 0,
      "end_offset" : 28,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "/User/zhuanghaotang/software/elasticsearch7.6.1",
      "start_offset" : 0,
      "end_offset" : 47,
      "type" : "word",
      "position" : 0
    }
  ]
}
```

```json
POST /_analyze
{
  "tokenizer": "whitespace",
  "filter": ["lowercase",{ "type":"stop","stopwords":["the"] }], 
  "text": "The Elasticsearch is the distributed search and analytics engine."
}

{
  "tokens" : [
    {
      "token" : "elasticsearch",
      "start_offset" : 4,
      "end_offset" : 17,
      "type" : "word",
      "position" : 1
    },
    {
      "token" : "is",
      "start_offset" : 18,
      "end_offset" : 20,
      "type" : "word",
      "position" : 2
    },
    {
      "token" : "distributed",
      "start_offset" : 25,
      "end_offset" : 36,
      "type" : "word",
      "position" : 4
    },
    {
      "token" : "search",
      "start_offset" : 37,
      "end_offset" : 43,
      "type" : "word",
      "position" : 5
    },
    {
      "token" : "and",
      "start_offset" : 44,
      "end_offset" : 47,
      "type" : "word",
      "position" : 6
    },
    {
      "token" : "analytics",
      "start_offset" : 48,
      "end_offset" : 57,
      "type" : "word",
      "position" : 7
    },
    {
      "token" : "engine.",
      "start_offset" : 58,
      "end_offset" : 65,
      "type" : "word",
      "position" : 8
    }
  ]
}
```



#### 1.自定义分词器的语法

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "char_filter": {
        "my_char_filter_name":{ // 自定义的char filter
          "type":""
        }
      },
      "tokenizer": {
        "my_tokenizer_name":{ // 自定义的tokenizer
          "type":""
        }
      },
      "filter": { 
        "my_filter_name":{ // 自定义的token filter
          "type":""
        }
      },
      "analyzer": {
        "my_analyzer_name":{ // 自定义的analyzer名称
          "type":"custom", // 类型固定是custom，表示这是一个自定义的analyzer
          "char_filter":["my_char_filter","es_inner_char_filter"], 
          "tokenizer":"my_tokenizer/es_inner_tokenizer",
          "filter":["my_token_filter","es_inner_token_filter"] 
        }
      }
    },
    "index":{
      "analysis.analyzer.default.type":"my_analyzer_name" // 使索引的所有字段都使用该自定义的analyzer（全局）
    }
  },"mappings":{
    "properties": {
      "property_name":{ 
        "type": "text",
        "analyzer": "standard" // 让索引的某个字段使用standard分词器（局部）
      }
    }
  }
}
```

#### 2.具体示例

```json
PUT /famous_person
{
  "settings": {
    "analysis": {
      "char_filter": {
        "to_duck":{
          "type":"mapping",
          "mappings":["Donald Trump => duck"]
        }
      },"analyzer": {
        "to_duck_analyzer":{
          "type":"custom",
          "char_filter":["to_duck","html_strip"],
          "tokenizer":"standard",
          "filter":["lowercase","stop"]
        }
      }
    }
  },"mappings": {
    "properties": {
      "name":{
        "type":"text",
        "analyzer": "to_duck_analyzer"
      }
    }
  }
}
```

#### 3.测试分词器

```json
POST /famous_person/_analyze
{
  "field": "name",
  "text": "Donald Trump"
}

{
  "tokens" : [
    {
      "token" : "duck",
      "start_offset" : 0,
      "end_offset" : 12,
      "type" : "<ALPHANUM>",
      "position" : 0
    }
  ]
}
```

#### 4.测试搜索

```json
//创建一个文档
POST /famous_person/_doc
{
  "name":"Donald Trump"
}

//搜索
GET /famous_person/_search

//指定duck搜索
GET /famous_person/_search
{
  "query": {
    "match": {
      "name": "duck"
    }
  }
}

//指定Donald Trump搜搜
GET /famous_person/_search
{
  "query": {
    "match_phrase": {
      "name": "Donald Trump"
    }
  }
}

//都返回同一个文档
{
  "_index": "famous_person", 
  "_type": "_doc", 
  "_id": "XeM8Q3MBJUEZUyF9j-6U", 
  "_score": 0.2876821, 
  "_source": {
    "name": "Donald Trump"
  }
}
```

分词器只会影响倒排索引中的单词词典（决定文档如何被搜出来），不会改变索引时文档的原始值，文档在索引时的原始值是什么那么被搜索出来时就是什么。

虽然文档的原始值已经被转换并且进行分词，但是仍然能够通过完整的原始值来进行搜索（单词词典维护了完整的原始值）



Search Analyzer。。









## 9.多字段特性

ElasticSearch提供多字段的特性，可以为一个字段添加多个子字段。

多字段可以为同一个文本使用不同的分词器进行分词，并且使用子字段来进行搜索。

比如中文内容可以使用其英文名称和拼音来进行搜索（通过子字段），比如"清华大学"，可以使用"Tsinghua University"和"qing hua da xue"来进行搜索。

### 9.1 自定义分词器

```json
PUT /school
{
  "settings": {
    "analysis": {
      "char_filter": {
        "to_english_name":{ 
          "type":"mapping",
          "mappings":["清华大学 => Tsinghua University","北京大学 => Peking University","中国人民大学 => Renmin University of China"]
        },"to_pinyin_name":{ 
          "type":"mapping",
          "mappings":["清华大学 => qing hua da xue","北京大学 => bei jing da xue","中国人民大学 => zhong guo ren min da xue"]
        }
      },
      "filter": { 
        "stop_word":{ 
          "type":"stop",
          "stopwords":["的","是","在","in","is"]
        }
      },
      "analyzer": {
        "university_english_analyzer":{ 
          "type":"custom",
          "char_filter":["to_english_name","html_strip"],
          "tokenizer":"standard",
          "filter":["lowercase","stop_word"] 
        },"university_pinyin_analyzer":{ 
          "type":"custom",
          "char_filter":["to_pinyin_name","html_strip"],
          "tokenizer":"standard"
        }
      }
    }
  }
}
```

### 9.2 修改索引的Mapping

```json
PUT /school/_mapping
{
  "properties": {
    "school_name":{
      "type":"text",
      "fields": {
        "english_name":{
          "type":"text",
          "analyzer":"university_english_analyzer"
        },"pinyin_name":{
          "type":"text",
          "analyzer":"university_pinyin_analyzer"
        }
      }
    }
  }
}
```

### 9.3 使用子字段进行搜索

```json
POST /school/_doc
{
  "school_name":"清华大学"
}

GET /school/_search
{
  "query": {
    "match": {
      "school_name": "清华"
    }
  }
}

GET /school/_search
{
  "query": {
    "match": {
      "school_name.english_name": "Tsinghua"
    }
  }
}

GET /school/_search
{
  "query": {
    "match": {
      "school_name.pinyin_name": "qing hua"
    }
  }
}

#都返回同一个文档
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.5753642,
    "hits" : [
      {
        "_index" : "school",
        "_type" : "_doc",
        "_id" : "WuMjQ3MBJUEZUyF9ne73",
        "_score" : 0.5753642,
        "_source" : {
          "school_name" : "清华大学"
        }
      }
    ]
  }
}
```





## 10.关于Keyword与Text类型

使用keyword类型来表示精确值，精确值即数字、日期、具体的一个字符串等，比如姓名，公司名称等等，ElasticSearch不会对keyword类型进行分词。

使用text类型来表示文本值，文本值即非结构化的文本数据，比如文章的标题、摘要、文章的内容、评论等等。

keyword类型只能通过精确值进行搜索，对应term和match DSL，而text类型只能通过全文搜索和全文短语搜索，对应match与match_pharse DSL。

keyword类型虽然不会被分词，但是也会建立倒排索引，单词词典中直接存储精确值，同时倒排索引项中只包含doc_id属性，单词词典与倒排列表是一对一的关系。

<img src="https://i.loli.net/2020/07/12/V7PQjYms1hnxSMU.png" align="left" style="zoom:60%">



## 11.Index Template

每个索引都拥有Mappings和Settings两个配置，Mappings用于定义索引中字段的名称与类型，同时为索引中的字段进行相关的配置，而Settings则为索引进行相关的配置。

正常情况下用户需要手动的为每个索引定义自己的Mappings和Settings，为了易于管理，ES提供了Index Template的功能，通过创建模板，自动的为索引添加默认的Mappings和Settings配置。

### 11.1 创建模板

```
PUT /_template/default_template
{
  "index_patterns": ["*"],  // 匹配所有的索引
  "order": 0, // 模板的序号，如果一个索引匹配多个模板，那么会按照模板的序号从小到大执行，后执行的配置将会覆盖先执行的配置。
  "mappings": {
    "date_detection": false,
    "numeric_detection": true
  },
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}
```

如果用户在创建索引时就为索引定义了Mappings和Settings，那么用户自定义的Mappings和Settings中的配置将会优于模板中配置的。

### 11.2 查看模板

```
GET /_template/default_template
```



## 12.Dynamic Template

ES提供的Dynamic Mapping可以自动的为索引中的字段推断出类型，由于Dynamic Mapping中类型的映射总是固定的，如果想修改类型的映射关系那么可以使用ES提供的Dynamic Template功能。

Dynamic Template可以将符合指定规则的字段映射成指定的类型。

### 12.1 定义Dynamic Template

```
PUT /user
{
  "mappings": {
    "dynamic_templates": [
      {
        "string_as_boolean": { // Dynamic Template的名称
          "match_mapping_type": "string",  // 匹配字符串类型
          "path_match": ["is_*","enable_*"],  // 匹配字段的名称
          "mapping": {
            "type": "boolean" // 将JSON的字符串类型同时以is_和enable_开头的字段映射成boolean类型
          }
        }
      }, 
      {
        "auto_keyword": {
          "match_mapping_type": "string", 
          "mapping": {
            "type": "keyword" // 将JSON的字符串类型映射成keyword类型
          }
        }
      }
    ]
  }
}
```

当定义了Dynamic Template之后，如果为索引新增字段，或者新增的文档的索引不存在时，那么将会通过Dynamic Template以及Dynamic Mapping进行类型的映射，Dynamic Template的优先级比Dynamic Mapping要高。





ElasticSearch的动态扩容

当往ElasticSearch集群当中增加节点，或者从ElasticSearch集群当中剔除节点时，能够自动的进行负载均衡，迁移分片。

<img src="/Users/zhuanghaotang/data/图片/md/elasticsearch/es扩容模型.jpg" align="left" style="zoom:50%"/>

当从ElasticSearch集群中查找一个文档时，会随机连接集群中的一个节点，该节点会从所有存有相关数据的节点中收集数据，并且将最终的结果返回给客户端（从主分片或者主分片对应的副本分片中查询）



## Query Then Fetch

Query Then Fetch原则包含Query和Fetch两个阶段。

Query阶段：协调节点从所有的主副分片中各选择一个，然后分别路由到这些分片当中，每个分片上都需要根据文档的分数进行倒序排序，同时返回前size个文档的id以及分数值。

Fetch阶段：协调节点对每个分片返回的前from+size个文档的分数再进行倒序排序，然后拿到前from+size个文档的id，然后再根据文档的id从这些分片当中获取文档的详细数据。 	

