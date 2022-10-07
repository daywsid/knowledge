​                                

​                                                           **二进制部署1.23.4版本k8s集群**

# 一、系统安装及环境准备

## （一）、致谢

> 这篇文章参考了博客园的文章和老男孩王导的视频，在此表示感谢和致敬！[博客园文章地址](https://www.cnblogs.com/wgh2008/p/16001375.html)

## （二）、安装CentOS操作系统

> 系统镜像：CentOS-7-x86_64-DVD-2009.iso
> 安装过程略。

## （三）、环境准备

1. 修改主机名

```shell
hostnamectl set-hostname k8s11.host.com
hostnamectl set-hostname k8s12.host.com
hostnamectl set-hostname k8s21.host.com
hostnamectl set-hostname k8s22.host.com
hostnamectl set-hostname harbor.host.com
```

2. 修改IP地址

```shell
nmcli connection add con-name ens33 type ethernet ifname ens33 ipv4.method auto autoconnect yes
nmcli connection modify ens33 ipv4.addresses "192.168.3.11/24 192.168.3.1"  ipv4.method manual
nmcli connection reload ens33
nmcli connection up ens33
```

3. 关闭IPV6地址

```shell
cat /etc/default/grub            #GRUB_CMDLINE_LINUX中增加ipv6.disable=1

GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="ipv6.disable=1 crashkernel=auto spectre_v2=retpoline rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
```

```shell
grub2-mkconfig -o /boot/grub2/grub.cfg       #使用grub2-mkconfig命令使开机启动参数修改生效
```

4. 关闭SELinux

```shell
sed -i '/^SELINUX=/  s/enforcing/disabled/' /etc/selinux/config
```

5. 关闭防火墙

```shell
systemctl stop firewalld
systemctl disable firewalld
```

6. 关闭邮件服务

```shell
systemctl stop postfix
systemctl disable postfix
```

7. 安装常用软件

```shell
yum install -y wget net-tools telnet tree nmap sysstat lrzsz dos2unix bind-utils vim gcc gcc-c++ yum-utils
```

8. 调整base源、EPEL源，添加K8S源

```shell
mv /etc/yum.repos.d/CentOS-Base.repo{,.bak}                                             ## 备份
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo  ## 使用阿里镜像源
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo           ## 安装epel源

cat > /etc/yum.repos.d/k8s.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
EOF

yum clean all
yum makecache -y                     ## 清除系统yum缓存，重新生成
yum repolist enabled                 ## 查看系统可用yum源和所有yum源
```

9. 时间同步

```shell
timedatectl set-timezone Asia/Shanghai      ## 修改系统时间、时区
yum -y install chrony                       ## 安装chrony

cat /etc/chrony.conf
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server ntp1.aliyun.com
server ntp2.aliyun.com
server ntp3.aliyun.com                      ## 配置chrony

systemctl start chronyd                     ## 启动 chrony
systemctl enable chronyd                    ## 设为开机自启动
```

10. 关闭swap分区

```shell
sed -i '/swap/ s/^\(.*\)$/#&/g' /etc/fstab
```

11. 内核优化

```shell
cat /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0

sysctl --system 
```

12. 其他4台机器同样操作，然后重启机器



# 二、安装DNS服务

## （一）、安装DNS服务

>为什么要安装bind9？
>
>K8S中，使用Ingress进行7层流量调度，需要使用域名，进行7层调度。
>
>以前使用绑定host的方法，来进行域名和IP地址的解析。
>
>在K8S里，没有好的办法给容器绑定host，必须自建DNS，让容器能够服从DNS解析。
>
>DNS，就是把域名和IP地址绑定。
>
>在 k8s11主机上操作

1. 安装bind9

```shell
[root@k8s11 ~]# yum install bind -y
[root@k8s11 ~]# rpm -qa bind
bind-9.11.4-26.P2.el7_9.9.x86_64

[root@k8s11 ~]# rpm -qc bind
/etc/logrotate.d/named
/etc/named.conf
/etc/named.iscdlv.key
/etc/named.rfc1912.zones
/etc/named.root.key
/etc/rndc.conf
/etc/rndc.key
/etc/sysconfig/named
/var/named/named.ca
/var/named/named.empty
/var/named/named.localhost
/var/named/named.loopback
[root@k8s11 ~]# rpm -ql bind
[root@k8s11 ~]# rpm -q --scripts bind          查看bind安装时执行的脚本
```

2. 修改主配置文件

>编辑`/etc/named.conf`文件，修改如下部分。
>`listen-on port 53` 配置为本机的IP地址。
>`allow-query` 允许任意主机查询。
>`forwarders` 虚拟机的网关，可以访问外网。

```shell
vim /etc/named.conf

listen-on port 53 { 192.168.3.11; };
allow-query     { any; };
forwarders      { 192.168.3.1; };
recursion yes;
dnssec-enable no;
dnssec-validation no;

[root@k8s11 ~]# named-checkconf        语法检查
```

3. 区域配置文件

>配置主机域（host.com）和业务域（wg.com）。
>
>编辑`/etc/named.rfc1912.zones`文件，在文件最后增加下面内容。
>`file`：域配置文件名称。
>`allow-update`：填写DNS服务器的IP地址，允许dns服务器更新。

```shell
zone "host.com" IN {
    type master;
    file "host.com.zone";
    allow-update { 192.168.3.11; };
};

zone "wg.com" IN {
    type master;
    file "wg.com.zone";
    allow-update { 192.168.3.11; };
};
```

>生产上规划主机名，和业务没有任何关系。不会使用mysql01，hadoop01等等名称。
>
>主机名：地域+IP后两位
>
>主机域：用比较好记忆的，没有实际意义的名称，如host.com
>
>业务域：wg.com
>
>新建文件：`/var/named/host.com.zone`，文件内容如下：

```shell
vim /var/named/host.com.zone
$TTL 86400
@   IN  SOA     dns.host.com. dnsadmin.host.com. (
        2022081701  ;Serial
        3600        ;Refresh
        1800        ;Retry
        604800      ;Expire
        86400       ;Minimum TTL
)

              NS   dns.host.com.
$TTL 8640
dns           IN  A   192.168.3.11
k8s11         IN  A     192.168.3.11
k8s12         IN  A     192.168.3.12
k8s21         IN  A     192.168.3.21
k8s22         IN  A     192.168.3.22
harbor        IN  A     192.168.3.200
```

> 新建文件：`/var/named/wg.com.zone`，文件内容如下：

```shell
vim /var/named/wg.com.zone
$TTL 86400
@   IN  SOA     dns.wg.com. dnsadmin.wg.com. (
        2022081701  ;Serial
        3600        ;Refresh
        1800        ;Retry
        604800      ;Expire
        86400       ;Minimum TTL
)

              NS   dns.wg.com.
$TTL 8640
dns           IN  A   192.168.3.11
harbor        IN  A   192.168.3.200
```

> 语法检查

```shell
# named-checkconf /etc/named.conf
# named-checkzone wg.com.zone /var/named/wg.com.zone
# named-checkzone host.com.zone /var/named/host.com.zone
```

4. 启动named服务

```shell
# 启动服务
[root@k8s11 ~]# systemctl start named
# 开启自启动
[root@k8s11 ~]# systemctl enable named
# 查看网络监听端口
[root@k8s11 ~]# netstat -luntp | grep 53
```

5.  域名解析检查

```shell
[root@k8s11 ~]# dig -t A harbor.host.com @192.168.3.11 +short
#能正常解析，说明DNS服务以及能正常解析了
```

6. 配置客户端

>让客户端能正常使用DNS服务。
>
>一种方法上直接修改/etc/resolv.conf文件，另一种方法是修改网络配置文件中的DNS1，修改成DNS服务器地址。本例修改网络配置文件。
>
>在11主机上的resolv.conf文件中，添加search host.com

```shell
[root@k8s11 ~]# vim /etc/sysconfig/network-scripts/ifcfg-eth0
DNS1=192.168.3.11

[root@k8s11 ~]# systemctl restart network

[root@k8s11 ~]# cat /etc/resolv.conf
# Generated by NetworkManager
search host.com
nameserver 192.168.3.11

```



# 三、部署架构及根证书签发

## （一）、系统架构

系统架构图(没有修改IP地址，供参考)如下：

![1940699-20220314100236625-252675457](https://wsidtypora.oss-cn-beijing.aliyuncs.com/img/1940699-20220314100236625-252675457.jpg)

>本次部署使用了5台虚拟机，各主机功能分配如下：
>
>2台是代理，分别是4层和7层代理。L7反代Ingress，L4反代Apiserver。还用来安装DNS服务以及etcd。2个节点上面跑一个VIP，其ip地址是192.168.3.10。
>
>2台部署K8S核心服务，Master节点和Node节点部署在一起，这两台主机既充当主控节点，又充当运算节点，如果资源充分，主控节点和运算节点可以分开部署；
>
>1台作运维主机，主要功能上Docker资源仓库，K8S资源清单仓库，提供了K8S共享存储服务（NFS），还用来签发证书。etcd部署在12、21、22三台主机上

## （二）、集群网络规划

集群三条网络规划如下：
节点网络：192.168.3.0/24
Pod网络：172.16.0.0/16
Service网络：虚网络，地址为10.100.0.0/16

两台宿主机IP分别是：192.168.3.21和192.168.3.22
宿主机上Pod网络分别是：172.7.21.0/24和172.7.22.0/24

Pod网络和宿主机网络对应关系：Pod网络的第三位和宿主机网络的最后一位对应。这样可以很方便的定位Pod在哪台宿主机上。

## （三）、系统资源

系统资源如下图：

| 主机名 | IP            | 服务器类型     | 环境配置                                                     |
| ------ | ------------- | -------------- | ------------------------------------------------------------ |
|        | 192.168.3.10  | VIP            | 无                                                           |
| k8s11  | 192.168.3.11  | 负载均衡（主） | Bind、Nginx、Keepalived                                      |
| k8s12  | 192.168.3.12  | 负载均衡（从） | etcd、Nginx、Keepalived                                      |
| k8s21  | 192.168.3.21  | Master+Node    | Ingress、kubelet、kube-proxy、pod、etcd、apiserver、controller-manager、scheduler |
| k8s22  | 192.168.3.22  | Master+Node    | Ingress、kubelet、kube-proxy、pod、etcd、apiserver、controller-manager、scheduler |
| harbor | 192.168.3.200 | 运维主机       | cfssl、harbor、NFS                                           |

## （四）、证书签发环境

证书签发环境部署在运维主机200上。
证书保存路径：/opt/certs

1. 安装cfssl工具

cfssl版本：1.6.1

```shell
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64  -O /usr/bin/cfssl
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64  -O /usr/bin/cfssl-json
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl-certinfo_1.6.1_linux_amd64  -O /usr/bin/cfssl-certinfo
chmod +x /usr/bin/cfssl*
```

> + cfssl: 用于签发证书，输出json格式文本；
> + cfssl-json: 将cfssl签发生成的证书(json格式)变成文件承载式文件；
> + cfssl-certinfo: 验证查看证书信息。

2. cfssl工具使用说明

> cfssl工具的子命令包括：
>
> - genkey: 生成一个key(私钥)和CSR(证书签名请求)
> - certinfo: 输出给定证书的证书信息
> - gencert: 生成新的key(密钥)和签名证书，该命令的参数如下：
>   - -initca：初始化一个新ca，生成根CA时需要。
>   - -ca：指明ca的证书（ca.pem）
>   - -ca-key：指明ca的私钥文件（ca-key.pem）
>   - -config：指明证书请求csr的json文件（ca-config.json）
>   - -profile：与-config中的profile对应，是指根据config中的profile段来生成证书的相关信息

3. 生成根证书

自签证书时，首先需要一个根证书，也叫CA证书。

创建生成CA证书签名请求（CSR）的JSON配置文件，文件路径及内容：

`/opt/certs/ca-csr.json`

```json
{
    "CN": "kubernetes",
    "hosts": [
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "k8s",
            "OU": "system"
        }
    ],
    "ca": {
        "expiry": "438000h"
    }
}
```

> 说明：
>
> > - CN：Common Name，一般写域名。kube-apiserver从证书中提取该字段作为请求的用户名（User Name）。浏览器使用该字段验证网站是否合法。
> > - hosts：网络请求url中合法的主机名或域名。
> > - key：密钥信息，其内容一般也比较固定就是：{"algo":"rsa”,"size":2048}，表示使用的加密算法rsa,密文长度2048。
> > - names：证书对外公开显示的信息，常见的有：
> >   - C：Country，所在国家简称
> >   - ST：State，所在州/省份简称
> >   - L：Locality，所在地区/城市简称
> >   - O：Organization Name，组织名称或公司名称，kube-apiserver从证书中提取该字段作为请求用户所属的组（Group）
> >   - OU：Organization Unit Name，组织单位名称或公司部门
> >
> > expiry：过期时间，438000h表示有效期20年。

生成CA证书和私钥命令

```shell
cfssl gencert -initca ca-csr.json | cfssl-json -bare ca
```

> 生成的文件：
>
> > ca.pem ：CA根证书
> >
> > ca-key.pem：CA 根证书的私钥
>
> 至此，CA根证书及其私钥文件已经生成，可以用它们来签发其它证书了。

4. 创建CA根证书策略文件

CA根证书策略文件，一般命名为ca-config.json，用于配置根证书的使用场景（profile）和具体参数（usage、过期时间、服务端认证、客户端认证、加密等），后续签名其它证书时需要指定特定场景（profile）

`/opt/certs/ca-config.json`

```json
{
    "signing": {
        "default": {
            "expiry": "438000h"
        },
        "profiles": {
            "server": {
                "expiry": "438000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry":"438000h",
                "usages": [
                    "signing",
                    "key enchiperment",
                    "client auth"
                ]
            },
            "kubernetes": {
                "expiry":"438000h",
                "usages": [
                    "signing",
                    "key enchiperment",
                    "server auth",
                    "client auth"
                ]
            },            
            "peer": {
                "expiry": "438000h",
                "usages": [
                    "signing",
                    "key enchiperment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
```

> - profile：指定证书使用场景，后续签名生成证书及其私钥时需要指定场景名称
>   - server auth：表示 client 可以用该该证书对 server 提供的证书进行验证。
>   - client auth：表示 server 可以用该该证书对 client 提供的证书进行验证。
> - signing：表示该证书可用于签名其它证书，生成的ca.pem证书中CA=TRUE
>
> 文件中定义了多个profile，可以根据情况使用。
>
> - server：服务端使用，客户端以此验证服务端身份。
> - client：客户端使用，用于服务端认证客户端。
> - peer/kubernetes：双向证书，通信双方都需要证书，用于集群成员间通信。
>
> 后续签发证书时，需要指定文件名（ca-config.json）以及使用的profile，本例中，为了简化操作，全部使用kubernetes。



# 四、部署etcd集群

## （一）、集群规划

| 主机名         | 角色        | IP           |
| -------------- | ----------- | ------------ |
| k8s12.host.com | etcd leader | 192.168.3.12 |
| k8s21.host.com | etcd follow | 192.168.3.21 |
| k8s22.host.com | etcd follow | 192.168.3.22 |

以下在运维主机200上操作。

## （二）、创建生成自签证书签名请求CSR文件

创建etcd证书，客户端访问与节点互相访问使用同一套证书。

`/opt/certs/etcd-csr.json`

```json
{
    "CN": "k8s-etcd",
    "hosts": [
      "localhost",
      "0.0.0.0",
      "127.0.0.1",
      "192.168.3.11",
      "192.168.3.12",
      "192.168.3.21",
      "192.168.3.22"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "k8s",
            "OU": "system"
        }
    ]
}
```

> - CN
>   - 本处CN可随便定义
> - hosts
>   - etcd安装的主机IP地址，必须是IP地址，不能是网段，可能的主机都列出来
>   - 这里列出的主机和`--advertise-client-urls`定义有关
>   - 如果有新增主机不在列表中，需要重新签发证书
> - names中的配置
>   - C：国家
>   - ST：州/省
>   - L：市
>   - O：组织，二进制部署随便定义，使用kubeadm时，要求值为`system:masters`
>   - OU：部门

## （三）、生成etcd证书和私钥

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json|cfssl-json -bare etcd
```

## （四）、检查生成的证书、私钥

```shell
[root@harbor certs]# ll etcd*
-rw-r--r-- 1 root root  400 Mar 12 19:06 etcd-csr.json
-rw------- 1 root root 1675 Mar 12 19:07 etcd-key.pem
-rw-r--r-- 1 root root 1078 Mar 12 19:07 etcd.csr
-rw-r--r-- 1 root root 1448 Mar 12 19:07 etcd.pem
```



## （五）、创建etcd用户

> etcd程序使用etcd用户启动，需要创建该用户。
>
> 在12、21、22主机上操作。

```shell
useradd -s /sbin/nologin -M etcd
```



## （六）、下载软件、解压，做软链接

> etcd版本：3.4.20，提前下载好并上传到主机的/opt/src目录下。
>
> 在12、21、22主机上操作。

```shell
mkdir /opt/src
wget https://github.com/etcd-io/etcd/releases/download/v3.4.20/etcd-v3.4.20-linux-amd64.tar.gz -O /opt/src
tar -zxvf etcd-v3.4.20-linux-amd64.tar.gz -C /opt
cd /opt
mv etcd-v3.4.20-linux-amd64 etcd-v3.4.20
ln -s etcd-v3.4.20 etcd
cp /opt/etcd-v3.4.20/etcdctl /usr/local/bin/
cp /opt/etcd-v3.4.20/etcd /usr/local/bin/
```



## （七）、创建目录，拷贝证书，私钥

>在12、21、22主机上操作。
>
>创建目录
>
>/opt/etcd/certs保存etcd集群通信使用证书和私钥。
>
>/data/etcd保存etcd数据库
>
>/data/logs/etcd-server保存etcd日志文件。

```shell
mkdir -p /opt/etcd/certs /data/etcd/ /data/logs/etcd-server
```

拷贝证书和私钥：把以下根CA证书、etcd证书和etcd私钥三个文件从200主机上拷贝过来

注意：私钥的权限 400

```shell
[root@k8s12 certs]# ll
总用量 12
-rw-r--r-- 1 etcd etcd 1314 9月  14 16:54 ca.pem
-rw------- 1 etcd etcd 1679 9月  14 16:54 etcd-key.pem
-rw-r--r-- 1 etcd etcd 1472 9月  14 16:54 etcd.pem
```



## （八）、创建etcd启动脚本

在12、21、22主机上操作。

`/opt/etcd/etcd-server-startup.sh`

```shell
#!/bin/bash
./etcd --name etcd-server-3.12 \
       --data-dir /data/etcd/etcd-server \
       --listen-peer-urls https://192.168.3.12:2380 \
       --listen-client-urls https://192.168.3.12:2379,http://127.0.0.1:2379 \
       --initial-advertise-peer-urls https://192.168.3.12:2380 \
       --initial-cluster etcd-server-3.12=https://192.168.3.12:2380,etcd-server-3.21=https://192.168.3.21:2380,etcd-server-3.22=https://192.168.3.22:2380 \
       --initial-cluster-token etcd-cluster-k8s \
       --initial-cluster-state new           \
       --advertise-client-urls https://192.168.3.12:2379,http://127.0.0.1:2379 \
       --client-cert-auth=true \
       --trusted-ca-file=/opt/etcd/certs/ca.pem \
       --cert-file=/opt/etcd/certs/etcd.pem \
       --key-file=/opt/etcd/certs/etcd-key.pem \
       --peer-client-cert-auth=true \
       --peer-trusted-ca-file=/opt/etcd/certs/ca.pem \
       --peer-cert-file=/opt/etcd/certs/etcd.pem \
       --peer-key-file=/opt/etcd/certs/etcd-key.pem \
       --log-output stdout
       --listen-metrics-urls=https://192.168.3.12:2381 \
       --enable-pprof=false

#!/bin/bash
./etcd --name etcd-server-3.21 \
       --data-dir /data/etcd/etcd-server \
       --listen-peer-urls https://192.168.3.21:2380 \
       --listen-client-urls https://192.168.3.21:2379,http://127.0.0.1:2379 \
       --initial-advertise-peer-urls https://192.168.3.21:2380 \
       --initial-cluster etcd-server-3.12=https://192.168.3.12:2380,etcd-server-3.21=https://192.168.3.21:2380,etcd-server-3.22=https://192.168.3.22:2380 \
       --initial-cluster-token etcd-cluster-k8s \
       --initial-cluster-state new           \
       --advertise-client-urls https://192.168.3.21:2379,http://127.0.0.1:2379 \
       --client-cert-auth=true \
       --trusted-ca-file=/opt/etcd/certs/ca.pem \
       --cert-file=/opt/etcd/certs/etcd.pem \
       --key-file=/opt/etcd/certs/etcd-key.pem \
       --peer-client-cert-auth=true \
       --peer-trusted-ca-file=/opt/etcd/certs/ca.pem \
       --peer-cert-file=/opt/etcd/certs/etcd.pem \
       --peer-key-file=/opt/etcd/certs/etcd-key.pem \
       --log-output stdout
       --listen-metrics-urls=https://192.168.3.21:2381 \
       --enable-pprof=false

#!/bin/bash
./etcd --name etcd-server-3.22 \
       --data-dir /data/etcd/etcd-server \
       --listen-peer-urls https://192.168.3.22:2380 \
       --listen-client-urls https://192.168.3.22:2379,http://127.0.0.1:2379 \
       --initial-advertise-peer-urls https://192.168.3.22:2380 \
       --initial-cluster etcd-server-3.12=https://192.168.3.12:2380,etcd-server-3.21=https://192.168.3.21:2380,etcd-server-3.22=https://192.168.3.22:2380 \
       --initial-cluster-token etcd-cluster-k8s \
       --initial-cluster-state new           \
       --advertise-client-urls https://192.168.3.22:2379,http://127.0.0.1:2379 \
       --client-cert-auth=true \
       --trusted-ca-file=/opt/etcd/certs/ca.pem \
       --cert-file=/opt/etcd/certs/etcd.pem \
       --key-file=/opt/etcd/certs/etcd-key.pem \
       --peer-client-cert-auth=true \
       --peer-trusted-ca-file=/opt/etcd/certs/ca.pem \
       --peer-cert-file=/opt/etcd/certs/etcd.pem \
       --peer-key-file=/opt/etcd/certs/etcd-key.pem \
       --log-output stdout
       --listen-metrics-urls=https://192.168.3.22:2381 \
       --enable-pprof=false

```

> etcd成员之间通信，2380端口
>
> 外部访问etcd，2379端口
>
> 参数说明

```shell
name：etcd节点成员名称，在一个etcd集群中必须唯一性，可使用Hostname或者machine-id
data-dir：etcd数据保存目录
listen-peer-urls：和其它成员节点间通信地址，每个节点不同，必须使用IP，使用域名无效。
listen-client-urls：对外提供服务的地址，127.0.0.1允许非安全方式访问，使用域名无效。
initial-advertise-peer-urls：节点监听地址，集群成员使用该地址访问本节点，并会通告集群其它节点
initial-cluster：集群中所有节点信息，格式为：节点名称+监听的本地端口，多个节点用逗号隔开，即：name=https://initial-advertise-peer-urls
initial-cluster-state：加入集群的当前状态，new是新集群，existing表示加入已有集群
initial-cluster-token：集群引导创建期间所使用的TOKEN。
advertise-client-urls：节点成员客户端url列表，对外公告此节点客户端监听地址，可以使用域名
client-cert-auth：客户端访问本节点时，是否需要证书认证
trusted-ca-file：本节点2379使用的CA证书
cert-file：本节点2379所使用的证书
key-file：本节点2379所使用的密钥
peer-client-cert-auth：集群成员访问本节点时，是否需要证书认证
peer-trusted-ca-file：本节点2380所使用的CA证书
peer-cert-file：本节点2380所使用的证书
peer-key-file：本节点2380所使用的密钥
log-outputs：日志输出方式
listen-metrics-urls：metrics数据的获取地址
enable-pprof：通过`url/debug/pprof/`获取启动时的状态，建议禁用
```



## （九）、调整权限(12、21、22都操作)

```shell
[root@k8s12 etcd]# chmod +x etcd-server-startup.sh
[root@k8s12 etcd]# chown -R etcd.etcd /opt/etcd-v3.4.20/ /data/etcd /data/logs/
[root@k8s12 etcd]# chown -R etcd.etcd /opt/etcd
```



## （十）、安装supervisor

```shell
[root@k8s12 etcd]# yum install supervisor -y
[root@k8s12 etcd]# systemctl enable supervisord  --now
[root@k8s12 etcd]# systemctl status supervisord
```





## （十一）、创建etcd-server启动配置文件

在12、21、22主机上操作。

`/etc/supervisord.d/etcd-server.ini`

```shell
#12机器
[program:etcd-server-3.12]
command=/opt/etcd/etcd-server-startup.sh
numprocs=1
directory=/opt/etcd
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=etcd
redirect_stderr=true
stdout_logfile=/data/logs/etcd-server/etcd-stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout__capture_maxbytes=10MB
stdout_events_enabled=false


#21机器
[program:etcd-server-3.21]
command=/opt/etcd/etcd-server-startup.sh
numprocs=1
directory=/opt/etcd
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=etcd
redirect_stderr=true
stdout_logfile=/data/logs/etcd-server/etcd-stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout__capture_maxbytes=10MB
stdout_events_enabled=false


#22机器
[program:etcd-server-3.22]
command=/opt/etcd/etcd-server-startup.sh
numprocs=1
directory=/opt/etcd
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=etcd
redirect_stderr=true
stdout_logfile=/data/logs/etcd-server/etcd-stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout__capture_maxbytes=10MB
stdout_events_enabled=false

```



## （十二）、启动etcd服务并检查

```shell
[root@k8s12 etcd]# supervisorctl update
etcd-server-3.12: added process group
[root@k8s12 etcd]# supervisorctl status
etcd-server-3.12                RUNNING   pid 12270, uptime 0:00:48
[root@k8s12 etcd]# netstat -luntp | grep etcd
```

确保监听了2379、2380和2381三个端口。



## （十三）、检查集群状态

```shell
# 12、21、22都部署完成以后查看集群节点
[root@k8s12 ~]# etcdctl --cacert=/opt/etcd/certs/ca.pem --cert=/opt/etcd/certs/etcd.pem --key=/opt/etcd/certs/etcd-key.pem --endpoints="https://192.168.3.12:2379,https://192.168.3.21:2379,https://192.168.3.22:2379" member list -w table

[root@k8s12 ~]# etcdctl --cacert=/opt/etcd/certs/ca.pem --cert=/opt/etcd/certs/etcd.pem --key=/opt/etcd/certs/etcd-key.pem --endpoints="https://192.168.3.12:2379,https://192.168.3.21:2379,https://192.168.3.22:2379" endpoint status -w table

[root@k8s12 ~]# etcdctl --cacert=/opt/etcd/certs/ca.pem --cert=/opt/etcd/certs/etcd.pem --key=/opt/etcd/certs/etcd-key.pem --endpoints="https://192.168.3.12:2379,https://192.168.3.21:2379,https://192.168.3.22:2379" endpoint health -w table
```

至此，etcd集群部署完成。



## 备注etcd受损恢复：

```shell
etcdctl --cacert=/opt/etcd/certs/ca.pem --cert=/opt/etcd/certs/etcd.pem --key=/opt/etcd/certs/etcd-key.pem --endpoints="https://192.168.3.12:2379,https://192.168.3.21:2379,https://192.168.3.22:2379" member remove f59e2de0c3bc3821

etcdctl --cacert=/opt/etcd/certs/ca.pem --cert=/opt/etcd/certs/etcd.pem --key=/opt/etcd/certs/etcd-key.pem --endpoints="https://192.168.3.12:2379,https://192.168.3.21:2379,https://192.168.3.22:2379" member add etcd-server-3.12 --peer-urls=https://192.168.3.12:2380

删除受损etcd节点的数据,修改etcd启动参数，重启etcd
将etcd的--initial-cluster-state启动参数，改为--initial-cluster-state=existing
```



# 五、部署Master节点服务

## （一）、安装Docker

在21、22、200三台机器上安装Docker。安装命令：

```shell
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun   #脚本安装

#手动安装，卸载旧版本
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
#设置stable镜像仓库
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
#使用国内阿里云仓库链接下载，更新yum软件包索引
yum makecache fast
#安装Docker 引擎
yum -y install docker-ce
#yum -y install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

1. 配置Docker

`/etc/docker/daemon.json`

```shell
{
    "graph": "/data/docker",
    "storage-driver": "overlay2",
    "insecure-registries": ["registry.access.redhat.com","quay.io","harbor.wg.com"],
    "registry-mirrors": ["https://q2gr04ke.mirror.aliyuncs.com"],
    "bip": "172.16.21.1/24",
    "exec-opts": ["native.cgroupdriver=systemd"],
    "live-restore": true
}
```

说明：

> exec-opts：CPU/MEM的资源管理方式
>
> registry-mirrors：镜像源
>
> insecure-registries：信任的HTTP镜像仓库
>
> bip根据不同的主机修改：
>
> 21:172.7.21.1/24
>
> 22:172.7.22.1/24
>
> 200: 172.7.200.1/24

创建目录

```shell
[root@k8s21 ~]# mkdir -pv /data/docker
```

2. 启动docker

```shell
~]# systemctl enable docker
~]# systemctl start docker
~]# systemctl status docker -l
~]# docker info
~]# docker version
```



## （二）、部署kube-apiserver集群

1. ###  集群规划

   | 主机名         | 角色                         | IP           |
   | -------------- | ---------------------------- | ------------ |
   | k8s21.host.com | kube-apiserver               | 192.168.3.21 |
   | k8s22.host.com | kube-apiserver               | 192.168.3.22 |
   | k8s11.host.com | 4层负载均衡 Nginx+Keepalived | 192.168.3.11 |
   | k8s12.host.com | 4层负载均衡 Nginx+Keepalived | 192.168.3.12 |

   注意：这里192.168.3.11和192.168.3.12使用nginx做4层负载均衡，用keepalived跑一个vip：192.168.3.10，代理两个kube-apiserver，实现高可用。

2. 下载软件、解压、做软链接

在21、22主机上操作

本例使用1.23.4版本。[下载地址](https://dl.k8s.io/v1.23.4/kubernetes-server-linux-amd64.tar.gz)

```shell
[root@k8s21 src]# tar zxvf kubernetes-server-linux-amd64.tar.gz -C /opt/
[root@k8s21 opt]# mv /opt/kubernetes /opt/kubernetes-v1.23.4
[root@k8s21 opt]# ln -s /opt/kubernetes-v1.23.4  /opt/kubernetes
[root@k8s21 opt]# cd /opt/kubernetes 
[root@k8s21 kubernetes]# rm -rf kubernetes-src.tar.gz 
[root@k8s21 kubernetes]# cd server/bin
[root@k8s21 bin]# rm *.tar -f
[root@k8s21 bin]# rm *_tag -f
[root@k8s21 opt]# vim /etc/profile
export PATH=$PATH:/opt/etcd:/opt/kubernetes/server/bin
[root@k8s21 opt]# source /etc/profile
```

3. 签发kube-apiserver证书

在200主机上操作

`/opt/certs/kube-apiserver-csr.json`

```json
{
    "CN": "kube-apiserver",
    "hosts": [
        "127.0.0.1",
        "10.100.0.1",
        "kubernetes",        
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local",
        "192.168.3.10",
        "192.168.3.21",
        "192.168.3.22",
        "192.168.3.23"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "system:masters",
            "OU": "system"            
        }
    ]
}
```

> 说明：
>
> - CN：K8S会提取CN字段的值作为用户名，实际是指K8S的"RoleBinding/ClusterRoleBinding"资源中，“subjects.kind”的值为“User"，
> - hosts：包括所有Master节点的IP地址，LB节点、LB集群节点、ClusterIP的首个IP，K8S的“ClusterIP”的范围在“--service-cluster-ip-range”中指定，取值为10.100.0.0/16，此处配置为10.100.0.1
> - names
>   - C：CN
>   - ST：
>   - L：
>   - O：“system:masters”，定义“O”值的原因：apiserver向kubelet发起请求时，将复用此证书，参看官方文档。K8S默认会提取“O”字段的值作为组，这实际是指K8S的“RoleBinding/ClusterRoleBinding”资源中"subjects.kind"的值为“Group”

生成证书

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-apiserver-csr.json|cfssl-json -bare kube-apiserver
```

4. 拷贝证书至各运算节点

在21、22主机上操作，把6张证书和密钥从200主机上拷贝到certs目录下

```shell
cd /opt/kubernetes/server/bin/certs
[root@k8s21 certs]# ll
total 24
-rw------- 1 root root 1675 Sep 15 23:21 ca-key.pem
-rw-r--r-- 1 root root 1314 Sep 15 23:21 ca.pem
-rw------- 1 root root 1679 Sep 15 23:21 etcd-key.pem
-rw-r--r-- 1 root root 1472 Sep 15 23:21 etcd.pem
-rw------- 1 root root 1679 Sep 15 23:21 kube-apiserver-key.pem
-rw-r--r-- 1 root root 1659 Sep 15 23:21 kube-apiserver.pem
```

5. 生成token.csv文件

该文件的作用是，在工作节点（kubelet）加入K8S集群的过程中，向kube-apiserver申请签发证书。

`/opt/kubernetes/bin/certs/kube-apiserver.token.csv`

```shell
[root@k8s21 certs]# head -c 16 /dev/urandom | od -An -t x | tr -d " "
946167f96df0137ae70e0502ad32a94e
[root@k8s21 certs]# echo 946167f96df0137ae70e0502ad32a94e,kubelet-bootstrap,10001,"system:kubelet-bootstrap" > kube-apiserver.token.csv
[root@k8s21 certs]# cat kube-apiserver.
kube-apiserver.pem        kube-apiserver.token.csv  
[root@k8s21 certs]# cat kube-apiserver.token.csv 
946167f96df0137ae70e0502ad32a94e,kubelet-bootstrap,10001,system:kubelet-bootstrap
```

6. 创建启动脚本

在21、22主机上操作， 创建启动脚本

`/opt/kubernetes/server/bin/kube-apiserver-startup.sh`

```shell
#21机器
#!/bin/bash
./kube-apiserver \
  --runtime-config=api/all=true \
  --anonymous-auth=false \
  --bind-address=0.0.0.0 \
  --advertise-address=192.168.3.21 \
  --secure-port=6443 \
  --tls-cert-file=./certs/kube-apiserver.pem \
  --tls-private-key-file=./certs/kube-apiserver-key.pem \
  --client-ca-file=./certs/ca.pem \
  --etcd-cafile=./certs/ca.pem \
  --etcd-certfile=./certs/etcd.pem \
  --etcd-keyfile=./certs/etcd-key.pem \
  --etcd-servers=https://192.168.3.12:2379,https://192.168.3.21:2379,https://192.168.3.22:2379 \
  --kubelet-client-certificate=./certs/kube-apiserver.pem \
  --kubelet-client-key=./certs/kube-apiserver-key.pem \
  --service-account-key-file=./certs/ca.pem \
  --service-account-signing-key-file=./certs/ca-key.pem \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \
  --enable-bootstrap-token-auth=true \
  --token-auth-file=./certs/kube-apiserver.token.csv \
  --allow-privileged=true \
  --service-cluster-ip-range=10.100.0.0/16 \
  --service-node-port-range=8000-20000 \
  --authorization-mode=RBAC,Node \
  --enable-aggregator-routing=true \
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
  --v=2 \
  --audit-log-path=/data/logs/kubernetes/kube-apiserver/audit-log \
  --log-dir=/data/logs/kubernetes/kube-apiserver
  
#22机器
#!/bin/bash
./kube-apiserver \
  --runtime-config=api/all=true \
  --anonymous-auth=false \
  --bind-address=0.0.0.0 \
  --advertise-address=192.168.3.22 \
  --secure-port=6443 \
  --tls-cert-file=./certs/kube-apiserver.pem \
  --tls-private-key-file=./certs/kube-apiserver-key.pem \
  --client-ca-file=./certs/ca.pem \
  --etcd-cafile=./certs/ca.pem \
  --etcd-certfile=./certs/etcd.pem \
  --etcd-keyfile=./certs/etcd-key.pem \
  --etcd-servers=https://192.168.3.12:2379,https://192.168.3.21:2379,https://192.168.3.22:2379 \
  --kubelet-client-certificate=./certs/kube-apiserver.pem \
  --kubelet-client-key=./certs/kube-apiserver-key.pem \
  --service-account-key-file=./certs/ca.pem \
  --service-account-signing-key-file=./certs/ca-key.pem \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \
  --enable-bootstrap-token-auth=true \
  --token-auth-file=./certs/kube-apiserver.token.csv \
  --allow-privileged=true \
  --service-cluster-ip-range=10.100.0.0/16 \
  --service-node-port-range=8000-20000 \
  --authorization-mode=RBAC,Node \
  --enable-aggregator-routing=true \
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
  --v=2 \
  --audit-log-path=/data/logs/kubernetes/kube-apiserver/audit-log \
  --log-dir=/data/logs/kubernetes/kube-apiserver
```

7. 调整权限和目录

```shell
[root@k8s21 bin]# chmod +x kube-apiserver-startup.sh
[root@k8s21 bin]# mkdir -pv /data/logs/kubernetes/kube-apiserver
```

8. 创建supervisor配置

`/etc/supervisord.d/kube-apiserver.ini`

```shell
#21机器
[program:kube-apiserver-3.21]
command=/opt/kubernetes/server/bin/kube-apiserver-startup.sh
numprocs=1
directory=/opt/kubernetes/serrver/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-apiserver/apiserver-stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=5
stdout_capture_maxbytes=1MB
stdout_events_enabled=false


#22机器
[program:kube-apiserver-3.22]
command=/opt/kubernetes/server/bin/kube-apiserver-startup.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-apiserver/apiserver-stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=5
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
```

9. 启动服务并检查

```shell
[root@k8s21 bin]# supervisorctl update
[root@k8s21 bin]# supervisorctl status
etcd-server-3.21                RUNNING   pid 12536, uptime 2:29:07
kube-apiserver-3.21             RUNNING   pid 13122, uptime 0:00:40
[root@k8s21 bin]# netstat -luntp|grep kube-apiser
tcp        0      0 0.0.0.0:6443            0.0.0.0:*               LISTEN      11338/./kube-apiser
```

10. 检查集群状态

```shell


[root@k8s21 bin]# curl --insecure https://192.168.3.21:6443/
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}[root@k8s21 bin]# curl --insecure https://192.168.3.22:6443/
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
```

11. 配置4层反向代理

apiserver监听端口为6443

用keepalived跑一个192.168.3.10的vip

用192.168.3.10上的7443，反向代理192.168.3.21和192.168.3.22上的6443端口。

下面的操作在11和12主机上进行。

**安装nginx**

```shell
~]# yum install nginx -y
~]# yum install nginx-mod-stream -y
```

**配置nginx**

把下面内容，添加到`/etc/nginx/nginx.conf`文件最后，也就是并列在http模块的后面。

```shell
stream {
    upstream kube-apiserver {
        server 192.168.3.21:6443    max_fails=3 fail_timeout=30s;
        server 192.168.3.22:6443    max_fails=3 fail_timeout=30s;
    }
    server {
        listen 7443;
        proxy_connect_timeout 2s;
        proxy_timeout 900s;
        proxy_pass kube-apiserver;
    }
}
```

```shell
[root@k8s12 ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

**启动nginx**

```shell
~]# systemctl start nginx
~]# systemctl enable nginx
~]# systemctl status nginx
```

检查状态

```shell
[root@k8s11 ~]# netstat -luntp | grep nginx
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      16734/nginx: master 
tcp        0      0 0.0.0.0:7443            0.0.0.0:*               LISTEN      16734/nginx: master 
[root@k8s11 ~]# 
```

**安装keepalived**

```shell
[root@k8s11 ~]# yum install keepalived -y
[root@k8s12 ~]# yum install keepalived -y
```

创建监听脚本

`/etc/keepalived/check_port.sh`

```shell
#!/bin/bash
CHK_PORT=$1
if [ -n "$CHK_PORT" ];then
    PORT_PROCESS=`ss -lnt | grep $CHK_PORT | wc -l`
    if [ $PORT_PROCESS -eq 0 ];then
        echo "Port $CHK_PORT Is Not Used, End."
        exit 1
    fi
else
    echo "Check Port Cant Be Empty!"
fi
```

增加执行权限

```shell
~]# chmod +x /etc/keepalived/check_port.sh
```

测试脚本

```shell
[root@k8s11 ~]# /etc/keepalived/check_port.sh
Check Port Cant Be Empty!
[root@k8s11 ~]# /etc/keepalived/check_port.sh 7443
[root@k8s11 ~]# echo $?
0
[root@k8s11 ~]# /etc/keepalived/check_port.sh 7445
Port 7445 Is Not Used, End.
[root@k8s11 ~]# echo $?
1
[root@k8s11 ~]# 
```

**keepalived主配置**

在11主机上操作，删除原配置文件

`/etc/keepalived/keepalived.conf`

```shell
! Configuration File for keepalived

global_defs {
    router_id 192.168.3.11
}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_port.sh 7443"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 251
    priority 100
    advert_int 1
    mcast_src_ip 192.168.3.11
    nopreempt  #设置非抢占式，当主服务down，vip漂移到备机，当主机服务up，vip依然在备机上

    authentication {
        auth_type PASS
        auth_pass 11111111
    }

    track_script {
        chk_nginx
    }

    virtual_ipaddress {
        192.168.3.10
    }
}
```

**keepalived从配置**

在12主机上操作，删除原配置文件

`/etc/keepalived/keepalived.conf`

```shell
! Configuration File for keepalived
 
global_defs {
    router_id 192.168.3.12
}
 
vrrp_script chk_nginx {
    script "/etc/keepalived/check_port.sh 7443"
    interval 2
    weight -20
}
 
vrrp_instance VI_1 {
    state BACKUP
    interface ens33                                 
    virtual_router_id 251
    priority 90
    advert_int 1
    mcast_src_ip 192.168.3.12
    !  注意 备机上不能有 nopreempt 配置
    authentication {
        auth_type PASS
        auth_pass 11111111
    }
 
    track_script {
        chk_nginx
    }
 
    virtual_ipaddress {
        192.168.3.10
    }
}
```

12. 启动代理并检查

在11主机上操作

```shell
[root@k8s11 ~]# systemctl enable keepalived.service 
Created symlink from /etc/systemd/system/multi-user.target.wants/keepalived.service to /usr/lib/systemd/system/keepalived.service.
[root@k8s11 ~]# systemctl start keepalived.service 
[root@k8s11 ~]# systemctl status keepalived.service 
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-09-18 23:46:10 CST; 6s ago
  Process: 17131 ExecStart=/usr/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 17132 (keepalived)
    Tasks: 3
   CGroup: /system.slice/keepalived.service
           ├─17132 /usr/sbin/keepalived -D
           ├─17133 /usr/sbin/keepalived -D
           └─17134 /usr/sbin/keepalived -D

Sep 18 23:46:10 k8s11.host.com Keepalived_healthcheckers[17133]: Opening file '/etc/keepalived/keepalived.conf'.
Sep 18 23:46:11 k8s11.host.com Keepalived_vrrp[17134]: VRRP_Instance(VI_1) Transition to MASTER STATE
Sep 18 23:46:12 k8s11.host.com Keepalived_vrrp[17134]: VRRP_Instance(VI_1) Entering MASTER STATE
Sep 18 23:46:12 k8s11.host.com Keepalived_vrrp[17134]: VRRP_Instance(VI_1) setting protocol VIPs.
Sep 18 23:46:12 k8s11.host.com Keepalived_vrrp[17134]: Sending gratuitous ARP on ens33 for 192.168.3.10
Sep 18 23:46:12 k8s11.host.com Keepalived_vrrp[17134]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs o....10
Sep 18 23:46:12 k8s11.host.com Keepalived_vrrp[17134]: Sending gratuitous ARP on ens33 for 192.168.3.10
Sep 18 23:46:12 k8s11.host.com Keepalived_vrrp[17134]: Sending gratuitous ARP on ens33 for 192.168.3.10
Sep 18 23:46:12 k8s11.host.com Keepalived_vrrp[17134]: Sending gratuitous ARP on ens33 for 192.168.3.10
Sep 18 23:46:12 k8s11.host.com Keepalived_vrrp[17134]: Sending gratuitous ARP on ens33 for 192.168.3.10
Hint: Some lines were ellipsized, use -l to show in full.
[root@k8s11 ~]# 
```

在12主机上操作

```shell
[root@k8s12 ~]# systemctl enable keepalived.service 
[root@k8s12 ~]# systemctl start keepalived.service 
[root@k8s12 ~]# systemctl status keepalived.service 
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-09-18 23:47:48 CST; 5s ago
  Process: 17952 ExecStart=/usr/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 17953 (keepalived)
    Tasks: 3
   CGroup: /system.slice/keepalived.service
           ├─17953 /usr/sbin/keepalived -D
           ├─17954 /usr/sbin/keepalived -D
           └─17955 /usr/sbin/keepalived -D

Sep 18 23:47:48 k8s12.host.com Keepalived_vrrp[17955]: SECURITY VIOLATION - scripts are being executed but sc...ed.
Sep 18 23:47:48 k8s12.host.com Keepalived_vrrp[17955]: VRRP_Instance(VI_1) removing protocol VIPs.
Sep 18 23:47:48 k8s12.host.com Keepalived_vrrp[17955]: Using LinkWatch kernel netlink reflector...
Sep 18 23:47:48 k8s12.host.com Keepalived_vrrp[17955]: VRRP_Instance(VI_1) Entering BACKUP STATE
Sep 18 23:47:48 k8s12.host.com Keepalived_vrrp[17955]: VRRP sockpool: [ifindex(2), proto(112), unicast(0), fd...1)]
Sep 18 23:47:48 k8s12.host.com Keepalived_healthcheckers[17954]: Initializing ipvs
Sep 18 23:47:48 k8s12.host.com systemd[1]: Started LVS and VRRP High Availability Monitor.
Sep 18 23:47:48 k8s12.host.com Keepalived_vrrp[17955]: VRRP_Script(chk_nginx) succeeded
Sep 18 23:47:48 k8s12.host.com Keepalived_healthcheckers[17954]: Opening file '/etc/keepalived/keepalived.conf'.
Sep 18 23:47:48 k8s12.host.com Keepalived_healthcheckers[17954]: Unknown keyword 'onfiguration'
Hint: Some lines were ellipsized, use -l to show in full.
[root@k8s12 ~]# 
```

在11上用`ip addr`命令，能看到VIP

```shell
[root@k8s11 ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:75:c1:63 brd ff:ff:ff:ff:ff:ff
    inet 192.168.3.11/24 brd 192.168.3.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.3.10/32 scope global ens33
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:00:07:4f brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN group default qlen 1000
    link/ether 52:54:00:00:07:4f brd ff:ff:ff:ff:ff:ff
[root@k8s11 ~]# 
```

Nginx+Keepalived高可用测试

在11主机上，停止nginx，vip不存在了。

```shell
[root@k8s11 ~]# systemctl stop nginx
[root@k8s11 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:75:c1:63 brd ff:ff:ff:ff:ff:ff
    inet 192.168.3.11/24 brd 192.168.3.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:00:07:4f brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN group default qlen 1000
    link/ether 52:54:00:00:07:4f brd ff:ff:ff:ff:ff:ff
[root@k8s11 ~]# 
```

12主机上查看，vip跑在了12主机上

```shell
[root@k8s12 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:05:68:09 brd ff:ff:ff:ff:ff:ff
    inet 192.168.3.12/24 brd 192.168.3.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.3.10/32 scope global ens33
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:3f:c4:20 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN group default qlen 1000
    link/ether 52:54:00:3f:c4:20 brd ff:ff:ff:ff:ff:ff
[root@k8s12 ~]# 
```

重启nginx，vip还在12主机上。

这是因为在11上配置了 nopreempt ，设置非抢占式，当主服务down，vip漂移到备机，当主机服务up，vip依然在备机上。

如果要想vip回到11上，重新启动keepalived。



## （三）、部署kubectl组件

1.  集群规划

| 主机名         | 角色    | IP           |
| -------------- | ------- | ------------ |
| k8s21.host.com | kubectl | 192.168.3.21 |
| k8s22.host.com | kubectl | 192.168.3.22 |

2. 签发kubectl证书

在运维主机200上操作

生成kubectl证书请求csr文件

`/opt/certs/kubectl-csr.json`

```shell
{
    "CN": "clusteradmin",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "system:masters",
            "OU": "system"            
        }
    ]
}
```

> 说明
>
> - CN：kubectl证书中的CN值没有意义，随便取值
> - O：因为希望使用kubectl时有完整的集群操作权限，所以取值为“system:masters”，K8S默认会提取O字段的值作为组，这实际是指K8S里“RoleBinding/ClusterRoleBinding”资源中"subjects.kind"的值为“Group”
> - 后续 kube-apiserver 使用 RBAC 对客户端(如 kubelet、kube-proxy、Pod)请求进行授权；
>   kube-apiserver 预定义了一些 RBAC 使用的 RoleBindings，如 cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver 的所有 API的权限；
> - O指定该证书的 Group 为 system:masters，kubelet 使用该证书访问 kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限；
> - 这个证书，是将来生成管理员用的kube config 配置文件用的，现在我们一般建议使用RBAC 来对kubernetes 进行角色权限控制， kubernetes 将证书中的CN 字段 作为User， O 字段作为 Group；
>   "O": "system:masters", 必须是system:masters，否则后面kubectl create clusterrolebinding报错。

生成证书

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubectl-csr.json | cfssl-json -bare kubectl
```

3. 把证书拷贝到21和22主机上

```shell
[root@harbor certs]# scp kubectl*.pem root@k8s21:/opt/kubernetes/server/bin/certs/
[root@harbor certs]# scp kubectl*.pem root@k8s22:/opt/kubernetes/server/bin/certs/
```

4. 生成kubeconfig配置文件

生成kubectl组件的kubectl.kubeconfig配置文件，该文件包含访问kube-apiseerver的所有信息，如kube-apiserver的地址，CA证书和自身使用的证书。

有了这个文件，便可以在任何机器上以超级管理员身份对K8S集群做任何操作，请务必保证此文件的安全性。

kubectl命令默认使用的配置文件为：`~/.kube/config`

以下在21主机上操作，完成后把生成的文件拷贝到其余Master节点，本例是22主机。

生成创建配置文件的脚本

```shell
#!/bin/bash
KUBE_CONFIG="/root/.kube/config"
KUBE_APISERVER="https://192.168.3.10:7443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config set-credentials clusteradmin \
  --client-certificate=/opt/kubernetes/server/bin/certs/kubectl.pem \
  --client-key=/opt/kubernetes/server/bin/certs/kubectl-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config set-context default \
  --cluster=kubernetes \
  --user=clusteradmin \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

> 说明：
>
> - 集群名称：描述集群信息的标记，没有实际意义。
> - certificate-authority：K8S集群的根CA证书
> - server：指向kube-apiserrver负载均衡器VIP的地址
> - kubeconfig：生成的kubeconfig文件
> - 用户名称：clusteradmin为定义一个用户，在kubeconfig配置文件中，这个用户用于关联一组证书，这个证书对K8S集群来说无实际意义，真正重要的是证书中的O字段与CN字段的定义。
> - client-certificate：客户端证书
> - client-key：客户端私钥
> - 上下文：default用于将kubeconfig配置文件“clusteradmin”和“kubernetes”作关联。
> - cluster：set-cluster命令配置的集群名称。
> - cluster：set-credentials命令配置的用户名称。

执行脚本

```shell
[root@k8s21 ~]# mkdir ~/.kube
[root@k8s21 ~]# mkdir k8s-shell
[root@k8s21 ~]# cd k8s-shell/
[root@k8s21 k8s-shell]# vim kubectl-config.sh
[root@k8s21 k8s-shell]# chmod +x kubectl-config.sh
[root@k8s21 k8s-shell]# ./kubectl-config.sh
Cluster "kubernetes" set.
User "clusteradmin" set.
Context "default" created.
Switched to context "default".
[root@k8s21 k8s-shell]#
```

查看集群状态

```shell
[root@k8s21 k8s-shell]# kubectl cluster-info
Kubernetes control plane is running at https://192.168.3.10:7443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[root@k8s21 k8s-shell]# kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                        ERROR
scheduler            Unhealthy   Get "https://127.0.0.1:10259/healthz": dial tcp 127.0.0.1:10259: connect: connection refused
controller-manager   Unhealthy   Get "https://127.0.0.1:10257/healthz": dial tcp 127.0.0.1:10257: connect: connection refused
etcd-1               Healthy     {"health":"true"}
etcd-2               Healthy     {"health":"true"}
etcd-0               Healthy     {"health":"true"}
[root@k8s21 k8s-shell]# kubectl get all -A
NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   2d5h
[root@k8s21 k8s-shell]#
```

把21主机上生成的kubeconfig文件拷贝到22节点

```shell
[root@k8s22 certs]# mkdir ~/.kube
[root@k8s22 certs]# scp root@k8s21:/root/.kube/config ~/.kube/
[root@k8s22 certs]# ll ~/.kube/
总用量 8
-rw------- 1 root root 6228 9月  20 19:00 config
[root@k8s22 certs]#
```

在22上查看集群状态

```shell
[root@k8s22 certs]# kubectl cluster-info
Kubernetes control plane is running at https://192.168.3.10:7443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[root@k8s22 certs]# kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                        ERROR
scheduler            Unhealthy   Get "https://127.0.0.1:10259/healthz": dial tcp 127.0.0.1:10259: connect: connection refused
controller-manager   Unhealthy   Get "https://127.0.0.1:10257/healthz": dial tcp 127.0.0.1:10257: connect: connection refused
etcd-1               Healthy     {"health":"true"}
etcd-0               Healthy     {"health":"true"}
etcd-2               Healthy     {"health":"true"}
[root@k8s22 certs]# kubectl get all -A
NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   2d6h
[root@k8s22 certs]#
```

至此，kubectl主机部署完成。



## （四）、部署controller-manager

1. 集群规划

| 主机名         | 角色               | IP           |
| -------------- | ------------------ | ------------ |
| k8s21.host.com | controller-manager | 192.168.3.21 |
| k8s22.host.com | controller-manager | 192.168.3.22 |

2. 生成kube-controller-manager证书

在运维主机200上操作。

生成证书请求文件

`/opt/certs/kube-controller-manager-csr.json`

```shell
{
    "CN": "system:kube-controller-manager",
    "hosts": [
        "127.0.0.1",
        "192.168.3.11",
        "192.168.3.12",
        "192.168.3.21",
        "192.168.3.22",
        "192.168.3.23"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "system:masters",
            "OU": "system"            
        }
    ]
}
```

> 说明：
>
> - CN：这里的CN值非常重要，kube-controller-manager能否正常与kubee-apiserver通信与此值有关，K8S默认会提取CN字段的值作为用户名，这实际是指K8S的“RoleBinding/ClusterRoleBinding”资源中“subjects：kind”的值为“User”
> - hosts：kube-controller-manager运行节点的IP地址。
> - O：无实际意义。
> - OU：无实际意义。
> - hosts 列表包含所有 kube-controller-manager 节点 IP；
> - CN 为 system:kube-controller-manager、O 为 system:kube-controller-manager，kubernetes 内置的 ClusterRoleBindings system:kube-controller-manager 赋予 kube-controller-manager 工作所需的权限

生成证书

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssl-json -bare kube-controller-manager
```

把证书拷贝到21和22主机上

```shell
[root@harbor certs]# scp kube-controller-manager*.pem k8s21:/opt/kubernetes/server/bin/certs/
[root@harbor certs]# scp kube-controller-manager*.pem k8s22:/opt/kubernetes/server/bin/certs/
```

3. 生成kube-controller-manager的kubeconfig配置文件

配置文件路径：`/opt/kubernetes/server/cfg/`

编写生成kubeconfig配置文件的脚本

```shell
#!/bin/bash
KUBE_CONFIG="/opt/kubernetes/server/cfg/kube-controller-manager.kubeconfig"
KUBE_APISERVER="https://192.168.3.10:7443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config set-credentials kube-controller-manager \
  --client-certificate=/opt/kubernetes/server/bin/certs/kube-controller-manager.pem \
  --client-key=/opt/kubernetes/server/bin/certs/kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-controller-manager \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

生成配置文件

```shell
[root@k8s21 k8s-shell]# cd /root/k8s-shell
[root@k8s21 k8s-shell]# vim kube-controller-manager-config.sh
[root@k8s21 k8s-shell]# chmod +x kube-controller-manager-config.sh
[root@k8s21 k8s-shell]# ./kube-controller-manager-config.sh
Cluster "kubernetes" set.
User "kube-controller-manager" set.
Context "default" created.
Switched to context "default".
[root@k8s21 k8s-shell]#
```

把生成的配置文件拷贝到22主机上。

```shell
[root@k8s21 k8s-shell]# scp -r /opt/kubernetes/server/cfg root@k8s22:/opt/kubernetes/server/
```

在22主机上查看

```shell
[root@k8s22 cfg]# ll /opt/kubernetes/server/cfg/
总用量 8
-rw------- 1 root root 6370 9月  20 19:21 kube-controller-manager.kubeconfig
[root@k8s22 cfg]#
```

4. 创建启动脚本

在21、22主机上操作

`/opt/kubernetes/server/bin/kube-controller-manager-startup.sh`

```shell
#!/bin/sh
./kube-controller-manager \
  --cluster-name=kubernetes \
  --bind-address=127.0.0.1 \
  --service-cluster-ip-range=10.100.0.0/16 \
  --leader-elect=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --kubeconfig=/opt/kubernetes/server/cfg/kube-controller-manager.kubeconfig \
  --tls-cert-file=./certs/kube-controller-manager.pem \
  --tls-private-key-file=./certs/kube-controller-manager-key.pem \
  --cluster-signing-cert-file=./certs/ca.pem \
  --cluster-signing-key-file=./certs/ca-key.pem \
  --cluster-signing-duration=438000h0m0s \
  --use-service-account-credentials=true \
  --root-ca-file=./certs/ca.pem \
  --service-account-private-key-file=./certs/ca-key.pem \
  --log-dir=/data/logs/kubernetes/kube-controller-manager \
  --v=2
```

说明：

```shell
--secure-port=10252  这个参数去掉，状态才能正常。
--cluster-cidr string
  CIDR Range for Pods in cluster. Requires --allocate-node-cidrs to be true
本例中，allocate-node-cidrs和cluster-cidr两个参数不配置，使用docker的bip。
```

创建目录，调整权限

```shell
mkdir -p /data/logs/kubernetes/kube-controller-manager
chmod +x /opt/kubernetes/server/bin/kube-controller-manager-startup.sh
```

5. 创建supervisor配置文件

`/etc/supervisord.d/kube-controller-manager.ini`

```shell
#21主机
[program:kube-controller-manager-3.21]
command=/opt/kubernetes/server/bin/kube-controller-manager-startup.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-controller-manager/controller-stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false


#22主机
[program:kube-controller-manager-3.22]
command=/opt/kubernetes/server/bin/kube-controller-manager-startup.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-controller-manager/controller-stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
```

6. 启动supervisor

```shell
[root@k8s21 cfg]# supervisorctl update
kube-controller-manager-3.21: added process group
[root@k8s21 ~]# supervisorctl status
etcd-server-3.21                 RUNNING   pid 40849, uptime 1:45:08
kube-apiserver-3.21              RUNNING   pid 11337, uptime 2 days, 6:36:34
kube-controller-manager-3.21     RUNNING   pid 41978, uptime 0:01:17
[root@k8s21 ~]# netstat -luntp | grep kube
tcp        0      0 0.0.0.0:6443            0.0.0.0:*               LISTEN      11338/./kube-apiser
tcp        0      0 127.0.0.1:10257         0.0.0.0:*               LISTEN      41979/./kube-contro
[root@k8s21 ~]#
```

至此，controller-manager主机部署完成。

## （五）、部署kube-scheduler

1. 集群规划

| 主机名         | 角色           | IP           |
| -------------- | -------------- | ------------ |
| k8s21.host.com | kube-scheduler | 192.168.3.21 |
| k8s22.host.com | kube-scheduler | 192.168.3.22 |

2. 生成kube-scheduler证书

创建证书请求csr文件

`/opt/certs/kube-scheduler-csr.json`

```shell
{
    "CN": "system:kube-scheduler",
    "hosts": [
        "127.0.0.1",
        "192.168.3.11",
        "192.168.3.12",
        "192.168.3.21",
        "192.168.3.22",
        "192.168.3.23"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "system:masters",
            "OU": "system"            
        }
    ]
}
```

生成证书

```shell
[root@harbor certs]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssl-json -bare kube-scheduler
```

3. 把证书拷贝到21和22节点。

```shell
[root@harbor certs]# scp kube-scheduler*.pem k8s22:/opt/kubernetes/server/bin/certs/
kube-scheduler-key.pem                                                                                                      100% 1675     1.2MB/s   00:00
kube-scheduler.pem                                                                                                          100% 1493     1.2MB/s   00:00
[root@harbor certs]# scp kube-scheduler*.pem k8s21:/opt/kubernetes/server/bin/certs/
kube-scheduler-key.pem                                                                                                      100% 1675     1.2MB/s   00:00
kube-scheduler.pem                                                                                                          100% 1493     1.1MB/s   00:00
[root@harbor certs]#
```

4. 生成kubeconfig配置文件

```shell
#!/bin/bash
KUBE_CONFIG="/opt/kubernetes/server/cfg/kube-scheduler.kubeconfig"
KUBE_APISERVER="https://192.168.3.10:7443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config set-credentials kube-scheduler \
  --client-certificate=/opt/kubernetes/server/bin/certs/kube-scheduler.pem \
  --client-key=/opt/kubernetes/server/bin/certs/kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-scheduler \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

执行脚本

```shell
[root@k8s21 k8s-shell]# vim kube-scheduler-config.sh
[root@k8s21 k8s-shell]# chmod +x kube-scheduler-config.sh
[root@k8s21 k8s-shell]# ./kube-scheduler-config.sh
Cluster "kubernetes" set.
User "kube-scheduler" set.
Context "default" created.
Switched to context "default".
[root@k8s21 k8s-shell]#
```

把kubeconfig 文件拷贝到22主机

```shell
scp  /opt/kubernetes/server/cfg/kube-scheduler.kubeconfig  root@k8s22:/opt/kubernetes/server/cfg/
```

5. 创建kube-scheduler启动脚本在21、22操作

`/opt/kubernetes/server/bin/kube-scheduler-startup.sh`

```shell
#!/bin/sh
./kube-scheduler \
  --address=127.0.0.1 \
  --leader-elect=true \
  --kubeconfig=/opt/kubernetes/server/cfg/kube-scheduler.kubeconfig \
  --log-dir=/data/logs/kubernetes/kube-scheduler \
  --v=2
```

创建脚本，调整权限

```shell
vim /opt/kubernetes/server/bin/kube-scheduler-startup.sh
chmod +x /opt/kubernetes/server/bin/kube-scheduler-startup.sh
mkdir -p /data/logs/kubernetes/kube-scheduler
```

6. 创建supervisor配置文件

`/etc/supervisord.d/kube-scheduler.ini`

```shell
#21主机
[program:kube-scheduler-3.21]
command=/opt/kubernetes/server/bin/kube-scheduler-startup.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-scheduler/scheduler-stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false

#22主机
[program:kube-scheduler-3.22]
command=/opt/kubernetes/server/bin/kube-scheduler-startup.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-scheduler/scheduler-stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
```

7. 启动kube-scheduler服务

```shell
[root@k8s21 k8s-shell]# supervisorctl update
kube-scheduler-3.21: added process group
[root@k8s21 k8s-shell]# supervisorctl status
etcd-server-3.21                 RUNNING   pid 40849, uptime 2:03:54
kube-apiserver-3.21              RUNNING   pid 11337, uptime 2 days, 6:55:20
kube-controller-manager-3.21     RUNNING   pid 41978, uptime 0:20:03
kube-scheduler-3.21              RUNNING   pid 42200, uptime 0:00:33
[root@k8s21 k8s-shell]# netstat -luntp | grep kube
tcp        0      0 0.0.0.0:6443            0.0.0.0:*               LISTEN      11338/./kube-apiser
tcp        0      0 127.0.0.1:10257         0.0.0.0:*               LISTEN      41979/./kube-contro
tcp        0      0 0.0.0.0:10259           0.0.0.0:*               LISTEN      42201/./kube-schedu
[root@k8s21 k8s-shell]#
```

8. 查看集群状态

```shell
[root@k8s22 cfg]# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
[root@k8s22 cfg]#
```

查看集群资源

```shell
[root@k8s22 cfg]# kubectl get sa -A
NAMESPACE         NAME                                 SECRETS   AGE
default           default                              1         23m
kube-node-lease   default                              1         23m
kube-public       default                              1         23m
kube-system       attachdetach-controller              1         23m
kube-system       bootstrap-signer                     1         23m
kube-system       certificate-controller               1         23m
kube-system       clusterrole-aggregation-controller   1         23m
kube-system       cronjob-controller                   1         23m
kube-system       daemon-set-controller                1         23m
kube-system       default                              1         23m
kube-system       deployment-controller                1         23m
kube-system       disruption-controller                1         23m
kube-system       endpoint-controller                  1         23m
kube-system       endpointslice-controller             1         23m
kube-system       endpointslicemirroring-controller    1         23m
kube-system       ephemeral-volume-controller          1         23m
kube-system       expand-controller                    1         23m
kube-system       generic-garbage-collector            1         23m
kube-system       horizontal-pod-autoscaler            1         23m
kube-system       job-controller                       1         23m
kube-system       namespace-controller                 1         23m
kube-system       node-controller                      1         23m
kube-system       persistent-volume-binder             1         23m
kube-system       pod-garbage-collector                1         23m
kube-system       pv-protection-controller             1         23m
kube-system       pvc-protection-controller            1         23m
kube-system       replicaset-controller                1         23m
kube-system       replication-controller               1         23m
kube-system       resourcequota-controller             1         23m
kube-system       root-ca-cert-publisher               1         23m
kube-system       service-account-controller           1         23m
kube-system       service-controller                   1         23m
kube-system       statefulset-controller               1         23m
kube-system       token-cleaner                        1         23m
kube-system       ttl-after-finished-controller        1         23m
kube-system       ttl-controller                       1         23m
[root@k8s22 cfg]#
```

至此，Master节点部署完成。



# 六、部署Node节点服务

本例中Master节点和Node节点部署在同一台主机上。

## （一）、部署kubelet

1. 集群规划

| 主机名         | 角色    | IP           |
| -------------- | ------- | ------------ |
| k8s21.host.com | kubelet | 192.168.3.21 |
| k8s22.host.com | kubelet | 192.168.3.22 |

在21主机上操作。

2. 生成kubelet的kubeconfig配置文件

```shell
#!/bin/bash
KUBE_CONFIG="/opt/kubernetes/server/cfg/kubelet-bootstrap.kubeconfig"
KUBE_APISERVER="https://192.168.3.10:7443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config set-credentials kubelet-bootstrap \
  --token=$(awk -F "," '{print $1}' /opt/kubernetes/server/bin/certs/kube-apiserver.token.csv) \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}

kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

> 说明：
>
> 关于`kubectl create clusterrolebinding`命令中的"kubelet-bootstrap"
>
> 第一个"kubelet-bootstrap"：会在K8S集群中创建一个名为"kubelet-bootstrap"的"ClusterRoleBinding"资源，用`kubectl get clusterrolebinding`查看
>
> 第二个"--user=kubelet-bootstrap"：表示将对应"ClusterRoleBinding"资源中的"subjects.kind"="User"、"subjects.name"="kubelet-bootstrap"
>
> 用`kubectl get clusterrolebinding kubelet-bootstrap -o yaml`查看
>
> 在经过本命令的配置后，KUBE-APISERVER的"kube-apiserver.token.csv"配置文件中的用户名"kubelet-bootstrap"便真正的在K8S集群中有了意义

执行脚本

```shell
[root@k8s21 k8s-shell]# vim kubelet-config.sh
[root@k8s21 k8s-shell]# chmod +x kubelet-config.sh
[root@k8s21 k8s-shell]# ./kubelet-config.sh
Cluster "kubernetes" set.
User "kubelet-bootstrap" set.
Context "default" created.
Switched to context "default".
clusterrolebinding.rbac.authorization.k8s.io/kubelet-bootstrap created
[root@k8s21 k8s-shell]#
```

把生成的kubeconfig文件拷贝到22主机上

```shell
[root@k8s21 k8s-shell]# scp  /opt/kubernetes/server/cfg/kubelet-bootstrap.kubeconfig  root@k8s22:/opt/kubernetes/server/cfg/
```

3. 创建kubelet启动脚本，在21、22操作

`/opt/kubernetes/server/bin/kubelet-startup.sh`

```shell
#21主机
#!/bin/sh
./kubelet \
  --v=2 \
  --log-dir=/data/logs/kubernetes/kubelet \
  --hostname-override=k8s21.host.com \
  --network-plugin=cni \
  --cluster-domain=cluster.local \
  --kubeconfig=/opt/kubernetes/server/cfg/kubelet.kubeconfig \
  --bootstrap-kubeconfig=/opt/kubernetes/server/cfg/kubelet-bootstrap.kubeconfig \
  --config=/opt/kubernetes/server/cfg/kubelet-config.yml \
  --cert-dir=/opt/kubernetes/server/bin/certs \
  --pod-infra-container-image=ibmcom/pause:3.1
  
#22主机  
#!/bin/sh
./kubelet \
  --v=2 \
  --log-dir=/data/logs/kubernetes/kubelet \
  --hostname-override=k8s22.host.com \
  --network-plugin=cni \
  --cluster-domain=cluster.local \
  --kubeconfig=/opt/kubernetes/server/cfg/kubelet.kubeconfig \
  --bootstrap-kubeconfig=/opt/kubernetes/server/cfg/kubelet-bootstrap.kubeconfig \
  --config=/opt/kubernetes/server/cfg/kubelet-config.yml \
  --cert-dir=/opt/kubernetes/server/bin/certs \
  --pod-infra-container-image=ibmcom/pause:3.1
```

说明： 本例中为了方便测试，先删除`--network-plugin`

配置参数文件

`/opt/kubernetes/server/cfg/kubelet-config.yml`

```shell
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: systemd
clusterDNS:
- 10.100.0.2
clusterDomain: cluster.local 
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/server/bin/certs/ca.pem 
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
```

生成上面两个文件

```shell
vim  /opt/kubernetes/server/bin/kubelet-startup.sh
chmod +x /opt/kubernetes/server/bin/kubelet-startup.sh
mkdir -pv /data/logs/kubernetes/kubelet
vim /opt/kubernetes/server/cfg/kubelet-config.yml
```

创建supervisor启动文件

`/etc/supervisord.d/kube-kubelet.ini`

```shell
#21主机
[program:kube-kubelet-3.21]
command=/opt/kubernetes/server/bin/kubelet-startup.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kubelet/kubelet-stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false


#22主机
[program:kube-kubelet-3.22]
command=/opt/kubernetes/server/bin/kubelet-startup.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kubelet/kubelet-stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
```

启动服务

```shell
[root@k8s21 k8s-shell]# supervisorctl update
kube-kubelet-3.21: added process group
[root@k8s21 k8s-shell]# supervisorctl status
etcd-server-3.21                 RUNNING   pid 40849, uptime 2:36:51
kube-apiserver-3.21              RUNNING   pid 11337, uptime 2 days, 7:28:17
kube-controller-manager-3.21     RUNNING   pid 41978, uptime 0:53:00
kube-kubelet-3.21                RUNNING   pid 42570, uptime 0:01:02
kube-scheduler-3.21              RUNNING   pid 42200, uptime 0:33:30
[root@k8s21 k8s-shell]#
```

4. 批准kubelete证书申请并加入集群

查看kubelet证书请求

```shell
[root@k8s22 cfg]# kubectl get csr
NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-70ZWMwCqLntoowCvR4LeC5gBMpO8u8Rfdg3Z1WlpyBk   2m17s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
node-csr-HK_LbGddo2Z_Kdw2SGmhcAWDH0g4PLFzsMtzWVG-Nvw   2m10s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
[root@k8s22 cfg]#
```

批准证书

```shell
[root@k8s22 cfg]# kubectl certificate approve node-csr-70ZWMwCqLntoowCvR4LeC5gBMpO8u8Rfdg3Z1WlpyBk
certificatesigningrequest.certificates.k8s.io/node-csr-70ZWMwCqLntoowCvR4LeC5gBMpO8u8Rfdg3Z1WlpyBk approved
[root@k8s22 cfg]# kubectl certificate approve node-csr-HK_LbGddo2Z_Kdw2SGmhcAWDH0g4PLFzsMtzWVG-Nvw
certificatesigningrequest.certificates.k8s.io/node-csr-HK_LbGddo2Z_Kdw2SGmhcAWDH0g4PLFzsMtzWVG-Nvw approved
[root@k8s22 cfg]#
```

查看节点

```shell
[root@k8s22 cfg]# kubectl get no
NAME             STATUS     ROLES    AGE   VERSION
k8s21.host.com   NotReady   <none>   44s   v1.23.4
k8s22.host.com   NotReady   <none>   34s   v1.23.4
[root@k8s22 cfg]#
```

由于没有安装网络插件，节点状态为NotReady



## （二）、部署kube-proy

1. 集群规划

| 主机名         | 角色       | IP           |
| -------------- | ---------- | ------------ |
| k8s21.host.com | kube-proxy | 192.168.3.21 |
| k8s22.host.com | kube-proxy | 192.168.3.22 |

#### 2.2 生成kube-proxy的kubeconfig文件

在运维主机200上操作

`/opt/certs/kube-proxy-csr.json`

```shell
{
    "CN": "system:kube-proxy",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "system:masters",
            "OU": "system"            
        }
    ]
}
```

生成证书

```shell
[root@harbor certs]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssl-json -bare kube-proxy
```

把生成的证书拷贝到21和22节点

```shell
[root@harbor certs]# scp kube-proxy*.pem k8s21:/opt/kubernetes/server/bin/certs/ 
[root@harbor certs]# scp kube-proxy*.pem k8s22:/opt/kubernetes/server/bin/certs/   
[root@harbor certs]# 
```

创建脚本

```shell
#!/bin/bash
KUBE_CONFIG="/opt/kubernetes/server/cfg/kube-proxy.kubeconfig"
KUBE_APISERVER="https://192.168.3.10:7443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config set-credentials kube-proxy \
  --client-certificate=/opt/kubernetes/server/bin/certs/kube-proxy.pem \
  --client-key=/opt/kubernetes/server/bin/certs/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

执行脚本

```shell
[root@k8s21 ~]# vim kube-proxy-config.sh
[root@k8s21 ~]# cd k8s-shell/
[root@k8s21 k8s-shell]# vim kube-proxy-config.sh
[root@k8s21 k8s-shell]# chmod +x kube-proxy-config.sh 
[root@k8s21 k8s-shell]# ./kube-proxy-config.sh
Cluster "kubernetes" set.
User "kube-proxy" set.
Context "default" created.
Switched to context "default".
[root@k8s21 k8s-shell]# 
```

把生成kubeconfig文件拷贝到22主机。

```shell
scp  /opt/kubernetes/server/cfg/kube-proxy.kubeconfig  root@k8s22:/opt/kubernetes/server/cfg/
```



## （三）、加载ipvs模块

编写脚本

```shell
#!/bin/bash
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs"
for i in $(ls $ipvs_mods_dir | grep -o "^[^.]*")
do
  /sbin/modinfo -F filename $i &>/dev/null
  if [ $? -eq 0 ];then
    /sbin/modprobe $i
  fi
done
```

在21上操作

```shell
[root@k8s21 k8s-shell]# vim ipvs.sh
[root@k8s21 k8s-shell]# chmod +x ipvs.sh 
[root@k8s21 k8s-shell]# ./ipvs.sh
[root@k8s21 k8s-shell]# lsmod | grep ip_vs
ip_vs_wrr              12697  0 
ip_vs_wlc              12519  0 
ip_vs_sh               12688  0 
ip_vs_sed              12519  0 
ip_vs_rr               12600  0 
ip_vs_pe_sip           12740  0 
nf_conntrack_sip       33780  1 ip_vs_pe_sip
ip_vs_nq               12516  0 
ip_vs_lc               12516  0 
ip_vs_lblcr            12922  0 
ip_vs_lblc             12819  0 
ip_vs_ftp              13079  0 
ip_vs_dh               12688  0 
ip_vs                 145458  24 ip_vs_dh,ip_vs_lc,ip_vs_nq,ip_vs_rr,ip_vs_sh,ip_vs_ftp,ip_vs_sed,ip_vs_wlc,ip_vs_wrr,ip_vs_pe_sip,ip_vs_lblcr,ip_vs_lblc
nf_nat                 26583  3 ip_vs_ftp,nf_nat_ipv4,nf_nat_masquerade_ipv4
nf_conntrack          139264  8 ip_vs,nf_nat,nf_nat_ipv4,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack_sip,nf_conntrack_ipv4
libcrc32c              12644  4 xfs,ip_vs,nf_nat,nf_conntrack
[root@k8s21 k8s-shell]# 
```

在22主机上同样操作。本处略。



## （四）、创建kube-proxy启动脚本

`/opt/kubernetes/server/bin/kube-proxy-startup.sh`，在21、22操作

```shell
#!/bin/sh
./kube-proxy \
  --v=2 \
  --log-dir=/data/logs/kubernetes/kube-proxy \
  --config=/opt/kubernetes/server/cfg/kube-proxy-config.yml
```

生成文件、调整权限，创建目录

```shell
[root@k8s21 bin]# vim /opt/kubernetes/server/bin/kube-proxy-startup.sh
[root@k8s21 bin]# chmod +x /opt/kubernetes/server/bin/kube-proxy-startup.sh
[root@k8s21 bin]# mkdir -p /data/logs/kubernetes/kube-proxy
[root@k8s21 bin]# 
```

配置参数文件

`/opt/kubernetes/server/cfg/kube-proxy-config.yml`

```shell
#21主机

kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/server/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s21.host.com
clusterCIDR: 10.100.0.0/16

#22主机

kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/server/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s22.host.com
clusterCIDR: 10.100.0.0/16
```

创建supervisor启动文件

`/etc/supervisord.d/kube-proxy.ini`

```shell
#21主机
[program:kube-proxy-3.21]
command=/opt/kubernetes/server/bin/kube-proxy-startup.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-proxy/kube-proxy-stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false



#22主机
[program:kube-proxy-3.22]
command=/opt/kubernetes/server/bin/kube-proxy-startup.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-proxy/kube-proxy-stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
```

启动服务，在21、22主机操作

```shell
[root@k8s21 bin]# supervisorctl update
kube-proxy-3.21: added process group
[root@k8s21 bin]# supervisorctl status
etcd-server-3.21                 RUNNING   pid 56543, uptime 0:25:17
kube-apiserver-3.21              RUNNING   pid 11337, uptime 2 days, 9:51:54
kube-controller-manager-3.21     RUNNING   pid 41978, uptime 3:16:37
kube-kubelet-3.21                RUNNING   pid 42570, uptime 2:24:39
kube-proxy-3.21                  RUNNING   pid 59538, uptime 0:01:19
kube-scheduler-3.21              RUNNING   pid 49757, uptime 1:21:45
[root@k8s21 bin]# 
```

至此，kubernetes集群部署完成。



# 七、安装Harbor



## （一）、下载解压并作软链接

在harbor.host.com上操作。

下载harbor离线包，上传到200主机的/opt/src目录下，[下载地址](https://github.com/goharbor/harbor/releases/download/v2.6.0/harbor-offline-installer-v2.6.0.tgz)

```shell
[root@harbor src]# pwd
/opt/src
[root@harbor src]# ll
总用量 738404
-rw-r--r-- 1 root root 756124163 9月  10 22:56 harbor-offline-installer-v2.6.0.tgz
[root@harbor src]# tar -zxvf harbor-offline-installer-v2.6.0.tgz -C /opt
[root@harbor src]# cd /opt
[root@harbor opt]# mv harbor harbor-v2.6.0
[root@harbor opt]# ln -s /opt/harbor-v2.6.0 /opt/harbor
```



## （二）、修改配置文件

```shell
[root@harbor opt]# cd harbor
[root@harbor harbor]# cp harbor.yml.tmpl harbor.yml
[root@harbor harbor]# mkdir -pv /data/harbor/logs
[root@harbor harbor]# vim harbor.yml

# 修改主机名
hostname: harbor.wg.com
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  # 修改端口号
  port: 180
# 注释https
# https related config
#https:
#  # https port for harbor, default is 443
#  port: 443
#  # The path of cert and key files for nginx
#  certificate: /your/certificate/path
#  private_key: /your/private/key/path
# 修改密码
harbor_admin_password: Eswin@2022
# 修改数据存储位置
# The default data volume
data_volume: /data/harbor
# 修改日志存储位置
    location: /data/harbor/logs
```



## （三）、安装docker编排工具docker-compose

```shell
[root@harbor harbor]# yum install docker-compose -y
[root@harbor harbor]# rpm -qa |grep docker-compose
docker-compose-1.18.0-4.el7.noarch
[root@harbor harbor]#
```



## （四）、启动harbor

```shell
# 修改配置文件后，需要执行./prepare
[root@harbor harbor]# ./prepare
...
Successfully called func: create_root_cert
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir

[root@harbor harbor]# ./install.sh
...
✔ ----Harbor has been installed and started successfully.----

```



## （五）、检查harbor启动情况

```shell
[root@harbor harbor]# docker-compose ps
      Name                     Command               State             Ports
--------------------------------------------------------------------------------------
harbor-core         /harbor/entrypoint.sh            Up
harbor-db           /docker-entrypoint.sh 96 13      Up
harbor-jobservice   /harbor/entrypoint.sh            Up
harbor-log          /bin/sh -c /usr/local/bin/ ...   Up      127.0.0.1:1514->10514/tcp
harbor-portal       nginx -g daemon off;             Up
nginx               nginx -g daemon off;             Up      0.0.0.0:180->8080/tcp
redis               redis-server /etc/redis.conf     Up
registry            /home/harbor/entrypoint.sh       Up
registryctl         /home/harbor/start.sh            Up
[root@harbor harbor]# 
```



## （六）、安装nginx并配置

实际使用时，不能直接使用IP地址，而是通过域名访问，因此需要安装反向代理。

1. 安装nginx

```shell
[root@harbor harbor]# yum install nginx -y
[root@harbor harbor]# rpm -qa|grep nginx
nginx-filesystem-1.20.1-9.el7.noarch
nginx-1.20.1-9.el7.x86_64
[root@harbor harbor]#
```

2. 配置nginx

`/etc/nginx/conf.d/harbor.host.com.conf`

```shell
[root@harbor harbor]# cat /etc/nginx/conf.d/harbor.host.com.conf
server {
    listen    80;
    server_name harbor.wg.com;
    client_max_body_size 1000m;
    location / {
        proxy_pass http://127.0.0.1:180;
    }
}


# 语法检查
[root@harbor harbor]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
# 启动nginx并设置为开机启动
[root@harbor harbor]# systemctl enable nginx
Created symlink from /etc/systemd/system/multi-user.target.wants/nginx.service to /usr/lib/systemd/system/nginx.service.
[root@harbor harbor]# systemctl start nginx
# 检查状态
[root@cfzx55-200 harbor]# systemctl status nginx
```

3. 访问测试

```shell
[root@harbor harbor]# curl  harbor.wg.com
<!doctype html>
<html>

。。。
```



## （七）、配置harbor的dns内网解析

在11主机上操作

添加harbor的A记录，注意serial序列号前滚一个序号，此前已配置本次无需操作。

```shell
cat /var/named/wg.com.zone
```



## （八）、浏览器打开

修改主机（运行虚拟机的电脑）配置文件，在文件的最后增加下面一行内容。

```shell
❯ sudo vim /etc/hosts
❯ cat /etc/hosts
10.211.55.200       harbor.od.com
```

用浏览器访问：`http://harbor.wg.com/`

用户名：admin

密码：Eswin@2022

![login-harbor1668627685](https://wsidtypora.oss-cn-beijing.aliyuncs.com/img/login-harbor1668627685.png)



## （九）、新建项目

![create-harbor-projects](https://wsidtypora.oss-cn-beijing.aliyuncs.com/img/create-harbor-projects.png)

项目名称：public

访问级别：public

![new-harbor-projects6](https://wsidtypora.oss-cn-beijing.aliyuncs.com/img/new-harbor-projects6.png)

结果如下图：

![result-create830989419](https://wsidtypora.oss-cn-beijing.aliyuncs.com/img/result-create830989419.png)



## （十）、配置http访问

```shell
[root@harbor harbor]# cat /etc/docker/daemon.json
{
  "graph": "/data/docker",
  "storage-driver": "overlay2",
  "insecure-registries": ["registry.access.redhat.com","quay.io","harbor.wg.com"],
  "registry-mirrors": ["https://q2gr04ke.mirror.aliyuncs.com"],
  "bip": "172.16.200.1/24",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "live-restore": true
}
[root@harbor harbor]#
```

daemon.json文件中，insecure-registries中配置了harbor.od.com，这样可以不使用证书直接用http方式访问。

## （十一）、给自建仓库上传镜像

```shell
# 拉取镜像
[root@harbor harbor]# docker pull nginx:1.7.9
1.7.9: Pulling from library/nginx
Image docker.io/library/nginx:1.7.9 uses outdated schema1 manifest format. Please upgrade to a schema2 image for better future compatibility. More information at https://docs.docker.com/registry/spec/deprecated-schema-v1/
a3ed95caeb02: Pull complete
...
c9cec474c523: Pull complete
Digest: sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
Status: Downloaded newer image for nginx:1.7.9
docker.io/library/nginx:1.7.9
#查询镜像信息
[root@harbor harbor]# docker image inspect nginx
# 打标签
[root@harbor harbor]# docker images | grep 1.7.9
nginx                         1.7.9     84581e99d807   7 years ago    91.7MB
[root@harbor harbor]# docker tag 84581e99d807 harbor.wg.com/public/nginx:v1.7.9
# 登录harbor
[root@harbor harbor]# docker login http://harbor.wg.com
Username: admin
Password:
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[root@harbor harbor]#

# 上传镜像
[root@harbor harbor]# docker push harbor.wg.com/public/nginx:v1.7.9
The push refers to repository [harbor.od.com/public/nginx]
5f70bf18a086: Pushed
4b26ab29a475: Pushed
ccb1d68e3fb7: Pushed
e387107e2065: Pushed
63bf84221cce: Pushed
e02dce553481: Pushed
dea2e4984e29: Pushed
v1.7.9: digest: sha256:b1f5935eb2e9e2ae89c0b3e2e148c19068d91ca502e857052f14db230443e4c2 size: 3012
[root@cfzx55-200 harbor]#
```

查看上传结果

![check-push-1006191484](https://wsidtypora.oss-cn-beijing.aliyuncs.com/img/check-push-1006191484.png)





