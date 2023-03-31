#### Longhorn介绍

[longhorn](https://links.jianshu.com/go?to=lhttps%3A%2F%2Flonghorn.io%2F) 是开源的Kubernetes云原生分布式块存储解决方案，以微服务方式运行在k8s集群上。

优点 ：

- 企业级分布式块存储，无单点故障；
- 支持增量快照和远程备份恢复(NFS/S3兼容对象存储)；
- 定期快照和备份；
- 提供UI页面，管理方便；

#### Longhorn原理

按照官网的说法，longhorn分为数据平面和控制平面，longhorn engine对应数据平面的存储控制器，longhorn manager对应于控制平面，大概的用途这里总结了一下：

- longhorn manager：基于daemonset方式在longhorn集群每个节点上运行，负责在k8s中创建和管理卷，并处理来自longhorn ui 和 k8s卷插件的api调用；
- longhorn engine ：当manager创建卷时，并在卷所在节点创建一个engine实例，并给卷创建副本放置到其他节点上，确保卷的高可用性；

![longhorn-01](.\png\longhorn-01.png)

#### Longhorn搭建

- 在安装longhorn存储前，首先要准备好一个k8s环境，具体信息如下(master节点不调度)；



```bash
$ kubectl get nodes
NAME    STATUS   ROLES                  AGE   VERSION
node1   Ready    control-plane,master   12h   v1.23.8
node2   Ready    <none>                 12h   v1.23.8
node3   Ready    <none>                 12h   v1.23.8

$ helm version
v3.8.2
```

- 这里给集群每个节点准备一块数据盘，挂载到指定的数据目录上作为longhorn的存储目录；



```bash
$ fdisk /dev/sdb
$ mkfs.xfs /dev/sdb1
$ mkdir -p /data/longhorn
$ echo "/dev/sdb1 /data/longhorn xfs defaults 0 0" >> /etc/fstab
$ mount -a
```

- 此外，longhorn还需要如下几个依赖支持，在所有集群节点安装；



```bash
# centos
yum install -y iscsi-initiator-utils nfs-utils && systemctl enable iscsid --now

# ubuntu
apt-get -y install open-iscsi && systemctl enable iscsid --now
```

- [获取](https://links.jianshu.com/go?to=https%3A%2F%2Fartifacthub.io%2Fpackages%2Fhelm%2Flonghorn%2Flonghorn) longhorn部署的chart包 ；



```bash
$ helm repo add longhorn https://charts.longhorn.io
$ helm repo update
$ helm pull longhorn/longhorn --version v1.4.1
$ tar xf longhorn-1.4.1.tgz 
$ cd longhorn/
```

- 修改配置参数如下；



```bash
$ vim values.yaml

service:
  ui:
    type: NodePort   # 需要在集群外看到页面，调整webUI访问模式为nodeport,默认cluster
    nodePort: 30012  # 设置访问端口

persistence:
  defaultFsType: xfs # 默认ext4,这里改成和本地的磁盘格式一致即可
  defaultClassReplicaCount: 2 # 因为master不参与调度，所以卷副本数改成2,和node节点数一样

defaultSettings
  defaultDataPath: "/data/longhorn" # 设置默认数据目录,默认存放到/var/lib/longhorn
```

- 安装longhorn；



```bash
$ kubectl create ns longhorn-system
$ helm install longhorn -n longhorn-system . # 将longhorn存储安装到longhorn命名空间
$ helm list -A
$ kubectl -n longhorn-system get pod  # 确保所有pod均为running状态表示部署成功
```

- 访问集群任意节点的30012打开webI页面查看仪表盘；



​	

```
修改svc
```

![img](https:////upload-images.jianshu.io/upload_images/27968893-8079bf38e9d180e5.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image-20230214111607294.png

- 查看集群的存储类；



```bash
$ kubectl get sc
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn (default)   driver.longhorn.io   Delete          Immediate           true                   3m16s
```

#### Longhorn挂载

- 通过longhorn sc 创建一个pvc卷并给pod挂载；



```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  storageClassName: longhorn # 指定调用的sc
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      restartPolicy: Always
      containers:
        - image: nginx:stable-alpine
          name: nginx
          imagePullPolicy: IfNotPresent
          ports:
          - containerPort: 80
            name: http
          volumeMounts:
          - mountPath: /usr/share/nginx/html
            name: data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: app-pvc
```

- 查看pod和pvc状态；



```bash
$ kubectl get pod,pvc,pv  # 通过sc创建pvc时会自动创建一个pv实例绑定
```

- 也可以在ui页面查看卷的挂载情况；

![img](https:////upload-images.jianshu.io/upload_images/27968893-ed51dc495e9a1edd.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



