---
layout:       post
title: MySQL数据/数据结构输出到本地
subtitle:     ""
date:         2021-02-23
author:       "ZhuXingda"
header-mask:  0.3
catalog:      true
multilingual: false
comments: true
tags:
  - Mysql
---
经常遇到需要将数据库导出成文件，然后在本地操作，总结常用方法有：
- 导出数据：利用mysql命令的-e参数
  mysql -h[hostname] -P[port] -u[username] -p[password] [dbname] -e"查询sql" > [file path]
  然后用sz命令将file.txt传回本地
  
- 导出数据/数据结构：利用mysqldump
    - 导出建表脚本（包括数据） mysqldump -h[hostname] -P[port] -u[username] -p[password] [dbname] < [table1name] <[table2name]>> > [file path]
    - 导出建表脚本（不包括数据） mysqldump -h[hostname] -P[port] -u[username] -p[password] -d [dbname] < [table1name] <[table2name]>> > [file path]
  
  
