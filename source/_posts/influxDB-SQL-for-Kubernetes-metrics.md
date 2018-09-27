---
title: influxDB-SQL-for-Kubernetes-metrics.md
date: 2018-09-27 23:57:31
tags:
	- InfluxDB
	- Heapster
	- Kubernetes
	- monitoring
---
# 节点监控指标汇总
- 这里以 10.35.48.177为例，可以访问172.25.3.194上面的influxdb进行测试
- InfluxDB 包含mean、group by 等语法，可以视需要使用
- 本文中仅为示例用法
- 有些指标是固定的，比如总的的文件系统空间、节点总内存及可分配总内存、总cpu核数、GPU总显存，所以一般搜最新的就好

## 容器总的cpu利用率(单位%)

SELECT *  FROM k8s."default"."cpu/node_utilization" WHERE (nodename =~ /^10\.35\.48\.177$/ AND type = 'node') AND time >= now() - 1h;

或者

SELECT mean(value) FROM k8s."default"."cpu/node_utilization" WHERE (nodename =~ /^10\.35\.48\.177$/ AND type = 'node') AND time >= now() - 1h GROUP BY time(1m)


## 总cpu核数
SELECT mean(value) / 1000 FROM k8s."default"."cpu/node_capacity" WHERE (type = 'node' AND nodename =~ /^10.35.48.177$/) AND time >= now() - 5m GROUP BY time(30s)
部分sql语法可以自己斟酌下哈

## 内存利用率(单位%)
每1分钟聚合的内存利用率均值: `SELECT mean(value) FROM k8s."default"."memory/node_utilization" WHERE (nodename =~ /^10\.35\.48\.177$/ AND type = 'node') AND time >= now() - 1h GROUP BY time(1m)`

## 节点总内存 字节
SELECT mean(value) / 1000 FROM k8s."default"."cpu/node_capacity" WHERE (type = 'node' AND nodename =~ /^10.35.48.177$/) AND time >= now() - 1h GROUP BY time(30s)

## 节点总的可分配内存 字节
SELECT mean(value) FROM k8s."default"."memory/node_allocatable" WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 1h GROUP BY time(30s)

## 节点上容器使用的总内存
SELECT value FROM k8s."default"."memory/usage" WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 1h

## 节点上容器使用的RSS内存
SELECT value FROM k8s."default"."memory/rss" WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 1h

## 节点上容器使用的cache内存
SELECT value FROM k8s."default"."memory/cache" WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 1h

## 节点上容器使用的working_set内存
SELECT value FROM k8s."default"."memory/working_set" WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 1h

- memory_usage = RAM usage include pages that have not been accessed in a
long time.
- working set = memory usage - inactive memory.

## 网络读速率
SELECT value FROM k8s."default"."network/rx_rate" WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 1h

## 网络写速率
SELECT value FROM k8s."default"."network/tx_rate" WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 1h

## 磁盘读速率(暂时忽略，上游有bug，会显示负数)
SELECT mean(value) FROM k8s."default"."disk/io_read_bytes_rate" WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 1h GROUP BY time(30s)

## 磁盘写速率(暂时忽略)
SELECT mean(value) FROM k8s."default"."disk/io_write_bytes_rate" WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 1h GROUP BY time(30s)

## 可用的文件系统空间(按照磁盘分类)
SELECT value FROM k8s."default"."filesystem/available" WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 10m  group by resource_id

## 已使用的文件系统空间(按照磁盘分类)
SELECT value FROM k8s."default"."filesystem/usage" WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 10m  group by resource_id

## 总的的文件系统空间
SELECT value FROM k8s."default"."filesystem/limit" WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 1m  group by resource_id

## GPU总显存
SELECT mean(value) FROM k8s."default"."accelerator/memory_total" WHERE nodename =~ /^10\.30\.21\.153$/ AND time >= now() - 1h  GROUP BY time(1m)

## GPU已使用的显存
SELECT mean(value) FROM k8s."default"."accelerator/memory_used" WHERE nodename =~ /^10\.30\.21\.153$/ AND time >= now() - 1h  GROUP BY time(1m)

## GPU核心利用率
SELECT value FROM k8s."default"."accelerator/duty_cycle" WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 10m


## uptime 启动时间
SELECT value FROM k8s."default".uptime WHERE (type = 'node' AND nodename =~ /^10\.35\.48\.177$/) AND time >= now() - 1m


# 容器内指标

