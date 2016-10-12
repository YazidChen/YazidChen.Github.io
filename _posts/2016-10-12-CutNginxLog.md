---
layout: post
title:  "定时切分Nginx日志"
categories: Nginx
description: 设定定时任务自动切分Nginx日志，Tomcat For Ubuntu16.04。
keywords: Nginx,日志,log,切分Nginx日志,定时切割Nginx日志，切分日志
---

# 一、编写日志自动切割脚本 #

```shell
vim /usr/local/nginx/sbin/cut_nginx_log.sh
```

脚本编写：

```shell
#!/bin/bash
# author YazidChen

# 一般设定切割时间是0点，切除昨天的日志。
DATE=$(date -d "yesterday" +"%Y%m%d")
LOG_PATH=/usr/local/nginx/logs/
ACCESS_LOG_NAME=access.log
ERROR_LOG_NAME=error.log
CUT_ACCESS_LOG_NAME=${DATE}.${ACCESS_LOG_NAME}
CUT_ERROR_LOG_NAME=${DATE}.${ERROR_LOG_NAME}
PID_PATH=/usr/local/nginx/logs/nginx.pid

# 重命名原始日志文件
mv ${LOG_PATH}${ACCESS_LOG_NAME} ${LOG_PATH}${CUT_ACCESS_LOG_NAME}
mv ${LOG_PATH}${ERROR_LOG_NAME} ${LOG_PATH}${CUT_ERROR_LOG_NAME}

# 向nginx主进程发信号，重新打开日志文件，当原始文件不存在时，重新生成新的日志文件access.log及error.log
kill -USR1 `cat ${PID_PATH}`
```

# 二、设置定时执行脚本 #

**1、**给脚本执行权限：
```shell
chmod +x cut_nginx_log.sh 
```

**2、**使用crontab工具添加定时任务：
```shell
# 编辑crontab文件
crontab -e
```

**3、**编写定时任务，保存退出：
```shell
# 每天0点执行任务
0 0 * * * /bin/sh /usr/local/nginx/sbin/cut_nginx_log.sh
```

为方便测试将时间修改：

![](http://i.imgur.com/PmS5NG6.png)

测试过程中校验了日志内容，发现没有内容缺失，测试成功：

![](http://i.imgur.com/cFDuNKZ.png)

测试中可能会使用到以下命令：
```shell
# 列出crontab文件内容
crontab -l
# 删除crontab文件
crontab -r
```

# 参考 #

本文参考以下文章，在此对原作者表示感谢！

[Linux下定时切割nginx日志并删除指定天数前的日志记录](http://pvbutler.blog.51cto.com/7662323/1653088)
