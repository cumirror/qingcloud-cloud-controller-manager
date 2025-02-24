# 青云负载均衡器配置指南

> 大部分的配置都是基于需要使用LB插件对应Service的`Annotation`，简单来讲，`Annotation`记录了关于当前服务的额外配置信息，参考<https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/>了解如何使用

## 一、负载均衡器属性配置

- `ServiceAnnotationLoadBalancerType`：在Service的`annotations`中添加此Key，表示使用yunify负载均衡器，不添加此Key的Service在Event中会有错误。其值范围和[create_loadbalancer](https://docs.qingcloud.com/product/api/action/lb/create_loadbalancer.html)中的type相同。

- 青云负载均衡器支持http/s协议的负载均衡，如果想要使用青云负载均衡器更多的功能，请将服务的特定的端口name写为`http`或`https`，如下：
```yaml
kind: Service
apiVersion: v1
metadata:
  name:  mylbapp
  annotations:
    service.beta.kubernetes.io/qingcloud-load-balancer-type: "0"
spec:
  selector:
    app:  mylbapp
  type:  LoadBalancer 
  ports:
  - name:  http  #这里指定port name,如果有多个http端口，都可以指定为http
    port:  8088
    targetPort:  80
```
将ServicePort命名为http之后，就可以在青云控制台上查看http监控。更多的功能支持诸如`转发规则定义`、`透明代理`等功能还在开发中

## 二、公网IP配置
> 使用青云负载均衡器时需要配合一个公网ip，cloud-controller带有IP管理功能。

### 手动配置公网ip
这是本插件的默认方式，即用户需要在Service中添加公网ip的annotation。首先在Service annotation中添加如下key: `service.beta.kubernetes.io/qingcloud-load-balancer-eip-source`，并设置其值为`mannual`。如果不设置此Key，默认是`mannual`。然后必须继续添加`service.beta.kubernetes.io/qingcloud-load-balancer-eip-ids`的key，其值公网IP的ID，如果需要绑定多个IP，以逗号分隔。请确保此IP处于可用状态。如下：
```yaml
kind: Service
apiVersion: v1
metadata:
  name:  mylbapp
  annotations:
    service.beta.kubernetes.io/qingcloud-load-balancer-eip-ids: "eip-xxxxx"  #这里设置EIP
    service.beta.kubernetes.io/qingcloud-load-balancer-type: "0"
spec:
  selector:
    app:  mylbapp
  type:  LoadBalancer 
  ports:
  - name:  http
    port:  8088
    targetPort:  80
```

### 自动获取IP
自动获取ip有三种方式，分别是：

1. 自动获取当前账户下处于可用的EIP，如果找不到返回错误。
2. 自动获取当前账户下处于可用的EIP，如果找不到则申请一个新的
3. 不管账户下有没有可用的EIP，申请一个新EIP

如果开启EIP的自动获取的功能，需满足两个条件：
1. 集群的配置文件中`/etc/kubernetes/qingcloud.conf`配置`userID`，是因为一些用户API权限较大，会获取到其他用户的IP。
2. 配置Service Annotations中的`service.beta.kubernetes.io/qingcloud-load-balancer-eip-source`不为`mannual`，上述三种方式对应的值分别为：
   1. `use-available`
   2. `auto`
   3. `allocate`

参考下面样例：
```yaml
kind: Service
apiVersion: v1
metadata:
  name:  mylbapp
  annotations:
    service.beta.kubernetes.io/qingcloud-load-balancer-eip-source: "auto" #也可以填use-available,allocate
    service.beta.kubernetes.io/qingcloud-load-balancer-type: "0"
spec:
  selector:
    app:  mylbapp
  type:  LoadBalancer 
  ports:
  - name:  http
    port:  8088
    targetPort:  80
```

## 三、配置Service使用云上现有的LB
### 注意事项
1. 在此模式下，LB的一些功能属性，包括7层协议配置，LB容量，LB绑定的EIP都由用户配置，LB插件不会修改任何属性，除了增删一些监听器
2. 确保LB正常工作，并且和Service对应的端口不冲突。LB已有的监听器监听的端口不能和需要暴露的服务冲突。

### 如何配置
1. `service.beta.kubernetes.io/qingcloud-load-balancer-eip-strategy`设置为"reuse-lb"
2. `service.beta.kubernetes.io/qingcloud-load-balancer-id`设置为现有的LB ID，类似"lb-xxxxxx"

其余设置都会被LB插件忽略。

### 参考Service
```yaml
kind: Service
apiVersion: v1
metadata:
  name:  reuse-lb
  annotations:
    service.beta.kubernetes.io/qingcloud-load-balancer-eip-strategy: "reuse-lb"
    service.beta.kubernetes.io/qingcloud-load-balancer-id: "lb-oglqftju"
spec:
  selector:
    app:  mylbapp
  type:  LoadBalancer 
  ports:
  - name:  http
    port:  8090
    targetPort:  80
```

## 四、配置LB的监听器属性

### 如何配置
1. 设置监听器的健康检查方式，`service.beta.kubernetes.io/qingcloud-lb-listener-healthycheckmethod`，对于 tcp 协议默认是 tcp 方式，对于 udp 协议默认是 udp 方式
2. 设置监听器的健康检查参数，`service.beta.kubernetes.io/qingcloud-lb-listener-healthycheckoption`，默认是 "10|5|2|5"
3. 支持 roundrobin/leastconn/source 三种负载均衡方式，`service.beta.kubernetes.io/qingcloud-lb-listener-balancemode`，默认是 roundrobin

因为一个LB会有多个监听器，所以进行service注解设置时，通过如下格式区分不同监听器：`80:xxx,443:xxx`。

### 参考Service
```yaml
kind: Service
apiVersion: v1
metadata:
  name:  reuse-lb
  annotations:
    service.beta.kubernetes.io/qingcloud-load-balancer-eip-strategy: "reuse-lb"
    service.beta.kubernetes.io/qingcloud-load-balancer-id: "lb-oglqftju"
    service.beta.kubernetes.io/qingcloud-lb-listener-healthycheckmethod: "8090:tcp"
    service.beta.kubernetes.io/qingcloud-lb-listener-healthycheckoption: "8090:10|5|2|5"
    service.beta.kubernetes.io/qingcloud-lb-listener-balancemode: "8090:source"
spec:
  selector:
    app:  mylbapp
  type:  LoadBalancer
  ports:
  - name:  http
    port:  8090
    protocol: TCP
    targetPort:  80
```
监听器参数说明：https://docsv3.qingcloud.com/network/loadbalancer/api/listener/modify_listener_attribute/

## 配置内网负载均衡器
### 已知问题
k8s在ipvs模式下，kube-proxy会把内网负载均衡器的ip绑定在ipvs接口上，这样会导致从LB过来的包被drop（进来的是主网卡，但是出去的时候发现ipvs有这么一个ip，路由不一致）故目前无法在IPVS模式下使用内网负载均衡器。参考[issue](https://github.com/kubernetes/kubernetes/issues/79783)
### 注意事项
1. 必须手动指定`service.beta.kubernetes.io/qingcloud-load-balancer-network-type`为`internal`，如果不指定或者填写其他值，都默认为公网LB，需要配置EIP
2. 可选指定LB所在的Vxnet，默认为创建LB插件配置文件中的`defaultVxnet`，手动配置vxnet的annotation为`service.beta.kubernetes.io/qingcloud-load-balancer-vxnet-id`
3. 可选指定LB 内网ip，通过`service.beta.kubernetes.io/qingcloud-load-balancer-internal-ip`指定。**注意，当前不支持更换内网IP**
### 参考Service
```yaml
kind: Service
apiVersion: v1
metadata:
  name:  mylbapp
  namespace: default
  annotations:
    service.beta.kubernetes.io/qingcloud-load-balancer-type: "0"
    service.beta.kubernetes.io/qingcloud-load-balancer-network-type: "internal"
    service.beta.kubernetes.io/qingcloud-load-balancer-internal-ip: "192.168.0.1" ##如果要路由器自动分配删掉这一行
spec:
  selector:
    app:  mylbapp
  type:  LoadBalancer 
  ports:
  - name:  http
    port:  80
    targetPort:  80
```