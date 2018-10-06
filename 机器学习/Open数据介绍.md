##### 1.数据集介绍


==使用自己数据集是需要按照以下方式格式化：==

###### 训练数据集包含在以下三个文件：
```
entity2id.txt:实例到id的映射；

relation2id.txt：关系到id的映射；

train2id.txt：训练文件，第一行是训练三元组的行数。随后的三元组为 (e1, e2, rel)形式，其中rel为实例e1、e2的关系。注意这些数据的id包含在以上所述的两个文件中
```

>train2id.txt: training file, the first line is the number of triples for training. Then the following lines are all in the format (e1, e2, rel) which indicates there is a relation rel between e1 and e2 . Note that train2id.txt contains ids from entitiy2id.txt and relation2id.txt instead of the names of the entities and relations. If you use your own datasets, please check the format of your training file. Files in the wrong format may cause segmentation fault.

>entity2id.txt: all entities and corresponding ids, one per line. The first line is the number of entities.

>relation2id.txt: all relations and corresponding ids, one per line. The first line is the number of relations.


###### 测试数据集包含在以下两个文件

```
test2id.txt：测试文件。第一行是三元组的总行数，其他数据遵循此格式 (e1, e2, rel);

valid2id.txt：验证文件。格式同上。

//You can get this file through n-n.py in folder benchmarks/FB15K.
type_constrain.txt：类型限制文件。第一行是关系数量，其他数据是每个关系的类型限制。如下数据表示id为1200的关系有四个头实例分别是...，四个尾实例分别是...。

1200	4	3123	1034	58	5733
1200	4	12123	4388	11087	11088
```
>For testing, datasets contain additional two files (totally five files):

>test2id.txt: testing file, the first line is the number of triples for testing. Then the following lines are all in the format (e1, e2, rel) .

>valid2id.txt: validating file, the first line is the number of triples for validating. Then the following lines are all in the format (e1, e2, rel) .

>type_constrain.txt: type constraining file, the first line is the number of relations. Then the following lines are type constraints for each relation. For example, the relation with id 1200 has 4 types of head entities, which are 3123, 1034, 58 and 5733. The relation with id 1200 has 4 types of tail entities, which are 12123, 4388, 11087 and 11088. You can get this file through n-n.py in folder benchmarks/FB15K.

