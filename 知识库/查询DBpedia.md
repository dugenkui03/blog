网址：http://dbpedia.org/sparql

注意事项：
1. 属性路径都是`xxx/ontology/yyy`
2. `xxx/page/yyy`和`xxx/resource/yyy`会相互转换，但是将后者作为查询sql属性值
3. 


示例：
```
SELECT ?e ?p ?s
  WHERE
    {
      ?e <http://dbpedia.org/ontology/award>         <http://dbpedia.org/resource/Pulitzer_Prize_for_Fiction>  .

      ?e <http://dbpedia.org/ontology/award>  <http://dbpedia.org/resource/Nobel_Prize_in_Literature> .

            ?e       <http://dbpedia.org/ontology/birthPlace> <http://dbpedia.org/resource/Oak_Park,_Illinois> .
      
      ?e ?p ?s .

    }
```