## 各种服务的基本查询SQL
对于多副本(pod)的无状态服务/有状态服务/短任务，查询到的指标值应该是各个副本的和
### 无状态服务
```
SELECT mean(value) FROM k8s."default"."memory/usage" WHERE (type = 'rc' AND rc_name =~ /^autotest-template-server123-4$/ AND namespace_name =~ /^guxiawei-84$/) AND time >= now() - 1h GROUP BY time(1m)
SELECT mean(value) FROM k8s."default"."cpu/usage_rate" WHERE (type = 'rc' AND rc_name =~ /^autotest-template-server123-4$/ AND namespace_name =~ /^guxiawei-84$/) AND time >= now() - 1h GROUP BY time(1m)
```
### 短任务
```
SELECT mean(value) FROM k8s."default"."memory/usage" WHERE (type = 'job' AND job_name =~ /^job-createzy6vgx$/ AND namespace_name =~ /^liuguiying-61$/) AND time >= now() - 1h GROUP BY time(1m)
```
### 有状态服务
```
SELECT mean(value) FROM k8s."default"."memory/usage" WHERE (type = 'ss' AND ss_name =~ /^testservice110$/ AND namespace_name =~ /^wangmeng3-154$/) AND time >= now() - 1h GROUP BY time(1m)
```
### pod
```
SELECT mean(value) FROM k8s."default"."memory/usage" WHERE (type = 'pod' AND pod_name =~ /^testservice110-0$/ AND namespace_name =~ /^wangmeng3-154$/) AND time >= now() - 1h GROUP BY time(1m)
SELECT mean(value) FROM k8s."default"."cpu/usage_rate" WHERE (type = 'pod' AND pod_name =~ /^autotest-template-server123-4-87ms1$/ AND namespace_name =~ /^guxiawei-84$/) AND time >= now() - 1h GROUP BY time(1m)
```
### 容器
- 根据pod名称和namespace查询，并按照容器名分类
```
SELECT value FROM k8s."default"."memory/usage" WHERE (type='pod_container' AND pod_name='gb-groupdn-stable-779-1-rcv9x'  AND namespace_name =~ /^guanbo-225$/) AND time >= now() - 5m group by container_name
```
- 按照容器名和pod名和namespace查询
 ```
 SELECT value FROM k8s."default"."memory/usage" WHERE (type='pod_container' AND pod_name='gb-groupdn-stable-779-1-rcv9x'  AND namespace_name =~ /^guanbo-225$/ and container_name='media-node') AND time >= now() - 5m
 ```
- 按照容器名和namespace查询（不推荐，容器名可能重名）
 ```
 SELECT value FROM k8s."default"."memory/usage" WHERE (type='pod_container' AND namespace_name =~ /^guanbo-225$/ and container_name='media-node') AND time >= now() - 5m
 ```
 
### GPU
```
SELECT mean(value) FROM k8s."default"."accelerator/memory_used" WHERE (type = 'pod' AND pod_name =~ /^autotest-template-server123-4-87ms1$/ AND namespace_name =~ /^haoyuan-8$/) AND time >= now() - 1h GROUP BY time(1m)
```

对于容器，需要的指标有：
- cpu/limit 容器的cpu限额
- cpu/request 独占的CPU核数
- cpu/usage_rate cpu使用率
- disk/io_read_bytes_rate 磁盘读速率
- disk/io_write_bytes_rate 磁盘写速率
- memory/limit 容器的内存限额
- memory/request 容器独占的内存
- memory/rss  使用的rss内存
- memory/usage 内存使用量
- memory/cache cache内存
- accelerator/memory_total	Memory capacity of an accelerator.
- accelerator/memory_used	Memory used of an accelerator.
- accelerator/duty_cycle	Duty cycle of an accelerator.
- accelerator/request	Number of accelerator devices requested by container.
- network/rx_rate 网络读速率
- network/tx_rate 网络写速率


### 附录：所有heapster指标
- 参考： https://github.com/kubernetes/heapster/blob/master/docs/storage-schema.md
```
cpu/limit
cpu/node_allocatable
cpu/node_capacity
cpu/node_reservation
cpu/node_utilization
cpu/request
cpu/usage
cpu/usage_rate
disk/io_read_bytes
disk/io_read_bytes_rate
disk/io_write_bytes
disk/io_write_bytes_rate
filesystem/available
filesystem/inodes
filesystem/inodes_free
filesystem/limit
filesystem/usage
memory/cache
memory/limit
memory/major_page_faults
memory/major_page_faults_rate
memory/node_allocatable
memory/node_capacity
memory/node_reservation
memory/node_utilization
memory/page_faults
memory/page_faults_rate
memory/request
memory/rss
memory/usage
memory/working_set
accelerator/memory_total	Memory capacity of an accelerator.
accelerator/memory_used	Memory used of an accelerator.
accelerator/duty_cycle	Duty cycle of an accelerator.
accelerator/request	Number of accelerator devices requested by container.
network/rx
network/rx_errors
network/rx_errors_rate
network/rx_rate
network/tx
network/tx_errors
network/tx_errors_rate
network/tx_rate
restart_count
uptime
```
