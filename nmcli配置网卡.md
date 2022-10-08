# 一、常用命令查看网络信息

```shell
ifconfig   #查看网络接口信息 
ip add     #查看IP地址 
```



# 二、nmcli命令
```shell
nmcli dev status               #查看设备状态
nmcli con show                 #显示连接状态
nmcli connection add con-name <网络接口名称> type <接口类型> ifname <网卡名称>   #添加网络接口
nmcli connection modify  <网络接口名称>  [选项]  [参数]    #修改网络接口
#ipv4选项查看：对应选项输入ipv4，敲击tab键
```



# 三、使用nmcli命令添加网络连接并配置信息
```shell 
nmcli connection add con-name <接口名称> type <方式名|ethernet> ifname <网卡名称>  #添加网络接口（名、方式、网卡名称）
nmcli connection modify <接口名称> ipv4.addresses <IP地址/掩码缩写>                #设置IP地址、子网掩码
nmcli connection modify <接口名称> ipv4.method manual                             #修改配置方式为手动
nmcli connection up <接口名称>                                                    #启用该网络接口
nmcli connection                                                                 #查看配置是否成功
```

> 注意：必须先修改 ipv4.address，然后才能修改 ipv4.method！
>
> 用空引号`""`代替选项的值，可将选项设回默认值（以 ipv4.method 为例）：
>
> ```bash
> nmcli com mod ens33 ipv4.method ""
> ```

# 四、示例

```shell
#nmcli connection add con-name ens33 type ethernet ifname ens33 ipv4.method auto autoconnect yes
nmcli connection add con-name ens38 type ethernet ifname ens38 autoconnect yes
nmcli connection modify ens33 ipv4.addresses 192.168.1.211/24
nmcli con mod ens33 ipv4.gateway 192.168.1.1           #网关不需要可以不配置
nmcli con mod ens33 ipv4.method manual      
nmcli con mod ens33 ipv4.dns "8.8.8.8"                 #DNS不需要可以不配置
nmcli con mod ens33 +ipv4.dns 114.114.114.114          # 添加一个 DNS
nmcli con up ens33
nmcli connection down Wired\ connection\ 1
nmcli connection delete Wired\ connection\ 1

#配置WIFI连接，未实践仅供参考
sudo nmcli dev wifi connect '你的wifi名称' password '你的密码' wep-key-type key ifname wlan0
```

> [详细教程可参考](https://zhuanlan.zhihu.com/p/395236748)

# 五、双网卡的链路聚合(bond)

```nmcli
nmcli dev status                #查看网卡状态
ethtool ens224 |grep -i speed   #查看网卡速度
ethtool ens256 |grep -i speed   #查看网卡速度
nmcli con add con-name bond0 ifname bond0 type bond mode 6 
nmcli con modify bond0 ipv4.addresses 192.168.3.10/24  ipv4.method manual
nmcli connection modify bond0 ipv4.addresses "192.168.3.10/24 192.168.3.1"  ipv4.method manual
#创建一块虚拟网卡bond0
nmcli con add con-name bond-port1 ifname ens224 type bond-slave master bond0
nmcli con add con-name bond-port2 ifname ens256 type bond-slave master bond0
#为ens224和ens256两块网卡创建配置文件并将两块网卡作为bond0网卡的slave
nmcli con reload bond0
nmcli con up bond0
#重载并激活bond0网卡
ethtool bond0 |grep -i speed   #查看bond0网卡的带宽
ip a show bond0                #查看bond0网卡的ip地址
nmcli con show                 #查看bond0网卡的两块物理网卡的状态
cat /proc/net/bonding/bond0    #查看bond0的详细信息
```

