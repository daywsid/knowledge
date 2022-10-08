# 一、安装duplicity

> 官网地址：https://duplicity.gitlab.io/

```
yum -y install librsync-devel python3-devel  python3-pip   gcc gcc-c++  librsync-devel 
python -m pip install --upgrade pip
pip3 install -r requirements.txt  -i https://pypi.mirrors.ustc.edu.cn/simple/ 
pip3 install future
pip3 install fasteners
```

> 下载解压安装

```
python3 setup.py install   默认安装到/usr
python setup.py install --prefix=/usr/local
```

# 二、使用duplicity

## 1.测试用例

```
duplicity --archive-dir=/mnt/sda4/data1/to --name=012n /mnt/sda4/data/to/012n file:///mnt/sda4/backup
duplicity restore   --file-to-restore=/bt1600/dhjdsa file:///mnt/sda4/backup /tmp/restore/dhjdsa
duplicity restore   file:///mnt/sda4/backup /tmp/restore
```

## 2.常用备份语句

```
duplicity collection-status file:///opt/sda4/backup/   
#列出备份的状态,有多少个完整备份及增量备份,备份的时间等等.
duplicity collection-status --ssh-options="-oPort=8888 -oIdentityFile=/home/abc/sK/id_rsa" s ftp://mysftp@backup/  /upload/
#--ssh-options 指定链接所使用的端口号  #-oIdentityFile 指定ssh密钥的位置
duplicity remove-older-than 1Y --force  file:///opt/sda4/backup/ 
#--force 如果不加上这个这个参数，则是列出要删除的文件，而不会删除他们。
#remove-older-than 删除比指定时间要旧的文件
#--no-encryption   不加密
   
duplicity list-current-files  file:///aaa 
##/aaa为备份的路径  查看备份的文档
duplicity restore file:///aaa /data/restore  
##恢复备份， file:///aaa是备份的路径 ##/data/restore是还原到的路径
duplicity restore --file-to-restore=passwd file:///aaa /abc/passwd  
##--file-to-restore使用的是相对路径， file:///aaa是备份文件的路径，/abc/passwd是文件恢复的路径，passwd是指定的文件名
   
duplicity restore --restore-time "2022-01-01T19:53:52" file:///aaa /bbb/passwd4
duplicity restore  --time time  file:///aaa /bbb/passwd4
#恢复指定时间备份的数据

duplicity --full-if-older-than 7D /etc file:///aaa
# 对重要数据，应经常做全量备份，用--full-if-older-than指定全量备份时间间隔。  自动选择备份类型 

#利用 crontab -e 设定每天凌晨3点自定执行脚本timedbackup.sh，写入 0 3 */1 * * timedbackup.sh 。脚本timedbackup.sh的内容如下：
duplicity --full-if-older-than 7D /etc file:///aaa

duplicity incr /etc oss://bucket-name/keyfolder/  
#增量备份
duplicity full /etc oss://bucket-name/keyfolder/   
#全量备份

FTP_PASSWORD=mypassword duplicity /local/dir ftp://user@other.host/some_dir
#备份，通过 ftp 使用 ftp 密码 mypassword 写入档案
```

## 3.官方备份案例

```
duplicity /home/me sftp://uid@other.host/some_dir   
#使用 sftp 将 /home/me 备份到 other.host 机器上的 some_dir
duplicity full /home/me sftp://uid@other.host/some_dir
#如果上述重复运行，第一次将是完整备份，后续将是增量备份。要强制进行完整备份，
duplicity --full-if-older-than 1M /home/me sftp://uid@other.host/some_dir
duplicity sftp://uid@other.host/some_dir /home/me   
#现在假设我们不小心删除了 /home/me 并希望将其恢复为上次备份时的状态
duplicity -t 3D --file-to-restore Mail/article sftp://uid@other.host/some_dir /home/me/restored_file    
#如果我们只想将 /home/me 中的文件“Mail/article”恢复到三天前的 /home/me/restored_file 中
duplicity verify sftp://uid@other.host/some_dir /home/me   
#以上命令将最新备份与当前文件进行比较
duplicity --exclude /mnt --exclude /tmp --exclude /proc / file:///usr/local/backup  
#以上将备份根目录，但不包括 /mnt、/tmp 和 /proc
duplicity --include /home --include /etc --exclude '**' / file:///usr/local/backup  
#以上将仅备份根目录下的 /home 和 /etc目录
FTP_PASSWORD=mypassword duplicity /local/dir ftp://user@other.host/some_dir
#Duplicity 也可以通过 ftp 访问存储库。如果给出了用户名，则读取环境变量 FTP_PASSWORD 以确定密码：
```

## 4.动作命令

