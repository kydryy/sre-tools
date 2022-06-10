[参考地址](https://blog.csdn.net/aixiaoyang168/article/details/103022342 )
# 目的
使用consul来存储Prometheus的target，实现target的动态变更
# 环境
|软件|版本|
|--|--|
|Centos|7|
|docker|1.13|
|docker-compose|1.29.2|
|consul|1.12.2|
|prometheus|latest|
|服务器ip|10.0.0.1
# 安装
使用docker-compose安装启动consul和Prometheus
```
# mkdir -p prometheus/{data,config}
# mkdir -p consul
# chown 65534:65534 prometheus
# docker-compose up -d
```
# 配置
## consul使用
访问http://10.0.0.1:8085/ui/dc1/services，可以看到对应的service，默认只有一个consul
![image](https://user-images.githubusercontent.com/6283866/172286481-f0ba5ff3-b86f-4713-8b97-cf193c7248b8.png)

**API 注册服务到 Consul 

接下来，我们要注册服务到 Consul 中，可以通过其提供的 API 标准接口来添加。
那么先注册一个测试服务，该测试数据为本机 node-exporter 服务信息，服务地址及端口为 node-exporter 默认提供指标数据的地址，执行如下命令：
```
$ curl -X PUT -d '{"id": "node-exporter","name": "node-exporter-10.0.0.1","address": "10.0.0.1","port": 12345,"tags": ["test"],"checks": [{"http": "http://10.0.0.1:12345/metrics", "interval": "5s"}]}'  http://10.0.0.1:8500/v1/agent/service/register
```
执行完毕后，刷新一下 Consul Web 控制台页面，可以看到成功注册到 Consul 中。
![image](https://user-images.githubusercontent.com/6283866/172286445-65d2616c-4449-462f-b09a-2798de46296a.png)
## prometheus配置
```
...
- job_name: 'consul-prometheus'
  consul_sd_configs:
  - server: '10.0.0.1:8500'
    services: []  
```
通过http://10.0.0.1:9090/targets?search= 可以看到上面的exporter，分组为consul-prometheus

## 配置优化
按照上面的配置配置完后有四个问题
1. Prometheus默认会把所有的service全部集中到target上，默认的consul service也会加进去
2. Prometheus内的target的label只有instance和job，其他标签都默认属于 before relabeling
3. 如果需要自定义一些标签，例如 team、group、project 等关键分组信息，方便后边 alertmanager 进行告警规则匹配，该如何处理呢？
4. 所有 Consul 中注册的 Service 都会默认加载到  Prometheus 下配置的 consul_prometheus 组，如果有多种类型的 exporter，如何在 Prometheus 中配置分配给指定类型的组，方便直观的区别它们？  
这里我们需要使用prometheus的relabel_configs功能
Prometheus 允许用户在采集任务设置中，通过 relabel_configs 来添加自定义的 Relabeling 的额过程，来对标签进行指定规则的重写。 
Prometheus 加载 Targets 后，这些 Targets 会自动包含一些默认的标签，Target 以 __ 作为前置的标签是在系统内部使用的，这些标签不会被写入到样本数据中。
眼尖的会发现，每次增加 Target 时会自动增加一个 instance 标签，而 instance 标签的内容刚好对应 Target 实例的 __address__ 值，这是因为实际上 Prometheus 内部做了一次标签重写处理，默认 __address__ 标签设置为 <host>:<port> 地址，经过标签重写后，默认会自动将该值设置为 instance 标签，所以我们能够在页面看到该标签。  
replace: 根据 regex 的配置匹配 source_labels 标签的值（注意：多个 source_label 的值会按照 separator 进行拼接），并且将匹配到的值写入到 target_label 当中，如果有多个匹配组，则可以使用 ${1}, ${2} 确定写入的内容。如果没匹配到任何内容则不对 target_label 进行重新， 默认为 replace。  
keep: 丢弃 source_labels 的值中没有匹配到 regex 正则表达式内容的 Target 实例  
drop: 丢弃 source_labels 的值中匹配到 regex 正则表达式内容的 Target 实例  
hashmod:  将 target_label 设置为关联的 source_label 的哈希模块  
labelmap: 根据 regex 去匹配 Target 实例所有标签的名称（注意是名称），并且将捕获到的内容作为为新的标签名称，regex 匹配到标签的的值作为新标签的值  
labeldrop:  对 Target 标签进行过滤，会移除匹配过滤条件的所有标签  
labelkeep: 对 Target 标签进行过滤，会移除不匹配过滤条件的所有标签    
问题一，我们可以配置 relabel_configs 来实现标签过滤，只加载符合规则的服务。以上边为例，可以通过过滤  __meta_consul_tags 标签为 test 的服务，relabel_config 向 Consul 注册服务的时候，只加载匹配 regex 表达式的标签的服务到自己的配置文件。修改 prometheus.yml 配置如下：
```
...
- job_name: 'consul-prometheus'
  consul_sd_configs:
    - server: '10.0.0.1:8500'
      services: []  
  relabel_configs:
    - source_labels: [__meta_consul_tags]
      regex: .*test.*
      action: keep
```
解释下，这里的 relabel_configs 配置作用为丢弃源标签中 __meta_consul_tags 不包含 test 标签的服务，__meta_consul_tags 对应到 Consul 服务中的值为  "tags": ["test"]，默认 consul  服务是不带该标签的，从而实现过滤。重启 Prometheus 可以看到现在只获取了 node-exporter-10.0.0.1 这个服务了。  
 问题二和问题三可以归为一类，就是将系统默认标签或者用户自定义标签转换成可视化标签，方便查看及后续 Alertmanager 进行告警规则匹配分组。不过要实现给服务添加自定义标签，我们还得做一下修改，就是在注册服务时，将自定义标签信息添加到 Meta Data 数据中，具体可以参考 [consul http api]((https://www.consul.io/api-docs/agent/service) 官网说明，下边来演示一下如何操作。  
```
# vi consul-0.json
{
  "ID": "node-exporter",
  "Name": "node-exporter-10.0.0.1",
  "Tags": [
    "test"
  ],
  "Address": "10.0.0.1",
  "Port": 12345,
  "Meta": {
    "department": "support",
    "app": "test"
  },
  "EnableTagOverride": false,
  "Check": {
    "HTTP": "http://172.30.12.167:9100/metrics",
    "Interval": "10s"
  },
  "Weights": {
    "Passing": 10,
    "Warning": 1
  }
}
Meta里就是新增metadata，后面Prometheus可以获取到这个metadata并relabel
# curl --request PUT --data @consul-0.json http://10.0.0.1:8500/v1/agent/service/register?replace-existing-checks=1
```  
修改Prometheus配置
```
...
- job_name: 'consul-prometheus'
  consul_sd_configs:
    - server: '10.0.0.1:8500'
      services: []  
  relabel_configs:
    - source_labels: [__meta_consul_tags]
      regex: .*test.*
      action: keep
    - regex: __meta_consul_service_metadata_(.+)
      action: labelmap
```
增加的配置作用为匹配 __meta_consul_service_metadata_ 开头的标签，将捕获到的内容作为新的标签名称，匹配到标签的的值作为新标签的值，而我们刚添加的三个自定义标签，系统会自动添加 __meta_consul_service_metadata_app=test、__meta_consul_service_metadata_department=support 两个标签，经过  relabel 后，Prometheus 将会新增 app=test、department=support两个标签。重启 Prometheus 服务，可以看到新增了对应了三个自定义标签。  
第四个问题其实和上面的一样，就是使用不同的tag来表示不同的服务器分组，然后分别添加进去
```
 - job_name: 'consul-prometheus'
  consul_sd_configs:
    - server: '10.0.0.1:8500'
      services: []  
  relabel_configs:
    - source_labels: [__meta_consul_tags]
      regex: .*test.*
      action: keep
    - regex: __meta_consul_service_metadata_(.+)
      action: labelmap
 - job_name: 'groupb-prometheus'
  consul_sd_configs:
    - server: '10.0.0.1:8500'
      services: []  
  relabel_configs:
    - source_labels: [__meta_consul_tags]
      regex: .*groupb.*
      action: keep
    - regex: __meta_consul_service_metadata_(.+)
      action: labelmap
```
 
