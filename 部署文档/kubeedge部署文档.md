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
下载地址：https://github.com/kubeedge/kubeedge/releases
解压缩后运行version，可以看到是1.10版本
![image](https://user-images.githubusercontent.com/6283866/164159821-7d29c70a-43aa-407d-9805-3256bc46e2a8.png)
2. 在k8s master节点上运行安装程序
```
# keadm beta init --advertise-address="218.xx.xx.x,110.xx.xx.xx,10.20.3.30,10.20.3.31,10.20.3.32"  kube-config=/root/.kube/config
--advertise-address为他的外网IP，主要是为了从外网注册过来
不出意外的话，会生成很多CRD和一个deployment
# kubectl get deploy -n kubeedge
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
cloudcore   1/1     1            1           24h
# kubectl get pod -n kubeedge
NAME                           READY   STATUS    RESTARTS   AGE
cloudcore-7448499647-qlv9v     1/1     Running   0          4h12m
```
pod为Running状态即可
### 部署iptablesmanager
默认的部署方式cloudcore以hostnework的方式部署在master节点上，会自动执行
```
iptables -t nat -A OUTPUT -p tcp --dport 10350 -j DNAT --to $CLOUDCOREIPS:10003
```
其中dport和CLOUDCOREIPS由tunnelport那个configmap指定
我们修改成使用单独的容器的iptables manager来做这些操作
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
cloudcore-7448499647-qlv9v     1/1     Running   0          4h13m
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
# kubectl get svc cloudcore -n kubeedge
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                           AGE
cloudcore   NodePort   10.233.38.170   <none>        10000:30000/TCP,10001:30001/TCP,10002:30002/TCP,10003:30003/TCP,10004:30004/TCP   24h
```
### 部署edgecore
客户端部署也使用keadm安装，但是由于客户端的机器性能较差且在国内，keadm安装过程需要访问github.com来下载安装包，所以我们提前准备好文件传到客户端的服务器上,就不会再下载了  
需要准备的工具如下
|工具|版本|下载地址|
|---|---|---|
|keadm|v1.10.0|https://github.com/kubeedge/kubeedge/releases/download/v1.10.0/keadm-v1.10.0-linux-amd64.tar.gz|
|kubeedge-v1.10.0-linux-amd64.tar.gz|v1.10.0|https://github.com/kubeedge/kubeedge/releases/download/v1.10.0/kubeedge-v1.10.0-linux-amd64.tar.gz|
|edgecore.service|无|https://github.com/kubeedge/kubeedge/blob/master/build/tools/edgecore.service|
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
# wget https://githu/etc/kubeedgeb.com/kubeedge/kubeedge/releases/download/v1.10.0/keadm-v1.10.0-linux-amd64.tar.gz && tar -zxvf keadm-v1.10.0-linux-amd64.tar.gz && mv keadm-v1.10.0-linux-amd64/keadm /usr/local/bin/ 
# wget https://github.com/kubeedge/kubeedge/releases/download/v1.10.0/kubeedge-v1.10.0-linux-amd64.tar.gz
复制edgecore.service到/etc/kubeedge目录
```
2. 启动客户端安装  
```
# keadm join --cloudcore-ipport=218.XX.XX.XX:10000  --edgenode-name edgenode-106.xx.xx.xx --token $token -l edgenode=yes
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
NAME                      STATUS   ROLES                  AGE   VERSION
edgenode-106.xx.xx.xx     Ready    agent,edge             23h   v1.22.6-kubeedge-v1.10.0
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
