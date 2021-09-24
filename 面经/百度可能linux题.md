sed修改test.txt第二行，第1个root为ROOT

```
sed -i '2s/root/ROOT/' passwd
```

![img](https://img2018.cnblogs.com/blog/1256425/201907/1256425-20190707221622962-2024227912.png)

文件绝对路径

```
pwd | awk '{print $1"/functions.sh"}'
```

查找某文件中关键字

```
grep '关键字' /etc/passwd
```

显示文件第100行

```
head -100 data.txt | tail -1
awk 'NR==100' data.txt
```

查看使用80的进程

```
lsof -i:22
```

进程部署在什么位置

```
cd /proc/22
ls -l
cwd就是要查找的进程所在路径
```

杀掉这些进程

```
kill -s 9 1191
```

磁盘使用情况

```
df -h
```

内存使用情况

```
top
```

杀掉aaa.sh进程

```
ps -aux | grep aaa.sh
kill -s 9 <pid>
```

创建a/b/c

```
mkdir -p a/b/c
```

查看文件大小

```
ls -lh /home
du -h /home/a.txt
```

查看文件夹大小

```
du -h --max-depth=0 /home
```

查看大文件

```
find . -type f -size +800M
```

```
grep  `world` copy.log | less
tail  -n 10000 | less
sed -n '500000000q;499999900,500000000p'  file # 其中 -n 与 p : 表示只打印符合条件的行
```

当前目录下java文件个数

```
find . -name '*.java' | wc -l
```

