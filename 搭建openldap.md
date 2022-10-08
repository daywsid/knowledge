> ****LDAP的术语：\****entry：一个单独的单元，使用DN(distinguish name)区别 attribute:entry的属性，比如，如果entry是组织机构的话，那么它的属性包括地址，电话，传真号码等，属性分为可选和必选，必选的属性使用objectclass定义，这些属性可以在 /etc/openldap/slapd.d/cn=config/cn=schema/目录下面找到 LDIF: LDAP interchange format 是用来表示LDAP entry的文本格式，格式如下：[id] dn: distinguished_nameattribute_type: attribute_value…attribute_type: attribute_value…

> 部署前需要关闭防火墙和selinux

```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g'/etc/selinux/config
systemctl disable firewalld.service
systemctl stop firewalld.service
```

# 一、LDAP服务器端安装

> ****准备安装测试环境\****:服务器IP:192.168.0.1 操作系统：RHEL7.8 在安装之前，服务器上配置好 yum 源。

要求：在服务器上安装openldap软件，然后把系统中已有的帐号和组信息转存到LDAP中。之后，我们在其它服务器上安装配置ldap客户端，到LDAP服务器认证登录，并通过NFS方式挂载用户目录。

## 1、安装openLDAP服务器端软件包

```
 yum install –y openldap* migrationtools compat-openldap
#说明：migrationtool工具用于将本地系统帐号迁移至openldap。
```

## 2、设置LDAP服务器全局连接密码

```
[root@server0 ~]# slappasswd -s wsid123456 > /etc/openldap/passwd
[root@server0 ~]# cat /etc/openldap/passwd 
{SSHA}SjGeEJNdQFujSss9Z72U2CNTSXOgDV64
```

## 3、生成LDAP基础数据并设置其权限

```
#复制一份LDAP的配置模板（基础数据）
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG 
chown ldap:ldap /var/lib/ldap/*
# slaptest 测试
```

## 4、启动LDAP服务，并设置开机自启动

```
systemctl start/restart slapd
systemctl enable slapd
systemctl status slapd
```

## 5、设置防火墙规则允许LDAP服务被连接

```
firewall-cmd --permanet --add-service=ldap
firewall-cmd --reload
#关闭防火墙无需进行此操作
```

## 6、设置LDAP日志文件，保存日志信息

```
vi /etc/rsyslog.conf 添加如下一行
local4.* /var/log/ldap.log
systemctl restart rsyslog  --重启rsyslog服务
#备注:开启日志需要配置/etc/logrotate.d/,以免日志太多导致根目录满
```

# 二、配置LDAP本地服务器域

## 1、配置基础用户认证结构

```
cd /etc/openldap/schema/
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/collective.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/corba.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/duaconf.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/dyngroup.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/java.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/misc.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/openldap.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/pmi.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/ppolicy.ldif
```

## 2、配置自定义的结构文件并导入到LDAP服务器

> 2.1 创建/root/addrootpwd.ldif文件，并将下面的信息复制进去

```
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW:{SSHA}CHdpG/GUJdp1aeGuFBpMXmnXVEQa91R+

#specify the password generated above for “olcRootPW” section 配置openldap管理员密码
```

> 2.2创建/root/changepwd2.ldif文件，并将下面的信息复制进去

```
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW:{SSHA}CHdpG/GUJdp1aeGuFBpMXmnXVEQa91R+
#允许用户修改密码
```

> 2.3创建/root/changedomain.ldif文件，并将下面的信息复制进去

```
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read by dn.base="cn=admin,dc=example,dc=com" read by * none

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=example,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,dc=example,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}6WExotT6YDsUoe89DsGEtt03vSLS2yLB

```

> 2.4创建/root/add-memberof.ldif文件，并将下面的信息复制进去

```
dn: cn=module{0},cn=config
cn: modulle{0}
objectClass: olcModuleList
objectclass: top
olcModuleload: memberof.la
olcModulePath: /usr/lib64/openldap
```

> 2.5创建/root/refint1.ldif文件，并将下面的信息复制进去

```
dn: cn=module{0},cn=config
add: olcmoduleload
olcmoduleload: refint
```

> 2.6创建/root/refint2.ldif文件，并将下面的信息复制进去

```
dn: olcOverlay=refint,olcDatabase={2}hdb,cn=config
objectClass: olcConfig
objectClass: olcOverlayConfig
objectClass: olcRefintConfig
objectClass: top
olcOverlay: refint
olcRefintAttribute: memberof uniqueMember  manager owner
```

> 2.7创建/root/updatepass.ldif文件，并将下面的信息复制进去

```
dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: to attrs=userPassword
        by dn="cn=admin,dc=example,dc=com" write
        by dn="ou=user,dc=example,dc=com" write
        by anonymous auth
        by self write
        by * none
olcAccess: to *
        by dn="cn=admin,dc=example,dc=com" write
        by dn="ou=user,dc=example,dc=com" write
        by * read
```

## 3、将新的配置文件更新到slapd服务程序

```
ldapadd -Y EXTERNAL -H ldapi:/// -f /root/addrootpwd.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /root/changepwd2.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /root/changedomain.ldif
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f /root/add-memberof.ldif
ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f /root/refint1.ldif
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f /root/refint2.ldif
ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f /root/updatepass.ldif
```

> 3.1 创建/root/base.ldif文件，并将下面的信息复制进去

```
dn: dc=example,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: wsltw Company
dc: wsltw

dn: cn=admin,dc=example,dc=com
objectClass: organizationalRole
cn: admin

dn: ou=user,dc=example,dc=com
objectClass: organizationalUnit
ou: user

dn: ou=Group,dc=example,dc=com
objectClass: organizationalRole
cn: Group
```

> 3.2 创建目录的结构服务

```
ldapadd -x -D "cn=admin,dc=example,dc=com" -W -f /root/base.ldif
```

## 4、测试LDAP服务器上的用户认证信息

```
ldapsearch –x 用户可为空 -b "dc=example,dc=com"
#LDAP客户端配置
authconfig --enableldap --enableldapauth --ldapserver="192.168.0.2,192.168.0.3" --ldapbasedn="dc=exampl
```