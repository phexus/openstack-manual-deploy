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
#127.0.1.1      controller

# controller
10.109.252.247  controller

#compute1
10.109.252.248  compute1

#compute2
10.109.252.241  compute2

#compute3
10.109.252.242  compute3

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
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
# 2.NTP 网络时间协议
## 控制节点
安装包
```shell
# apt install chrony
# vim /etc/chrony/chrony.conf
```
编辑  /etc/chrony.conf 文件，这里可以根据需要将NTP_SERVER替换为合适的NTP服务器，在本架构中不包含NTP服务器，所以不用添加这一行。
```shell
server NTP_SERVER iburst
```
允许计算节点同步。（计算节点IP网段属于10.109.252.0）
```shell
allow 10.109.252.0/24
```
重启 NTP 服务
```shell
# service chrony restart
```
## 其他节点
```shell
# apt install chrony
# vim /etc/chrony/chrony.conf
```
编辑  /etc/chrony.conf 文件，确保从 controller 节点同步时间
```shell
server controller iburst
```
注释掉 `pool 2.debian.pool.ntp.org offline iburst` 这一行
```shell
# service chrony restart
```
具体文件可直接复制本repo内的，适当修改 allow
## 验证
在控制节点上同步时间
```shell
# chronyc sources

  210 Number of sources = 2
  MS Name/IP address         Stratum Poll Reach LastRx Last sample
  ===============================================================================
  ^- 192.0.2.11                    2   7    12   137  -2814us[-3000us] +/-   43ms
  ^* 192.0.2.12                    2   6   177    46    +17us[  -23us] +/-   68ms
```
在其余节点上同步时间
```shell
# chronyc sources

  210 Number of sources = 1
  MS Name/IP address         Stratum Poll Reach LastRx Last sample
  ===============================================================================
  ^* controller                    3    9   377   421    +15us[  -87us] +/-   15ms
```
带星号的是NTP当前同步的地址。确保其他节点输出结果同步的是controller
# 3. 安装 OpenStack 包
## 所有节点
```shell
# apt install software-properties-common
# add-apt-repository cloud-archive:newton
# apt update && apt dist-upgrade    //若有更新注意重启机器激活
# apt install python-openstackclient
```
# 4. 数据库
## 控制节点
```shell
# apt install mariadb-server python-pymysql
# vim /etc/mysql/mariadb.conf.d/99-openstack.cnf
```
添加内容，bind-address 替换为控制节点第一张网卡的 ip 地址，此时为10.109.252.247
```shell
[mysqld]
bind-address = 10.109.252.247

default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```
重启服务，并为数据库 root 用户设置密码，删除 anonymous 用户，关闭 root 远程访问
```shell
# service mysql restart
# mysql_secure_installation
```
# 5. Message queue 消息队列
## 控制节点
```shell
# apt install rabbitmq-server
# rabbitmqctl add_user openstack RABBIT_PASS
# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```
# 6. Memcached 缓存令牌
## 控制节点
```shell
# apt install memcached python-memcache
# vim /etc/memcached.conf
```
编辑 /etc/sysconfig/memcached文件并配置IP地址，将127.0.0.1改为控制节点IP。
```shell
#-l 127.0.0.1
-l 10.109.252.247
```
```shell
# service memcached restart
```
# 7. Identity service 认证服务
## 控制节点
```shell
# mysql
MariaDB[(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGESON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGESON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
```
```shell
# apt install keystone
# vim /etc/keystone/keystone.conf
```
添加内容
```shell
[database]
...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
...
[token]
...
provider = fernet
```
同步认证服务数据库、初始化Fernetkey仓库、引导认证服务
```shell
# su -s /bin/sh -c "keystone-manage db_sync" keystone
# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
# keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:35357/v3/ \
  --bootstrap-internal-url http://controller:35357/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```
