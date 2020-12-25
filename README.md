# Intel pmem ceph 性能测试报告



[TOC]

### 硬件信息

| 主机 ip       | CPU                                                          | 傲腾内存 | Sata SSD |
| ------------- | ------------------------------------------------------------ | -------- | -------- |
| 192.168.32.29 | 2 * Intel(R) Xeon(R) Gold 6252N CPU @ 2.30GHz ( 96 cpus )    | 512G     | 800G * 3 |
| 192.168.32.30 | 2 * Intel(R) Xeon(R) Gold 6252N CPU @ 2.30GHz ( 96 cpus )    | 512G     | 800G * 3 |
| 192.168.32.31 | 2 * Intel(R) Xeon(R) Platinum 8280L CPU @ 2.70GHz ( 112 cpus ) | 512G     | 800G * 3 |

### 软件信息

- 操作系统 CentOS Linux release 7.8.2003 (Core)

- 内核版本 3.10.0-1160.6.1.el7.x86_64

- Docker 版本 19.03.9

- Kubernetes 版本 v1.18.13

- Ceph 版本 Octopus 15.2.2

- Ceph 集群 3 节点，18 OSD

- Rook 版本 1.3.3


### 代码项目

```shell
git clone https://github.com/jing2uo/myrook.git        # 包含可执行代码，本次测试完整结果，本文档
```

### 部署流程

```shell
# 清理磁盘和遗留文件，编辑 clean.sh 修改 for 循环语句部分的盘符
# 执行脚本会将磁盘全部擦除，并划分两个 372G 的分区，请确保写对
...
for i in "sdb" "sdd" "sdf"
...
```

```shell
# 部署 rook operator
cd rook
kubectl apply -f rook/common.yaml
kubectl apply -f rook/operator.yaml

# 确保创建 pv 使用的节点 ip、磁盘信息和具体环境一致
kubectl apply -f rook/ssd-pv.yaml

# 部署 ceph 集群
kubectl apply -f rook/ssd-sc.yaml
kubectl apply -f rook/cluster-with-ssd.yaml
kubectl apply -f rook/toolbox.yaml
kubectl apply -f rook/cephcsi-storageclass.yaml
```
```shell
# 对象存储部分
kubectl create -n rook-ceph  -f cosbench/object.yaml
kubectl create -n rook-ceph  -f cosbench/object-bucket-claim-delete.yaml
kubectl create -n rook-ceph  -f cosbench/storageclass-bucket-delete.yaml
kubectl create -n rook-ceph  -f cosbench/rgw-external.yaml
```

### 测试流程

##### cockroach 测试过程

```shell
kubectl create -n rook-ceph -f cockroachdb/cockroachdb-statefulset.yaml
kubectl create -n rook-ceph -f cockroachdb/cluster-init.yaml
kubectl exec -n rook-ceph cockroachdb-0 -ti bash
# 进入容器后执行
cockroach workload init bank --drop
cockroach workload run bank  --duration=300s

cockroach workload init movr --drop
cockroach workload run movr --duration=300s

cockroach workload init tpcc --drop
cockroach workload run tpcc --duration=300s

cockroach workload init ycsb --drop
cockroach workload run ycsb --duration=300s
```


##### hammerdb 测试过程

```shell
# 镜像仓库地址需要修改
docker pull mysql:latest
docker tag mysql:latest 192.168.32.30:60080/cephtest/mysql:latest
docker push 192.168.32.30:60080/cephtest/mysql:latest
kubectl create -n rook-ceph -f hammerdb/hammerdb-statefulset.yaml
kubectl create -n rook-ceph -f hammerdb/mysql-statefulset.yaml 

# 进入 mysql
kubectl exec -ti -n rook-ceph mysql-0 bash
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';
mysql> flush privileges;

# 获取 mysql cluster ip
kubectl get -n rook-ceph pod mysql-0 -o wide

# 进入 hammerdb
kubectl exec -ti -n rook-ceph hammerdb-0 bash
./hammerdbcli 

# tpc-h 测试
dbset db mysql
dbset bm tpc-h
diset connection mysql_host 10.199.1.144 # 修改为上方获取的 cluster ip
diset tpch mysql_tpch_pass password
buildschema
loadscript
vurun

# tpc-c 测试
dbset db mysql
dbset bm tpc-c
diset connection mysql_host 10.199.1.144
diset tpcc mysql_pass password
diset tpcc mysql_driver timed
diset tpcc mysql_rampup 0
diset tpcc mysql_duration 1
buildschema
loadscript
vuset vu [ 1 2 4 8 ]  # 执行时选择
vucreate
vurun
```

