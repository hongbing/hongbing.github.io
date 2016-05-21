---
layout: post
title: 工作中常用的linux命令
description: 记录工作中常用的linux命令
categories: posts blog
---

本文记录了在工作中常用的linux命令<!-- more -->。

+ 查找某个端口运行的程序

```
lsof -i:[port]

netstat -antp | grep [port]
```
注意使用上述两个命令都需要root权限。

+ 创建软链

```
ln -s target_file_name symblic_file_name
```
target_file_name 为已经存在的文件名，symblic_file_name为需要创建的软链文件名。该命令会在当前目录创建一个名为symblic_file_name的软链文件，链接到target_file_name上。

+ 解除软链

```
unlink mySymblink
```

+ sar命令查看系统状况

```
// 查看磁盘io
sar -d 1 10

// 查看tcp状况
sar -n TCP,ETCP 1 10

// 查看设备读写速率
sar -n DEV 1 10
```

+ 查看cpu架构

```
lscpu
```

+ 查看系统版本,linux版本号

```
uname -a

uname -r

cat /proc/version

lsb_release -a
```

+ cpu绑定

```
taskset -pc [cid] [pid]
```

cid表示需要绑定的cpu id号，pid为进程号。参数没有cid表示查看当前pid的cpu绑定情况。通过
`cat /proc/cpuinfo | grep processor`查看当前的cpu核数.

 绑定过后，使用`pidstat 2 | grep [pid]`查看绑定的进程是否跑在绑定的那个核上。
 
+ 查看某一个进程占用的内存大小：

```
top -d 1 -p [pid]
```

top查看到的RES或者RSS是进程的实际内存，VSZ或者VIRT是虚拟内存（表示当前没有用到，可能会用到的内存大小）,然而，RES的内存大小还包括了共享内存的大小，使用`pmap -d [pid]`可以在最后一行查看到实际内存与共享内存的大小。

与top命令相关的：

```
top，shift + p // 各进程按CPU从高到低排序
top，shift + h // 查看各线程占用CPU情况
top，shift + m // 各进程按内存使用（RES）大小排序显示
```

+ 查看当前目录下所占存储最大的10个文件：

```
du -hsx * | sort -rh | head -10
```

+ 查看历史命令：

```
ctrl + r

history | grep "mysql"
```

使用CTRL+p和CTRL+n还可以向前和向后查找上一条或者下一条命令。

+ 查看某一个文件中的字符串出现的次数

``` 
grep -o 'helolo' info.txt | wc -l
```

+ 针对gz的文件，使用上面的命令无效，需要使用zcat

```
zcat info.gz | grep -o 'helolo' | wc -l
```

+ 查看系统连接状况

```
ss -s
```

+  TCP 连接状态数量统计

```
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'

netstat -ant | grep ':8082' | awk '{print $6}' | sort | uniq -c

cat /proc/net/sockstat`
```

+ 排序(对out.txt的第二列排序，-n表示对数字排序，-f表示大小写不敏感)

```
sort -n -k2 out.txt
```

+ 对第二列排序，并将第二列相同的整行输出

```bash
sort -k2 sort.log | awk '{
    n=a[$2]++;
    if (n==0) 
	v=$0;
    else {
	if (n==1) 
	    print v;
	print $0
    }
}'
```

+ 将所有数据转换成一整行

```
awk '{printf("%s ",$i)}' filename
```

+ 矩阵转置

```bash
# 使用数组的方式
awk '{
    for(i=1;i<=NF;i++){
	a[FNR,i]=$i
    }
}
END{
    for(i=1;i<=NF;i++){
	for(j=1;j<=FNR;j++){
	    printf a[j,i]" "
	}
	    print ""
    }
}' file.text

# 将相同的NF串联起来的方式
awk '{
    for (i = 1; i <= NF; ++i) {
        if (NR == 1)  s[i] = $i;
        else s[i] = s[i] " " $i;
    }
} END {
    for (i = 1; s[i] != ""; ++i) {
        print s[i];
    }
}' file.text
```

+ 按CPU利用率从大到小排列

```
ps -e -o "%C : %p : %z : %a"|sort -nr
```

+ 查看进程，按内存从大到小排列

```
ps -e -o "%C : %p : %z : %a"|sort -k5 -nr
```

+ 查看系统cpu物理个数，核数以及逻辑个数

```
cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l;cat /proc/cpuinfo| grep "cpu cores"| uniq;cat /proc/cpuinfo| grep "processor"| wc -l
```

+ 文件拷贝

1.将本地文件拷贝到远程机器上

```
scp 本地用户名@IP地址:文件名  远程用户名@IP地址 : 文件夹名
```

例：scp  info.log  root@10.210.230.46:/data7/jetty8950/logs
在本地机器上可以不用写“本地用户名@IP地址”，直接写文件名就可以。

2.将远程机器上的文件拷贝到本地

```
scp  远程用户名@IP地址：文件  本地文件夹

可能有用的几个参数 :
    -v 和大多数linux命令中的-v意思一样, 用来显示进度,可以用来查看连接, 认证, 或是配置错误
  　-C 使能压缩选项
  　-P 选择端口 注意 -p 已经被 rcp 使用
　　-4 强行使用 IPV4 地址
　　-6 强行使用 IPV6 地址
　　-r Recursively copy entire directories
```

+ 找到某进程打开的所有文件

```
lsof -c $processname | egrep 'w.+REG' | awk '{print $9}' | sort | uniq
```

+ 找到系统中已经删除的文件

```
lsof -n | grep deleted
```

在执行`df -h`查看磁盘空间的时候，看见磁盘空间已满，但是通过`du -sh *`查看目录下的文件，却没有大的文件。这时可以通过lsof命令查看是否存在已经删除了（在磁盘上已经看不到文件的存在了），但是仍然被某进程打开着的文件，找到打开该文件的进程号，kill掉，就可以释放空间了。

![linux-command](/images/linuxcommand/linux-command.png)

