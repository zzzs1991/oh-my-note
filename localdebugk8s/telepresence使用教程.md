## 使用telepresence连接远程k8s集群进行远程调试
### 一. mac电脑
#### 1. 安装telepresence
##### 1.1 通过homebrew安装

###### 1.1.1 安装home brew

> homebrew是mac系统的软件包管理器
> https://brew.sh/index_zh-cn
>
> 1 telepresence官方推荐使用homebrew安装
> 安装时需要使用原始源 配合 科学上网
>
> 安装常规软件可以使用国内阿里巴巴镜像源
> https://developer.aliyun.com/mirror/homebrew?spm=a2c6h.13651102.0.0.3e221b11SG1CwE

###### 1.1.2 安装telepresence

>brew cask install osxfuse
>brew install datawire/blackbird/telepresence
>
>需要配合科学上网 和 代理
>若公司网安装不上可以用手机热点尝试

#### 2. 安装docker

> https://www.docker.com/products/docker-desktop
> 安装后会有kubectl工具

#### 3. 配置测试环境k8s登录证书

> 将下列yaml文本存入 ~/.kube/config文件中

```yml
apiVersion: v1
kind: Config
clusters:
- name: "local"
  cluster:
    server: "https://192.168.1.250/k8s/clusters/c-vdfdn"
    certificate-authority-data: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM3akNDQ\
      WRhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFvTVJJd0VBWURWUVFLRXdsMGFHVXQKY\
      21GdVkyZ3hFakFRQmdOVkJBTVRDV05oZEhSc1pTMWpZVEFlRncweE9URXhNRFF4TmpRM01qaGFGd\
      zB5T1RFeApNREV4TmpRM01qaGFNQ2d4RWpBUUJnTlZCQW9UQ1hSb1pTMXlZVzVqYURFU01CQUdBM\
      VVFQXhNSlkyRjBkR3hsCkxXTmhNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ\
      0tDQVFFQTFtY2ExVEtTbkY4Rk5UNnQKRVlWclJvT3ZyLzhhcWpJZ1lXaEpEalpMNnloZHJoN3d3Z\
      jZMZ0d6UU12bXZ3d2NPMFo2L0ViSUw4STVGVCs4UwpTcDU5MWRjNjNCN2pnOCs1dSszbDIwZm9SN\
      npDYlVUa0ZFYWxMWXd6MWFnU0J3L1I0MWtndmpUS2psRWd2OEUwCmRHR1BWSnJNbVFlN0pYMFpYd\
      ThYSFZRRGs0akw3ckFsNmV2UHAyZG51VWR3a1dhU2Q5RURxQnR0WVNrclY1UHQKSjlrMmtlY2NTT\
      GNRNm1DWW4wZm5mbTZEaWxjWDMvM1gwMUg4SmR0VFN2MGUvTTNkOHZPWndieEN0L1RyT09OKwoxa\
      3hLYjVvV0EvM0JKSDB5cTdLMDBNeUlFSml4OGkvOFFqdkZyd0o4N1IxNWwvbmJQNGplUjdHZDlZN\
      DUxR1E1CmlOWHBXUUlEQVFBQm95TXdJVEFPQmdOVkhROEJBZjhFQkFNQ0FxUXdEd1lEVlIwVEFRS\
      C9CQVV3QXdFQi96QU4KQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBcUE3ZHM1clpNOGdEbnVLUjRhZ\
      GZ1aTFraUxuaFhIQlhCR2haR0tKMQozdmhrNDRTNlAyaGdWL2Y1S2JDYjdJVGdjV2NjVmNDMEc1W\
      jFrM0NNL296TFhJL3FJSnZTc2JZZXpJNy9NazdqCi82bkZsZm5adXBoaFgvc3l6R3J1MStvblJGa\
      0lRYnMzam9GN0VqL2piT0gwcmtOR1RZV1p1WVM0ZGdZV2NITlkKdGhwQWttREtHc0g2cVI4QSs3O\
      DNUbjN0ZVl3M0pvTWIyZ3FUVnhCSTQvTmZRTmllS2xCU0hqL1hBdkppV1ZmZAowNDR0ZVNJQTJqd\
      1BreUU3TWpmMWZiTzNHSnRDS1ZIdUk5OGJYTGhvVTIraEMrUWZ5YzRIUS80KzRRaVRyRDRsCmhYV\
      jV2Wk9CVWMvMFM4dXR1MWtTaXlKY0hpYi91Qy9vT3ZydDh0YzdwbzlLQUE9PQotLS0tLUVORCBDR\
      VJUSUZJQ0FURS0tLS0t"
- name: "local-dev10"
  cluster:
    server: "https://192.168.1.250:6443"
    certificate-authority-data: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN3akNDQ\
      WFxZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFTTVJBd0RnWURWUVFERXdkcmRXSmwKT\
      FdOaE1CNFhEVEU1TVRFd05ERTJOVE16T1ZvWERUSTVNVEV3TVRFMk5UTXpPVm93RWpFUU1BNEdBM\
      VVFQXhNSAphM1ZpWlMxallUQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ\
      0VCQU1KUXdWUm5DYjhECnhzMWJoM2pGa2dubzZRcDhBY1RLZE02Zndxay93dE9CdSt2MjdYK0wxT\
      1F5Ti80T0RxbmhncXhPZDV5RVh0Q1EKNXlrMEtWMTI2cjBSeFR3a3BGZncxMWxLditzMnJLTkM5N\
      mUrcm0yOVhlYVhwSXZBclhEa3NRdy9nY2tCL25wTQpzUE5ZTDgrOWtFRDBzU2lESTd6QXRmbFBKL\
      09rTXVheHNyNjBuU0kraFVrTnNIZjdLRUJ2eVVOREdsZ2VUOFUyCkNLTjFSSmR2RkVNclVtZVQwb\
      3ZMSUszeWJmRC9kU2hKaklRWk9mL2xNUkw1V2U1c2NtVWMzaVFwZ1dsZkU3NmkKMzFXSFljSUt4Y\
      lRyQkx0ekRKT0ZqVEx3TUNwanpYdTNMcjdqWGU1VjM1Z0RjcTZxRVFGZk84QnVCSDgzMjl5TApBZ\
      k94RnhlaDBNRUNBd0VBQWFNak1DRXdEZ1lEVlIwUEFRSC9CQVFEQWdLa01BOEdBMVVkRXdFQi93U\
      UZNQU1CCkFmOHdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBSERrcWpjSE96cGZxa0l2T05mSXdXd\
      mhNbnZtN1RMd0UvK0EKQTgrSStqbjdrNzZQbndSdEhZWWtzdFJBNzZ0U2Z5VFBBQW9ZaVI1VG95d\
      1ZUQkhzR0IvcnQrS2daamFSSWhqRwp6M2tQTTRaa05Va240YXgxaVltaUxmcGFRaXlXdmFLOCtHS\
      E8wMlZ0L2hNWHZDbVBRR3VyK3QxejhHM3RzVmtCCmdDem1PWmt4aHRNb3hXMmZWYWowa0ttRUJRd\
      G44SW1MaUlHWFlpSGFCSEowblU5NjRna0wzdFNSbW5oMmVCb3cKNVYxQzY2d2NhOWhqQ3FTUGVwe\
      kVmTzR6ZGkxMnREL3craG5pQ1FPckpEbkhqaVgxVHJrRXpwSkV0Q0Z2NjZtOQp4SEFwZ3lWQllUU\
      2Q1Tkd3QTg3NVp3aDd3V3d1Q05SVmJQRXc3cVhwLzFLRHUreWx3cDA9Ci0tLS0tRU5EIENFUlRJR\
      klDQVRFLS0tLS0K"

users:
- name: "local"
  user:
    token: "kubeconfig-user-ws2tr.c-vdfdn:g6fj9ct9bspghjtxqzc24twh7f64ng8p4nj7qtqftvkqc9shqsgrsx"

contexts:
- name: "local"
  context:
    user: "local"
    cluster: "local"
- name: "local-dev10"
  context:
    user: "local"
    cluster: "local-dev10"

current-context: "local"

```

