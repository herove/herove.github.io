---
layout:     post
title:      "Greenplum集群更换主机"
date:       2017-04-21 15:08:00
author:     "zhouleihao"
header-img: ""
tags:
    - Greenplum
---

> 执行前备份相关表
 
1. 集群停服
```
gpstop
``` 
 
2. 运维模式启动集群
```
gpstart -m
```
 
3. 更新gp_segment_configuration表，修改hostname到新的机器
```
# PGOPTIONS='-c gp_session_role=utility' psql
testDB=# set allow_system_table_mods='dml'; 
testDB=# update gp_segment_configuration set hostname="新机器" where hostname="坏掉的机器"
```

4. 重启集群
```
gpstop -M fast -r
```
 
5. 执行gprecoverseg全量恢复segment
```
gprecoverseg -F
```
 
6. 等到状态从resyning变成unbalance状态，执行以下命令恢复
```
gprecoverseg -r
```
 
