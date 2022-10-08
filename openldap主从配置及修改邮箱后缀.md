# openldap搭建此处省略

## Master机器配置,添加模块syncprov

```
ldapadd -Y EXTERNAL -H ldapi:/// -f syncprov_mod.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f syncprov.ldif

cat syncprov_mod.ldif
dn: cn=module,cn=config
objectClass: olcModuleList
cn: module
olcModulePath: /usr/lib64/openldap
olcModuleLoad: syncprov.la

cat syncprov.ldif
dn: olcOverlay=syncprov,olcDatabase={2}hdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: syncprov
olcSpCheckpoint: 1 1
olcSpSessionLog: 1024
```

## Slave机器配置

> 需要根据实际情况修改的参数：

- provider 同步来源，也就是主节点，可以包含多个主节点
- binddn 主节点管理账户
- credentials 主节点管理账户密码
- searchbase 根目录

```
ldapmodify -Y EXTERNAL -H ldapi:/// -f slave.ldif

cat slave.ldif
dn: olcDatabase={2}hdb,cn=config                        
changetype: modify
add: olcSyncRepl
olcSyncRepl:
  rid=001
  provider=ldap://192.168.0.1:389/
  bindmethod=simple
  binddn="cn=admin,dc=example,dc=com"
  credentials=password
  searchbase="dc=example,dc=com"
  scope=sub
  schemachecking=on
  type=refreshAndPersist
  retry="30 5 300 3"
  attrs="*,+"
  interval=00:00:00:02
-
add: olcDbIndex
olcDbIndex: uid eq,pres
olcDbIndex: uniqueMember eq,pres
olcDbIndex: uidNumber,gidNumber eq,pres
olcDbIndex: member,memberUid eq,pres
olcDbIndex: entryUUID eq
olcDbIndex: entryCSN eq
```

> /etc/openldap/slapd.d/cn=config目录可以查看dn: olcDatabase={2}hdb,cn=config

# 批量修改ldap邮箱后缀

## 导出用户信息

```
ldapsearch -x -b 'ou=user,dc=example,dc=com' -D "cn=admin,dc=example,dc=com" -W mail > mail.ldif
```

## 编辑修改方式

```
vim add.txt
changetype: modify
replace: mail
```

> 批量插入

```
sed -i '/^dn:/r add.txt' mail.ldif
```

> 批量修改邮箱后缀

```
sed -i 's/test1.com/test2.com/g' mail.ldif
```

> 导入更新信息

```
ldapmodify -x -D "cn=admin,dc=example,dc=com" -W  -f mail.ldif
```