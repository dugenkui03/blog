
github地址：https://github.com/dbpedia/lookup/tree/master

##### 1.介绍
DBpedia是一个根据相关关键字检索DBpedia URIs的web服务器。所谓“相关” 指的是资源匹配的标签或者是频繁用于维基百科页面指定特定资源的锚文本。==结果是通过指向其他维基百科页面的链接数量排名的==。

##### 2.web API

此项目提供了两个接口：关键字搜索和前缀搜索。托管版本的服务可以在[lookup.dbpedia](http://lookup.dbpedia.org/api/search/KeywordSearch?QueryClass=place&QueryString=berlin)中找到。

###### 2.1关键字查询
关键字查询接口可以找到与给定包含一个或者多个单词的关键字相关的DBpedia资源。实例如下：
- http://lookup.dbpedia.org/api/search/KeywordSearch?QueryClass=place&QueryString=berlin

###### 2.2前缀查询
前缀查询接口用来实现“输入自动补全autocomplete input boxes”，对于给定的“一般的单词”，比如Berl(in),API可以自动获取相关的资源：
http://dbpedia.org/resource/Berlin

示例：返回前五个与“berl”相关的资源
- http://lookup.dbpedia.org/api/search/PrefixSearch?QueryClass=&MaxHits=5&QueryString=berl

###### 2.3 参数解释
可以收的参数含义如下：
1. `QueryString:`DBpedia URI中应该 包含/模糊匹配 的字符串;
2. `QueryClass:`结果应该包含的、从Ontology中来的DBpedia中的类（owl#thing和无类型资源不用指定这个参数);
3. `MaxHits:`返回结果的数量，默认5；

###### 2.4 返回结果支持json格式
返回结果默认为XML，请求头包含`Accept: application/json`则返回JSON格式.

##### 3.运行服务的本地镜像

###### 3.1 可控并构建Lookup
```
git clone git://github.com/dbpedia/lookup.git
cd lookup
mvn clean install
```

###### 3.2下载并配置index
index获取页面:http://downloads.dbpedia-spotlight.org/dbpedia_lookup/

###### 3.3 运行服务
`./run Server [PATH TO THE INDEX]/[VERSION]/`
例如
`./run Server /opt/dbpedia-lookup/2015-04`

**注意：索引文件必须解压缩**
**现在允许的版本：见github页面**
**允许的语言：英语**

##### 4.重建索引
如果你想运行一个本地镜像，则你可以下载上边提到的预编译的索引。重建索引你需要：
- DBpedia 数据集；
- [Wikistatsextractor output](http://downloads.dbpedia-spotlight.org/) - [wikistatsextractor](https://github.com/jodaiber/wikistatsextractor)是 [pignlproc](https://github.com/dbpedia-spotlight/pignlproc)的替换选项；
- Unix

###### 4.1 获取数据集：
from：http://downloads.dbpedia.org/2015-10/core-i18n/en/

- redirects_en.nt (or .ttl)
- short_abstracts_en.nt (or .ttl)
- instance_types_en.nt (or .ttl)
- article_categories_en.nt (or .ttl)

from http://downloads.dbpedia.org/2015-10/core

- instance_types_en.ttl
- instance_types_sdtyped_dbo_en.ttl
- instance_types_transitive_en.ttl

###### 4.2 连接所有数据，并且通过URI排序
这一步非常重要，应为排序后的索引非常快：
```
  cat instance_types_en.nt (or .ttl)  \
      short_abstracts_en.nt (or .ttl) \
      article_categories_en.nt (or .ttl) \
      instance_types_en.ttl  \
      instance_types_sdtyped_dbo_en.ttl \
      instance_types_transitive_en.ttl | sort >all_dbpedia_data.nt (or .ttl)

```

###### 4.3 获取数据集redirects_en.nt (or .ttl)
重定向数据集不会被索引，他们作为lookup的目标被排除；

###### 4.4运行索引器Indexer
Indexer必须运行两次：
1. 有DBpedia数据
` ./run Indexer lookup_index_dir redirects_en.nt (or .ttl) all_dbpedia_data.nt (or .ttl)`
2. 有wikistatsextractor数据
 `./run Indexer lookup_index_dir redirects_en.nt (or .ttl) pairCounts`