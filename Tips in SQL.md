# SQL中的一些内容

**约束相关**

1. 删除约束

   > ```sql
   > alter table tablename drop constraint constraintname
   > ```

2. 新建约束，并给字段设置默认值

   > ```sql
   > //添加字段
   > Alter table tableName add column_name type 
   > //设置默认值
   > alter table tablename add constraint constraintname default '默认值' for 字段名
   > ```


