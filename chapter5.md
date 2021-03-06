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

在之前的例子上，你以独立进程的方式执行了curl命令，但是只是在pod最重要的容器中。这和容器中实际上的主进程service传输信息并没有什么区别。

**在service上配置session affinity**

如果多次执行同样的命令，每次调用执行可能是在不同的pod上。但是如下清单一样，可以将service的sessionAffinity属性设置为ClientIP（而不是None ，None是默认值）来避免此情况的发生。

代码清单5.2 sessionAffinity被设置成ClientIP的service的例子

```
apiVersion: v1
kind: Service
spec:
  sessionAffinity: ClientIP
...
```

这种方式将会使service proxy将来自同一个client ip地址的所有请求转发至同一个pod上。作为练习，创建额外的service并将session affinity设置为ClientIP,并尝试向其发送请求。

kubernetes仅仅支持两种形式的service session affinity：None和ClientIP。或许惊讶竟然不支持基于cookie的session affinity的选项，但是你要了解到kubernetes service不是在http层面上工作。Service处理TCP和UDP包，并不关心其中的载荷内容。因为cookie是http协议中的一部分，service并不知道他们，这就解释了为什么session affinity不能基于cookie。

**同一个service使用多个端口**

你创建的service使用（expose）一个端口，但是service也支持多个端口。比如，你的pod监听两个端口，比如说HTTP的8080和HTTPS的8443，你可以使用一个service从端口80和443转发至pod端口8080和8443。在这种情况下，不需要创建两个不同的服务。通过一个集群IP，在使用一个多端口的service就可以将服务的多个端口全部暴露出来。

> **注意**
> 在创建一个有多个端口的service的时候，必须指定多个端口的名字。

以下列表中显示了支持多端口的service spec：
代码清单5.3 在service定义中指定多端口

```
apiVersion: v1
kind: Service
metadata:
name: kubia
spec: ports:
  - name: http
    port: 80                    <-------
    targetPort: 8080            <-------pod的端口8080映射成端口80
  - name: https
    port: 443                   <-------
    targetPort: 8443            <-------pod的端口8443映射成端口443
  selector:
app: kubia                      <-------label selector适用于整个servcie
```

> **注意**
>  label selector应用于整个service，不能对每个端口做单独的配置。如果不同的pod有不同的端口映射关系，需要创建两个service

之前创建的kubia pod不在多个端口上侦听，因此可以练习创建一个多端口service和一个多端口pod。

**使用命名端口**

在这些例子里，通过数字来指定端口，但是在service spec里也可以给不同的端口号命名，通过名称来指定。这样对一些不是众所周知的端口号的情况下，使得service spec更加清楚。
举个例子，假设你的pod端口定义命名如下表所示：

代码清单5.4 在pod的定义中指定port名称

```
kind: Pod
spec:
  containers:
  - name: kubia
    ports:
    - name: http
      containerPort: 8080        <-------端口8080被命名为http
    - name: https
      containerPort: 8443        <-------端口8443被命名为https
```
可以在service spec中按名称引用这些端口，如下面清单所示：

代码清单5.5 在service中引用命名pod

```
apiVersion: v1
kind: Service
spec:
  ports:
  - name: http
port: 80
    targetPort: http             <-------将端口80映射到容器中被称为http的端口
  - name: https
    port: 443
    targetPort: https            <-------将端口443映射到容器中被称为https的端口
Port 80 is mapped to the container’s port called http.
Port 443 is mapped to the container’s port, whose name is https.
```

为什么要采用命名端口的方式？最大的好处就是即使更换端口号也无需更改service spec。你的pod现在对http服务用的是8080，但是假设过段时间你决定将端口更换为80呢？

但是如果你采用了命名的端口，你仅仅需要做的就是改变pod spec中的端口号（当然你的端口号的名字没有改变）。在你的pod向新端口更新时，根据pod收到的连接(端口8080在旧的pod上，端口80在新的pod上)，用户连接将会转发到对应的端口号上。

##5.1.2 发现服务

通过创建service，现在就可以通过个单一稳定的IP地址访问到pod。在service整个生命周期内这个地址保持不变。在service后面的pod可能消失重建，他们的IP地址可能改变，数量也会增减，但是始终可以通过service的单一不变的IP地址访问到这些pod。

