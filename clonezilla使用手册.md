# 一、clonezilla 再生龙软件介绍，软件下载，镜像制作

## 简介：

>Clonezilla是一个用于Linux，Free-Net-OpenBSD，Mac OS X，Windows以及Minix的分区和磁盘克隆程序。它支持所有主要的文件系统，包括EXT，NTFS，FAT，XFS，JFS和Btrfs，LVM2，以及VMWare的企业集群文件系统VMFS3和VMFS5。Clonezilla支持32位和64位系统，同时支持旧版BIOS和UEFI BIOS，并且同时支持MBR和GPT分区表。它是一个用于完整备份Windows系统和所有安装于上的应用软件的好工具，而我主要用来来做系统克隆，私有化部署。
>
>Clonezilla也可以使用dd命令来备份不支持的文件系统，该命令可以复制块而非文件，因而不必在意文件系统。简单点说，就是Clonezilla可以复制任何东西。（关于块的快速说明：磁盘扇区是磁盘上最小的可编址存储单元，而块是由单个或者多个扇区组成的逻辑数据结构。）
>
>Clonezilla分为两个版本：Clonezilla Live和Clonezilla Server Edition（SE）。Clonezilla Live对于将单个计算机克隆到本地存储设备或者网络共享来说是一流的。而Clonezilla SE则适合更大的部署，用于一次性快速多点克隆整个网络中的PC。Clonezilla SE是一个神奇的软件，我们将在今后讨论。今天，我们将创建一个Clonezilla Live USB存储棒，克隆某个系统，然后恢复它。

## **制作再生龙启动盘**

> 1.下载镜像
>
> 再生龙的镜像在https://clonezilla.org/downloads.php下载，有两个选择，一个是基于Ubuntu、一个是基于Debian，都是稳定版，使用其实没什么差别。

![image-20220812190953267](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaimage-20220812190953267.png)

> 2.制作启动盘
>
> 启动盘的制作有很多种方式，这里选择 **UltraISO** 工具。
>
> 在工具栏选择【文件】->【打开】，然后选择下载好的镜像文件：

![image-20220812191234079](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaimage-20220812191234079.png)

>在工具栏选择【启动】->【写入硬映像】
>
>选择被制作启动盘的U盘，然后【格式化】、【写入】。

![image-20220812191323487](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaimage-20220812191323487.png)





# 二、系统备份

>把我们之前制作的U盘插到物理机上,修改bios的boot引导项，选择优先走刚才所选的U盘加载，F10保存修改，重启电脑
>
>现在我们看到了clonezilla的系统启动界面，他提供了许多模式，我们选择默认的第一种,如下图:

![QQ图片20220812160744](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812160744.png)

> clonezilla语音选择，我们选择简体中文

![QQ图片20220812160843](https://wsidtypora.oss-cn-beijing.aliyuncs.com/img/QQ图片20220812160843.png)

> 操作系统键盘映射关系，我们选择默认

![QQ图片20220812160905](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812160905.png)

> 初次使用，建议不要使用命令行操作，选择默认

![QQ图片20220812160923](https://wsidtypora.oss-cn-beijing.aliyuncs.com/img/QQ图片20220812160923.png)

> 选择clonezilla存储镜像方式

![QQ图片20220812160948](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812160948.png)

> 选择clonezilla存储镜像目录，默认选择local_dev，按Enter继续

![QQ图片20220812161026](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161026.png)

>现在可以插入保存镜像image的U盘，等检测到以后按Ctrl+C 退出（建议提前插好U盘等介质，已更好的识别到）

![QQ图片20220812143650](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812143650.png)

>选择存放镜像的介质或者路径，这个硬盘可以是你的usb或者本地硬盘,这里默认选择本地U盘

![QQ图片20220812161202](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161202.png)

> 默认选项 ，跳过检查与修正来源分区的文件系统

![QQ图片20220812161223](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161223.png)

> 再生龙镜像仓库目录选择器，确定目录后，选择Done然后回车进行下一步

![QQ图片20220812161311](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161311.png)

> 备注：如果是还原的时候目录浏览器应该如下图，都是默认选择即可

![QQ图片20220812201524](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812201524-16603072373605.png)

> 默认的模式已经能够满足我们的需求了，有特殊需求的可以选择专家模式，里面有更多的自定义选项，如修改分区

![QQ图片20220812161334](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161334.png)

> 选定模式，储存本机硬盘为镜像文件

![QQ图片20220812161420](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161420.png)

> 数据存储目录命名（他这里并不是镜像文件，镜像文件还需要再次打包）

![QQ图片20220812161443](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161443.png)

> 选择需要克隆的硬盘,这里选择系统盘

![QQ图片20220812161515](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161515.png)

> 默认选项，跳过检查与修正来源分区的文件系统

![QQ图片20220812161702](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161702.png)

>默认选项，检查保存的镜像

![QQ图片20220812161719](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161719.png)

> 默认选项，不对镜像加密

![QQ图片20220812161805](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161805.png)

>默认选项，当所有操作完成，选择重启或关机，然后按Enter继续

![QQ图片20220812161836](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161836.png)

> 选择y继续下一步操作

![QQ图片20220812161911](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161911.png)

> 系统开始执行克隆，他会一个分区一个分区的克隆，下图为进度条展示

![QQ图片20220812161938](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812161938.png)

> 系统克隆完毕，并且校验通过，按Enter继续

![QQ图片20220812163401](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812163401.png)

>克隆完毕，重启系统，克隆教程到此结束

![QQ图片20220812163507](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812163507.png)





# 三、系统还原

> 系统还原之前的操作和系统备份是一样的（核心步骤包括存储路径选择，目录名设置，普通模式，重新按照系统备份流程走一遍，一直到出现这个界面），选择还原镜像文件到本地硬盘

![QQ图片20220812195143](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812195143.png)

> 选择还原的镜像名

![QQ图片20220812195237](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812195237.png)

> 选择目标机器的系统盘，数据会恢复到这里，并且建议新的系统盘大小要大于原来系统的系统盘

![QQ图片20220812201727](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812201727.png)

> 默认操作，在还原前检查数据是否可还原

![image-20220812203415809](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaclonezillaclonezillaimage-20220812203415809.png)

> 默认操作下一步，按Enter继续

![image-20220812202004012](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaimage-20220812202004012.png)

> 默认操作下一步，系统开始进行还原前检查

![QQ图片20220812201902](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812201902.png)

![QQ图片20220812202311](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812202311.png)

> 检查完毕后开始执行系统恢复流程，恢复完毕后选择重新启动

![QQ图片20220812203022](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812203022.png)

![QQ图片20220812203043](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812203043.png)

> 新操作系统启动成功

![QQ图片20220812204142](https://raw.githubusercontent.com/daywsid/typora-img/main/clonezillaQQ%25E5%259B%25BE%25E7%2589%258720220812204142.png)





# 四、文档参考及常见问题

> 参考文档地址：
>
> https://www.likecs.com/show-203792745.html
>
> https://www.isolves.com/it/rj/czxt/linux/2021-07-04/41092.html

> 常见问题：
>
> 因为只是做硬盘层面的恢复，所以最好做一下硬件的检测，比如网卡信息可能需要手工配置一下，修改下ip或者mac地址啥的
>
> 引导丢失系统会默认有grub工具，可以用于恢复
>
> 软件版本号一定要对上，先确认好该用32位的还是64位的
>
> 有时候我们觉得使用文件系统太麻烦，可以直接把所有的资料打包位一个iso镜像，选择产生恢复专用的再生龙，只需要在新机器安装这个镜像，就可以把系统打包过去，非常方便。