配置Apache服务器
```shell
# vim /etc/apache2/apache2.conf
```
在开头添加
```shell
ServerName controller
```
完成安装
```shell
# service apache2 restart
# rm -f /var/lib/keystone/keystone.db
```
切换到普通用户
```shell
$ 
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
```
本指南有一个service 项目，你添加的每一个服务都有唯一的用户。普通的任务不应该使用具有特权的项目和用户。作为示例，本指南创建一个demo项目和用户。
```shell
$ openstack project create --domain default \
  --description "Service Project" service
$ openstack project create --domain default \
  --description "Demo Project" demo
$ openstack user create --domain default \
  --password-prompt demo
$ openstack role create user
$ openstack role add --project demo --user demo user
```
## 验证
出于安全性的原因，禁用掉暂时的认证令牌机制。编辑`/etc/keystone/keystone-paste.ini`文件，并从`[pipeline:public_api]`, `[pipeline:admin_api]`, 和`[pipeline:api_v3]`选项中删除`admin_token_auth`
```shell
$ unset OS_AUTH_URL OS_PASSWORD
$ openstack --os-auth-url http://controller:35357/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue

Password:
+------------+-----------------------------------------------------------------+
| Field      | Value                                                           |
+------------+-----------------------------------------------------------------+
| expires    | 2016-02-12T20:14:07.056119Z                                     |
| id         | gAAAAABWvi7_B8kKQD9wdXac8MoZiQldmjEO643d-e_j-XXq9AmIegIbA7UHGPv |
|            | atnN21qtOMjCFWX7BReJEQnVOAj3nclRQgAYRsfSU_MrsuWb4EDtnjU7HEpoBb4 |
|            | o6ozsA_NmFWEpLeKy0uNn_WeKbAhYygrsmQGA49dclHVnz-OMVLiyM9ws       |
| project_id | 343d245e850143a096806dfaefa9afdc                                |
| user_id    | ac3377633149401296f6c0d92d79dc16                                |
+------------+-----------------------------------------------------------------+

$ openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name demo --os-username demo token issue

Password:
+------------+-----------------------------------------------------------------+
| Field      | Value                                                           |
+------------+-----------------------------------------------------------------+
| expires    | 2016-02-12T20:15:39.014479Z                                     |
| id         | gAAAAABWvi9bsh7vkiby5BpCCnc-JkbGhm9wH3fabS_cY7uabOubesi-Me6IGWW |
|            | yQqNegDDZ5jw7grI26vvgy1J5nCVwZ_zFRqPiz_qhbq29mgbQLglbkq6FQvzBRQ |
|            | JcOzq3uwhzNxszJWmzGC7rJE_H0A_a3UFhqv8M4zMRYSbS2YF0MyFmp_U       |
| project_id | ed0b60bf607743088218b0a533d5943f                                |
| user_id    | 58126687cbcc4888bfa9ab73a2256f27                                |
+------------+-----------------------------------------------------------------+
```
## 创建OpenStack客户端环境脚本
在前面章节中，我们使用环境变量和命令的组合来配置认证服务，为了更加高效和方便，我们创建一个脚本方便以后的操作。这些脚本包括一些公共的操作，但是也支持自定义的操作。

创建并编辑`admin-openrc`文件，并添加以下内容：
```shell
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

创建并编辑`demo-openrc`文件，并添加以下内容：
```shell
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=DEMO_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
使用脚本:加载脚本文件更新环境变量，并请求一个认证令牌；
```shell
$ . admin-openrc
$ openstack token issue

+------------+-----------------------------------------------------------------+
| Field      | Value                                                           |
+------------+-----------------------------------------------------------------+
| expires    | 2016-02-12T20:44:35.659723Z                                     |
| id         | gAAAAABWvjYj-Zjfg8WXFaQnUd1DMYTBVrKw4h3fIagi5NoEmh21U72SrRv2trl |
|            | JWFYhLi2_uPR31Igf6A8mH2Rw9kv_bxNo1jbLNPLGzW_u5FC7InFqx0yYtTwa1e |
|            | eq2b0f6-18KZyQhs7F3teAta143kJEWuNEYET-y7u29y0be1_64KYkM7E       |
| project_id | 343d245e850143a096806dfaefa9afdc                                |
| user_id    | ac3377633149401296f6c0d92d79dc16                                |
+------------+-----------------------------------------------------------------+
```
