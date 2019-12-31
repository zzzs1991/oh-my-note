### 配置工程
1. 参数配置
- 字符参数
	1. name: branch
	2. default: master
	3. description: 选定分支
- 选项参数
	1. name: namespace
	2. options: test6 - test11
	3. description: 选定测试环境 
- 布尔参数
	1. name: restart
	2. default: true
	3. description: 是否重启服务 
2. 源码管理
	git
3. 构建过程

```sh
# 没有选择namespace，直接退出构建
if [ "$namespace" = "请选择" ];
then 
	exit 1 
fi
```
```sh
# 拉取对应分支代码
echo  '[INFO] git pull'

git fetch --all

echo "change branch $branch:"
git checkout $branch
git reset --hard origin/$branch
```
```sh
# docker 构建
echo '[INFO] begin build image ...'
mvn clean package docker:build -DpushImageTag -DdockerImageTags=$branch  -Dmaven.test.skip=true -P docker
```
```sh
# 是否重启项目
if [ "$restart" = "false" ];
then 
	exit 0 
fi
```
```sh
# k8s 
echo '[INFO] restart rancher server ...'


cd /home/newcore/rancher-v2.2.0
cat <<EOF | ./rancher kubectl replace --force -f - 

apiVersion: apps/v1beta2
kind: Deployment
metadata:
  annotations:
    field.cattle.io/creatorId: user-wd96g
  labels:
    cattle.io/creator: norman
    workload.user.cattle.io/workloadselector: deployment-$namespace-$JOB_BASE_NAME
  name: $JOB_BASE_NAME
  namespace: $namespace
  selfLink: /apis/apps/v1beta2/namespaces/$namespace/deployments/$JOB_BASE_NAME
spec:
  minReadySeconds: 30
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      workload.user.cattle.io/workloadselector: deployment-$namespace-$JOB_BASE_NAME
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      annotations:
      creationTimestamp: null
      labels:
        workload.user.cattle.io/workloadselector: deployment-$namespace-$JOB_BASE_NAME
    spec:
      containers:
      - env:
        - name: JAVA_OPTS
          value: -Xmx1024m
        - name: spring.druid.url
          value: jdbc:mysql://mysql:3306/inventory_$namespace?autoReconnect=true&amp;useUnicode=true&amp;characterEncoding=UTF-8
        - name: spring.datasource.url
          value: jdbc:mysql://mysql:3306/inventory_$namespace?autoReconnect=true&amp;useUnicode=true&amp;characterEncoding=UTF-8
        - name: spring.session.store-type
          value: none
        image: 192.168.1.243:5000/inventory:$branch
        imagePullPolicy: Always
        name: $JOB_BASE_NAME
        resources:
          limits:
            cpu: "2"
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 512Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities: {}
          privileged: false
          procMount: Default
          readOnlyRootFilesystem: false
          runAsNonRoot: false
        stdin: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        tty: true
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30

EOF
```