```shell
# 执行下列命令,若显示如下六个节点则说明配置成功
➜  ~ kubectl get nodes
NAME     STATUS   ROLES               AGE    VERSION
dev1     Ready    <none>              164d   v1.15.5
dev10    Ready    controlplane,etcd   186d   v1.15.5
dev2     Ready    <none>              172d   v1.15.5
dev245   Ready    worker              66d    v1.15.5
dev7     Ready    worker              67d    v1.15.5
dev9     Ready    worker              186d   v1.15.5
```



#### 4. 通过telepresence来本地debug-以user-center为例

##### 4.1 执行如下命令

```shell
telepresence --namespace test7 -s user-center --expose 18080:8080
```

>--namespace test7 指定namespace
>-s user-center 将指定namespace中的user-center服务替换掉
>--expose 18080:8080 PORT[:REMOTE_PORT] 
>				新核云后端服务镜像监听的大多是8080端口
>				将8080端口映射到本地的18080端口

```shell
# 执行成功后
➜  ~ telepresence --namespace test7 -s user-center --expose 18080:8080
T: Starting proxy with method 'vpn-tcp', which has the following limitations:
T: All processes are affected, only one telepresence can run per machine, and
T: you can't use other VPNs. You may need to add cloud hosts and headless
T: services with --also-proxy. For a full list of method limitations see
T: https://telepresence.io/reference/methods.html
T: Volumes are rooted at $TELEPRESENCE_ROOT. See
T: https://telepresence.io/howto/volumes.html for details.
T: Starting network proxy to cluster by swapping out Deployment user-center
T: with a proxy
T: Forwarding remote port 8080 to local port 18080.

T: Guessing that Services IP range is 10.43.0.0/16. Services started after this
T:  point will be inaccessible if are outside this range; restart telepresence
T: if you can't access a new Service.
T: Connected. Flushing DNS cache.
T: Setup complete. Launching your command.

The default interactive shell is now zsh.
To update your account to use zsh, please run `chsh -s /bin/zsh`.
For more details, please visit https://support.apple.com/kb/HT208050.
@local|bash-3.2$

# 出现最后一行内容即为成功
```

>此时在rancher页面上可以观察到:
>
>原来的user-center镜像已经被替换掉
>user-center-72f7aed3120b4e3280b858f9be8e67aa
>
>使用的镜像为 
>datawire/telepresence-k8s:0.105

##### 4.2 本地debug启动对应项目

> IDEA中启动按钮左侧 Edit Configurations...
>
> SpringBoot
> 		XXXApplication
> 				Active profiles 推荐设置为local
>
> 复制resource.docker文件夹 为 resource.local
> 		设置application-local.yml 中 server.port为telepresence命令指定的端口
> 		修改logback-spring.xml中<properties name="LOG_NAME" value="本机文件位置"









