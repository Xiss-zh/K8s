



# K8s-V1.23.xx-Kubeadm安装

### 1、基础环境配置

```shell
# 配置主机名
hostnamectl set-hostname k8s-master01

# 配置域名解析
cat >> /etc/hosts << EOF
192.168.10.241 k8s-master01
192.168.10.242 k8s-master02
192.168.10.243 k8s-master03
192.168.10.244 k8s-node01
192.168.10.245 k8s-node02
EOF

# 关闭防火墙
systemctl stop firewalld && systemctl disable firewalld

# 关闭selinux
setenforce 0 && sed -ri 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
 
# 关闭swap交换空间
swapoff -a && sed -ri 's/.*swap.*/#&/' /etc/fstab

#关闭NetworkManager
systemctl disable --now NetworkManager


#加载所需内核模块
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1

vm.overcommit_memory=1
net.ipv4.conf.all.route_localnet = 1 
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720
 
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system

```

### 2、安装并加载ipvs作为流量转发

```shell
#1，在所有节点安装ipvs
yum install ipset ipvsadm -y

ipvsadm -l -n

#2，在所有节点运行
cat <<EOF> /etc/sysconfig/modules/ipvs.modules
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules
lsmod | grep -e ip_vs -e nf_conntrack


```

### 3、安装Docker

```shell
 #安装所需的软件包。yum-utils 提供了 yum-config-manager ，并且 device mapper 存储驱动程序需要 device-mapper-persistent-data 和 lvm2。
 yum install -y yum-utils  device-mapper-persistent-data lvm2
 
 # 选择国内的一些源地址：
 yum-config-manager --add-repo  http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
 
 # 安装最新版本的 Docker Engine-Community 和 containerd
 yum install docker-ce docker-ce-cli containerd.io -y
 
# 启动
systemctl enable --now docker
systemctl enable --now containerd
 
# 配置阿里镜像源，日志大小，cgroup
mkdir -p /etc/docker

cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# 加载配置文件重启
systemctl daemon-reload

# 重启
systemctl restart docker

# 查看docker 配置信息
docker info
```
