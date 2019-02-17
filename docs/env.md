
这里是贯穿本书的环境

## env-1

- os：Ubuntu Server 16.04×64 
- openstack: queen 版
- mariadb: 10.0
- keystone: 
- swift: 2.17.1dev20
- Python: 2.7
- rsync: 3.0

- 存储设置：4T
- 架构部署: 

| 主机名   |     IP      |  作用 |
|----------|:-------------:|------:|
| controller |  192.168.100.50 | controller node |
| keystone |  192.168.100.50 | keystone node |
| proxy |  192.168.100.50 | proxy |
| object1      |     192.168.100.105    |     存储节点1(zone1)| 
| object2      |      192.168.100.106   |     存储节点2(zone1)| 
| object3      |      192.168.100.107    |     存储节点3(zone1)| 
| object4      |     192.168.100.107     |     存储节点4(zone1)|  

其中 192.168.100.50 有一个另一个网卡IP 192.168.0.50

## env-2

- os：Ubuntu Server 16.04×64 
- openstack: queen 版
- mariadb: 10.2
- keystone: 
- swift: 2.17.1dev20
- Python: 2.7
- rsync: 3.0

- 存储设置：4T
- 架构部署: 

| 主机名   |     IP      |  作用 |
|----------|:-------------:|------:|
| controller |  192.168.10.13 | controller node |
| mysql |  192.168.10.13 | 存 keystone 中用户帐户密码等数据 |
| keystone |  192.168.10.13 | auth |
| Proxy |  192.168.10.13 | swift proxy node |
| Proxy |  192.168.10.12 | swift proxy node |
| Proxy |  192.168.10.11 | swift proxy node |
| object1      |     192.168.0.127    |     存储节点1(zone1)| 
| object2      |     192.168.0.134    |     存储节点2(zone1)| 
| object3      |     192.168.0.135    |     存储节点3(zone1)| 
| object4      |    192.168.0.180     |     存储节点4(zone1)| 
| object5      |    192.168.0.189     |     存储节点5(zone1)| 

