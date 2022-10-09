# **Cgroup 安装配置**

> 一般我们都使用 libcgroup 提供的配置方法，为了使用 cgroup，我们需要先安装libcgroup

```
yum install libcgroup libcgroup-tools -y
```

> 重启服务使配置生效

```
[root@localhost ~]# systemctl restart cgconfig.service
[root@localhost ~]# systemctl restart cgred.service
```

> 限制cpu使用 跟cpu相关的子系统有cpu、cpuacct和cpuset。其中：
>
> cpuset主要用于设置cpu的亲和性，可以限制cgroup中的进程只能在指定的cpu上运行。cpuacct包含当前cgroup所使用的CPU的统计信息。cpu：限制cgroup的cpu使用上限。
>
> cpu.stat
>
> 包含了下面三项统计结果
>
> nr_periods：表示过去了多少个cpu.cfs_period_us里面配置的时间周期
>
> nr_throttled：在上面的这些周期中，有多少次是受到了限制(即cgroup中的进程在指定的时间周期中用光了它的配额)
>
> throttled_time: cgroup中的进程被限制使用CPU持续了多长时间(纳秒)
>
> cpu：cpu使用时间限额 cpu.cfs_period_us和cpu.cfs_quota_us来限制该组中的所有进程在单位时间里可以使用的cpu时间。这里的cfs是完全公平调度器的缩写。cpu.cfs_period_us就是时间周期(微秒)，默认为100000，即百毫秒。cpu.cfs_quota_us就是在这期间内可使用的cpu时间(微秒)，默认-1，即无限制。(cfs_quota_us是cfs_period_us的两倍即可限定在双核上完全使用)。
>
> 参考文档：https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/resource_management_guide/sec-unmounting_a_hierarchy
>
> ```
> 注意
> 请重启 cgconfig 服务，让 /etc/cgconfig.conf 的更改生效。重启此服务会重建配置文件中指定的层级，但并不会影响所有挂载层级。您可以通过执行 systemctl restart 指令来重启服务，但是，建议您先停止 cgconfig：
> ~]# systemctl stop cgconfig
> 然后打开并编写配置文件。保存更改后，您可以用以下指令再次启动 cgconfig：
> ~]# systemctl start cgconfig
> 
> /etc/cgrules.conf 文件中的条目可包括以下额外符号：
> @ —— 被用作 user 前缀时，它代表一个群组而不是单独用户。例如：@admins 表示 admins 组群中的所有用户。
> * —— 代表“所有”。例如：subsystem 字段中的 * 代表所有子系统。
> % —— 代表与上一行相同的项目。例如：
> @adminstaff        net_prio   /admingroup
> @labstaff        %          %
> ```

> Cgroup 只要有两个配置文件: /etc/cgconfig.conf   /etc/cgrules.confg，示例如下：

```
#cgconfig.conf 
group userlimit {
    cpu{
            cpu.cfs_quota_us = 200000;
            cpu.cfs_period_us = 100000;
    }
    memory {
        memory.limit_in_bytes = 10G;
    }
}


template userlimit/%u {
     cpu{
            cpu.cfs_quota_us = 40000;
            cpu.cfs_period_us = 50000;
      }

   memory {
        memory.limit_in_bytes = 10G;
        memory.memsw.limit_in_bytes = 10G;
    }
}

#cgrules.conf
root    cpu,memory  userlimit/%u
@abc    cpu,memory  userlimit/%u
mount -t cgroup -o memory memory /sys/fs/cgroup/memory
mount -t cgroup cgroup /sys/fs/cgroup
mount -t cgroup -o cpu,cpuacct cpu /cgroup/cpu
yum install -y libcgroup-tools   libcgroup
测试cgroup
while :; do echo test > /dev/null; done
cd /sys/fs/cgroup/cpu,cpuacct
mkdir test

echo $$ ;   "echo 5456 > cgroup.procs"
rmdir /sys/fs/cgroup/cpu/mycgroup
#删除cgroup
cgdelete cpu:userlimit/root
cgdelete memory:usermemlimit/userid
cgdelete memory:usermemlimi  #所有用户删除完成以后才可执行

1.限制只能使用1个CPU（每250ms能使用250ms的CPU时间）
    # echo 250000 > cpu.cfs_quota_us /* quota = 250ms */
    # echo 250000 > cpu.cfs_period_us /* period = 250ms */

2.限制使用2个CPU（内核）（每500ms能使用1000ms的CPU时间，即使用两个内核）
    # echo 1000000 > cpu.cfs_quota_us /* quota = 1000ms */
    # echo 500000 > cpu.cfs_period_us /* period = 500ms */

3.限制使用1个CPU的20%（每50ms能使用10ms的CPU时间，即使用一个CPU核心的20%）
    # echo 10000 > cpu.cfs_quota_us /* quota = 10ms */
    # echo 50000 > cpu.cfs_period_us /* period = 50ms */
```

## 附录为测试cpu和内存的脚本程序

```
#测试cgroup内存
MemGB.c:
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#define GB (1024 * 1024 *1024)
int main(int argc, char *argv[])
{
char *p;
int i = 0;
while(1) {
p = (char *)malloc(GB);
memset(p, 0, GB);
printf("%dG memory allocated\n", ++i);
sleep(1);
}
return 0;
}$gcc -o MemGB MemGB.c
#测试CPU的脚本
cat cpu_up.sh
#!/bin/sh
PWD=$(cd $(dirname $0); pwd)
cpu_num=$1

if [ $# != 1 ]
then
  printf "\e[0;34mUSAGE: $0 <CPUs>\e[0m\n"
  exit 1
fi

for i in $(seq ${cpu_num})
do
    echo -ne "
    i=0;
while true
do
    i=i+1;
done" | /bin/sh &
    pid_array[${i}]=$! ;
done

for i in "${pid_array[@]}"; do
    printf "\e[0;32mkill ${i}\e[0m\n" >> ${PWD}/kill_cpu_up.log 2>&1
done
#测试时sh cpu_up.sh 2,cpu_up.sh是自定义的，这个不受限制,表示使用2个CPU，可以根据自己的情况去设定

while :; do echo test > /dev/null; done  
#测试1个CPU时使用
```