##### cosbench 测试过程

```shell
docker run --name cosbench -dti -p 19088:19088 -p 18088:18088 nexenta/cosbench:latest bash
docker exec -ti cosbench bash
cd cos
bash start-all.sh

# 浏览器打开 192.168.32.30:19088/controller/index.html，可以看到页面，域名部分需替换为自己节点的 ip
# ceph 集群部分需要获取 key，修改允许创建的 bucket 数量

kubectl exec -ti -n rook-ceph rook-ceph-tools-5dd4bb486d-gh9wh bash
radosgw-admin user list
radosgw-admin user create --display-name="cosbench" --uid=cosbench
radosgw-admin user modify --uid cosbench max_buckets=1000000
kubectl get svc -n rook-ceph rook-ceph-rgw-my-store-external

# 以上命令需要根据环境修改 ceph-tools pod名称
# 修改 bucket 限制后，会回显 access_key 和 secret_key，复制保留下来

# 使用以上信息修改 cosbench/s3-config.xml 文件，用于提交 cosbench 测试，密码部分可以忽略
...
  <auth type="none" config="username=cosbench;password=cosbench;auth_url=http://192.168.32.30:32059"/>
  <storage type="s3" config="accesskey=DX6Z8326O1D4Z3BFRBC2;secretkey=Xz9llJeTcxaUzz62rOTejtFsSHzNdN159QeWBTZg;endpoint=http://192.168.32.30:32059;timeout=60"/>
...
```



![](/home/jing2uo/Desktop/alauda/docs/case.png)



### case 1 :  使用 SSD 存储  Data 

##### 配置信息

```shell
1 OSD = 370G(data ssd)
6(block) * 3(node)  = 18 OSD 
```


##### cockroach 测试结果

```shell
# bank 部分结果
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0         151311          504.4    437.8     12.1    520.1  14495.5  94489.3  transfer

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
  300.0s        0         151311          504.4    437.8     12.1    520.1  14495.5  94489.3  
```

```shell
# movr 部分结果
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0            977            3.3      5.5      5.2      7.3     15.7     35.7  addUser

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0            316            1.1      8.6      8.4     10.5     15.2     18.9  addVehicle

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0            340            1.1      9.6      9.4     11.5     17.8     24.1  applyPromoCode

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0            101            0.3      5.5      5.5      7.3     14.7     15.2  createPromoCode

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0            239            0.8      5.6      5.2      7.3     15.7     17.8  endRide

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0          63421          211.4      1.0      1.0      1.3      1.5     60.8  readVehicles

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0           1303            4.3     15.0     15.2     18.9     26.2     39.8  startRide

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0           3276           10.9     62.2     60.8     79.7    167.8    285.2  updateActiveRides

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
  300.0s        0          69973          233.2      4.3      1.0     17.8     71.3    285.2  
```

```shell
# tpcc 部分结果
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0              5            0.0     51.0     52.4     56.6     56.6     56.6  delivery

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0             63            0.2     31.4     31.5     39.8     44.0     71.3  newOrder

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0              7            0.0      5.3      5.2      6.6      6.6      6.6  orderStatus

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0             61            0.2     23.3     17.8     60.8    104.9    130.0  payment

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0              5            0.0     16.7     16.8     19.9     19.9     19.9  stockLevel

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
  300.0s        0            141            0.5     26.8     26.2     52.4    104.9    130.0  
Audit check 9.2.1.7: SKIP: not enough delivery transactions to be statistically significant
Audit check 9.2.2.5.1: SKIP: not enough orders to be statistically significant
Audit check 9.2.2.5.2: SKIP: not enough orders to be statistically significant
Audit check 9.2.2.5.3: PASS
Audit check 9.2.2.5.4: PASS
Audit check 9.2.2.5.5: SKIP: not enough payments to be statistically significant
Audit check 9.2.2.5.6: SKIP: not enough order status transactions to be statistically significant

_elapsed_______tpmC____efc__avg(ms)__p50(ms)__p90(ms)__p95(ms)__p99(ms)_pMax(ms)
  300.0s       12.6  98.0%     31.4     31.5     39.8     39.8     44.0     71.3
```

```shell
# ycsb 部分结果
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0       14470912        48235.7      3.9      1.0     15.2     60.8    738.2  read

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0         759249         2530.8     13.7      8.9     39.8     92.3   1275.1  update

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
  300.0s        0       15230161        50766.5      4.4      1.1     16.8     65.0   1275.1  
```

##### hammerdb 测试结果

```shell
#tpch
Vuser 1:Completed 1 query set(s) in 81 seconds
Vuser 1:FINISHED SUCCESS
```

