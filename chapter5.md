#服务：使客户能够发现并与Pod交谈
本章内容涵盖
>* 创建service资源，利用单个地址访问一组pod
>* 发现集群中的service
>* 将service公开给外部客户
>* 从集群内部连接到外部service
>* 判断Pod是否已准备好成为服务的一部分
>* 排除服务故障

到现在介绍了Pod以及如何通过ReplicaSets和类似资源部署它们以确保它们持续运行。尽管特定的pod可以独立地根据外部激励做出正确的反应，现在大多数应用都是根据外部请求做出回应。例如就微服务而言，pod对来自集群内部的其他pod或者集群外部的客户端的http请求做出回应。

Pod需要找到其他pod的方法如果他们需要使用其他pod提供的服务，不像在没有kubernetes的世界里，系统管理员要在用户端配置文件中明确指出提供服务的精确的IP地址或者主机名来配置每个客户端应用，但是同样的方式在kubernetes中并不适用，因为

* _Pod并不是一成不变的_ —他们可以在任何时候启动或者关闭，无论是因为pod为了给其他pod提供空间被关闭或者主动减少pods的数量还是因为集群中的pod异常结束。
* _Kubernetes在Pod已安排到节点并启动之前为Pod分配IP地址_ —因此客户端不能提前知道提供服务的pod的IP地址
* _横向伸缩(Horizontal scaling)意味着多个pod可以提供相同的服务_ —每个pod都有自己的IP地址，客户端无需关心背后提供服务pod的数量以及各自对应的IP地址。他们无需记录每个pod IP地址。相反，所有的pod可以通过一个单一的IP地址进行访问。

为了解决上述问题，kubernetes也提供了一种资源类型—service，在本章中将对其进行阐述说明。
##5.1 介绍service
Kubernetes service是一种资源，可以为提供同样服务的pod提供单一、常用的接入点。当service存在的情况下，该服务拥有固定的IP地址和端口。客户端通过IP和端口号建立连接，这些连接会被路由到提供该服务的任意一个pod上。通过这种方式，客户端不需要知道每个单独的提供该服务的pod的地址，允许这些pod可以在集群中移入或者移出。

**以示例解读服务**

回顾一下有前端Web服务器和后端数据库服务器的例子。有很多pod提供前端服务，而只有一个pod提供后台数据库服务。需要解决两个问题才能使系统发挥作用。

* 外部服务器连接到pod上，而无需关心有前端web服务器一个还是多个
* 前端的pod需要连接到后台的数据库上。由于数据库运行在一个pod上，它可能会在集群中移来移去，导致IP地址变化。当后台数据库被移动时，无需对前端pod重新配置。

为前端pod创建一个service，并且将其配置成可以在集群外部访问到。外部的客户端利用单一的固定的IP地址来连接这些pod。类似的，可以为后台数据库pod创建service，并为其分配一个固定的IP地址。尽管pod的IP地址会改变，但是service的IP地址固定不变。另外，通过创建service，前端的pod在知道后端的service的名字下可以通过环境变量或者DNS来轻松的访问到。系统中所有的元素都在图5.1中展示出来（两种service，支持这些service的两套pod，以及它们之间的相互依赖关系）
![图 5.1 内部和外部客户端通常通过service连接到pod](figures/Figure5.1.png )
到目前为止了解了service背后的基本理念。那么现在，去深入研究如何创建它们。
##5.1.1 创建service
就像了解到的，service可以由多个pod组成。所有的pod对service的连接负载均衡。但是如何准确的定义哪些pod属于service的一部分，哪些不是？

或许还记得label selecto以及它们在Replication-Controllers和其他Pod控制器中的用法，以指定哪些Pod属于同一组。service以相同的方式使用相同的机制，可以参考图5.2。

在之前的章节中，通过创建replication controller来运行三个pod ，每个pod包含node js 服务。再次创建replication controller并且确认pod启动并且运行，在这个之后为这三个pod创建一个service。

![图 5.2 Label selectors确定哪些pod属于服务](figures/Figure5.2.png )

**通过KUBECTL expose创建service**

