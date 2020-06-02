### 1. centos更改机器的dns
```shell
nmcli connection show查看网卡
cd /etc/sysconfig
cd networkscript
vim 对应的网卡文件
添加DNS1 DNS2
重启网卡
systemctl restart NetworkManager.service
cat /etc/resolv.conf
```
