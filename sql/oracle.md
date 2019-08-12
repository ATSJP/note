## oracle

### 常用

1复制表结构，建立新表

```
-- 建表结构
create table ins.acc_withdraw_info_20190315 as select * from ins.acc_withdraw_info where 1=2

-- 建表结构 数据
create table newtable as select * from oldtable

```
2、复制其他表的某个数据

```
insert into tab1 select * from tab2;
```

#### 高级函数

**1、语法格式：row_number() over(partition by 分组列 order by 排序列 desc)**

**link**:https://blog.csdn.net/qq_25221835/article/details/82762416

其中，row_number得到的是分组的序号

#### 恢复表到某个时间点

```sql
alter table <tableName> enable row movement;
flashback table <tableName> to timestamp to_timestamp('2019-01-02 10:05:00','yyyy-mm-dd HH24:MI:SS');
```
#### 开启事务

```
FOR UPDATE
```

### 坑

#### 时间格式

```sql
格式时间 yyyy-mm-dd hh24:mi:ss
```
#### oracle部分版本不支持 两层以上嵌套自查询（外两层的列，无法识别）
[老外怎么说](
https://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:1853075500346799932#185916940034636142)