但客户端pod如何知道service的IP和端口？是否需要先创建服务，然后手动查找其IP地址并将IP传递给客户端pod的配置选项？当然不。 Kubernetes还为客户端提供了发现service的IP和端口的方式。

**通过环境变量发现服务**

在pod开始运行的时候，kubernetes会初始化一系列的环境变量指向现在存在的service。如果你创建service早于客户端pod的创建，pod上的进程可以根据环境变量获得service的IP地址和端口号。

在一个运行pod上检查环境，去了解这些环境变量。现在已经了解了通过`kubectl exec`命令在pod上运行一个命令，但是由于service的创建晚于pod的创建的，那么关于这个service的环境变量并没有设置，这个问题也需要被解决。

在查看service的环境变量之前，首先需要删除掉所有的pod使得replicationcontroller创建全新的pod。在无需知道pod的名字的情况下就能删除所有的pod，就像这样：

```
$ kubectl delete po --all
pod "kubia-7nog1" deleted
pod "kubia-bf50t" deleted
pod "kubia-gzwli" deleted
```
现在列出所有新的pod(确信大家知道如何操作)，然后选择一个作为`kubectl exec`命令的执行目标。一旦选择了目标pod，通过在容器里运行`env`来列出来所有的环境变量，如下表所示。

代码清单5.6 容器中和service相关的环境变量

```
$ kubectl exec kubia-3inly env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=kubia-3inlyKUBERNETES_SERVICE_HOST=10.111.240.1 
KUBERNETES_SERVICE_PORT=443
... 
KUBIA_SERVICE_HOST=10.111.249.153 
KUBIA_SERVICE_PORT=80
...
Here’s the cluster IP of the service.
And here’s the port the service is available on.
```
在cluster中定义了两个service：`kubernetes`和`kubia`服务（之前在用`kubectl get svc`命令的时候应该见过）；所以，列表中显示了和这两个service相关的环境变量。在本章开始部分，创建了`kubia` service，在和其有关的环境变量中有KUBIA_SERVICE _HOST 和KUBIA_SERVICE_PORT ，分别代表了`kubia` service的IP地址和端口号。 

回顾本章开始部分的前端-后端的例子，当前端pod需要后端数据库服务pod，可以通过名为`backend-database`的service将后端pod显露(expose)出来，然后前端pod通过环境变量BACKEND\_DATABASE\_SERVICE\_HOST 和 BACKEND\_DATABASE\_SERVIC\E_PORT去获得IP地址和端口信息。
> **注意**
> 服务名称中的破折号被转换为下划线，并且当服务名称用作环境变量名称中的前缀时，所有的字母都是大写的。

环境变量是获得service IP地址和端口号的一种方式，为什么不用DNS域名？为什么kubernetes中没有DNS服务器，并允许通过DNS来获得所有service的IP地址？事实证明，它的确如此！

**通过DNS发现service**

还记得第三章中在`kube-system`命名空间下列出的所有pod的名字吗？其中一个pod被称作`kube-dns`，当前的`kube-system`的命名空间中中也包含了一个具有相同名字的响应service。

就像名字的暗示，这个pod运行DNS服务，在集群中的其他pod都被配置成使用其作为dns（kubernetes通过修改每个容器的`/etc/resolv.conf`文件实现）。运行在pod上的进程DNS查询都会被kubernetes自身的DNS server响应，该服务器知道系统中运行的所有service。

> **注意**
> pod是否使用内部的DNS服务器是根据pod spec中`dnsPolicy`属性来决定的

每个service从内部DNS server中获得一个DNS条目，客户端的pod在知道service名字的情况下可以通过全限定域名（FQDN）来访问,而不是诉诸于环境变量。

**通过FQDN连接服务**

再次回顾前端-后端的例子，前端pod可以通过打开到以下FQDN的连接来访问后端数据库服务：

`backend-database.default.svc.cluster.local`

`backend-database`对应于服务名称，`default`表示服务在其中定义的名称空间，而`svc.cluster.local`是在所有集群本地服务名称中使用的可配置集群域后缀。
> **注意**
> 客户端仍然必须知道service的端口号。如果service使用标准端口号（例如 http的80端口，Postgres的5432端口），这样是没问题的。如果并不是标准端口，客户端可以从环境变量中获取端口号。

