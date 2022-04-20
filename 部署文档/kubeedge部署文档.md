# kubeedge部署文档
## 介绍
KubeEdge是一个开源系统，用于将容器化应用程序编排功能扩展到Edge的主机。它基于kubernetes构建，并为网络应用程序提供基础架构支持。云和边缘之间的部署和元数据同步。
![image](https://user-images.githubusercontent.com/6283866/164159086-29a5ca40-3029-4b71-a743-f7eacdac964b.png)
官网：https://kubeedge.io
## 准备
### 组件版本
|组件|版本|
|--|--|
|K8S|1.21|
|kubeege|v1.10.0|
### 部署cloudcore
这里我们使用keadm以容器的方式部署kubeedge，注意，cloudcore和edgecore都需要使用同一个版本的安装文件
1. 在k8s master节点上下载keadm
下载地址：https://github.com/kubeedge/kubeedge/releases，解压缩后运行version，可以看到是1.10版本
![image](https://user-images.githubusercontent.com/6283866/164159821-7d29c70a-43aa-407d-9805-3256bc46e2a8.png)
2. 在k8s master节点上运行安装程序
```
keadm beta init --advertise-address="218.xx.xx.x,110.xx.xx.xx,10.20.3.30,10.20.3.31,10.20.3.32"  kube-config=/root/.kube/config
```
具体参数--advertise-address为他的外网IP，主要是为了从外网注册过来
不出意外的话，会生成很多CRD和一个deployment
### 部署iptablesmanager
默认的部署方式是以k8s的master节点的iptables作为kubectl log/exec的转发，会自动执行
```
iptables -t nat -A OUTPUT -p tcp --dport 10350 -j DNAT --to $CLOUDCOREIPS:10003
```
其中dport和CLOUDCOREIPS由tunnelport那个configmap指定
我们修改成使用单独的容器的iptables来做
以下操作均在k8s master上执行
1. 部署cloud-iptables-manager
由于官方提供的iptablesmanager镜像有问题无法下载，只能自己打包docker。
```
下载github上的kubeedge代码
# git clone https://github.com/kubeedge/kubeedge
把iptablesmanager的dockfile复制到代码根目录
# cd kubeedge
# cp build/iptablesManager/Dockerfile .
编译docker镜像
# docker build -t yourrepo/iptablesmanager:latest .
修改iptablesmanager的部署yaml文件
# vi build/iptablesmanager/iptablesmanager-ds.yaml
---
修改前
image: kubeedge/iptables-manager:latest
修改后
image: yourrepo/iptablesmanager:latest
---
运行部署
# kubectl apply -f build/iptablesmanager/iptablesmanager-ds.yaml
确认部署完成,pod均为Running状态
# kubectl get pod -n kubeedge
NAME                           READY   STATUS    RESTARTS   AGE
cloud-iptables-manager-cqmp7   1/1     Running   0          4h13m
cloud-iptables-manager-dkf4t   1/1     Running   0          4h13m
cloud-iptables-manager-fh962   1/1     Running   0          4h13m
```
![image](https://user-images.githubusercontent.com/6283866/164164867-ef6f8d07-84e0-44f4-95c6-ed17ece2d484.png)

3. 修改configmap cloudcore，添加以下内容
```
# kubectl edit cm cloudcore -n kubeedge
---
iptablesManager:
  enable: true
  mode: "external"
---
```
![image](https://user-images.githubusercontent.com/6283866/164162521-3e3544a1-502c-4db1-bdd1-c450e65e1399.png)
3. 修改cloudcore的deployment
```
# kubectl edit deploy cloudcore -n kubeedge
删除hostNetwork: true
```
因为默认使用的hostNetwork方式运行的cloudcore，修改成使用外部iptables以后必须修改这里，之后他就会自然把iptables规则指向对应的svc
4. 修改cloudcore的service，从clusterIP修改成Nodeport，并且将对应的port改成30000-30004
```
# kubectl edit svc cloudcore -n kubeedge
---
  ports:
  - name: cloudhub
    nodePort: 30000
    port: 10000
    protocol: TCP
    targetPort: 10000
  - name: cloudhub-quic
    nodePort: 30001
    port: 10001
    protocol: TCP
    targetPort: 10001
  - name: cloudhub-https
    nodePort: 30002
    port: 10002
    protocol: TCP
    targetPort: 10002
  - name: cloudstream
    nodePort: 30003
    port: 10003
    protocol: TCP
    targetPort: 10003
  - name: tunnelport
    nodePort: 30004
    port: 10004
    protocol: TCP
    targetPort: 10004
  selector:
    k8s-app: kubeedge
    kubeedge: cloudcore
  sessionAffinity: None
  type: NodePort
---
```
### 部署edgecore

