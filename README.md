
## 什么是Consul?


Consul是一种服务网络解决方案，使团队能够管理服务之间以及跨本地和多云环境和运行时的安全网络连接。Consul提供服务发现、服务网格(service mesh)、流量管理和网络基础设施设备的自动更新。可以在单个Consul部署中单独或一起使用这些功能。


## 架构介绍


Consul提供了一个控制平面(control plane)，允许注册、查询和保护部署在网络上的服务。控制平面是网络基础设施的一部分，它维护一个中央注册表来跟踪服务及其各自的IP地址。它是一个在节点(如物理服务器、云实例、虚拟机或容器)集群上运行的分布式系统。


Consul通过代理与数据平面交互。数据平面是处理数据请求的网络基础设施的一部分


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213231521121-1028075312.png)


Consul的核心工作流程由以下几个阶段组成：


* **注册**：团队将服务添加到Consul目录中 \-\- 一个中心注册表，该注册表允许服务自动发现彼此，而不需要人工操作修改应用程序代码、部署额外的负载均衡器或硬编码IP地址。它是所有服务及其地址的运行时真实来源。团队可以使用 CLI 或 API 手动定义和[注册](https://github.com)服务，也可以通过[服务同步](https://github.com)在Kubernetes中实现流程自动化。服务还包括健康检查，以便 Consul 监控不健康的服务。
* **查询**：Consul的基于身份的DNS支持在Consul目录中查找健康的服务。在Consul注册的服务提供健康信息、接入点和其他帮助控制网络中的数据流的数据。服务只能根据自定义的基于身份的策略通过其本地代理访问其他服务。
* **安全**：在服务定位上游后，Consul 确保服务和服务之间的通信经过身份验证、授权和加密。Consul服务网格使用mTLS保护微服务架构，并可以根据服务身份标识允许或限制访问，而不管计算环境和运行时的差异。


### 基本术语


#### 数据中心


Consul控制平面包含一个或多个数据中心。数据中心是Consul基础设施中可以执行基本Consul操作的最小单元。数据中心至少包含一个Consul服务器代理，实际部署中包含三到五个服务器代理和几个Consul客户端代理。可以创建多个数据中心，并允许不同数据中心的节点相互交互。参阅[Bootstrap一个数据中心](https://github.com)获取有关如何创建数据中心的信息。


#### 集群


相互感知的Consul 代理 的集合称为集群。术语*数据中心*和*集群*经常互换使用。然而，在某些情况下，集群仅指Consul服务器代理，某些情况下集群可能指的是客户端代理的集合。


#### 代理


可通过运行Consul二进制文件来启动Consul*代理*，它们是实现Consul控制平面功能的守护进程。可以将代理作为服务器或客户端启动。所有节点都要允许代理。参阅[Consul代理](https://github.com)获取更多信息。


##### 服务器代理


个人理解，服务器代理，即以服务器模式运行Consul代理的节点，对应上图中的`Consul Server`。Consul服务器代理存储所有状态信息，包括服务和节点IP地址、健康检查和配置。建议在集群中部署三到五台服务器。部署的服务器越多，发生故障时的弹性和可用性就越大。然而，更多的服务器会减慢 [consensus](https://github.com)，这是使Consul能够高效和有效地处理信息的关键服务器功能。


###### Consensus 协议


Consul集群通过一个称为*consensus*的过程选择一台服务器作为*leader*。*leader*处理所有查询和事务，防止在包含多个服务器的集群中发生冲突的更新。


当前不充当集群*leader*的Consul Server称为*followers*。*followers*将客户端代理的请求转发给集群 *leader* 。*leader* 将请求复制到集群中的所有其他服务器。请求副本可确保当 *leader*不可用时，群集中的其他服务器可以在不丢失任何数据的情况下选举新 *leader*。


Consul服务器使用Raft算法在端口`8300`上建立*consensus*。


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213231646730-572331012.png)


##### 客户端代理


个人理解，客户端代理，即以客户端模式运行Consul代理的节点，对应上图中的`Consul Client`。Consul客户端向Consul集群报告节点和服务健康状态。在典型的部署中，必须在数据中心的每个计算节点上运行客户端代理。客户端使用远程过程调用（RPC）与服务器交互。默认情况下，客户端通过端口 `8300`上的服务器发送RPC请求。


可以与Consul一起使用的客户端代理或服务的数量没有限制，但生产部署应将服务分布在多个Consul数据中心。使用多数据中心部署可以增强基础设施的弹性，并限制控制平面问题。建议每个数据中心最多部署5000个客户端代理。一些大型组织在多数据中心部署中部署了数万个客户端代理和数十万个服务实例。参阅[跨数据中心请求](https://github.com)以获取更多信息。


还可以使用部署Envoy代理但不部署客户端代理的备用服务网格配置运行Consul。请参阅[使用Consul数据平面的简化服务网格](https://github.com)了解更多信息。


#### 局域网八卦池(LAN gossip pool)


客户端和服务器代理加入局域网gossip池，以便他们可以分发和执行节点[健康检查](https://github.com)。池中的代理在整个群集中传播健康检查信息。代理使用`UDP`通过端口`8301`上进行gossip通信。如果`UDP`不可用，则采用TCP。请参阅[Gossip协议](https://github.com)获取更多信息。


以下简化图显示了服务器和客户端之间的交互。


*LAN gossip pool*


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213231704769-1989653744.png)


*RRC*


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213231719875-1198547463.png)


#### 跨数据中心请求


每个Consul数据中心都维护自己的服务目录及其运行状况。默认情况下，信息不会在数据中心之间复制。广域网联邦(WAN federation)和集群对等是两种多数据中心部署模型，可实现跨数据中心的服务连接。


##### 广域网联邦(WAN federation)


WAN federation是一种连接多个Consul数据中心的方法。它要求指定一个*主数据中心*(primary datacenter)，该数据中心包含关于所有数据中心的权威信息，包括服务网格配置和访问控制列表(`ACL`)资源。


在此模型中，当客户端代理请求远程辅助数据中心的资源时，本地Consul服务器将RPC请求转发给有权访问该资源的远程Consul服务器。远程服务器将结果发送到本地服务器。如果远程数据中心不可用，则其资源也不可获取。默认情况下，WAN\-federated 服务器通过TCP端口`8300`发送跨数据中心请求。


可以配置控制平面和数据平面流量通过网格网关，简化网络要求。



> 要在启用ACL系统时使服务能够跨数据中心通信，请参阅[多个数据中心的ACL复制](https://github.com)教程。


###### 广域网八卦池(WAN gossip pool)


服务器还可以加入WAN gossip pool，该池针对互联网带来的更大延迟进行了优化。该池使服务器能够交换信息，如地址和健康状况，并在发生故障时优雅地处理连接丢失。


在下图中，每个数据中心的服务器通过在端口通过`TCP/UDP` `8302`端口发送数据，加入WAN gossip pool。参阅[Gossip协议](https://github.com)获取更多信息。


WAN gossip pool


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213231736655-346857006.png)


PRC


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213231802477-1778431191.png)


##### **集群对等(Cluster peering)**


可以在两个或多个独立集群之间创建对等连接，以便部署到不同数据中心或管理分区的服务可以通信。[管理分区](https://github.com)是Consul Enterprise中的一项功能，能够定义使用相同Consul服务器的隔离网络区域。在集群对等模型中，可以在其中一个数据中心或分区中创建令牌，并配置另一个数据核心或分区以显示令牌以建立连接。


参阅[什么是集群对等？](https://github.com)获取更多信息。


### 参考链接


[https://developer.hashicorp.com/consul/docs/intro](https://github.com)


[https://developer.hashicorp.com/consul/docs/architecture](https://github.com)


## 服务发现(*Service discovery*)


### 什么是服务发现？


服务发现可帮助发现、跟踪和监视网络中服务的运行状况。服务发现在*服务目录*(*service catalog*)中注册并维护所有服务的记录。此服务目录充当一个单一的事实来源，允许服务相互查询和通信。


#### 举例说明


问题：当一项服务存在于多个主机节点上时，client端如何获取相应的 ip 和 port 呢。


如下，在传统情况下，都会使用静态配置的方法来存储服务配置信息，然后根据这些配置信息访问对应的服务。


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213231852250-1353243143.png)


如上图，假设客户端请求的某个接口，需要调用服务1\-N。客户端必须要知道所有服务的网络位置(ip\+端口)，以往的做法将服务器信息存储在配置文件，或者数据库中。这里就带出几个问题：


* 需要配置N个服务的网络位置，加大配置的复杂性
* 服务的网络位置变化，都需要改变每个调用者的配置
* 集群的情况下，难以做负载（反向代理的方式除外）


总结起来一句话：服务多了，配置很麻烦。


为了解决这个麻烦的问题，引入了服务发现，且看下图。


与之前的处理方式不同。图中增加了一个服务发现模块。服务1\-N把当前自己的网络位置注册到服务发现模块，同时服务发现模块以K\-V的形式记录这些注册信息。（`K` 一般设计为服务名，`V` 一般设计为 `ip:port`。此外，服务发现模块定时的轮询查看这些服务能不能访问(这就是健康检查)。


客户端在调用服务1\-N的时候，直接请求服务发现模块，查询对应服务的的网络位置，然后再调用对应的服务。不难发现，此时客户端不需要记录这些服务的网络位置信息，实现客户端与服务端的完全解耦。


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213231923550-585790505.png)


### 服务发现的好处


服务发现为所有组织带来了好处，从简化的可扩展性到应用程序弹性的提高。服务发现的一些好处包括：


* 动态IP地址和端口发现
* \-简化了横向服务扩展
* 将发现逻辑从应用程序中抽象出来
* 通过健康检查确保可靠的服务沟通
* 跨健康服务实例负载均衡请求
* 通过高速发现实现更快的部署时间
* 自动服务注册和注销


### 服务发现是任何工作的？


服务发现使用服务的身份，而不是传统的访问信息（IP地址和端口）。这允许动态映射服务并跟踪服务目录中的任何变更。然后，服务消费者（用户或其他服务）使用DNS从服务目录中动态检索其他服务的访问信息。服务的生命周期可能如下：


服务消费者通过服务目录提供的唯一Consul DNS条目与“Web”服务通信。


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213231941737-1458039843.png)


“Web”服务的一个新实例使用其IP地址和端口将自己注册到服务目录中。当服务的新实例注册到服务目录中时，它们将加入负载均衡池，以处理服务消费者的请求。


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213232001985-1270975186.png)


随着服务的新实例的添加和旧的或不健康的服务实例的删除，服务目录会动态更新。删除的服务将不再加入处理服务消费者请求的负载均衡池。


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213232017878-1560979715.png)


### 什么是微服务中的服务发现？


在微服务应用程序中，活动服务实例集在大型动态环境中经常发生变化。这些服务实例依赖于服务目录从相应的服务中检索最新的访问信息。可靠的服务目录对于微服务中的服务发现尤为重要，以确保健康、可扩展和高度响应的应用程序操作。


### 服务发现的两种主要模式


有两种主要的服务发现模式：*客户端*发现(*client\-side* discovery)和*服务器端*发现(*server\-side* discovery)。


在使用客户端发现的系统中，服务消费者负责确定可用服务实例的访问信息以及它们之间的负载均衡请求。


1. 服务消费者查询服务目录
2. 服务目录检索并返回所有访问信息
3. 服务消费者选择健康的下游服务，并直接向其发出请求


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213232502014-1321102231.png)


在使用服务器端发现的系统中，服务消费者使用中介查询服务目录并向其发出请求。


the service consumer uses an intermediary to query the service catalog and make requests to them.


1. 服务消费者查询中间服务(Consul)
2. 中间服务查询服务目录，并将请求路由到可用的服务实例。


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213232042579-1732090690.png)


对于现代应用程序，这种发现方法是有利的，因为开发人员可以通过解耦和集中服务发现逻辑使他们的应用程序更快、更轻量


### 服务发现 vs 负载均衡


服务发现和负载平衡在将请求分发到后端服务方面有相似之处，但在许多重要方面有所不同。


传统的负载均衡器不是为服务的快速注册和注销而设计的，也不是为高可用性而设计的。相比之下，服务发现系统使用多个节点来维护服务注册表状态，并使用对等状态管理系统来提高任何类型基础设施的弹性。


对于现代的基于云的应用程序，服务发现是将流量引导到合适的服务提供商的首选方法，因为它能够扩展并保持弹性，独立于基础设施。


### 参考链接


[https://developer.hashicorp.com/consul/docs/concepts/service\-discovery](https://github.com)


## 启动代理



```
consul agent 

```

常见选项：


* `-datacenter=` 代理所属数据中心
* `-auto-reload-config` 监视配置文件的更改，当配置文件被修改时，自动重新加载文件
* `-bind=` 设置群集通信的绑定地址。它是集群中所有其他节点都应该可以访问的IP地址。默认值为`0.0.0.0`，意味着Consul将绑定到本地计算机上的所有地址，并将第一个可用的私有IPv4地址通告给集群的其余部分。这个参数对于确保集群内部通信的稳定性和可靠性至关重要‌。
* `-advertise`：通知展现地址，用来改变我们给集群中的其他节点展现的地址，一般情况下\-bind地址就是展现地址，但在某些情况下，可能存在无法绑定的可路由地址，这时会激活一个不同的地址来供应。如果这个地址不可路由，其他节点会将视该节点为故障。这种情况下，advertise参数就派上用场了，该参数允许配置一个不同的地址来通告给集群，以确保节点能够正确地被集群中的其他成员发现。
* `-bootstrap`: 将服务器设置为引导模式。一个数据中心只能有一个server处于`bootstrap`模式，当一个server处于`bootstrap`模式时，可以自己选举为`leader`。
* `-bootstrap-expect=` 将服务器设置为期望的引导模式\-\-期望的 server 数量，当提供该值时，consul会一直等待直到指定sever数目的时候才会引导整个集群，该选项不能和`-bootstrap`同时使用
* `-check_output_max_size=` 设置此代理上检查的最大输出大小
* `-client=` 设置用于客户端访问的绑定地址。这包括RPC，DNS，HTTP、HTTPS和gRPC（如果已配置），缺省值为 `127.0.0.1`
* `-config-dir=` 读取配置文件的目录路径。这将读按字母排列顺序，读取该目录中以'.json'结尾的每个文件作为配置，可以多次指定。
* `-config-file=` 配置文件路径，可以多次指定。
* `-config-format=` 配置文件格式，与扩展名无关，必须为'hcl' 或者 'json'
* `-data-dir=` 存储代理状态的数据目录路径。
* `-default-query-time=` 阻塞查询在Consul强制返回响应之前等待的时间。此值可以被`wait`查询参数覆盖。
参数。
* `-dev` 在开发模式下启动代理。
* `-disable-host-node-id` 将此设置为`true`将阻止Consul使用主机信息生成节点ID，并将使Consul生成随机节点ID。
* `-dns-port=` 要使用的DNS端口。
* `-domain=` 用于DNS接口的域。
* `-enable-local-script-checks` 启用配置文件中的健康检查脚本。
* `-enable-script-checks` 启用健康检查脚本。
* `-encrypt=` 提供八卦(gossip)加密密钥。
* `-grpc-port=` 设置要监听的gRPC API端口
* `-grpc-tls-port=` 设置要监听的gRPC\-TLS API端口。
* `-http-port=` 设置要监听的HTTP API端口，默认8500，同时也用于Web UI。
* `-https-port=` 设置要侦听的HTTPS API端口。
* `-log-file=` 日志文件的路径
* `-log-json` 以JSON格式输出日志。
* `-log-level=` 代理日志级别。
* `-log-rotate-bytes=` 应写入日志文件的最大字节数
* `-log-rotate-duration=` 设置多久过后需要执行日志轮换
* `-log-rotate-max-files=` 要保留的最大日志文件数量
* `-max-query-time=` 阻塞查询在Consul强制返回响应之前可等待的最大时间。Consul将抖动应用于等待时间。这个抖动(jitter)时间将限制为`MaxQueryTime`。
* `-node=` 此节点的名称。在群集中必须是唯一的。
* `-node-id=` 此节点在空间和时间上的唯一ID。默认为一个随机生成并保存在数据目录中的ID。
* `-node-meta=` 此节点的任意元数据键/值对，格式为`key:value`。可以指定多次。
* `-pid-file=` 存储代理进程ID的文件路径。
* `-primary-gateway=` 主数据中心中要使用网格网关的地址。用于启用重试时，在启动时引导WAN联邦。可以多次指定。
* `-protocol=` 设置协议版本。默认为 `latest`。
* `-raft-protocol=` 设置Raft协议版本。默认为 `latest`。
* `-read-replica` （仅限企业版）此标志用于使服务器不参与Raft仲裁，让它只接收数据复制流。当需要对服务器进行大量读取的时，这可用于增加读取可扩展性
* `-recursor=` 上游DNS服务器的地址。可以多次指定。
* `-rejoin` 忽略之前的离开，尝试重新加入集群。
* `-retry-interval=` 两次加入重试之间等待时间
* `-retry-join=` 当启用重试时，要在启动时加入的代理地址，可以包含端口号（默认值`8301`），形如`192.168.1.105:8301`。 可以多次指定。如下，如果在配置文件中配置该参数，写成列表的形式，支持配置多个。



```
{
  "retry_join": ["192.168.1.105:8301", "192.168.1.106"]
}

```

不管采用哪种方式配置，Consul 代理启动时，都会不断尝试连接这些地址直到加入集群成功或者达到配置的最大重试次数。
* `-retry-max=` 重试加入的最大重试次数。默认为0，表示无限重试。
* `-serf-lan-allowed-cidrs value` Serf LAN允许的网络(比如`192.168.1.0/24`) 。可多次指定
* `-serf-lan-bind value` 需要绑定的Serf LAN要监听地址。缺省使用`-bind`地址
* `-serf-lan-port value` Serf LAN要监听的端口。
* `-serf-wan-allowed-cidrs value` Serf WAN允许的网络(比如`192.168.1.0/24`) 。可多次指定
* `-serf-wan-bind value` 需要绑定的Serf WAN要监听地址。缺省使用`-bind`地址
* `-serf-wan-port value` Serf WAN要监听的端口。
* `-server` 以服务器模式启动代理。`-server=false` 表示以客户端模式启动代理
* `-server-port=` 设置需要监听的服务器端口
* `-syslog` 启用记录日志到系统日志(syslog)
* `-ui` 启用内置静态web UI服务器。
* `-ui-content-path=` 将外部UI路径设置为字符串。默认为： `/ui/`
* `-ui-dir=` 包含web UI资源的目录路径。


部分参数也可以存放在配置文件，通过`-config-file`参数配置从配置文件读取，配置详情参考链接：[https://developer.hashicorp.com/consul/docs/agent\#common\-configuration\-settings](https://github.com)


更多选项可通过以下命令查看



```
# ./consul agent --help

```

服务器节点1（`192.168.88.133`）：



```
# mkdir -p /data/consul
# consul agent -server -bootstrap -datacenter=testDC -ui=true -data-dir=/data/consul -node=server1 -bind=192.168.88.133 -client=0.0.0.0 -serf-lan-port=8303 -serf-wan-port=8305 -dns-port=8601 -http-port=8603 -syslog -log-level=INFO 

```

服务器节点1（`192.168.88.134`）：



```
# mkdir -p /data/consul
# consul agent -server=true -datacenter=testDC -data-dir=/data/consul --node=server2 -bind=192.168.88.134 -client=0.0.0.0 -retry-join=192.168.88.133 -serf-lan-port=8303 -serf-wan-port=8305 -dns-port=8601 -http-port=8603 -syslog -log-level=INFO

```

说明：启动后加入集群`192.168.88.134:8301`


**浏览器访问验证**


说明：个人理解，Web UI访问访问地址(图示的`192.168.88.133`）为启动参数`-retry-join`配置的可用地址(可通过该地址成功加入集群)，端口`8603`为启动参数`-http-port`配置的端口。


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213232117678-875114802.png)


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213232138027-924461343.png)


参考链接：


[https://developer.hashicorp.com/consul/install](https://github.com):[Flowercloud 机场订阅加速](https://flowercloud6.com)


[https://developer.hashicorp.com/consul/docs/agent](https://github.com)


### 查看集群状态



```
# consul operator raft list-peers
Error getting peers: Failed to retrieve raft configuration: Get "http://127.0.0.1:8500/v1/operator/raft/configuration": dial tcp 127.0.0.1:8500: connect: connection refused
# consul operator raft list-peers -http-addr=192.168.88.133:8603
Node     ID                                    Address              State     Voter  RaftProtocol  Commit Index  Trails Leader By
server1  114b7fb3-0027-e179-19ff-3554d4f754f9  192.168.88.133:8300  follower  true   3             607           0 commits
server2  9e2a3388-bd5c-1a36-6a6c-479a8cd46012  192.168.88.134:8300  leader    true   3             607           -

```

### 查看成员状态



```
# consul members
Error retrieving members: Get "http://127.0.0.1:8500/v1/agent/members?segment=_all": dial tcp 127.0.0.1:8500: connect: connection refused
# consul members -http-addr=192.168.88.133:8603
Node     Address              Status  Type    Build   Protocol  DC      Partition  Segment
server1  192.168.88.133:8303  alive   server  1.19.2  2         testdc  default    
server2  192.168.88.134:8303  alive   server  1.19.2  2         testdc  default    

```

## 注册服务




| 方法 | 请求路径 | 内容格式 |
| --- | --- | --- |
| `PUT` | `/agent/service/register` | `application/json` |


### **查询参数**


* `replace-existing-checks` \- 如果请求中缺少健康检查，将从代理中删除对应服务的健康检查(官方原文：`Missing health checks from the request will be deleted from the agent`)。使用此参数可以幂等地注册服务及其检查，而无需手动注销检查。
* `ns` `(string: "")` \- 指定注册服务所属的名称空间。也可以[通过其它方法指定命名空间](https://github.com)。企业版才支持该参数。


### JSON请求体服务


对于JSON key命名，`/agent/service/register`端点支持驼峰和下划线命名方法。下划线命名是一种约定，包含两个或多个单词的键之间有下划线。下划线命名确保与代理配置文件格式的兼容性


* `Name` (`string:` ) \- 指定服务的逻辑名称。许多服务实例可能共享相同的逻辑服务名称。建议对服务定义名称使用有效的DNS标签。
* `ID` `(string: "")` \- 指定服务的唯一ID。每个服务代理的ID必须保持唯一。 如果未提供，则默认为`Name` 参数值。
* `Tags` `(array: nil)` \- 指定分配给服务的tag列表。 `Tags`方便在查询服务时根据tag进行筛选服务，并在Consul API中暴露。
* `Address` `(string: "")` \- 指定服务的地址。如果未提供，则代理的地址将在DNS查询期间用作服务的地址。
* `TaggedAddresses` `(map: nil)` \- 为服务实例指定显式LAN和WAN地址的映射。地址和端口都可以在映射值中指定。
* `Meta` `(map: nil)` \- 指定链接到服务实例的任意KV元数据
* `Namespace` `(string: "")` \- 指定注册服务所属名称空间。该参数优先于`ns`查询参数。 企业版才支持该参数。
* `Port` `(int: 0)` \- 指定服务端口
* `Kind` `(string: "")` \- 服务类型，默认为 `""`, 表示典型的Consule服务。可配置为以下值：
	+ `"connect-proxy"` 适用[service mesh](https://github.com) 代理，表示另一个服务。
	+ `"mesh-gateway"` 适用 [mesh gateway](https://github.com)实例
	+ `"terminating-gateway"` 适用 [terminating gateway](https://github.com)实例
	+ `"ingress-gateway"` 适用[ingress gateway](https://github.com)实例
* `Proxy` `(Proxy: nil)` \- 从1\.2\.3开始，指定服务网格代理实例的配置。这仅在`Kind`定义为代理或网关时有效。参阅[服务网格代理配置参考](https://github.com)了解全部细节
* `Connect` `(Connect: nil)` \- 指定[服务网格配置](https://github.com)。连接子系统提供Consul的服务网格功能。参阅 连接结构(Connect Structure)部分，以查看支持的字段。
* `Check` `(Check: nil)` \- 指定某种检查。参阅[检查文档](https://github.com)获取有关可接受字段的更多信息。如果没有为检查提供名称或id，则自动生成。要提供自定义id和/或名称，请设置`CheckID`和/或`Name`字段。
* `Checks` `(array: nil)` \- 指定一个检查列表。如果没有为检查提供名称或id，则自动生成。提供自定义id和/或名称，请设置`CheckID`和/或`Name`字段。自动生成的`Name`和`CheckID`取决于检查在数组中的位置，因此即使行为是确定的，建议所有检查要么通过将字段留空/省略让consul来设置`CheckID`，要么提供一个唯一的值。
* `EnableTagOverride` `(bool: false)` \- 指定为此服务的标记禁用反熵功能(anti\-entropy feature)。如果`EnableTagOverride`设置为`true`，则外部代理可以在[目录](https://github.com)中更新此服务并修改标签。此代理的后续本地同步操作将忽略更新的标签。例如，如果外部代理修改了此服务的标签和端口，并且`EnableTagOverride`设置为`true`，则在下一个同步周期后，服务的端口将恢复为原始值，但标签将保持更新后的值。作为反例，如果外部代理修改了此服务的标签和端口，并且`EnableTagOverride`设置为`false`，则在下一个同步周期后，服务的端口和标签将恢复为原始值，所有修改都将丢失。
* `Weights` `(Weights: nil)` \- 指定服务的权重。请参阅[服务配置参考](https://github.com)获取更多信息。默认值为`{"Passing": 1, "Warning": 1}`。权重仅适用于本地注册的服务。如果多个节点注册了相同的服务，则每个节点独立实现`EnableTagOverride`和其他服务配置项。更新在一个节点上注册的服务的标签不一定会更新在另一个节点中注册的同名服务上的相同标签。如果未指定`EnableTagOverride`，则默认值为`false`。参见[反熵同步](https://github.com)获取更多信息。
* `Partition` `(string: "")` \- 使用的分区。如果没有提供，则根据请求的ACL令牌推断分区，或默认为`default`分区。企业版才支持该参数。


### 连接结构(Connect Structure)


对于 `Connect` 字段,参数如下：


* `Native` `(bool: false)` \- 指定此服务是否[原生](https://github.com)支持[Consul服务网格](https://github.com)协议。如果为`true`，那么服务网格代理、DNS查询等将能够发现此服务。
* `SidecarService` `(ServiceDefinition: nil)` \-指定要注册的可选嵌套服务定义。请参阅[部署sidecar服务](https://github.com)获取更多信息。


### 注册服务请求体示例



```
{
  "ID": "redis1",
  "Name": "redis",
  "Tags": ["primary", "v1"],
  "Address": "127.0.0.1",
  "Port": 8000,
  "Meta": {
    "redis_version": "4.0"
  },
  "EnableTagOverride": false,
  "Check": {
    "DeregisterCriticalServiceAfter": "90m",
    "Args": ["/usr/local/bin/check_redis.py"],
    "Interval": "10s",
    "Timeout": "5s"
  },
  "Weights": {
    "Passing": 10,
    "Warning": 1
  }
}

```

### 注册服务请求示例



```
$ curl --request PUT \
    --data @payload.json \
    http://127.0.0.1:8500/v1/agent/service/register?replace-existing-checks=true

```

或者类似如下



```
$ curl -X PUT -d '                                      
{                                                    
    "id": "redis-node-exporter",                                   
    "name": "redis-node-exporter",   
    "Tags": ["primary"],
    "address": "192.168.88.131",                      
    "port": 9100,                       
    "EnableTagOverride": false
}' http://192.168.88.133:8603/v1/agent/service/register

```

说明：`127.0.0.1:8500`、`192.168.88.133:8603`同 Consul Web UI访问地址


注册效果：


*注册后可以在Services看到注册的服务*


![](https://img2024.cnblogs.com/blog/1569452/202412/1569452-20241213232419890-1773942214.png)


参考链接：[https://developer.hashicorp.com/consul/api\-docs/agent/service\#register\-service](https://github.com)


## 注销服务




| 方法 | 请求路径 | 内容格式 |
| --- | --- | --- |
| `PUT` | `/agent/service/deregister/:service_id` | `application/json` |


注意：


1、如果要注销的服务不存在，则不采取任何行动。


2、代理将负责在目录中注销服务。如果有相关的检查，也会取消注册。


### 路径参数


* `service_id` `(string: )` \- 指定要注销服务的ID


### 查询参数


* `ns` `(string: "")` \- 指定待注销服务所属的名称空间。也可以[通过其它方法指定命名空间](https://github.com)。企业版才支持该参数
* `partition` `(string: "")` 使用的分区。如果没有提供，则根据请求的ACL令牌推断分区，或默认为`default`分区。企业版才支持该参数。



```
$ curl --request PUT http://127.0.0.1:8500/v1/agent/service/deregister/my-service-id

```

示例：



```
$ curl --request PUT http://192.168.88.133:8603/v1/agent/service/deregister/redis-node-exporter

```

参考链接：[https://developer.hashicorp.com/consul/api\-docs/agent/service\#deregister\-service](https://github.com)