连接一个service可能比这更简单。如果前端pod和数据库pod在同一个命名空间下，你可以省略`svc.cluster.local`后缀，甚至命名空间。因此你可以使用`backend-database`来指代service。这是不可思议的简单，不是吗？

尝试一下。尝试使用FQDN来代替IP去访问`kubia`service。另外，必须在一个存在的pod上才能这样做。已经知道通过`kubectl exec`在一个pod的容器上去执行一个简单的命令。但是这一次不是直接的使用`curl`命令，而是使用`bash`shell，这样的话，可以在容器上运行多条命令。在第二章中，当想进入容器时，启动docker时调用`docker exec -it bash`命令，这与此很相似。

**在pod容器上运行shell**

可以通过`kubectl exec`命令在一个pod容器上运行`bash`（或者其他形式的shell）。通过这种方式，可以随意浏览容器，而无需为每个要运行的命令执行`kubectl exec`。
> **注意**
> shell的二进制可执行文件必须在容器映像中可用才能使用。

为了正常的使用shell，`kubectl exec`需要-it选项

```
$ kubectl exec -it kubia-3inly bash
root@kubia-3inly:/#
```
这样的话就处于container内部，根据下述的任何一种方式使用`curl`命令来访问`kubia`service

```
root@kubia-3inly:/# curl http://kubia.default.svc.cluster.local
You’ve hit kubia-5asi2
root@kubia-3inly:/# curl http://kubia.default
You’ve hit kubia-3inly
root@kubia-3inly:/# curl http://kubia
You’ve hit kubia-8awf3
```

在请求的URL中，可以将service的名称作为主机名来访问service。因为根据每个pod容器DNS解析器配置的方式，可以将命名空间和`svc.cluster.local`后缀省略掉。查看一下容器内的`/etc/resilv.conf`文件就明白了。

```
root@kubia-3inly:/# cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local ...
```

**理解ping不通service的IP**

在继续之前还有最后一问题。了解了如何创建service，所以很快的去自己创建一个。但是，不知道任何原因，无法访问创建的service？

大家可能会尝试通过进入现有的pod，并尝试像上一个示例中那样访问该service来找出问题所在。然后，如果仍然无法使用简单的`curl`命令访问service，也许会尝试ping service IP以查看service是否已启动。现在来尝试一下:

```
root@kubia-3inly:/# ping kubia
PING kubia.default.svc.cluster.local (10.111.249.153): 56 data bytes
^C--- kubia.default.svc.cluster.local ping statistics ---
54 packets transmitted, 0 packets received, 100% packet loss
```

嗯，curl这个service是工作的，但是却ping不通。这是因为service的集群IP是一个虚拟IP，并且只有在与service端口结合时才有意义。将在第11章中解释这意味着什么以及服务是如何工作的。在这里提到这一点，这是因为用户疏于警惕,在尝试调试异常的服务时所做的第一件事。
##5.2 访问集群外部的service

到现在为止，讨论了由集群内运行的一个或多个pod支持的service。但是还有另外一种情况的发生，就是想通过Kubernetes service功能访问（expose）外部service。不要让service将连接重定向到集群中的pod，而是希望它重定向到外部IP和端口。

这样做可以让充分利用服务负载平衡和服务发现。在集群中运行的客户端pod可以像连接到内部服务一样连接到外部服务。
##5.2.1 介绍service endpoint

在进入如何做到这一点之前，先阐述一下service。service并不是和pod直接相连。相反，有一种资源介于两者之间—它就是Endpoint 资源。如果之前在service上运行过`kubectl describe`，可能已经了解到endpoint，如下表所示

代码清单5.7 用kubectl describe展示service的全部细节

```
Name:               kubia
Namespace:          default
Labels:             <none>
Selector:           app=kubia                                             <-------用于创建endpoint列表的service pod selector                                      
Type:               ClusterIP
IP:                 10.111.249.153
Port:               <unset> 80/TCP
Endpoints:          10.108.1.4:8080,10.108.2.5:8080,10.108.2.6:8080       <-------代表service endpoint的pod的IP和端口列表
Session Affinity:        None
No events.
```

