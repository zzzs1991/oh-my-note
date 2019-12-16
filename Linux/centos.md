```shell


以下就是关闭SELinux的方法系统版本：centos 6.5 mini1、查看selinux状态查看selinux的详细状态，如果为enable则表示为开启
# /usr/sbin/sestatus -v查看selinux的模式
# getenforce

2、关闭selinux
2.1:永久性关闭（这样需要重启服务器后生效）
# sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
2.2:临时性关闭（立即生效，但是重启服务器后失效）
# setenforce 0 
#设置selinux为permissive模式（即关闭）
# setenforce 1 
#设置selinux为enforcing模式（即开启）这样就关闭SELinux了，当遇到问题时可以考虑关闭SELinux再进行安装



```

