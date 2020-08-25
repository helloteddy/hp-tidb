### 1.集群机器和拓扑
此次测试机器配置如下(带宽为1Mbps)：

| ip |cpu  |内存  | 磁盘规格 |
| --- | --- | --- | --- | 
| 192.168.0.6 | 2核 | 8G | 高性能云盘(IOPS 2000)| 
| 192.168.0.7 | 2核 | 8G | 高性能云盘(IOPS 2000) | 
| 192.168.0.8 | 2核 | 8G | 高性能云盘(IOPS 2000) |
| 192.168.0.9 | 2核 | 8G | 高性能云盘(IOPS 2000) | 

集群拓扑如下：

| 角色 | 机器 |
| --- | --- |
| pd | 192.168.0.7 |
| tidb |192.168.0.7  |
| tikv | 192.168.0.6 |
| tikv | 192.168.0.7 |
| tikv | 192.168.0.8 |
| tiflash | 192.168.0.9 |

### 2.集群配置调整

### 3.测试案例
#### 3.1 sysbench
* Sysbench 是一个采用 lua 编写的比较简单的测试数据库负载的程序
* 主要包括随机点查询（point select），简单的点更新（update where id=）和范围查询
* 用户可以根据自己的业务场景，改写 sysbench 的 .lua 脚本来进行测试
##### 3.1.1 安装
方式一：
源码编译安装 https://github.com/akopytov/sysbench
方式二：
``` sh
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash

sudo yum -y install sysbench
```
##### 3.1.2 配置和运行
配置文件
```
mysql-host=192.168.0.6
mysql-port=4000
mysql-user=root
mysql-password=
mysql-db=sbtest
time=600
threads=4
report-interval=10
db-driver=mysql
```
导入数据步骤
1. 创建数据库
``` sql
create database sbtest;
```
2. 导入数据之前先设置为乐观事务模式，导入结束后再设置回悲观模式
``` sql
 set global tidb_disable_txn_auto_retry = off;
 set global tidb_txn_mode="optimistic";
```
3. 执行导入命令导入数据
```
sysbench --config-file=config_1 oltp_point_select --tables=8 --table-size=1000000 prepare
```
测试步骤
1. 调整回悲观事务
```sql
set global tidb_txn_mode="pessimistic" ;
```
2. Point select 测试命令
``` sh
sysbench --config-file=config_1 oltp_point_select --threads=8 --tables=8 --table-size=1000000 run
```
3. Update index 测试命令
```
sysbench --config-file=config_1 oltp_update_index --threads=8 --tables=8 --table-size=1000000 run
```
4. Read-only 测试命令
```
sysbench --config-file=config_1 oltp_read_only --threads=8 --tables=8 --table-size=1000000 run
```
##### 3.1.3 测试结果
1. Point select 测试结果
```
SQL statistics:
    queries performed:
        read:                            5465308
        write:                           0
        other:                           0
        total:                           5465308
    transactions:                        5465308 (9108.74 per sec.)
    queries:                             5465308 (9108.74 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.0058s
    total number of events:              5465308

Latency (ms):
         min:                                    0.20
         avg:                                    0.88
         max:                                   77.99
         95th percentile:                        1.73
         sum:                              4796105.25

Threads fairness:
    events (avg/stddev):           683163.5000/604.85
    execution time (avg/stddev):   599.5132/0.00
```
监控截图
* TiDB Query Summary 中的 qps 与 duration
![2134fc8a84e8f484d696f7b4004c7a65.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p660)
* TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标
cpu
![3b5b8147d97d124d597eb70570608e6d.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p661)
QPS
![8799b785ca3874b4b384a3b5c5de8d8a.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p662)

* TiKV Details 面板中 grpc 的 qps 以及 duration
qps(没有看到qps指标)
duration
![e3c0d5eb56b59fa53e5aa7d32a60c754.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p664)
2. Update index 测试结果
```
SQL statistics:
    queries performed:
        read:                            0
        write:                           360024
        other:                           352
        total:                           360376
    transactions:                        360376 (600.61 per sec.)
    queries:                             360376 (600.61 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.0109s
    total number of events:              360376

Latency (ms):
         min:                                    0.53
         avg:                                   13.32
         max:                                  294.07
         95th percentile:                       19.65
         sum:                              4799579.33

Threads fairness:
    events (avg/stddev):           45047.0000/25.25
    execution time (avg/stddev):   599.9474/0.00
```
监控截图
* TiDB Query Summary 中的 qps 与 duration
![44c1362bbd194936252f5fe3dc3d6390.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p665)

* TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标
cpu
![d20da3efb7fd0c5553a56053a7a2cfcd.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p667)
qps
![4a2cf3d0232be083085a767bc23219d6.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p668)

* TiKV Details 面板中 grpc 的 qps 以及 duration
qps(没有看到qps指标)
duration
![84a7f0be14da24638828f0615b5e0241.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p669)

3. Read-only 测试结果
```
SQL statistics:
    queries performed:
        read:                            2061976
        write:                           0
        other:                           294568
        total:                           2356544
    transactions:                        147284 (245.45 per sec.)
    queries:                             2356544 (3927.19 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.0574s
    total number of events:              147284

Latency (ms):
         min:                                    6.59
         avg:                                   32.59
         max:                                  310.74
         95th percentile:                       61.08
         sum:                              4799896.44

Threads fairness:
    events (avg/stddev):           18410.5000/20.98
    execution time (avg/stddev):   599.9871/0.02
```
监控截图
* TiDB Query Summary 中的 qps 与 duration
![4601f8d11c32238279394662ebb9d054.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p672)

* TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标
cpu
![d75e33fa16d6dd00da1b24b826e75f46.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p673)
qps
![60d7fdf605d25df628830875a9aa06f9.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p674)


* TiKV Details 面板中 grpc 的 qps 以及 duration
qps(没有看到qps指标)
duration
![dca6544cec45649a53a77a0862fe2dfa.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p675)

#### 3.2 go-ycsb
* go-ycsb 是 PingCAP 在雅虎开源的 ycsb 基础上用 golang 开发的数据
库性能测试工具
##### 3.2.1 安装
1. 下载地址：https://github.com/pingcap/go-ycsb
2. 下载后执行 make 即可完成安装
##### 3.2.2 配置和运行
go-ycsb 中有负载类型 workloada ~ workloadf
1. 数据载入
```
./bin/go-ycsb load mysql -P workloads/workloada -p recordcount=1000000 -p mysql.host=localhost -p mysql.port=4000 --threads 8
```
2. 运行测试
```
./bin/go-ycsb run mysql -P workloads/workloada -p operationcount=1000000 -p mysql.host=localhost -p mysql.port=4000 --threads 8
```
##### 3.2.3 测试结果
测试结果
```
READ   - Takes(s): 837.5, Count: 500310, OPS: 597.4, Avg(us): 1689, Min(us): 647, Max(us): 110883, 99th(us): 11000, 99.9th(us): 28000, 99.99th(us): 37000
UPDATE - Takes(s): 837.5, Count: 499690, OPS: 596.6, Avg(us): 11629, Min(us): 4607, Max(us): 524844, 99th(us): 39000, 99.9th(us): 192000, 99.99th(us): 273000
```
监控截图
* TiDB Query Summary 中的 qps 与 duration
![233ee16afcf96e52dd8a68f2a41b2a58.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p676)

* TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标
cpu
![79e42333eadbe43164f87e79713e74b5.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p677)
qps
![ee1b4f047afdb2a0da29979c7391c76e.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p678)

* TiKV Details 面板中 grpc 的 qps 以及 duration
qps(没有看到qps指标)
duration
![539371d553b9ac4e6143d41aff2d6b7e.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p679)

#### 3.3 go-tpc
* TPC-C 是专门针对联机交易处理系统（OLTP 系统）的规范，一般情况下 我们也把这类系统称为业务处理系统。1992年7月发布。
* TPC-H 是国际上认定的分析性数据库测试的标准
* PingCAP 使用 golang 编写了 TPC-C 和 TPC-H 的测试程序，下载链接
和使用说明见：https://github.com/pingcap/go-tpc
##### 3.3.1 安装
1. 安装方式
``` sh
git clone https://github.com.cnpmjs.org/pingcap/go-tpc.git

make build
```

