[sparql规范](https://www.w3.org/TR/sparql11-query/#groupby)

[Apache Jena执行sparql查询](http://blog.csdn.net/beirdu/article/details/78927437)



#### 一.基本介绍

##### 1.1 实体属性

实体属性分为3中：对象属性、数据属性、功能属性。还有总的属性**Property**。

```
select DISTINCT(?props) AS ?P
where{
 ?sub <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://www.w3.org/1999/02/22-rdf-syntax-ns#Property> .
 ?sub <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> ?props
} 

http://www.w3.org/1999/02/22-rdf-syntax-ns#Property
http://www.w3.org/2002/07/owl#ObjectProperty
http://www.w3.org/2002/07/owl#FunctionalProperty
http://www.w3.org/2002/07/owl#DatatypeProperty
```
###### 1）数据属性
年龄、性别和出生日期都是数据属性，但是类型分别是整型、String和date。部分数据属性值以`^^`链接其类型。示例

```
//海明威数据属性的键值对

select ?pred ?value
where{
 <http://dbpedia.org/resource/Ernest_Hemingway> ?pred ?value .
 ?pred <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://www.w3.org/2002/07/owl#DatatypeProperty> .
}

output：
http://dbpedia.org/ontology/deathDate	
"1961-7-2"^^<http://www.w3.org/2001/XMLSchema#date>
```
- 数据属性取值范围一般是<https://www.w3.org/2001/XMLSchema#date/string/integer>
- 
###### 2）对象属性
对象属性会有其取值范围是对象，比如某人的老婆一定是人；

```
select ?pred,?range
where{
 ?pred <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://www.w3.org/2002/07/owl#ObjectProperty> .
 ?pred <http://www.w3.org/2000/01/rdf-schema#range> ?range .
} limit 100


output:
http://dbpedia.org/ontology/birthPlace	http://dbpedia.org/ontology/Place
http://dbpedia.org/ontology/thumbnail	http://dbpedia.org/ontology/Image
http://dbpedia.org/ontology/academicAdvisor	http://dbpedia.org/ontology/Person

```


#### 二.关键字使用

关键字大小写通用，主要介绍：`optional、

##### 2.1 OPTIONAL 可选过滤
表示条件是可选的，可以使用多个OPTIONAL表示多个{}中的条件都是可选的
```
//如下，如果?sub有child属性，则罗列出其属性值，没有的话就空出来

select ?sub ?value
where{
 ?sub <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://dbpedia.org/ontology/Person> .
 optional{?sub <http://dbpedia.org/ontology/child>  ?value }.
 optional{...}
}

output：
...
http://dbpedia.org/resource/Junichirō_Koizumi	http://dbpedia.org/resource/Shinjirō_Koizumi
http://dbpedia.org/resource/Junichirō_Koizumi	http://dbpedia.org/resource/Kotaro_Koizumi
http://dbpedia.org/resource/Andreas_Ekberg	
http://dbpedia.org/resource/Danilo_Tognon
...

```


##### 2.2 order by 可选过滤
默认升序，使用DESC(?var1)可以在指定字段使用降序

```
select ?birthDate
where{
 ?sub <http://dbpedia.org/ontology/birthDate> ?birthDate .
 FILTER REGEX(?birthDate,"^19") 
} order by ?birthDate 

output:
"1901-6-13"^^<http://www.w3.org/2001/XMLSchema#date>
"1901-6-14"^^<http://www.w3.org/2001/XMLSchema#date>
```
不同的字段使用不同的方式排序
```

select ?sub,?birthDate,?deadDate
where{
 ?sub <http://dbpedia.org/ontology/birthDate> ?birthDate .
 ?sub <http://dbpedia.org/ontology/deathDate> ?deadDate .
 FILTER ( REGEX(?birthDate,"^19") && regex(?deadDate,"^19")).
} order by ?birthDate,DESC(?deadDate)

output:
http://dbpedia.org/resource/Grace_Hartman_(politician)	
"1900-0-0"^^<http://www.w3.org/2001/XMLSchema#date>
1998-05-23
http://dbpedia.org/resource/Ho_Sin_Hang	
"1900-0-0"^^<http://www.w3.org/2001/XMLSchema#date>
1997-12-04
http://dbpedia.org/resource/Fateh_Chand_Badhwar	
"1900-0-0"^^<http://www.w3.org/2001/XMLSchema#date>
1995-10-10
```

##### 2.3 筛选FILTER、HAVING
FILTER表示对结果集进行筛选，可以放在OPTIONAL子句中针对可选子句的结果集进行筛选。

###### 1) 直接使用>,<,=,!=进行筛选，符号两侧可以是常量或者变量

```
select ?sub,?age
where{
 ?sub <http://dbpedia.org/ontology/birthDate> ?age .
 filter (?age!='1861-7-2').
} limit 100
```

###### 2)使用正则表达式 FILTER REGEX(?变量,"regex","param")

[sparql可使用正则语法](https://www.w3.org/TR/xpath-functions/#regex-syntax)。

```
//类型值含有person，且不区分大小写

select DISTINCT(?class) as ?class
where{
 ?sub <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> ?class 
 FILTER REGEX(?class,"person","i")
} 

output：

http://xmlns.com/foaf/0.1/Person
http://dbpedia.org/ontology/Person
```

3）HAVING 对最后的结果集进行筛选

跟group by一块使用，注意筛选的时候不能用别名。

```
//出生人数超过1000人的时间及出生人数统计

select ?birth,(COUNT(?birth) AS ?cou)
where{
 ?sub <http://dbpedia.org/ontology/birthDate> ?birth 
} 
group by ?birth 
having (COUNT(?birth)>1000) //fixme:不能使用 having (?cou>1000)


output:
...
"1971-1-1"^^<http://www.w3.org/2001/XMLSchema#date>
1803
"1968-1-1"^^<http://www.w3.org/2001/XMLSchema#date>
1361
"1940-1-1"^^<http://www.w3.org/2001/XMLSchema#date>
1234
"1947-1-1"^^<http://www.w3.org/2001/XMLSchema#date>
1722
"1958-1-1"^^<http://www.w3.org/2001/XMLSchema#date>
1644
"1959-1-1"^^<http://www.w3.org/2001/XMLSchema#date>
1709
...

```



##### 四.常用函数

##### 1. 字符串
字符串的长度`strlen(str)`、截取字符串substr("str",start_index,length/可选)，字符出现的位置

###### 字符串长度
```
//长度小于14的名字
select ?name,(strlen(?name) as ?nameLength)
where{
 ?sub <http://xmlns.com/foaf/0.1/givenName> ?name 
 filter (strlen(?name)<15)
} order by DESC(?nameLength)

"Clarence Brown"@en 14 
"Brothers Grimm"@en 14
"Samuel Johnson"@en 14

```
###### 截取字符串substr(datasource,start_index,length)
还不知道怎么查询某个字母第一次出现在字符串中的位置，todo

- substr(datasource,start_index,length):length可以省略，表示截取从开始索引到最后的位置；
- 有些?var不是string类型的，需要用str(?var)转化一下

```
//作家生辰

select (substr(str(?bir),1,4) AS ?birYear),(COUNT(?writer) as ?cou)
where{
    ?writer <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://dbpedia.org/ontology/Writer> .
    ?writer <http://dbpedia.org/ontology/birthDate> ?bir .
}group by substr(str(?bir),1,4)
having (COUNT(?writer)>1)
order by desc(COUNT(?writer))


output:
birYear	cou
1947 586
1948 585
1951 577
```



###### group by+聚合字符串：group_concat(?sub ; separator=",")

用某个字段分组并且希望知道合并的记录数以及将某个字段所有记录合并为一个字段

```
//以首都分组，查询首都对应的国家

select ?capital,(COUNT(?capital) as ?cou),(group_concat(?country ; separator=",") as ?countries)
where{
?country <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://dbpedia.org/ontology/Country> .
?country <http://dbpedia.org/ontology/capital> ?capital .
}group by ?capital
having (COUNT(?capital)>1)

output：
http://dbpedia.org/resource/Dresden	
2
http://dbpedia.org/resource/Kingdom_of_Saxony,http://dbpedia.org/resource/Electorate_of_Saxony
http://dbpedia.org/resource/Beirut	
2
http://dbpedia.org/resource/Lebanon,http://dbpedia.org/resource/Greater_Lebanon

```
