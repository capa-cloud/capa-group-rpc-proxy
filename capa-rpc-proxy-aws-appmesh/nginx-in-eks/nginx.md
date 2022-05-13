## Nginx基础知识

### Nginx的架构是典型的多进程模型：

#### 1. 高可用

master进程类似于应用服务器中的集群中的主监视Node节点，而work进程类似于应用服务器集群中的slave节点，
因为是进程模式，所以1个worker进程崩溃，不会导致整个Nginx服务器的崩溃，除非是Master进程崩溃，Nginx才崩溃，
但是Master进程却并不参与应用请求，所以从图中可以看到client只与worker进程交互，大并发的压力都在worker进程中，
这种模式也是为什么Nginx稳定的原因，试想一下，在java服务器中，因为JVM仅仅就是一个进程，
所以一旦出现什么风吹草动的话，那么，JVM直接就会崩溃掉，而JVM崩溃掉，哪还有什么线程运行，服务器安在？

这就是Nginx为什么不推荐使用多线程的主要的原因，为的就是一个多进程稳定可靠；

#### 2. 并发争用

线程锁的消耗；

#### 3. 并发代码

进程编程肯定比线程容易

#### 4. 线程切换

epoll这种IO多路复用机制下的1进程1线程的模式，比多线程的模式效率要高，没有大量的线程切换

### Nginx作为负载均衡集群

增加域名这种手段，其实是DNS的活，但是DNS有两大知名的缺陷：

一个是机器坏掉，DNS配置之后仍旧会请求坏掉的机器，这种应该类似有一个心跳检测的东西，但是不好意思，DNS没有； 

其次，DNS负载分配不平衡，以上图为例，如果反向代理手段使用的是DNS，那么极有可能内部服务器有的压力很大，
有的闲的无聊， 而上述的2点原因，都是Nginx能解决的，并且负载策略Nginx和Apache都有n种策略，
而这也就是Nginx作为反向代理的作用。 

---

## K8S ingress nginx

### k8s ingress是啥东东

上篇文章介绍service时有说了暴露了service的三种方式ClusterIP、NodePort与LoadBalance，
这几种方式都是在service的维度提供的，service的作用体现在两个方面，对集群内部，它不断跟踪pod的变化，
更新endpoint中对应pod的对象，提供了ip不断变化的pod的服务发现机制，对集群外部，他类似负载均衡器，
可以在集群内外部对pod进行访问。但是，单独用service暴露服务的方式，在实际生产环境中不太合适：

* ClusterIP的方式只能在集群内部访问。
* NodePort方式的话，测试环境使用还行，当有几十上百的服务在集群中运行时，NodePort的端口管理是灾难。
* LoadBalance方式受限于云平台，且通常在云平台部署ELB还需要额外的费用。

所幸k8s还提供了一种集群维度暴露服务的方式，也就是ingress。ingress可以简单理解为service的service，
他通过独立的ingress对象来制定请求转发的规则，把请求路由到一个或多个service中。这样就把服务与请求规则解耦了，
可以从业务维度统一考虑业务的暴露，而不用为每个service单独考虑。
举个例子，现在集群有api、文件存储、前端3个service，可以通过一个ingress对象来实现图中的请求转发：

### ingress与ingress-controller

要理解ingress，需要区分两个概念，ingress和ingress-controller：

#### ingress对象：

指的是k8s中的一个api对象，一般用yaml配置。作用是定义请求如何转发到service的规则，可以理解为配置模板。

#### ingress-controller：

具体实现反向代理及负载均衡的程序，对ingress定义的规则进行解析，根据配置的规则来实现请求转发。
简单来说，ingress-controller才是负责具体转发的组件，通过各种方式将它暴露在集群入口，
外部对集群的请求流量会先到ingress-controller，而ingress对象是用来告诉ingress-controller该如何转发请求，
比如哪些域名哪些path要转发到哪些服务等等。

