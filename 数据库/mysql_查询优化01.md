[博文链接](http://blog.csdn.net/beirdu/article/details/76719063)

##### 1.索引注意事项
1. like左匹配模式可以走到索引，但是同><一样是range类型的索引，复合索引走到这里就会停止；
2. `in (a,b,c)`走的不是rang类型索引，而是“多值相等”，复合索引走到这里不会停止；但是要注意不要连续使用多个“多字相等”，诸如`in (a,b,...10个) in (1,2,3,4...10)`排列组合会有一百种；
3. 覆盖索引，即索引可以覆盖到所查询的字段，此时直接可以在索引中取值。“非覆盖索引”及时查询的字段数量一样，但也会比前者慢，因为需要回表取值，此时可以使用...
4. 延迟关联，即先根据查询条件查询数据**主键**id集合，然后根据id获取结果集；
```sql
explain
select oute.username,oute.sync_bbs 
from XXX oute
inner join
( 
select id 
from XXX inne
where inne.username in('女士','李娜','先生') and sex in('FEMALE','MALE')
)tem
on oute.id=tem.id;
```
5.索引应该单独放在比较符号的一侧，如果是表达式的一部分，则走不到索引：
```
where a=1; -- 可以走到索引
where a-1=0; -- 走不到索引；
```
6.选择性低的字段，如果每次查询都用到，则应该作为符合索引的第一项，比如性别、通讯工具是否在线等等；
7. 等值传播
```
explain extended
select *
from xxxlows_rfnd fr
inner join xxxetails od
on fr.ord_id=od.ord_id 
where  od.ord_id=2819460;

show warnings;
-- 重构的查询
/* select#1 */
...
where ((`od`.`ord_id` = 2819460) and (`ent_portal`.`fr`.`ord_id` = 2819460))
```
##### 2.相关操作
1. 使用`explain`可以查看查询计划；
2. 使用`explain extended`+`show warnings`可以查看重构后的查询；
3. drap table table_name,delete from tab_name where con_XXX,truncate table tab_name的区别
```
1.drop table tab_name:删除表，后两者为删除表数据；
2.delete from tab_name where con_XXX:根据条件删除表中记录；auto-incrementing keys不会重置；执行后会计算影响的记录数，也因此比truncate table tab_name 慢；可以使用事务回滚操作；仅仅是行锁
3.truncate table tab_name:清空表，重置auto-incrementing key字段为1，保留索引；

```