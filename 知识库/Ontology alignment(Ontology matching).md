[维基链接：只有计算机科学方面翻译](https://en.wikipedia.org/wiki/Ontology_alignment)

##### 概述
>Ontology alignment, or ontology matching, is the process of determining correspondences between concepts in ontologies. A set of correspondences is also called an alignment. The phrase takes on a slightly different meaning, in computer science, cognitive science or philosophy.

本体对齐或本体匹配是确定本体中概念(concepts)之间对应关系的过程。 是“一致性”的集合，也被成为对齐。 这个短语在计算机科学，认知科学或哲学中的含义略有不同。

##### 计算机科学
>For computer scientists, concepts are expressed as labels for data. Historically, the need for ontology alignment arose out of the need to integrate heterogeneous databases, ones developed independently and thus each having their own data vocabulary. In the Semantic Web context involving many actors providing their own ontologies, ontology matching has taken a critical place for helping heterogeneous resources to interoperate. Ontology alignment tools find classes of data that are "semantically equivalent," for example, "Truck" and "Lorry." The classes are not necessarily logically identical. According to Euzenat and Shvaiko (2007),[1] there are three major dimensions for similarity: syntactic, external, and semantic. Coincidentally, they roughly correspond to the dimensions identified by Cognitive Scientists below. A number of tools and frameworks have been developed for aligning ontologies, some with inspiration from Cognitive Science and some independently.

对于计算机科学家来说，概念(concepts)被表达为数据标签(labels of data)。 从历史上看，本体对齐的需求是出于集成异构数据的需要—他们独立开发并拥有自己的数据词汇表。 在涉及许多行为体的提供他们自己的本体的语义网络上下文中，本体匹配已经成为帮助异构数据进行交互操作的关键因素。 本体对齐工具可以找到语义上相同的数据类，例如“卡车”和“货车”。 这些类不一定是逻辑上不一定一致。 根据Euzenat和Shvaiko（2007），相似度有三个主要维度：语法，外部(external)和语义。 巧合的是，它们大致对应于认知科学家所确定的维度。 已经开发了一些工具和框架来对齐本体，其中一些工具和框架来自Cognitive Science(认知科学）和一些独立的灵感。

>Ontology alignment tools have generally been developed to operate on database schemas,[2] XML schemas,[3] taxonomies,[4] formal languages, entity-relationship models,[5] dictionaries, and other label frameworks. They are usually converted to a graph representation before being matched. Since the emergence of the Semantic Web, such graphs can be represented in the Resource Description Framework line of languages by triples of the form <subject, predicate, object>, as illustrated in the Notation 3 syntax. In this context, aligning ontologies is sometimes referred to as "ontology matching".

本体对齐工具通常被开发来在以下形式中进行数据操作：操作数据库模式database schemas，XML schemas，分类法taxonomies，formal languages，实体关系模型，字典和其他标签框架。 它们通常在匹配之前转换为图形表示graph representation。 语义网出现以后，这些图可以在资源描述框架语言用<subject, predicate, object>形式的三元组来表示，如Notation3（N3） 语法所示。 在这种情况下，对齐本体有时被称为“本体匹配”。

```
//n3语法

<rdf:RDF
    xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#"
    xmlns:dc="http://purl.org/dc/elements/1.1/">
  <rdf:Description rdf:about="http://en.wikipedia.org/wiki/Tony_Benn">
    <dc:title>Tony Benn</dc:title>
    <dc:publisher>Wikipedia</dc:publisher>
  </rdf:Description>
</rdf:RDF>
```

>The problem of Ontology Alignment has been tackled recently by trying to compute matching first and mapping (based on the matching) in an automatic fashion. Systems like DSSim, X-SOM[6] or COMA++ obtained at the moment very high precision and recall.[3] The Ontology Alignment Evaluation Initiative aims to evaluate, compare and improve the different approaches.

最近通过尝试以自动方式automatic fashion计算 匹配第一和映射（基于匹配）/matching first and mapping (based on the matching) 来解决本体对齐问题。 像DSSim，X-SOM或COMA ++等系统在此刻获得了非常高的精度和召回率。 本体对齐评估旨在评估、比较和改进不同的方法。

>More recently, a technique useful to minimize the effort in mapping validation and visualization has been presented which is based on Minimal Mappings. Minimal mappings are high quality mappings such that i) all the other mappings can be computed from them in time linear in the size of the input graphs, and ii) none of them can be dropped without losing property i).

最近，提出了一种有助于最小化映射验证和可视化工作量工作的技术，该技术基于最小映射（Minimal Mappings）。最小映射是高质量映射，所有其他映射可以根据输入图的大小按时间线性计算，并且它们在没有丢失属性时都不会被丢掉。

###### 形式化定义 formal definition
![image](https://wx4.sinaimg.cn/mw1024/006Xp67Kly1focq8jkbxyj318n0hodjv.jpg)

**给定两个本体i=<C<sub>i</sub>,R<sub>i</sub>,I<sub>i</sub>,A<sub>i</sub>>和j=<C<sub>j</sub>,R<sub>j</sub>,I<sub>j</sub>,A<sub>j</sub>>,我们能够在他们的数据项(term)之间定义不同类型的关系。这样的关系被称为对齐，并且可以按照不同的维度分类**：
- 相似和逻辑：这是在匹配（关于本体数据项相似度的预测）和映射（逻辑公理，通常表达逻辑等价或包含）；
- 原子与复杂度：考虑的对齐是否是一对一的，或者是否可以在“类查询公式a query-like formulation ”中设计更多的术语，例如LAV / GAV映射；
- 同质和异质：同构还是异构：对齐只能预测同类型的术语，比如类只能关联类、实体只能关联实体，或者允许关系中的异构（类关联实体？）；
- 对齐类型：与对齐相关联的语义，可以是包含、等价、相交、部分，或者是任意用户指定的的关系。

包含、原子和同质是更加丰富对齐方式的基础，并且他们在描述逻辑中有一个明确的语义。现在让我们介绍更多正式的本体匹配和映射。

一个原子同构的匹配是一个相似度范围在`[0,1]`的对齐，它描述两个输入本体i和j的数据项的相似度。既能够用启发算法的方式计算匹配，也能够从其他的匹配中推断出来。

我们可以正式的为匹配和映射下定义：一个匹配是一个四元组m=<id,t<sub>i</sub>,t<sub>j</sub>,s>,t<sub>i</sub>、t<sub>j</sub>是同构的本体数据项，s是四元组m的相似度。一个映射为一个二元组u=<t<sub>i</sub>,t<sub>j</sub>>,t<sub>i</sub>,t<sub>j</sub>是同构的本体数据项。
