数学成绩最高的同学信息

```
select * from 学生表 where 学号 in
(select top 1 学号 from 成绩表 where 课程号 in 
(select 课程号 from 课程表 where 课程名='数学') 
order by 分数 desc)
```

日志选取最后5个用户信息按ID排序

```

```

各城市人数大于2的信息

```

```

10个相同邮箱，重复只留一个

```

```

name字段中所有张三换为李四

```

```

成绩前五名

```
```

年龄第二大的员工信息

```
```

