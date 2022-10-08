# 一、常用命令查看网络信息

```
ifconfig   #查看网络接口信息 
ip add     #查看IP地址 
```

# 二、nmcli命令

```
nmcli dev status               #查看设备状态
nmcli con show                 #显示连接状态
nmcli connection add con-name <网络接口名称> type <接口类型> ifname <网卡名称>   #添加网络接口
nmcli connection modify  <网络接口名称>  [选项]  [参数]    #修改网络接口
#ipv4选项查看：对应选项输入ipv4，敲击tab键
```

# 三、使用nmcli命令添加网络连接并配置信息

```
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
> ```
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
nmcli dev wifi connect '你的wifi名称' password '你的密码' wep-key-type key ifname wlan0
```