# openstack-manual-deploy
Docs and config files of OpenStack-manual-deploy
# 说明
本文参考 https://docs.openstack.org/newton/install-guide-ubuntu/ 官方文档手动安装 Newton 版本（Ocata 版本有不可抗拒的 bug，暂不更新），用于日常实验环境的搭建，并对部分步骤给出比必要的解释，相关的配置文件可在对应的目录中找到，用以备忘。
# 目录
<!-- toc -->
* [环境](#环境)
  * [架构](#架构)
  * [网络](#网络)

<!-- toc stop -->
# 环境
## 架构
根据官方文档，本文架构采用一个控制节点和一个计算节点。
（The example architecture requires at least twonodes (hosts) to launch a basic virtual machine or instance. ）

控制节点运行认证服务、镜像服务、计算服务的管理部分、网络服务的管理部分、各种网络代理以及Dashboard，还包括一些基础服务如数据库、消息队列以及NTP。

计算节点上运行计算服务中管理实例的管理程序部分。默认情况下，计算服务使用 KVM。还运行网络代理服务，用来连接实例和虚拟网络以及通过安全组给实例提供防火墙服务。
## 网络
### 公有网络
公有网络选项以尽可能简单的方式通过layer-2（网桥/交换机）服务以及VLAN网络的分割来部署OpenStack网络服务。实际上，它将虚拟网络桥接到物理网络，并依靠物理网络基础设施提供layer-3服务(路由)。另外，DHCP服务为实例提供IP地址。
### 私有网络
私有网络选项扩展了公有网络选项，增加了layer-3（路由）服务，使用VXLAN类似的方式。本质上，它使用NAT路由虚拟网络到物理网络。另外，这个选项也提供高级服务的基础，比如LBaas和FWaaS。

这里我们选择私有网络。
## 物理机配置
```
############################################################################################

# 节点名称：
#                controller          compute1     

# 配置信息：
#                24核16G/4*300G      24核16G/4*300G      

# 网络信息：
#     eno2       10.109.252.247      10.109.252.248
#     eno3       access              access

# 操作系统：
#                ubuntu 16.04 LTS

############################################################################################
```
## 安全
下面是各个需要密码的服务以及解释，可以采用同一个，也可以按照下表设置密码。

| Password      | Description           |
| ------------- |:---------------------:|
| Database Password      | Root password for the database（can share the same pwd with linux user） |
| ADMIN_PASS      | Password of user admin |
| DEMO_PASS       | Password of user demo  |
| RABBIT_PASS       | Password of user guest of RabbitMQ  |
| KEYSTONE_DBPASS      | Database password of Identity service |
| GLANCE_DBPASS      | Database password for Image service |
| GLANCE_PASS      | Password of Image service user glance |
| NOVA_DBPASS      | Database password for Compute service |
| NOVA_PASS      | Password of Compute service user nova |
| NEUTRON_DBPASS      | Database password for the Networking service |
| NEUTRON_PASS      | Password of Networking service user neutron |
| DASH_DBPASS      | Database password for the dashboard |

# 1.网络配置
## 控制节点

/etc/network/interfaces 网卡1 eno2 设置为静态IP，网卡2 eno3 如下设置
```shell
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eno2
iface eno2 inet static
address 10.109.252.247
netmask 255.255.255.0
gateway 10.109.252.1
dns_nameservers 10.3.9.4 10.3.9.5

# The provider network interface
auto eno3
iface eno3 inet manual
up ip link set dev $IFACE up
down ip link set dev $IFACE down
dns_nameservers 10.3.9.4 10.3.9.5
```
/etc/hosts 配置地址解析，需注释掉 127.0.1.1 这一行
```shell
127.0.0.1       localhost
# 127.0.1.1     controller

# controller
10.109.252.247    controller

#compute1
10.109.252.248    compute1
```
注意！需要禁掉网卡2 eno3 的 ipv6，否则在创建网络时会一直处于创建过程中，无法成功！（尽管ipv6被禁，但是 provider 网络架构下实例仍然具备 ipv6 访问能力）通过修改 /etc/sysctl.conf 文件实现对 /proc 进行永久修改。
```shell
net.ipv6.conf.eno3.disable_ipv6 = 1
```
执行以下命令使修改立即生效。
```shell
sudo sysctl -p /etc/sysctl.conf 
```
## 计算节点
同样需要配置网卡、地址解析、禁用第二张网卡的 ipv6，参考控制节点。
## 验证
控制节点与计算节点的互通性，在控制节点与计算节点执行
```shell
ping -c 4 compute1
ping -c 4 controller
```
# 2.NTP
