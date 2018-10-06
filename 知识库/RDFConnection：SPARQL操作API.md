##### 1.引言
`RDFConnection`提供了基于RDF数据的SPARQL操作。主要操作有三种：查询、更新和sparql图存储。对于url和本地数据，RDFConnection提供的接口是同意的，都是用了HTTP和SPARQL协议。

###### 两个示例：连接到远程数据.未运行
```java
//connection代表连接数据集。参数类型为String，则代表一个url；参数类型也可以Dataset
try ( RDFConnection conn = RDFConnectionFactory.connect(...) ) {
//加载rdf文件到数据集(conn连接的)的默认图，
    conn.load("data.ttl") ;
    conn.querySelect("SELECT DISTINCT ?s { ?s ?p ?o }", (qs)->
       Resource subject = qs.getResource("s") ;
       System.out.println("Subject: "+subject) ;
    }) ;
}
```
```
RDFConnection conn = RDFConnectionFactory.connect(...)
conn.load("data.ttl") ;
QueryExecution qExec = conn.query("SELECT DISTINCT ?s { ?s ?p ?o }") ;
ResultSet rs = qExec.execSelect() ;
while(rs.hasNext()) {
    QuerySolution qs = rs.next() ;
    Resource subject = qs.getResource("s") ;
    System.out.println("Subject: "+subject) ;
}
qExec.close() ;
conn.close() ;
```
- 

##### 2.事务
使用事务是一种更好的工作方式，由`Txn`类提供，示例如下：
- ==事务暗含在使用的代码中==
```
try (RDFConnection conn = RDFConnectionFactory.connect("") ) {
            Txn.executeWrite(conn, ()-> {
                conn.load("data1.ttl") ;
                conn.load("data2.ttl") ;
                //下局查询用到了事务
                conn.querySelect("SELECT DISTINCT ?s { ?s ?p ?o }",
                        (qs)-> {
                            Resource subject = qs.getResource("s");
                            System.out.println("Subject: " + subject);
                        }
                 );
            });
         }catch (Exception e){
            e.printStackTrace();
        }
```
- ==也可以使用传统的方式，明确使用提开始、提交等函数==`begin()、commit()`,函数`abort()`防止意外:
```
try ( RDFConnection conn = RDFConnectionFactory.connect(...) ) {
    conn.begin(ReadWrite.WRITE) ;
    try {
        conn.load("data1.ttl") ;
        conn.load("data2.ttl") ;
        conn.querySelect("SELECT DISTINCT ?s { ?s ?p ?o }", (qs)->
           Resource subject = qs.getResource("s") ;
           System.out.println("Subject: "+subject) ;
        }) ;
        conn.commit() ;
    } finally { conn.end() ; }
}

```

##### 3.图存储协议GSP(graph store protocal)
[GSP](https://www.w3.org/TR/sparql11-http-rdf-update/)是工作在一个数据集的整个图上（一些列操作）.GSP提供了操作数据集中数据的标准方式。这些操作是==取得一个图，然后将RDF数据放进图中，添加更多的数据到图中，从数据集中删除一个图==。比如，加载两个文件：
```java
try ( RDFConnection conn = RDFConnectionFactory.connect(...) ) {
    conn.load("data1.ttl") ;
    conn.load("data2.nt") ;
  }
```
- 文件后缀名用于检测数据存储格式；

##### 4.远程和本地
>GSP operations work on while models and datasets. When used on a remote connection, the result of a GSP operation is a separate copy of the remote RDF data. When working with local connections, 3 isolations modes are available:

>Copy – the models and datasets returned are independent copies. Updates are made to the return copy only. This is most like a remote connection and is useful for testing.

>Read-only – the models and datasets are made read-only but any changes to the underlying RDF data by changes by another route will be visible. This provides a form of checking for large datasets when "copy" is impractical.

>None – the models and datasets are passed back with no additional wrappers and they can be updated with the changes being made the underlying dataset.

>The default for a local RDFConnection is "none". When used with TDB, accessing returned models must be done with transactions in this mode.

