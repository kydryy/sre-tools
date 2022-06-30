# acme-trojan-go部署
## 具体思路
使用acme生成域名证书，配置trojan-go作为服务端，具体原理用途不讲了，多搜索就好了
## 部署过程
### 生成挂载的本地目录
**config目录** ：存储acme.sh生成的证书以及trojan server端的config.json文件  
**nginxhtml目录**：存储nginx的默认静态文件
用途是为了能正确的通过acme获取到https证书
```
# mkdir -p /home/trojan-go/{config,nginxhtml} 
# echo 123 > /home/trojan-go/nginxhtml/index.html
# vi /home/trojan-go/config.json
{
    "run_type": "server",
    "local_addr": "0.0.0.0",
    "local_port": 7443,
    "remote_addr": "127.0.0.1",
    "remote_port": 80,
    "password": [
        "yourpasswordchangeit"
    ],
    "ssl": {
        "cert": "/etc/trojan-go/tr.mydomain.com/fullchain.cer",
        "key": "/etc/trojan-go/tr.mydomain.com/tr.mydomain.com.key",
        "sni": "tr.mydomain.com"
    }
}
```
### 添加DNS解析
使用acme.sh的最简单的webroot方式，提前启动一个nginx并监听80端口，由acme.sh自动去指定目录生成_well_knowns文件，需要该文件能被访问到，所以需要添加dns解析  

### docker-compose一键启动  
```
# docker-compose up -d
```
启动后需要等带acme.sh的容器运行完成，如果发现acme.sh运行有误请自行解决  
如果发现trojan一直启动失败，证书生成正常后重启容器即可
### 添加cron
主要是为了定期更新证书
```
# crontab -l
34 0 * * * /usr/sbin/docker run --rm  -it -v /home/trojan-go/config:/acme.sh --net=host neilpang/acme.sh --cron
35 0 * * * /usr/sbin/docker restart webserver
```
### 客户端配置
使用clash作为客户端，配置文件添加一个proxies即可
```
proxies:
......
  - name: "trojan"
    type: trojan
    server: youripaddress
    port: 7443
    password: yourpasswordchangeit
    # udp: true
    sni: tr.mydomain.com # aka server name
```