```shell
# tpc-c

hammerdb>vudestroy
hammerdb>vuset vu 1
hammerdb>vucreate
hammerdb>vurun
Vuser 1:TEST RESULT : System achieved 2762 MySQL TPM at 868 NOPM

hammerdb>vudestroy
hammerdb>vuset vu 2
hammerdb>vucreate
hammerdb>vurun
Vuser 1:TEST RESULT : System achieved 4791 MySQL TPM at 1571 NOPM

hammerdb>vudestroy
hammerdb>vuset vu 4
hammerdb>vucreate
hammerdb>vurun
Vuser 1:TEST RESULT : System achieved 7007 MySQL TPM at 2263 NOPM

hammerdb>vudestroy
hammerdb>vuset vu 8
hammerdb>vucreate
hammerdb>vurun
Vuser 1:TEST RESULT : System achieved 7718 MySQL TPM at 2544 NOPM
```

##### cosbench 测试结果

![](/home/jing2uo/Desktop/alauda/ssd/cosbench.png)

### case 2 :  使用 pmem 存储 metadata 

##### 配置信息

```
1 OSD = 370G(data ssd) + 80G(metadata pmem)
6(block) * 3(node)  = 18 OSD
```

##### cockroach 测试结果

```shell
# bank 结果
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0         194645          648.8    340.9     11.0    453.0  11811.2  60129.5  transfer

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
  300.0s        0         194645          648.8    340.9     11.0    453.0  11811.2  60129.5  
```

```shell
# movr 结果
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0           1050            3.5      4.9      4.7      7.6     13.1     15.7  addUser

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0            344            1.1      7.8      7.9     10.0     13.6     15.2  addVehicle

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0            369            1.2      9.0      9.4     12.1     15.7     19.9  applyPromoCode

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0            109            0.4      4.7      4.5      6.8     12.6     13.6  createPromoCode

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0            257            0.9      5.0      4.7      8.4     13.1     14.7  endRide

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0          67483          224.9      1.0      1.0      1.4      1.6     27.3  readVehicles

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0           1382            4.6     13.4     13.1     18.9     30.4     50.3  startRide

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0           3511           11.7     57.8     56.6     71.3    151.0    251.7  updateActiveRides

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
  300.0s        0          74505          248.3      4.0      1.0     15.7     60.8    251.7  
```

```shell
# tpcc 结果
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0       14619294        48730.6      3.9      1.2     14.7     54.5    486.5  read

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0         767120         2557.0     13.4      9.4     35.7     75.5    872.4  update

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
  300.0s        0       15386414        51287.7      4.4      1.4     16.8     56.6    872.4  
```

```shell
# ycsb 结果
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0       14619294        48730.6      3.9      1.2     14.7     54.5    486.5  read

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0         767120         2557.0     13.4      9.4     35.7     75.5    872.4  update

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
  300.0s        0       15386414        51287.7      4.4      1.4     16.8     56.6    872.4  
```

##### hammerdb 测试结果

```shell
# tpch 
Vuser 1:Completed 1 query set(s) in 95 seconds
```

```shell
# tpcc

hammerdb>vudestroy
hammerdb>vuset vu 1
hammerdb>vucreate
hammerdb>vurun
Vuser 1:TEST RESULT : System achieved 3062 MySQL TPM at 940 NOPM

hammerdb>vudestroy
hammerdb>vuset vu 2
hammerdb>vucreate
hammerdb>vurun
Vuser 1:TEST RESULT : System achieved 5479 MySQL TPM at 1848 NOPM

hammerdb>vudestroy
hammerdb>vuset vu 4
hammerdb>vucreate
hammerdb>vurun
Vuser 1:TEST RESULT : System achieved 7773 MySQL TPM at 2701 NOPM

hammerdb>vudestroy
hammerdb>vuset vu 8
hammerdb>vucreate
hammerdb>vurun
Vuser 1:TEST RESULT : System achieved 8535 MySQL TPM at 2844 NOPM
```

##### cosbench 测试结果

![](/home/jing2uo/Desktop/alauda/meta/cosbench.png)



### case 3 : 使用 opencas 存储 data

##### 配置信息

```
1 OSD = 1 CAS (data) + 20G(metadata pmem)
1 CAS = 370G(core ssd) + 60G(cache pmem)
6(block) * 3(node)  = 18 OSD
```

##### cockroach 测试结果

```shell
# bank 结果
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0         167678          558.9    395.9     10.5    335.5  11811.2 103079.2  transfer

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
  300.0s        0         167678          558.9    395.9     10.5    335.5  11811.2 103079.2  
```

