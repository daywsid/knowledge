# 一、安装配置ldapscripts

1. 安装依赖包 `sharutils`

```
yum install sharutils  -y            #需要配置yum源，已在搭建openldap中配置
```

1. 手动安装资源包 `Ldapscripts`

```
wget -O /opt/ldapscripts-2.0.8.tar.gz https://github.com/martymac/ldapscripts/archive/refs/tags/ldapscripts-2.0.8.tar.gz
tar zxf ldapscripts-2.0.8.tgz
cd  ldapscripts-2.0.8
make install
#需要自定义安装位置使用make PREFIX=/target/directory install
```

1. 配置 `ldapscripts`

```
vim /usr/local/etc/ldapscripts/ldapscripts.conf
SERVER="ldap://192.168.3.10:389"
SUFFIX="dc=software,dc=com"
BINDDN="cn=admin,dc=software,dc=com"
```

> 将 `SERVER="ldap://localhost"` 改成`SERVER="ldap://192.168.3.10:389"`
>
> 将 `SUFFIX="dc=example,dc=com"` 改成 `SUFFIX="dc=software,dc=com"`
>
> 将 `BINDDN="cn=Manager,dc=example,dc=com"` 改成 `BINDDN="cn=admin,dc=software,dc=com"`

1. 修改 `/usr/local/etc/ldapscripts/ldapscripts.passwd` 文件

```
echo "Wg@2022" > /usr/local/etc/ldapscripts/ldapscripts.passwd
```

1. `ldapscripts`命令

```
ldapadduser <username> <groupname | gid> [uid]               #新建用户，uid可不指定自动生成
ldapsetpasswd <username | uid> [encoded password]            #设置用户密码
ldapdeleteuser <username | uid>                              #删除用户
ldapaddusertogroup <username | uid | dn> <groupname | gid>   #给用户新增组权限
```

# 二、安装phpldapadmin

1. 安装 `phpldapadmin`

```
yum install -y phpldapadmin          #需要配置epel源，已在搭建openldap中配置
```

> yum安装时，会自动安装apache和php的依赖。注意：phpldapadmin很多没更新了，只支持php5，如果你服务器的环境是php7，则会有问题，页面会有各种报错。可以使用`php -v`来查看php版本。

1. 修改apache的phpldapadmin配置文件

```
vim /etc/httpd/conf.d/phpldapadmin.conf
  <IfModule mod_authz_core.c>
    # Apache 2.4
    Require all granted
  </IfModule>
```

> 修改中的内容，放开外网访问，这里只改了 2.4 版本的配置，因为 redhat7 默认安装的 apache 为 2.4 版本。所以只需要改 2.4 版本的配置就可以了. 如果不知道自己apache版本，执行 rpm -qa|grep httpd 查看 apache 版本。

1. 修改配置用DN登录ldap

```
vim /etc/phpldapadmin/config.php
$servers->setValue('login','attr','dn');         #398行，默认是使用uid进行登录，我这里改为dn，也就是用户名
$servers->setValue('login','anon_bind',false);   #460行，关闭匿名登录，否则任何人都可以直接匿名登录查看所有人的信息
$servers->setValue('unique','attrs',array('mail','uid','uidNumber','cn','sn'));
#519行，设置用户属性的唯一性，这里我将cn,sn加上了，以确保用户名的唯一性
```

1. 启动apache

```
systemctl start httpd
systemctl enable httpd
```

1. web 端登录LDAP

> 启动了apache服务后，采用dn登录方式登录 web 端LDAP。
>
> 在浏览器上访问: http://192.168.3.10/ldapadmin，然后使用上面定义的用户，进行登录，如下：

```
登录DN: cn=admin,dc=software,dc=com
密码：  Wg@2022
```