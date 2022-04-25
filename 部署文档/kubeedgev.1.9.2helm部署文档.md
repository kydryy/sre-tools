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
|kubeege|v1.9.2|
|helm|v3.8.2|
### 部署cloudcore
和v1.10.0版本不同，v1.9.2要部署到k8s里还需要使用helm而不是使用keadm，注意，cloudcore和edgecore都需要使用同一个版本的安装文件  
1. 在k8s master节点上下载helm
下载地址：https://github.com/helm/helm/releases
解压缩后复制到/usr/local/bin/内
```
# wget https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz
# tar -zxvf helm-v3.8.2-linux-amd64.tar.gz
# cp linux-amd64/helm /usr/local/bin
# helm version
version.BuildInfo{Version:"v3.8.2", GitCommit:"6e3701edea09e5d55a8ca2aae03a68917630e91b", GitTreeState:"clean", GoVersion:"go1.17.5"}
```  
注意PATH
2. 在k8s master节点上下载kubeedge代码并切换到release-1.9分支
下载地址：https://github.com/kubeedge/kubeedge/releases
```
# wget https://github.com/kubeedge/kubeedge
# cd kubeedge
# git checkout release-1.9
```
3. 修改helm的配置

### 部署edgecore
客户端部署也使用keadm安装，但是由于客户端的机器性能较差且在国内，keadm安装过程需要访问github.com来下载安装包，所以我们提前准备好文件传到客户端的服务器上,就不会再下载了  
需要准备的工具如下
|工具|版本|下载地址|
|---|---|---|
|keadm|v1.9.2|https://github.com/kubeedge/kubeedge/releases/download/v1.9.2/keadm-v1.9.2-linux-amd64.tar.gz|
|kubeedge-v1.9.2-linux-amd64.tar.gz|v1.10.0|https://github.com/kubeedge/kubeedge/releases/download/v1.9.2/kubeedge-v1.9.2-linux-amd64.tar.gz|
|edgecore.service|无|https://github.com/kubeedge/kubeedge/blob/master/build/tools/edgecore.service|
|checksum_kubeedge-v1.9.2-linux-amd64.tar.gz.txt|v1.9.2|https://github.com/kubeedge/kubeedge/releases/download/v1.9.2/checksum_kubeedge-v1.9.2-linux-amd64.tar.gz.txt|
以下操作在master上运行
```
# keadm gettoken
```
记录token，后面用来注册node
以下操作在客户端node上运行  
1. 复制三个文件到对应目录  
```
# mkdir -p /etc/kubeedge
# cd /etc/kubeedge
# wget https://github.com/kubeedge/kubeedge/releases/download/v1.9.2/keadm-v1.9.2-linux-amd64.tar.gz && tar -zxvf keadm-v1.9.2-linux-amd64.tar.gz && mv keadm-v1.9.2-linux-amd64/keadm /usr/local/bin/ 
# wget https://github.com/kubeedge/kubeedge/releases/download/v1.9.2/kubeedge-v1.9.2-linux-amd64.tar.gz
# wget https://github.com/kubeedge/kubeedge/releases/download/v1.9.2/checksum_kubeedge-v1.9.2-linux-amd64.tar.gz.txt
复制edgecore.service到/etc/kubeedge目录
```  

2. 启动客户端安装  
```
# keadm join --cloudcore-ipport=218.XX.XX.XX:10000  --edgenode-name edgenode-106.xx.xx.xx --token $token -l edgenode=yes --kubeedge-version=v1.9.2
```
--cloudcore-ipport: 公网注册的ip端口
--edgenode-name：注册后的node名字
--token：上面在master上获取到的token
-l：指定edgenode的label
3. 测试安装是否完成  
在master节点上运行
```
# kubectl get node
[root@node1 ~]# kubectl get node
NAME                              STATUS   ROLES                  AGE     VERSION
edgenode-taobao-122.xx.xx.xx   Ready    agent,edge             40m     v1.21.4-kubeedge-v1.9.2
```
### 实现kubectl logs/exec操作edgenode  
新版本不需要手动添加iptables了，但是还是需要手动启动edgecore端的配置  
**代码注释里写着edgeStream的enable默认为true，可是实际上是false！**
```
# vi /etc/kubeedge/config/edgecore.yaml
---
  edgeStream:
    enable: true
---
# systemctl restart edgecore
```
修改即可
### 获取edgenode的度量  
度量同上，只需要安装metrics-server0.4.1版本以上就可以，因为他默认从kubelet获取node的度量，目前只支持cpu和内存
### 防止calico等服务跑到edgenode上去
1. 设置edgenode的污点
```
#  kubectl taint nodes edgenode-106.xx.xx.xx node-role.kubernetes.io/edge=:NoSchedule
```

添加affinity即可
```
affinity:
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
      - matchExpressions:
          - key: node-role.kubernetes.io/edge
            operator: DoesNotExist
```
### 允许服务运行到edgenode上
1. 添加容忍度  
```
      tolerations:
        - key: node-role.kubernetes.io/edge
          operator: Exists
          effect: NoSchedule
```
容忍度介绍看kubernetes官方文档  
2. 添加nodeSelector
```
      nodeSelector:
        edgenode: 'yes'
```
### 删除edgenode节点  
待补充
### 删除kubeedge  
待补充