```shell
# movr 结果
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0           1105            3.7      4.2      3.8      7.9     11.5     16.3  addUser

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0            364            1.2      7.6      7.3     11.5     15.2     54.5  addVehicle

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0            380            1.3      8.9      8.9     12.6     16.8     23.1  applyPromoCode

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0            115            0.4      4.0      3.7      6.8     10.5     13.1  createPromoCode

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0            272            0.9      4.4      3.9      9.4     11.5     26.2  endRide

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0          71022          236.7      1.0      1.0      1.3      1.6     44.0  readVehicles

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0           1452            4.8     12.5     12.1     18.9     28.3     67.1  startRide

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0           3688           12.3     54.0     52.4     75.5    134.2    209.7  updateActiveRides

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
  300.0s        0          78398          261.3      3.8      1.0     14.7     56.6    209.7  
```

```shell
# tpcc 结果
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0              5            0.0     47.2     48.2     52.4     52.4     52.4  delivery

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0             67            0.2     34.7     29.4     92.3    134.2    142.6  newOrder

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0              5            0.0      5.4      4.7      9.4      9.4      9.4  orderStatus

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0             55            0.2     18.4     15.7     27.3     58.7     79.7  payment

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0              5            0.0    125.5     18.9    318.8    318.8    318.8  stockLevel

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
  300.0s        0            137            0.5     30.9     25.2     75.5    268.4    318.8  
Audit check 9.2.1.7: SKIP: not enough delivery transactions to be statistically significant
Audit check 9.2.2.5.1: SKIP: not enough orders to be statistically significant
Audit check 9.2.2.5.2: SKIP: not enough orders to be statistically significant
Audit check 9.2.2.5.3: PASS
Audit check 9.2.2.5.4: PASS
Audit check 9.2.2.5.5: SKIP: not enough payments to be statistically significant
Audit check 9.2.2.5.6: SKIP: not enough order status transactions to be statistically significant

_elapsed_______tpmC____efc__avg(ms)__p50(ms)__p90(ms)__p95(ms)__p99(ms)_pMax(ms)
  300.0s       13.4 104.2%     34.7     29.4     39.8     92.3    134.2    142.6

```

```shell
# ycsb 结果
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0       15037198        50123.8      3.8      1.3     14.2     48.2    738.2  read

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__total
  300.0s        0         789092         2630.3     12.7      8.4     35.7     79.7   1006.6  update

_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
  300.0s        0       15826290        52754.1      4.2      1.4     16.3     50.3   1006.6  
```

##### hammerdb 测试结果

```shell
# tpch 
Vuser 1:Completed 1 query set(s) in 91 seconds
```

```shell
# tpcc

hammerdb>vudestroy
hammerdb>vuset vu 1
hammerdb>vucreate
hammerdb>vurun
Vuser 1:TEST RESULT : System achieved 3752 MySQL TPM at 1176 NOPM

hammerdb>vudestroy
hammerdb>vuset vu 2
hammerdb>vucreate
hammerdb>vurun
Vuser 1:TEST RESULT : System achieved 6650 MySQL TPM at 2307 NOPM

hammerdb>vudestroy
hammerdb>vuset vu 4
hammerdb>vucreate
hammerdb>vurun
Vuser 1:TEST RESULT : System achieved 9623 MySQL TPM at 3106 NOPM

hammerdb>vudestroy
hammerdb>vuset vu 8
hammerdb>vucreate
hammerdb>vurun
Vuser 1:TEST RESULT : System achieved 10578 MySQL TPM at 3525 NOPM
```

##### cosbench 测试结果

![](/home/jing2uo/Desktop/alauda/cas/cosbench.png)



### 参考链接

删除持续 terminating 状态的资源  https://blog.csdn.net/solaraceboy/article/details/1000405241

对象存储配置   https://zhuanlan.zhihu.com/p/107083375

ceph 用户管理 https://docs.ceph.com/en/latest/man/8/radosgw-admin/

pmem-csi安装文档  https://github.com/intel/pmem-csi/blob/devel/docs/install.md

ndctl 排错  https://docs.pmem.io/ndctl-user-guide/troubleshooting

casadm 使用 https://open-cas.github.io/guide_configuring.html

opencas 安装 https://github.com/Open-CAS/open-cas-linux



### 有用的命令

查询命名空间下资源

```shell
kubectl api-resources -o name --verbs=list --namespaced | xargs -n 1 kubectl get --show-kind --ignore-not-found -n rook-ceph
```

擦除磁盘开始扇区

```
dd if=/dev/zero of=/dev/pmem0 bs=512 count=8
```