Endpoint资源就是暴露一个服务的IP地址和端口的列表，endpoint资源和kubernetes其他资源一样，所以可以使用`kubectl info`来获取它的基本信息。

```
$ kubectl get endpoints kubia
NAME    ENDPOINTS                                         AGE
kubia   10.108.1.4:8080,10.108.2.5:8080,10.108.2.6:8080   1h
```
尽管在service spec中定义了pod selector，但在重定向传入连接时不会直接使用它。相反，selector用于构建IP和端口列表，然后存储在endpoint资源中。当客户端连接到服务时，服务代理选择这些IP和端口对中的一个，并将传入连接重定向到在该位置监听的服务器。

##5.2.2 手动配置service endpoint

或许已经意识到这一点，service endpoint与service解耦后，可以分别的手动配置和更新它们。

如果创建了不包含pod selector的service，Kubernetes将不会创建endpoint资源（毕竟，缺少selector，将不会知道service中包含哪些pods）。这样就需要创建endpoint资源来指定该service的endpoint列表。

要使用手动配置endpoints的方式创建service，需要创建service和endpoint资源。
 
**创建无selector的service**
 
首先为service创建一个YAML文件，如下表所示

代码清单5.8 不含pod selector的service：external-service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: external-service       <------service 的名字必须和endpoint的宁子相匹配
spec:
  ports:                       <------service中没有定义selector
  - port: 80
```

定义一个名为external-service的服务，它将接受端口80上的传入连接。并没有为service定义一个pod selector。

**为缺少selector的service创建endpoint资源**

Endpoint 是一个单独的资源并不是service的一个属性。由于创建的资源中并不包含selector，相关的endpoint资源并没有自动的创建，所以必须额外来创建。下表列出了YAML manifest。

代码清单5.9 手动创建endpoint资源：external-service-endpoints.yaml

```
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service            <--------endpoint的名字必须和service的名字像匹配（见之前的代码清单）
subsets:
  - addresses:
    - ip: 11.11.11.11               <--------service将连接重定向到endpoint的IP地址
    - ip: 22.22.22.22
    ports:
    - port: 80                      <--------endpoint的目标端口
```

Endpoints对象需要与service具有相同的名称，并包含该service的目标IP地址和端口列表。service和endpoing资源都发布到服务器后，这样service就可以像具有pod selector那样的service正常使用。在service创建后创建的容器将包含service的环境变量，并且与其IP:port对的所有连接都将在服务端点之间进行负载均衡。

图5.4显示了三个pod连接到具有外部endpoint的service

![图 5.4 pod使用（consuming）具有两个外部endpoint的service上](figures/Figure5.4.png )

如果稍后决定将外部servicee迁移到Kubernetes内运行的pod，为service添加selector，从而对endpoint进行自动管理。反过来也是一样的--将selector从service中移除，kubernetes将停止更新endpoint。这意味着service的IP地址可以保持不变，同时service的实际实现却发生了改变。

##5.2.3 为外部的service创建别名 

除了手动配置service的endpoint来代替公开外部service方法之外，有种更简单的方法就是通过其完全限定域名（FQDN）访问外部service 

**创建一个外部名称的service**

要创建一个具有别名的外部服务的service时，要将创建service资源的一个`type`字段设置为`ExternalName`。例如。设想下在api.somecompany.com有公共可用的API。可以定义一个指向它的服务，如下面的清单所示。

代码清单5.10 type是ExternalName的service：external-service-externalname.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName                         <-------service的type被设置成ExternalName
  externalName: someapi.somecompany.com      <-------实际服务的完全限定域名
  ports:
  - port: 80
```

service创建完成后，pod可以通过`external-service.default.svc.cluster.local`域名（甚至是`external-service`）连接到外部服务，而不是使用服务的实际FQDN。这隐藏了实际的service名称及其使用该服务的pod的位置，允许修改service定义，并且在以后如果将其指向不同的service，只是简单的修改`externalName`属性或者将类型重新变回`ClusterIP`并为service创建endpoint—无论是手动创建，或者对service上指定label selector使其自动的创建。

