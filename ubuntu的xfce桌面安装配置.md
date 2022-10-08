# 一、安装操作系统，并修改默认的源

```
cat /etc/apt/sources.list
deb http://cn.archive.ubuntu.com/ubuntu/ bionic main restricted
deb http://cn.archive.ubuntu.com/ubuntu/ bionic-updates main restricted
deb http://cn.archive.ubuntu.com/ubuntu/ bionic universe
deb http://cn.archive.ubuntu.com/ubuntu/ bionic-updates universe
deb http://cn.archive.ubuntu.com/ubuntu/ bionic multiverse
deb http://cn.archive.ubuntu.com/ubuntu/ bionic-updates multiverse
deb http://cn.archive.ubuntu.com/ubuntu/ bionic-backports main restricted universe multiverse
deb http://security.ubuntu.com/ubuntu bionic-security main restricted
deb http://security.ubuntu.com/ubuntu bionic-security universe
deb http://security.ubuntu.com/ubuntu bionic-security multiverse
```

> 修改为国内阿里源

```
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
```

# 二、安装xfce

## 1.安装lightdm服务

```
apt-get update
apt-get install lightdm  lightdm-gtk-greeter  -y
```

## 2.配置lightdm服务

```
#默认会生成/etc/lightdm/lightdm.conf配置文件，如果未生成，手动配置
vim /etc/lightdm/lightdm.conf
[SeatDefaults]
xserver-allow-tcp=true
greeter-show-remote-login=true
[XDMCPServer]
enabled=true
port=177
```

## 3.启动lightdm服务，并设置开机自启动

```
systemctl start lightdm
systemctl enable lightdm
```

> 还需要禁止gdm和gdm3开机自启动

```
systemctl disable gdm
systemctl disable gdm3
然后执行dpkg-reconfigure lightdm，选择lightdm方式
```

## 4.关闭防火墙

```
systemctl stop ufw
systemctl disable ufw
```

## 5.安装xfce4和xfce4-terminal

```
apt-get install xfce4 -y
apt-get install xfce4-terminal -y
```

> 默认登录时需要选择桌面类型，可以把xfce作为默认桌面

```
mkdir /usr/share/xsessions/bak
mv /usr/share/xsessions/ubuntu* /usr/share/xsessions/bak/
```

## 6.重启机器验证配置

## 7.配置ETX登录时需要增加以下命令

```
/usr/bin/xfce4-session --display @d  &
```