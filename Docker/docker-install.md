


### 卸载旧版本
```bash
➜  ~ apt-get remove docker \
> docker-engine \
> docker.io
```
### 安装必要的一些系统工具
```bash
➜  ~ apt-get -y install apt-transport-https ca-certificates curl software-properties-common 
```
### 安装 GPG 证书
```bash
➜  ~ curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
OK
```
### 更新并安装 Docker CE
```bash
sudo apt-get -y update
sudo apt-get -y install docker-ce
```
### 启动 Docker CE
```bash
sudo systemctl enable docker
sudo systemctl start docker
```
### 建立 docker 用户组
```bash
sudo groupadd docker
```
### 将当前用户加入docker组
```bash
sudo usermod -aG docker $USER
```
### 测试 Docker 是否安装正确
```bash
docker run hello-world
```
### 镜像加速
```bash
编辑
/etc/docker/daemon.json
```
```json
{
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}
```
重启docker服务
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```