创建service的最简单方法是通过`kubectl expose`，在第2章中已经使用它将前面创建的ReplicationController可以被访问到。像创建replication controller时使用的pod selector那样，利用`expose`命令和pod selector来创建service资源，从而通过单个的IP和端口来访问所有的pod。

现在，除了使用expose命令之外，可以通过将配置的YAML文件传递到kubernetes API server来手动创建service

**通过YAML描述文件来创建service**

使用以下列表的内容创建一个名为kubia-svc.yaml的文件

代码清单 5.1 service的定义：kubia-svc.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
  - port: 80             <---------service的可用端口
    targetPort: 8080     <---------service将连接转发到的容器端口
  selector:
app: kubia               <---------具有app=kubia标签的都属于service的一部分
```

创建了一个名叫kubia的service，它将在端口80接受请求并将连接路由到具有label selector是`app=kubia`的pod的8080端口上。

接下来通过使用`kubectl create`发布文件来创建服务.

**检测新的service**

在传递完YAML文件后，可以在命名空间下列出来所有的service资源，并可以发现新的service已经被分配了一个内部的集群IP。

```
$ kubectl get svc
NAME         CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   10.111.240.1     <none>        443/TCP   30d
kubia        10.111.249.153   <none>        80/TCP    6m       <-------这个service
```

列表显示service拥有10.111.249.153的IP地址。因为只是集群的IP地址，只能在集群内部可以被访问到。service的主要目标就是使集群内部的其他pod可以访问到当前这组pod，但通常也希望在外部公开（expose）service。如何实现将在之后讲解。现在，从集群内部使用创建好的service并了解service的功能。

**通过内部集群测试service**

在集群内部有若干方法向service发送请求

*  显而易见的方法是创建一个pod ，它将请求发送到service的集群IP并记录响应。可以通过查看pod日志了解service的回复
*  使用`ssh`远程登录到kubernetes节点上，然后使用`curl`命令
*  可以通过`kubectl exec`命令在一个已经存在的pod上执行`curl`命令

针对最后一个选项，学习如何在现有的pod中运行命令。

**在运行的容器中远程执行命令**

可以使用`kubectl exec`命令远程的在一个已经存在的pod容器上执行任何命令。这样就可以很方便了解pod的内容、状态以及环境。用`kubectl get pods`命令列出所有的pods并且选择其中一个作为exec命令的执行目标（在下述例子中，选择`kubia-7nog1` pod作为目标）。也可以获得service的集群（cluster）IP （比如使用`kubectl get svc`命令），当执行下述命令时，请确保替换对应pod的名称以及service IP地址

```
$ kubectl exec kubia-7nog1 -- curl -s http://10.111.249.153
You’ve hit kubia-gzwli
```

如果之前使用过`ssh`命令登录到一个远程系统，会发现`kubectl exec`没有特别大的不同之处

>**为什么是双破折号(-)?**
>
双破折号代表着`kubectl`命令选项的结束。在两个破折号之后的内容是指在pod内部需要执行的命令。如果需要执行的命令并没有以破折号开始的参数，双破折号也不是必须的。如下情况，如果这里不使用双破折号，-s的选项会被解析成`kubectl exec`的选项，会导致结果异常和歧义错误。

```
$ kubectl exec kubia-7nog1 curl -s http://10.111.249.153
The connection to the server 10.111.249.153 was refused – did you
     specify the right host or port?
```
>service拒绝连接之外什么都不做。这是因为kubectl并不能连接到位于10.111.249.153的API server（-s的选项被解析成kubectl的一部分，导致kubectl去连接不同的API server而不是默认值）。 

重温一下在运行命令期间发生了什么。图5.3显示了事件的顺序。在一个pod容器上，利用kubernetes去执行curl命令。Curl向service iP发送出一个HTTP请求，service后面三个pod做支撑，kubernetes service 代理截取的该连接，在三个pod中任意选择了一个pod，之后将请求转发给它。在pod中运行的Node js处理请求，并返回带有pod名称的HTPP应答。Curl接收应答并向标准的输出打印，然而应答也kubectl被截取，打印在本地机器的标准输出上。

![图 5.3 使用kubectl exec其中一个容器中运行`curl`来测试与服务的连接](figures/Figure5.3.png )
