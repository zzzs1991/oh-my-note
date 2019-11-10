## /etc 

​	一直都好奇，为什么linux下所有的配置文件都要放置于/etc目录下，这个etc是什么意思。网上有这么几种说法。

---

### 1.拉丁文解释

> etc不是缩写，是and so on的意思 
>
> 来源于 法语的 et cetera 翻译成中文就是 等等 的意思。
>
> 至于为什么在/etc下面存放配置文件， 按照原始的UNIX的说法(linux文件结构参考UNIX的教学实现MINIX) 这下面放的都是一堆零零碎碎的东西, 就叫etc, 这其实是个历史遗留.

---

### 2.英文缩写解释

> "Editable Text Configuration"：
>
> 所以常看到这样的目录介绍
> /etc - Usually contain the configuration files for all the programs that run on your Linux/Unix system.



---

### 3./etc下文件解析

```
/etc目录
　　包含很多文件.许多网络配置文件也在/etc 中. 

/etc/rc  or/etc/rc.d  or/etc/rc*.d 
　　启动、或改变运行级时运行的scripts或scripts的目录. 

/etc/passwd 
　　用户数据库，其中的域给出了用户名、真实姓名、家目录、加密的口令和用户的其他信息. 

/etc/fdprm 
　　软盘参数表.说明不同的软盘格式.用setfdprm 设置.

/etc/fstab 
　　启动时mount -a命令(在/etc/rc 或等效的启动文件中)自动mount的文件系统列表.Linux下，也包括用swapon -a启用的swap区的信息.

/etc/group 
　　类似/etc/passwd ，但说明的不是用户而是组. 

/etc/inittab 
　　init 的配置文件. 

/etc/issue 
　　getty在登录提示符前的输出信息.通常包括系统的一段短说明或欢迎信息.内容由系统管理员确定. 

/etc/magic 
　　file 的配置文件.包含不同文件格式的说明，file 基于它猜测文件类型.

/etc/motd 
　　Message Of TheDay，成功登录后自动输出.内容由系统管理员确定.经常用于通告信息，如计划关机时间的警告. 

/etc/mtab 
　　当前安装的文件系统列表.由scripts初始化，并由mount 命令自动更新.需要一个当前安装的文件系统的列表时使用，例如df命令. 

/etc/shadow 
　　在安装了影子口令软件的系统上的影子口令文件.影子口令文件将/etc/passwd 文件中的加密口令移动到/etc/shadow中，而后者只对root可读.这使破译口令更困难. 

/etc/login.defs 
　　login 命令的配置文件. 

/etc/printcap 
　　类似/etc/termcap ，但针对打印机.语法不同. 

/etc/profile , /etc/csh.login ,/etc/csh.cshrc 
　　登录或启动时Bourne或Cshells执行的文件.这允许系统管理员为所有用户建立全局缺省环境. 

/etc/securetty 
　　确认安全终端，即哪个终端允许root登录.一般只列出虚拟控制台，这样就不可能(至少很困难)通过modem或网络闯入系统并得到超级用户特权. 

/etc/shells 
　　列出可信任的shell.chsh 命令允许用户在本文件指定范围内改变登录shell.提供一台机器FTP服务的服务进程ftpd检查用户shell是否列在 /etc/shells 文件中，如果不是将不允许该用户登录. 

/etc/termcap 
```