```
full <folder> <url>   
#执行完整备份。即使签名可用于增量备份，也会启动新的备份链。
incr <folder> <url>  
#如果要求这样做，将执行增量备份。如果找不到旧签名，则 Duplicity 将中止。
duplicity collection-status file:///opt/sda4/backup/   
#列出备份的状态,有多少个完整备份及增量备份,备份的时间等等.
duplicity list-current-files  file:///aaa 
##/aaa为备份的路径  查看备份的文档
duplicity remove-older-than 1Y --force  file:///opt/sda4/backup/ 
 #--force 如果不加上这个这个参数，则是列出要删除的文件，而不会删除他们。
 #remove-older-than 删除比指定时间要旧的文件
  
remove-all-but-n-full <count> [--force] <url>   
#--force 如果不加上这个这个参数，则是列出要删除的文件，而不会删除他们。
#删除所有比完整备份更早的所有备份集的增量集（换句话说，只保留旧的完整备份而不保留它们的增量），count为1 意味着只有一个最近的备份链将保持不变
  
cleanup [--force] [--extra-clean] <url> 
#删除给定后端上无关的重复文件。不会删除非重复文件或完整数据集中的文件。只有在重复会话失败或过早中止后才需要这样做。请注意，需要--force来删除文件，而不是仅仅列出它们
```

## 5.选项options

```
--compare-data  --比较数据
在操作验证时启用常规文件的数据比较。出于性能原因，默认情况下禁用此功能
--copy-links   --复制链接
在备份期间解析符号链接。启用此功能将解析和备份符号链接的文件/文件夹数据，而不是符号链接本身，这可能会增加备份的大小。

--dry-run     --空运行
计算将要做什么，但不执行任何后端操作
--exclude  排除与shell_pattern匹配的文件。如果一个目录被匹配，那么该目录下的文件也将被匹配。
--exclude-device-files -排除设备文件

--exclude-filelist filename 排除文件名中列出的文件，文件列表的每一行都按照与--include和--exclude 相同的规则进行解释
--exclude-if-present filename 如果存在文件名，则排除目录。此选项需要位于任何其他包含或排除选项之前。

--exclude-older-than time 排除修改日期早于指定时间的任何文件。这可用于生成仅包含最近更改的文件的部分备份。
--exclude-other-filesystems  排除其他文件系统的文件
--exclude-regexp regexp  正则表达式
--extra-clean  清理时，更积极地节省空间。例如，这可能会删除旧备份链的签名文件。

--file-to-restore path 此选项可以在恢复模式下提供，只恢复路径而不是备份存档的全部内容。路径应该相对于备份目录的根目录给出。

--full-if-older-than time 如果请求增量备份，但集合中最新的完整备份早于给定时间，则执行完整备份。
--force  即使可能导致数据丢失，也要继续。Duplicity 将让用户知道何时需要此选项。

--include  类似于--exclude但包含匹配的文件。与--exclude不同，此选项还将匹配匹配文件的父目录
--include-filelist filename  与--exclude-filelist类似，但包含列出的文件。
--include-regexp regexp
--name symbolicname  符号名称目前仅用于影响--archive-dir的扩展
--no-compression不要使用 GZip 压缩远程系统上的文件。

--no-encryption  不要使用 GnuPG 加密远程系统上的文件。
--no-print-statistics  默认情况下，duplicity 将在成功备份后打印有关当前会话的统计信息。此开关禁用该行为。
--progress-rate number  设置 duplicity 输出上传进度消息的更新速率（需要--progress选项）。默认是每 3 秒提示一次状态。
```

## 6.环境变量

```
FTP_PASSWORD   大多数支持密码的后端都支持。
PASSPHRASE  密码短语，这个密码被传递给 GnuPG。如果未设置，将提示用户输入密码。
```

## 7.网址格式url

```
本地文件路径  file://[relative|/absolute]/local/path   文件：//[相对|/绝对]/本地/路径
ftp[s]://user[:password]@other.host[:port]/some_dir

Rsync via daemon

rsync://user[:password]@host.com[:port]::[/]module/some_dir
rsync://user@host.com[:port]/[relative|/absolute]_path   仅秘钥

scp://.. or
sftp://user[:password]@other.host[:port]/[relative|/absolute]_path
```

## 8.时间格式

```
-t、--time和--restore-time选项采用时间字符串，可以以多种格式给出：
字符串“now”（指当前时间）

一个数字序列，例如“123456890”（以秒为单位表示纪元之后的时间）

日期时间格式的字符串，如“2002-01-25T07:00:00+02:00”

间隔，它是一个数字，后跟字符 s、m、h、D、W、M 或 Y 中的一个（分别表示秒、分钟、小时、天、周、月或年），或一系列这样的对。在这种情况下，字符串指的是当前时间之前的时间间隔长度。例如，“1h78m”表示 1 小时 78 分钟前的时间。这里的日历很简单：一个月总是30天，一年总是365天，一天总是86400秒。

格式为 YYYY/MM/DD、YYYY-MM-DD、MM/DD/YYYY 或 MM-DD-YYYY 的日期格式，表示相关日期的午夜，相对于当前时区设置。例如，“2002/3/5”、“03-05-2002”和“2002-3-05”均表示 2002 年 3 月 5 日。
```

## 9.文件选择

```shell
--exclude
--exclude-device-files
--exclude-filelist
--exclude-regexp
--include
--include-filelist
--include-regexp


duplicity --include /usr/local/bin --exclude /usr/local /usr scp://user@host/backup 
将备份 /usr/local/bin 目录（及其内容），但不备份 /usr/local/doc。


例如，如果文件“list.txt”包含以下行：

/usr/local
- /usr/local/doc
/usr/local/bin
+ /var
- /var

然后--include-filelist list.txt将包括 /usr、/usr/local 和 /usr/local/bin。它将排除 /usr/local/doc、/usr/local/doc/pyth
```