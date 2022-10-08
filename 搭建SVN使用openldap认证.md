# 搭建SVN使用openldap认证

## 关闭防火墙和selinux

```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g'/etc/selinux/config
systemctl disable firewalld.service
systemctl stop firewalld.service
```

## 安装软件包

```
yum install –y *sasl* subversion mod_ldap httpdmod_dav_svn
```

## 配置**sasl使用ldap认证**

```
sed -i 's/MECH=shadow/MECH=ldap/g' /etc/sysconfig/saslauthd
```

## 编辑saslauthd.conf文件，默认不存在，需要自己添加，具体内容依照ldap服务端配置

```
ldap_servers: ldap://192.168.0.2
ldap_bind_dn: cn=admin,dc=example,dc=com
ldap_bind_pw: wsid123456
ldap_search_base: ou=user,dc=example,dc=com
ldap_filter: uid=%U
ldap_password_attr: userPassword
```

## 配置svn通过ldap验证,在ldap服务器中添加svn.conf文件，默认没有该文件，需要自己添加

```
cat /etc/sasl2/svn.conf
pwcheck_method: saslauthd
mech_list: PLAIN LOGIN
```

## 新建svn仓库

```
svnadmin create /opt/svn/XW
```

## svn服务器中修改svn服务器配置

```
sed -i '/use-sasl/s/^# //' /opt/svn/XW/conf/svnserve.conf
```

## 启动svn服务

```
svnserve -d -r /opt/svn
ps -ef | grep svnserve
```

## 配置subversion

```
Cat /etc/httpd/conf.d/subversion.conf
LoadModule dav_svn_module modules/mod_dav_svn.so
LoadModule authz_svn_module modules/mod_authz_svn.so
LoadModule dontdothat_module  modules/mod_dontdothat.so

<Location />

DAV svn
SVNParentPath /opt/svn
SVNListParentPath on
AuthzSVNAccessFile /opt/svn/conf/authz

AuthType Basic
AuthName "Authorization SVN"

AuthBasicProvider ldap
AuthLDAPURL "ldap://192.168.0.2:389/ou=user,dc=example,dc=com?uid?sub?(objectclass=*)"
AuthLDAPBindDN "cn=admin,dc=example,dc=com"
AuthLDAPBindPassword "wsid123456"

Require valid-user
</Location>
```

> 同时修改

```
cat /etc/httpd/conf/httpd.conf
ServerName localhost:80
<Directory />
    AllowOverride none
    Require all granted
    Allow from all
</Directory>
DocumentRoot "/opt/svn"
```