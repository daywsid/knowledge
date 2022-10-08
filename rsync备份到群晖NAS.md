# 一、首先开启群晖NAS的ssh和rsync服务

## （一）、开启ssh服务

> 群晖控制面板==》终端机和SNMP ==》 启动SSH功能，并设置端口保存。

![图片](https://wsidtypora.oss-cn-beijing.aliyuncs.com/img/640)

## （二）、开启rsync服务

> 群晖控制面板==》文件服务 ==》 rsync ==》 启动rsync服务，并设置端口保存。

![图片](https://wsidtypora.oss-cn-beijing.aliyuncs.com/img/640)

# 二、群晖开启root账号ssh登录

## （一）、ssh登录群晖

> 通过xshell使用admin登录ssh==》sudo -i （切换root用户，提示输入用户密码）==》 synouser --setpw root 密码 （设置root密码）。

```
sudo -i                             #切换最高权限账号命令
synouser --setpw root 你的密码      #设置root密码命令
```

## (二)、修改ssh配置文件

> 修改/etc/ssh/sshd_config文件，并重启服务

```
vim /etc/ssh/sshd_config
#PermitRootLogin prohibit-password
PermitRootLogin yes

systemctl restart sshd    #重启sshd服务
```