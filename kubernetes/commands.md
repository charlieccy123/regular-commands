# Kubernetes&容器



## Docker

```bash
docker login --username=admin --password=PASSWORD REPO_URL
docker login --username=admin --password=PASSWORD REPO_URL
docker logout REPO_URL

docker save -o xxx.tar 镜像:tag
docker load -i ./xxx.tar

docker tag 源镜像:TAG  目标仓库:TAG

# 清理没有使用的数据包括镜像数据和已经停止的容器，这个命令慎用
docker system prune -a
systemctl stop docker && systemctl start docker

# 提供DOCKER整体磁盘使用情况，镜像、容器
docker system df

# 清理 72小时没有使用的镜像
docker image prune -a --force --filter "until=72h"

# 强制删除所有镜像
docker image prune --force --all

# 清理所有构建缓存
docker builder prune -a

# 清理磁盘 删除没用的容器 无用的数据卷，包括构建cache
docker system prune --all

# 拉指定平台的
docker pull --platform amd64 nginx

# 指定dockerfile并制定构建那种平台的镜像，-t为镜像标签
docker build -f ./docker/Dockerfile_testnet --platform linux/x86_64 -t index-f7711f7 .
```

## kaniko 构建

```bash
docker login --username=admin --password=PASSWORD REPO_URL

docker run -ti --rm -v `pwd`:/workspace -v /Users/charlie/.docker/config.json:/kaniko/.docker/config.json:ro gcr.io/kaniko-project/executor:latest --dockerfile=./Dockerfile --destination=aaa:1.0 --no-push
```

## helm

```bash
helm template kube-prometheus-stack -n monitoring --name-template prometheus --output-dir ~/Downloads
```

## Containerd

```bash
# runC 是创建和启动容器的，containerd是管理容器生命周期的

# 列出镜像
ctr images ls

# 下载镜像，它默认会下载你当前平台的镜像，注意下载镜像的镜像仓库要写完整
ctr images pull docker.io/library/nginx:1.9.0
ctr images pull --all-platforms docker.io/library/nginx:1.9.0  # 下载所有平台的镜像

# 挂载镜像，把镜像挂载到指定目录，这样就可以看镜像里面的内容
ctr images mount docker.io/library/nginx:1.9.0 /mnt
umount /mnt   # 解除挂载

# 导出镜像  nginx.img 是导出的文件名字这个随意  后面是仓库地址
ctr images export nginx.img docker.io/library/nginx:1.9.0
ctr i export --all-platforms nginx.img docker.io/library/nginx:1.9.0

ctr images import ./nginx.img  # 导入

# 增加TAG
ctr images tag docker.io/library/nginx:1.9.0 myhuber.abc.com/infra/nginx:1.9.0

# 镜像删除，下面几个子命令都可以，删除镜像要写镜像仓库全名，写ID是不行的
ctr images rm|remove|delete nginx.img docker.io/library/nginx:1.9.0

# 查看容器等效 docker ps -a 也就是无论容器是否运行都会显式出来
ctr c ls

# 查看已运行的容器以及其在主机上的PID，显式运行中的容器，也就是动态容器
# 静态容器就是 容器创建了，但是并没有运行
ctr tasks|t|task ls

# 列出这个容器里的进程ID
ctr task ps 容器名称

# 拷贝这个过去，否则运行容器会提示找不到 runtime，尤其是二进制安装containerd和runc的场景下
# 二进制安装下载的 containerd 里面有3个目录，etc opt 和 usr，实际containerd-shim-runc-v2就在 usr/local/bin里面
# 本身就是拷贝过去的，所以还要拷贝到 /usr/bin目录下
cp /usr/local/bin/containerd-shim-runc-v2 /usr/bin

# 创建容器
ctr container|c create docker.io/library/nginx:1.9.0 nginx1
# 创建后启动容器
ctr task start -d nginx1
ctr task ps nginx1

# 进入容器，这个 --exec-id 1 是个随机数，写啥都行
ctr task exec --exec-id 1 nginx1 /bin/bash

# 创建并运行一个容器，而且使用本机网络
ctr run -d --net-host docker.io/library/nginx:1.9.0 nginx1

# 暂停和恢复一个容器
ctr task pause nginx1
ctr task resume nginx1

# 停止一个容器，但是没有删除，只是不运行了，容器状态变成STOPPED
ctr task kill nginx1
# 删除容器，其实这里就是删除容器进程，容器本身没有删除
ctr task rm|remove|delete|del nginx1

# 删除静态容器，这个容器就不能有运行的TASK，需要先删除TASK然后使容器变成静态的，才能删除
ctr container delete nginx1

# 推送镜像
ctr images push --platform linux/amd64 --plain-http -u admin:PASSWORD
```



## Kubernetes

### Deployment、Statefulset和Daemonset相关

```bash
# 更新镜像
kubectl set image deployment/risk-wallet-validator risk-wallet-validator=hb.kucoin.net/kucoin/risk-wallet-validator:master-80

# 重启方式1
kubectl patch deployment <deployment-name> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container-name>","env":[{"name":"RESTART_","value":"'$(date +%s)'"}]}]}}}}'

# 重启方式2
kubectl scale deployment XXXX --replicas=0 -n {namespace}
kubectl scale deployment XXXX --replicas=1 -n {namespace}

# 重启daemonset
kubectl rollout restart daemonset NAME -n NAMESPACE
kubectl rollout restart deployment NAME -n NAMESPACE
```



StatefulSet实现滚动更新含金丝雀发布

