# com.k8s
K8S相关

#K8S部署Java项目

kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具。

这个工具能通过两条指令完成一个kubernetes集群的部署：

```
# 创建一个 Master 节点
$ kubeadm init

# 将一个 Node 节点加入到当前集群中
$ kubeadm join <Master节点的IP和端口 >
```

## 1. 安装要求

在开始之前，部署Kubernetes集群机器需要满足以下几个条件：

- 一台或多台机器，操作系统 CentOS7.x-86_x64
- 硬件配置：2GB或更多RAM，2个CPU或更多CPU，硬盘30GB或更多
- 可以访问外网，需要拉取镜像，如果服务器不能上网，需要提前下载镜像并导入节点
- 禁止swap分区

## 2. 准备环境

| 角色   | IP           |
| ------ | ------------ |
| master | 192.168.1.11 |
| node1  | 192.168.1.12 |
| node2  | 192.168.1.13 |

```
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0  # 临时

# 关闭swap
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久

# 根据规划设置主机名
hostnamectl set-hostname <hostname>

# 在master添加hosts
cat >> /etc/hosts << EOF
192.168.44.146 k8smaster
192.168.44.145 k8snode1
192.168.44.144 k8snode2
EOF

# 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  # 生效

# 时间同步
yum install ntpdate -y
ntpdate time.windows.com
```

## 3. 所有节点安装Docker/kubeadm/kubelet

Kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker。

### 3.1 安装Docker

```
$ wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
$ yum -y install docker-ce-18.06.1.ce-3.el7
$ systemctl enable docker && systemctl start docker
$ docker --version
Docker version 18.06.1-ce, build e68fc7a
```

```
$ cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```

### 3.2 添加阿里云YUM软件源

```
$ cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 3.3 安装kubeadm，kubelet和kubectl

由于版本更新频繁，这里指定版本号部署：

```
$ yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0
$ systemctl enable kubelet
```

## 4. 部署Kubernetes Master

在192.168.31.61（Master）执行。

```
$ kubeadm init \
  --apiserver-advertise-address=192.168.44.146 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.18.0 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16
```

由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址。

使用kubectl工具：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl get nodes
```

## 5. 加入Kubernetes Node

在192.168.1.12/13（Node）执行。

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：

```
$ kubeadm join 192.168.1.11:6443 --token esce21.q6hetwm8si29qxwn \
    --discovery-token-ca-cert-hash sha256:00603a05805807501d7181c3d60b478788408cfe6cedefedb1f97569708be9c5
```

默认token有效期为24小时，当过期之后，该token就不可用了。这时就需要重新创建token，操作如下：

```
kubeadm token create --print-join-command
```

## 6. 部署CNI网络插件

```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

默认镜像地址无法访问，sed命令修改为docker hub镜像仓库。

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

kubectl get pods -n kube-system
NAME                          READY   STATUS    RESTARTS   AGE
kube-flannel-ds-amd64-2pc95   1/1     Running   0          72s
```

## 7. 测试kubernetes集群

在Kubernetes集群中创建一个pod，验证是否正常运行：

```
$ kubectl create deployment nginx --image=nginx
$ kubectl expose deployment nginx --port=80 --type=NodePort
$ kubectl get pod,svc
```

访问地址：http://NodeIP:Port  







**=======================================================**
#K8S部署Java项目

容器交付流程
	开发代码阶段 —> 持续交付/集成 —> 应用部署 —> 运维

	开发代码阶段：编写代码、测试、编写DockerFile
	持续交付/集成：代码编译打包、制作镜像、上传镜像仓库
	应用部署：环境部署、Pod、Service、Ingress
	运维：监控、故障排查、升级优化
K8S部署项目流程
	制作镜像（dockerflie） -> 推送镜像仓库（阿里云）—> 控制器部署镜像（deployment） -> 对外部署应用(Service)

第一步，准备Java项目，进行打包
	编写dockerfile文件
	FROM openjdk:8-jdk-alpine
	VOLUME /tmp
	ADD ./target/demojenkins.jar demojenkins.jar
	ENTRYPOINT ["java","-jar","/demojenkins.jar", "&"]

使用mvn命令进行打包
	mvn clean package


第二步，制作镜像
	将jar传到服务器，进行制作
	*docker生成镜像（进入项目路径）
	docker build -t java-demo-01:latest .

	*查看docker镜像
	Docker images

	*启动制作好的镜像，看是否可以访问
	docker run -d -p 8111:8111 java-demo-01:latest -t


	
第三步，镜像推送到阿里云仓库
	根据阿里云文档操作，上传镜像
	1. 登录阿里云Docker Registry
	2.将镜像推送到Registry
	
	2.1 添加版本号
	sudo docker tag 3e39b5761734 registry.cn-hangzhou.aliyuncs.com/shenyidong/java-project-01:1.0.0

	2.2 推送
	sudo docker push registry.cn-hangzhou.aliyuncs.com/shenyidong/java-project-01:1.0.0


第四步，部署镜像，暴露应用
	*创建deployment
	kubectl create deployment javademo1 --image=registry.cn-hangzhou.aliyuncs.com/shenyidong/java-project-01:1.0.0 --dry-run -o yaml > javademo1.yaml

	kubectl apply -f  javademo1.yaml

	*查看pod
	kubectl get pods -o wide

	*扩容
	kubectl scale deployment javademo1 --replicas=3

	*暴露端口
	kubectl expose deployment javademo1 --port=8111 --target-port=8111 --type=NodePort 
	kubectl get svc


	#进行访问
	http://192.168.2.128:30577/user


#遇到外网无法访问docker的解决方案
1、关闭防火墙
2、vi /etc/sysctl.conf添加以下代码net.ipv4.ip_forward=1