`ExternalName` service仅在DNS级别实施 - 为service创建了简单的CNAME DNS记录。因此，连接到服务的客户端将直接连接到外部服务，完全绕过服务代理。出于这个原因，这些类型的服务甚至不会获得集群IP。
> **注意**
> CNAME记录指向完全限定的域名而不是数字IP地址。
##5.3 外部客户请求内部service

到目前为止，只讨论了集群内服务如何被pod使用。但是，你还需要向外部公开某些service 例如前端Web服务器，以便外部客户端可以访问它们，就像图5.5描述的那样
![图 5.5 将内部service公开给外部的客户端](figures/Figure5.5.png )

有几种方式可以在外部访问service：

* _将service的`type`设置成`NodePort`_ -每个群集节点都会在节点上打开一个端口，对于`NodePort`service，每个集群节点在节点本身（因此得名叫NodePort）上打开一个端口，并将在该端口上接收到的流量重定向到基础服务。该service仅在内部群集IP和端口上才可访问，但也可通过所有节点上的专用端口访问。
* _将service的`type`设置成`LoadBalance`,`NodePort`类型的一种扩展_ -这使得service可以通过一个专用的负载均衡器来访问，这是由Kubernetes中正在运行的云基础设施提供的。负载均衡器将流量重定向到跨所有节点的节点端口。客户端通过负载均衡器的IP连接到service。
* _创建一个Ingress资源，这是一个完全不同的机制，通过一个IP地址公开多个service_ -它运行在HTTP层（网络协议第七层），因此可以提供比工作在第4层的service更多的功能。我们将在5.4节介绍Ingress资源

##5.3.1 使用NodePost service

将一组pod公开给外部客户端的第一种方法是创建一个service并将其type设置为`NodePort`。通过创建NodePort服务，可以让Kubernetes在其所有节点上保留一个端口(所有节点上都使用相同的端口号)，并将传入的连接转发给作为服务部分的pods。

这与常规service类似（它们的实际类型是ClusterIP），但是不仅可以通过服务的内部群集IP访问NodePort service，还可以通过任何节点的IP和预留节点端口访问NodePort service。

当尝试与NodePort服务交互时，意义更加重大。

**创建nodeport service**

现在将创建一个NodePort service，以查看如何使用它。下面的清单显示了service的YAML。

代码清单5.11 NodePort service定义：kubia-svc-nodeport.yaml

```
apiVersion: v1
kind: Service
metadata: 
   name: kubia-nodeport
spec:
  type: NodePort                 <------将service的type属性设置成NodePort
  ports:
  - port: 80						   <------service集群IP的端口号
    targetPort: 8080             <------背后pod的目标端口号
    nodePort: 30123              <------通过集群node的30123端口可以访问该service
  selector:
app: kubi
```

将类型设置为NodePort并指定该service应该绑定到的所有集群节点的节点端口。指定端口不是强制性的。如果忽略它，Kubernetes将选择一个随机端口。
> **注意**
> 当在GKE中创建服务时，kubectl打印出一个关于必须配置防火墙规则的警告。接下来的章节讲述如何处理。

**检查你的nodeport service**
查看service(nodeport service)的基础信息来更好的了解它。

```
$ kubectl get svc kubia-nodeport
NAME             CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubia-nodeport   10.111.254.223   <nodes>       80:30123/TCP   2m
```
看看EXTERNAL-IP列。它显示<节点>，暗示service可通过任何集群节点的IP地址访问。 PORT（S）列显示cluster IP（80）的内部端口和节点端口（30123），可以通过以下地址访问该服务：

* 10.11.254.223:80
* <1stnode’sIP>:30123
* <2ndnode’sIP>:30123, 等等

图5.6显示了service暴露在两个集群节点的端口30123上（这适用于在GKE上运行的情况; Minikube只有一个节点，但原理相同）。到达任何一个端口的传入连接将被重定向到一个随机选择的pod，该pod是否是位于接收到连接的节点上是不确定的。
![图 5.6 外部客户端经过Node1 或者Node2 连接一个NodePort service](figures/Figure5.6.png )
在第一个node的端口30123收到的连接可以被重定向到第一node个上运行的pod也可能是到第二个node上运行的pod。
