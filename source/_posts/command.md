---
title: Linux命令备忘
date: 2016-03-01 17:18:06
author: linxianlong
tag: [Linux,Shell]
category: Linux
---

### 查看linux core文件位置.

```
	/proc/sys/kernel/core_pattern
```

### shell字符串的处

```shell
${varible##*string} 从左向右截取最后一个string后的字符串
${varible#*string}从左向右截取第一个string后的字符串
${varible%%string*}从右向左截取最后一个string后的字符串
${varible%string*}从右向左截取第一个string后的字符串
```

### shell 时间

```shell
#日期格式
date -d "20151120 -2 days" +"%Y%m%d"
#日期转时间戳
date -d "20151027" +%s
#时间戳转时间
date -d "1970-01-01 1357004952 sec UTC" +"%Y%m%d"
```

### 从一个日志字段中筛选去两个字段的值

```php
WARNING: 01-12 19:36:50:  ral-worker * 30449 [conf/manager/helper/pool.cpp:241][logid=2210608107 worker_id=140610281404160 optime=1452598610.631276 msg=service+not+found+in+share+memory service=Redis_doc_push]
```
日志如上，我想从这段日志中筛选出service字段和logid字段，`Redis_doc_push  2210608107`
#### 方法1.

```sh
 cat xxx.log | sed -r "s/.*?logid=([0-9]+).*?service=(\w+).*?/\1,\2/"
```
#### 方法2.

```sh
cat xxx.log |  grep -oP "logid=\d+.*service=\w+" | awk -F' ' '{print $1,$NF}'
```