### ingress-controller

ingress-controller并不是k8s自带的组件，实际上ingress-controller只是一个统称，用户可以选择不同的ingress-controller实现，
目前，由k8s维护的ingress-controller只有google云的GCE与ingress-nginx两个，其他还有很多第三方维护的ingress-controller，
具体可以参考官方文档。但是不管哪一种ingress-controller，实现的机制都大同小异，只是在具体配置上有差异。
一般来说，ingress-controller的形式都是一个pod，里面跑着daemon程序和反向代理程序。
daemon负责不断监控集群的变化，根据ingress对象生成配置并应用新配置到反向代理，
比如nginx-ingress就是动态生成nginx配置，动态更新upstream，并在需要的时候reload程序应用新配置。
为了方便，后面的例子都以k8s官方维护的nginx-ingress为例。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: abc-ingress
  annotations: 
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  tls:
  - hosts:
    - api.abc.com
    secretName: abc-tls
  rules:
  - host: api.abc.com
    http:
      paths:
      - backend:
          serviceName: apiserver
          servicePort: 80
  - host: www.abc.com
    http:
      paths:
      - path: /image/*
        backend:
          serviceName: fileserver
          servicePort: 80
  - host: www.abc.com
    http:
      paths:
      - backend:
          serviceName: feserver
          servicePort: 8080
```

与其他k8s对象一样，ingress配置也包含了apiVersion、kind、metadata、spec等关键字段。
有几个关注的在spec字段中，tls用于定义https密钥、证书。rule用于指定请求路由规则。
这里值得关注的是metadata.annotations字段。在ingress配置中，annotations很重要。
前面有说ingress-controller有很多不同的实现，而不同的ingress-controller就可以根据
"kubernetes.io/ingress.class:"来判断要使用哪些ingress配置，同时，
不同的ingress-controller也有对应的annotations配置，用于自定义一些参数。
列如上面配置的'nginx.ingress.kubernetes.io/use-regex: "true"',最终是在生成nginx配置中，
会采用location ~来表示正则匹配。

### ingress的部署

ingress的部署，需要考虑两个方面：

ingress-controller是作为pod来运行的，以什么方式部署比较好
ingress解决了把如何请求路由到集群内部，那它自己怎么暴露给外部比较好
下面列举一些目前常见的部署和暴露方式，具体使用哪种方式还是得根据实际需求来考虑决定。

#### Deployment+LoadBalancer模式的Service

如果要把ingress部署在公有云，那用这种方式比较合适。用Deployment部署ingress-controller，
创建一个type为LoadBalancer的service关联这组pod。大部分公有云，
都会为LoadBalancer的service自动创建一个负载均衡器，通常还绑定了公网地址。
只要把域名解析指向该地址，就实现了集群服务的对外暴露。

#### Deployment+NodePort模式的Service

同样用deployment模式部署ingress-controller，并创建对应的服务，但是type为NodePort。这样，ingress就会暴露在集群节点ip的特定端口上。由于nodeport暴露的端口是随机端口，一般会在前面再搭建一套负载均衡器来转发请求。该方式一般用于宿主机是相对固定的环境ip地址不变的场景。
NodePort方式暴露ingress虽然简单方便，但是NodePort多了一层NAT，在请求量级很大时可能对性能会有一定影响。

#### DaemonSet+HostNetwork+nodeSelector

用DaemonSet结合nodeselector来部署ingress-controller到特定的node上，然后使用HostNetwork直接把该pod与宿主机node的网络打通，直接使用宿主机的80/433端口就能访问服务。这时，ingress-controller所在的node机器就很类似传统架构的边缘节点，比如机房入口的nginx服务器。该方式整个请求链路最简单，性能相对NodePort模式更好。缺点是由于直接利用宿主机节点的网络和端口，一个node只能部署一个ingress-controller pod。比较适合大并发的生产环境使用。