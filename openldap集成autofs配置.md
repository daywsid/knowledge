# 服务端配置

## ldap服务添加autofs属性

```
ldapadd -Y EXTERNAL -H ldapi:/// -f autofs.ldif
cat autofs.ldif
dn: cn=autofs,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: autofs
olcAttributeTypes: {0}( 1.3.6.1.1.1.1.25 NAME 'automountInformation' DESC 'Information used by the autofs automounter' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 SINGLE-VALUE )
olcObjectClasses: {0}( 1.3.6.1.1.1.1.13 NAME 'automount' DESC 'An entry in an automounter map' SUP top STRUCTURAL MUST ( cn $ automountInformation $ objectclass ) MAY description )
olcObjectClasses: {1}( 1.3.6.1.4.1.2312.4.2.2 NAME 'automountMap' DESC 'An group of related automount objects' SUP top STRUCTURAL MUST ou )
```

## 导入配置automount.ldif

```
ldapadd -D "cn=admin,dc=example,dc=com" -W -f automount.ldif
cat automount.ldif

dn: ou=automount,dc=example,dc=com
ou: automount
objectClass: top
objectClass: organizationalUnit

dn: ou=auto.master,ou=automount,dc=example,dc=com
ou: auto.master
objectClass: top
objectClass: automountMap

dn: cn=/-,ou=auto.master,ou=automount,dc=example,dc=com
cn: /-
objectClass: automount
automountInformation: ldap:ou=auto.nfs,ou=automount,dc=example,dc=com --timeout=60 --ghost

dn: ou=auto.nfs,ou=automount,dc=example,dc=com
ou: auto.nfs
objectClass: automountMap

dn: cn=/test1,ou=auto.nfs,ou=automount,dc=example,dc=com
cn: /test1
objectClass: automount
automountInformation: -fstype=nfs 192.168.0.1:/test1

dn: cn=/test2,ou=auto.nfs,ou=automount,dc=example,dc=com
cn: /test2
objectClass: automount
automountInformation: -fstype=nfs,ro,fg,hard,intr,suid,proto=tcp,vers=3 192.168.0.1:/test2
```

> 备注：dn: cn=/-,ou=auto.master,ou=automount,dc=example,dc=com是指采用绝对路径配置
>
> 同时所有配置前提是已搭建好openldap服务,客户端已配置完成。

# 客户端配置

## 安装并设置开机自启动

```
yum install -y autofs
systemctl enable autofs.service --now
```

## 修改autofs配置

```
#echo "修改/etc/autofs.conf配置文件"
sed -i 's/^mount_nfs_default_protocol/#mount_nfs_default_protocol/g' /etc/autofs.conf
sed -i 's/#mount_nfs_default_protocol = 3/mount_nfs_default_protocol = 3/g' /etc/autofs.conf

#echo "修改/etc/sysconfig/autofs配置文件"
cat << EOF >>/etc/sysconfig/autofs
MASTER_MAP_NAME="ou=auto.master,ou=automount,dc=example,dc=com"
TIMEOUT=300
BROWSE_MODE="no"
MOUNT_NFS_DEFAULT_PROTOCOL=3
LOGGING="verbose"
LDAP_URI="ldap://192.168.0.2/"
SEARCH_BASE="ou=automount,dc=example,dc=com"
MAP_OBJECT_CLASS="automountMap"
ENTRY_OBJECT_CLASS="automount"
MAP_ATTRIBUTE="ou"
ENTRY_ATTRIBUTE="cn"
VALUE_ATTRIBUTE="automountInformation"
EOF
```

## 重启autofs

```
systemctl restart autofs.service
```