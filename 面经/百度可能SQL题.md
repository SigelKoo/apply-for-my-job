数学成绩最高的同学信息

```
select * 
from 学生表 
where 学号 in
(
select top 1 学号 
from 成绩表 
where 课程号 in 
(select 课程号 from 课程表 where 课程名='数学') 
order by 分数 desc
)
```

日志选取最后5个用户信息按ID排序

```
select * from table order by ID desc limit 5; 
```

各城市人数大于2的信息

```
?不懂表结构不知到要干嘛
```

10个相同邮箱，重复只留一个

```
select distinct 邮箱 from table;
```

name字段中所有张三换为李四

```
select
	case when name = '张三' then '李四'
	else name
from
	table
;
```

成绩前五名

```
select
	*
from
	table
order by 成绩 desc
limit 5
;
```

年龄第二大的员工信息

```
select
	*
from 
	table 
order by 年龄 desc 
limit 1 
offset 1
;
```

