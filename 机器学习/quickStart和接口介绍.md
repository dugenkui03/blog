链路预测可以用于预测使用那些模型。

#### 1.quickStart

##### 1.1 `training`的QuickStart
为了完成知识图谱嵌入knowledge graph embedding需要四个步骤：
1. 导入数据集；
2. 设置training的配置参数；
3. 训练模型；
4. 导出结果；
5. 可选：测试模型、验证训练结果。

```python
import config # 训练集配置
import models # 各种嵌入模型
import tensorflow as tf
import numpy as np #支持高级大量的维度数组与矩阵运算，此外也针对数组运算提供大量的数学函数库，是大量机器学习框架的基础库。
import os

'''
使用0号GUP：
    如果机器中有多块GPU，tensorflow会默认吃掉所有能用的显存;
     如果实验室多人公用一台服务器，希望指定使用特定某块GPU。
'''
os.environ['CUDA_VISIBLE_DEVICES']='0'


'''
1. 导入数据.Input test files from the same folder.
'''
#Validation and test files 用来验证训练结果，但他们不是必须的
con = config.Config()
con.set_in_path("./benchmarks/FB15K/")

'''
2. 设置训练参数
'''
con.set_test_link_prediction(True)
con.set_test_triple_classification(True)
con.set_work_threads(8) #分配几个线程来对正面样本和负面样本采样
con.set_train_times(10) #训练时间
con.set_nbatches(100) # n批
con.set_alpha(0.001)
con.set_margin(1.0)
con.set_bern(0)
con.set_dimension(100)
con.set_ent_neg_rate(1)
con.set_rel_neg_rate(0)
con.set_opt_method("SGD")

'''
3. 设置输出文件
'''
#Models will be exported via tf.Saver() automatically.fixme 设置模型输出文件
con.set_export_files("./res/model.vec.tf", 0)
#Model parameters will be exported to json files automatically. fixme 模型配置参数输出文件
con.set_out_files("./res/embedding.vec.json")


'''
4. 训练模型
'''
#Initialize experimental settings. fixme 初始化实验设置
con.init()
#Set the knowledge embedding model fixme 设置使用哪个模型
con.set_model(models.TransE)
#Train the model. fixme 训练模型
con.run()

'''
5. 训练模型后测试模型，需要
    1."set_test_flag(True)"；
    2.有相应的Validation and test 文件
'''
#To test models after training needs "set_test_flag(True)".
con.test()
con.show_link_prediction(2,1)
con.show_triple_classification(2,1,3)
```


##### 1.2 Testing的QuickStart
链路预测`Link prediction`、三元组分类`triple classification`和实现。

###### (1)链路预测`Link prediction`
链路预测致力于预测三元组(h,r,t)中缺失的头尾实例。在链路预测这项任务中，系统都是选出一批排序的实体集合，而非选出最好的一个。对于每个三元组，我们使用知识图谱中的实体替换头尾实体，并且按照得分函数fr的相似度分数对这些实体进行降序排序。我们使用两种措施作为我们的评估标准：

1. MR : mean rank of correct entities正确实体的平均排序；
2. MRR：the average of the reciprocal(倒数) ranks of correct entities正确实体的倒数平均数；
3. Hit@N : proportion of correct entities in top-N ranked entities。top-n排序实体中的正确实体比例。


>Link prediction aims to predict the missing h or t for a relation fact triple (h, r, t)。In this task, for each position of missing entity, the system is asked to rank a set of candidate entities from the knowledge graph, instead of only giving one best result.

>For each test triple (h, r, t), we replace the head/tail entity by all entities in the knowledge graph, and rank these entities in descending order of similarity scores calculated by score function fr.

###### (2)三元组分类triple classification
三元组分类意在评估一个给定的三元组是否是正确的。这是二元分类任务。对于此任务我们设置了一个关系相异阈值`δr`。如果分数小于阈值`δr`则表示正面样本，大于此值则为负面样本。阈值`δr`是通过在验证集上最大化分类精度得到的。

###### (3)测试模型

为了评估模型需要导入数据集并设置必要的配置参数，然后设置模型参数并且测试模型。示例如example_test_transe.py 来测试transE:

有四种方法可以测试模型。这里重点讲`tensorFlow`中保存模型和获取模型的方法。
```python
//todo
```

##### 1.3 获取嵌入矩阵`embedding matrix`

四种方法获取嵌入矩阵。

- 设置import files则OpenKE会自动通过`tf.Saver()`导入模型。
```python
con=config.Config()
con.set_in_path("./benchmarks/FB15K/")
con.set_test_link_prediction(True)
con.set_test_triple_classification(True)
con.set_work_thread(4)
con.set_dimension(50)
con.set_import_files("./res/model.vec.tf")

con.init()
con.set_model(models.TransE)

#Get the embeddings(numpy.array)
embeddings=con.get_parameters("numpy")

#Get the embeddings (python list)
embeddings=con.get_parameters();
```
- 从json文件中读取模型参数，并且手动manually导入参数。
```python
con=config.Config()
con.set_in_path("./benchamarks/FB15K/")
...

con.init()
con.set_model(models.TransE)

f=open("./res/embedding.vec.json","r")
embeddings=json.loads(f.read())
f.close()
```
- 通过 `tf.Saver()`手动的导入模型
- 训练完模型之后立即获取模型.
```python
...
...

#模型将会通过 tf.Saver()自动导出
con.set_export_files("./res/model.vec.tf",0)

# 模型参数将会自动导出为json文件
con.set_out_files("./res/embedding.vec.json")

#初始化实验设置
con.init()

#设置知识嵌入模型 knowledge embedding model
con.set_model(models.TransE)

#训练模型
con.run()

# 获取嵌入矩阵embeddings(numpy.array)
embeddings=con.get_parameters("numpy")

# 获取嵌入矩阵python list
embeddings=con.getParameters();
```


#### 二.接口`interfaces`

配置，模型
##### [参考链接](https://github.com/dugenkui03/OpenKE#interfaces)