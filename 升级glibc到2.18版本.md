# Centos7.X 默认情况下自带的glibc版本为glibc-2.17, 但很多运行在centos上的应用需要高版本glibc才能成功编译和安装

## **以下安装步骤在生产上验证通过**

> 安装步骤

```
wget http://ftp.gnu.org/gnu/glibc/glibc-2.18.tar.gz
tar -zxvf glibc-2.18.tar.gz 
cd glibc-2.18
mkdir build
cd build
../configure --prefix=/usr --disable-profile --enable-add-ons --with-headers=/usr/include --with-binutils=/usr/bin
make -j4
make install
```

> 安装验证

```
ldd --version
rpm -qa | grep glibc
strings /lib64/libc.so.6 |grep GLIBC_
```