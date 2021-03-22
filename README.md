# com.k8s
K8S相关

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