```bash
# 假设3个nginx的sts web-0 web-1 web-2，正常来讲滚动更新是从POD序号大到小进行更新。它还可以使用 partition参数来控制StatefulSet进行金丝雀发布

# 下面这个 "partition": 2 表示POD编号大于等于2才更新，这样就实现了金丝雀发布
kubectl patch sts web -p '{"spec": {"updateStrategy": {"rollingUpdate": {"partition": 2}}}}'

# 更新 
kubectl set image .....

# 然后改成0，这样就再次更新镜像的时候就把剩下的全更新了
kubectl patch sts web -p '{"spec": {"updateStrategy": {"rollingUpdate": {"partition": 0}}}}'
```





### Secret和Configmap相关

```bash
# 创建镜像拉取密钥 docker-registry 这个是一种类型
kubectl create secret docker-registry harbor-registry \
--docker-server=st-hb.kucoin.net --docker-username='admin' \
--docker-password='kAHtLbYBIc5deBQbxgUg' -n infra

# 查看secret内容
kubectl get secret harbor-secret --output="jsonpath={.data.\.dockerconfigjson}" |base64 -d
```



### RABC相关

创建一个普通查看权限的账号

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
labels:
  addonmanager.kubernetes.io/mode: Reconcile
  kubernetes.io/cluster-service: "true"
name: dev
namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
labels:
  kubernetes.io/bootstrapping: rbac-defaults
  rbac.authorization.k8s.io/aggregate-to-edit: "true"
name: dev
rules:
# 允许执行 exec 进入POD
- apiGroups:
- ""
resources:
- pods/exec
verbs:
- create
# 允许查看POD
- apiGroups:
- ""
resources:
- pods
verbs:
- get
- list
- watch
- delete
# 允许扩容
- apiGroups:
- extensions
- apps
resources:
- deployments/scale
verbs:
- patch
# 允许查看deployment
- apiGroups:
- extensions
- apps
resources:
- deployments
verbs:
- get
- list
- watch
- patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
name: dev
roleRef:
apiGroup: rbac.authorization.k8s.io
kind: ClusterRole
name: dev
subjects:
- kind: ServiceAccount
name: dev
namespace: default
```

然后执行下面命令获取token

```bash
kubectl get secret -n default | grep dev
kubectl get secret view-token-jf57n -o jsonpath={.data.token} -n default | base64 -d
```



### POD相关

```bash
# 查看pod标签
kubectl get pod --show-labels
# 列出符合标签条件的pod
kubectl get pods -l KEY=VALUE
# 通过标签集合来查找
kubectl get pods -l "KEY in (VALUE1, VALUE2, VALUE3)"

# 删除标签,KEY就是标签的key，通过后面一个减号来删除
kubectl label pod <POD_NAME> KEY-

# 批量删除POD
kubectl get pod | grep cloudx-scheduler | grep "Terminating" | awk '{print $1}' | xargs kubectl delete pod --force
```



### 节点相关

修改kubele 拉取镜像超时时长

```bash
vim /opt/cloud/cce/kubernetes/kubelet/kubelet
# 修改下面参数
--image-pull-progress-deadline=10m
systemctl restart kubelet
```

污点

```bash
# 节点打污点  https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/taint-and-toleration/
kubectl taint nodes NODE_NAME KEY1=VALUE1:NoSchedule
kubectl taint nodes NODE_NAME KEY1:NoSchedule

# 移除污点
kubectl taint nodes NODE_NAME KEY1=VALUE1:NoSchedule-
```

对于Deployment有2种配置方式

```yaml
# 应用要配置
spec:
tolerations:
  - key: "KEY1"
    value: "VALUE1"
    operator: "Equal"
    effect: "NoSchedule"

# 简单配置
spec:
tolerations:
  - key: "KEY1"
    operator: "Exists"
    effect: "NoSchedule"
```

给节点打标签并设置Deployment的节点选择器

```bash
# 给节点打标签
kubectl label node node2.noisedu.cn node=ssd
# 查看节点标签
kubectl get node --show-labels
```

```yaml
# 最简单的把POD调度到指定节点就是通过 nodeSelector
apiVersion: v1
kind: Pod
metadata:
name: pod-nodeselector
spec:
containers:
- name: demoapp
  image: 10.0.0.55:80/mykubernetes/pod_test:v0.1
  imagePullPolicy: IfNotPresent
nodeSelector:
  node: ssd
```

设置节点不可调度并排空节点

```bash
# 设置节点不可调度（但对其上面的POD不做任何操作，POD继续对外提供服务）
kubectl cordon <NODE_NAME>

# 驱逐指定节点的POD，驱逐方式是安全的，它遵循K8S的优雅终止。其实这个命令本身有2个操作
# 先驱逐节点上的POD，然后设置节点为不可调度。如果要想恢复调度，也是使用 kubectl uncordon 命令
# 之所以先用 kubectl cordon 设置节点不可调度，主要是防止在没有驱逐完成之前有新的POD调度过来
kubectl drain --ignore-daemonsets --delete-local-data <NODE_NAME>

# 设置节点为可调度（如果节点只是维护，维护完设置为可调度）
kubectl uncordon <NODE_NAME>

# 删除节点（如果节点有问题，驱逐之后，删除有问题的节点），虽然直接删除节点，也会做POD驱逐，但是
# 这种相对比较暴力，而且从MASTER上删除NODE后则不能通过Master来恢复调度，只能进入node，然后重启 kubelet 服务
# 进行重新注册，所以除非确定节点确实需要下线，否则不要执行。
kubectl delete nodes <NODE_NAME>
```