##### 3.3.2 配置和运行
###### 3.3.2.1 TPC-C
1. 加载数据
```
./bin/go-tpc tpcc -H localhost -P 4000 -D tpcc --warehouses 20 prepare
```
2. 运行测试
```
./bin/go-tpc tpcc -H localhost -P 4000 -D tpcc --warehouses 20 run --time 5m --threads 4
```
###### 3.3.2.2 TPC-H
1. 加载数据
```
./bin/go-tpc tpch prepare -H localhost -P 4000 -D tpch --sf 1 --analyze
```
2. 运行测试
```
 ./bin/go-tpc tpch run -H localhost -P 4000 -D tpch --sf 1 --time 5m --threads 4
```
##### 3.3.3 测试结果
###### 3.3.3.1 TPC-C
```
Finished
[Summary] DELIVERY - Takes(s): 299.0, Count: 920, TPM: 184.6, Sum(ms): 158897, Avg(ms): 172, 90th(ms): 256, 99th(ms): 512, 99.9th(ms): 512
[Summary] DELIVERY_ERR - Takes(s): 299.0, Count: 1, TPM: 0.2, Sum(ms): 77, Avg(ms): 77, 90th(ms): 80, 99th(ms): 80, 99.9th(ms): 80
[Summary] NEW_ORDER - Takes(s): 299.8, Count: 10677, TPM: 2136.7, Sum(ms): 594381, Avg(ms): 55, 90th(ms): 96, 99th(ms): 128, 99.9th(ms): 192
[Summary] NEW_ORDER_ERR - Takes(s): 299.8, Count: 1, TPM: 0.2, Sum(ms): 16, Avg(ms): 16, 90th(ms): 16, 99th(ms): 16, 99.9th(ms): 16
[Summary] ORDER_STATUS - Takes(s): 299.8, Count: 964, TPM: 193.0, Sum(ms): 11483, Avg(ms): 11, 90th(ms): 16, 99th(ms): 64, 99.9th(ms): 96
[Summary] PAYMENT - Takes(s): 299.9, Count: 10094, TPM: 2019.8, Sum(ms): 408633, Avg(ms): 40, 90th(ms): 64, 99th(ms): 112, 99.9th(ms): 160
[Summary] PAYMENT_ERR - Takes(s): 299.9, Count: 2, TPM: 0.4, Sum(ms): 40, Avg(ms): 20, 90th(ms): 32, 99th(ms): 32, 99.9th(ms): 32
[Summary] STOCK_LEVEL - Takes(s): 299.8, Count: 908, TPM: 181.7, Sum(ms): 14065, Avg(ms): 15, 90th(ms): 20, 99th(ms): 48, 99.9th(ms): 96
tpmC: 2136.7
```
监控截图
* TiDB Query Summary 中的 qps 与 duration
![b06980ced016d6b1350705a099886cf2.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p680)
* TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标
cpu
![2c71b6cc31edd3cec9113382b7b2d0a6.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p683)
qps
![8b62f3c45434d3e5c5245349762688c3.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p684)

* TiKV Details 面板中 grpc 的 qps 以及 duration
qps(没有看到qps指标)
duration
![38bfba1e6fe6258dcc31567ae563deed.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p685)


###### 3.3.3.2 TPC-H
测试结果
```
[Summary] Q1: 5.82s
[Summary] Q10: 3.82s
[Summary] Q11: 1.74s
[Summary] Q12: 3.37s
[Summary] Q13: 3.01s
[Summary] Q14: 3.76s
[Summary] Q15: 6.83s
[Summary] Q16: 0.91s
[Summary] Q17: 7.26s
[Summary] Q18: 9.92s
[Summary] Q19: 3.97s
[Summary] Q2: 3.33s
[Summary] Q20: 2.48s
[Summary] Q21: 5.79s
[Summary] Q22: 0.83s
[Summary] Q3: 4.86s
[Summary] Q4: 2.07s
[Summary] Q5: 5.76s
[Summary] Q6: 3.09s
[Summary] Q7: 5.95s
[Summary] Q8: 4.61s
[Summary] Q9: 12.50s
```
监控截图
* TiDB Query Summary 中的 qps 与 duration
![e99fc48e2e2e779e3a7d41fcfa9af049.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p687)

* TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标
cpu
![8812393d8a9ac041f670a1063828702f.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p689)
qps
![f32d468fca0e94215b6d7637b937c849.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p690)


* TiKV Details 面板中 grpc 的 qps 以及 duration
qps(没有看到qps指标)
duration
![45ca2311b8004de8b93467006134e0e0.png](evernotecid://13965CEC-8683-4BEC-8DB7-B696DE1F23B4/appyinxiangcom/26552848/ENResource/p691)


### 思考与分析
1. 此次测试没有修改tidb和tikv的配置，以默认的集群配置进行测试。
2. 当前集群的拓扑把tikv、tidb、pd三个角色放在一个节点上，会造成角色之间抢占资源，影响集群性能的问题。
3. 前面的测试从监控看来，主要受影响的资源有cpu和磁盘IO，对应的就是tikv的读操作。tikv的读操作包括三部分一个是kv_get、kv_batch_get和coprocessor。前面两个是读取数据的操作，最后一个是从tikv取数之后计算中间结果再返回到tidb做聚合。虽然说当前测试机器配置较低，但是个人认为集群的瓶颈会在tikv的coprocessor模块和storage模块。

