---
title: Shell从一段中筛选去两个字段的值
date: 2016-03-03 20:15:48
tags: [Shell]
category: Linux
---

```php
WARNING: 01-12 19:36:50:  ral-worker * 30449 [conf/manager/helper/pool.cpp:241][logid=2210608107 worker_id=140610281404160 optime=1452598610.631276 msg=service+not+found+in+share+memory service=Redis_doc_push]
```
日志如上，我想从这段日志中筛选出service字段和logid字段，`Redis_doc_push  2210608107`

#### 方法1.
通过awk获取指定的列后输出，缺点是如果日志打印的格式不规范（如没有用空格分割），则无法使用。

```sh
cat xxx.log |  grep -oP "logid=\d+.*service=\w+" | awk -F' ' '{print $1,$NF}'
```
#### 方法2.
用正则表达式捕获组，用sed将整行替换为两个组的值

```sh
 cat xxx.log | sed -r "s/.*?logid=([0-9]+).*?service=(\w+).*?/\1,\2/"
```

