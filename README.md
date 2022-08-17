# cilium-Document-zh-CN
cilium 官方文档中文版翻译项目，该项目的目标是翻译cilium官方文档，并收录社区中比较优秀的cilium相关文章，供大家学习探讨。

[Cilium Document](https://docs.cilium.io/en/stable/)

# 什么是cilium
Cilium是开源软件，用于透明地保护基于Linux容器管理平台（如Docker和Kubernetes）部署的应用程序服务之间的网络连接。
Cilium的基础是一种新的Linux内核技术，称为eBPF，它可以在Linux本身中动态插入强大的安全可见性和控制逻辑。由于eBPF在Linux内核内运行，因此可以应用和更新Cilium安全策略，而无需更改应用程序代码或容器配置。
# 什么是Hubble
hubble是一个完全分布式的网络和安全观测平台。它构建在Cilium和eBPF之上，以完全透明的方式深入了解服务的通信和行为以及网络基础设施。
依赖eBPF，所有可见性都是可编程的，并允许采用动态方法，最大限度地减少开销，同时根据用户的需要提供深度和详细的可见性。Hubble是专门为充分利用eBPF而设计的。
Hubble可以回答以下几个问题：
## 服务依赖关系和通信图

- 哪些服务正在相互通信？多久一次？服务依赖关系图是什么样子的？
- 正在进行哪些HTTP调用？服务消费或生产什么kafka主题？
## 网络监控和告警

- 是否有任何网络通信故障？为什么连接失败？是DNS吗？这是应用程序问题还是网络问题？第4层（TCP）或第7层（HTTP）上的通信是否中断？
- 在过去5分钟内，哪些服务遇到了DNS解析问题？哪些服务最近经历了TCP连接中断或连接超时？未响应的TCP SYN请求的速率是多少？

## 应用监控

- 特定服务或所有集群的5xx或4xx HTTP响应代码的速率是多少？
- 集群中，HTTP请求和响应之间的95%和99%延迟是多少？哪些服务表现最差？两个服务之间的延迟是多少？
## 安全可观测性

- 哪些服务的连接y因网络策略被阻止？从集群外部访问了哪些服务？哪些服务解析了特定的DNS名称？

# 为什么使用 clilium 和 hubble
eBPF以前所未有的粒度和效率实现了对系统和应用程序的可见性和控制。它以一种完全透明的方式执行，不需要应用程序以任何方式进行更改。eBPF同样具备处理现代容器工作负载以及更传统的工作负载（如虚拟机和标准Linux进程）的能力。
现代数据中心应用程序的开发已转向面向服务的架构，通常称为微服务，其中大型应用程序被拆分为小型独立服务，这些服务使用HTTP等轻量级协议通过API相互通信。微服务应用程序往往是高度动态的，随着应用程序的扩展/加入，以及在作为连续交付的一部分部署的滚动更新过程中，单个容器启动或销毁。
这种向高度动态微服务的转变在确保微服务之间的连接方面既是一个挑战，也是一个机遇。传统的Linux网络安全方法（例如iptables）过滤IP地址和TCP/UDP端口，但IP地址在动态微服务环境中经常发生变化。容器的高度不稳定的生命周期导致这些方法难以与应用程序并排扩展，因为负载平衡表和访问控制列表承载着几十万条需要不断更新的规则。出于安全目的，协议端口（例如HTTP流量的TCP端口80）不再用于区分应用程序流量，因为该端口用于跨服务的广泛消息。
另一个挑战是提供准确可观测性能力，因为传统系统使用IP地址作为主要身份识别工具，在微服务架构中，IP地址的使用寿命可能会大大缩短，只有几秒钟。
通过利用Linux eBPF，Cilium保留了透明插入安全可见性+强制的能力，但使用基于service/pod/cintainer的标识（与传统系统中的IP地址标识不同），并可以在应用层（例如HTTP）上进行过滤。因此，Cilium不仅可以通过将安全性与寻址解耦，使安全策略在高度动态的环境中应用变得简单，而且还可以通过在HTTP层操作，除了提供传统的第3层和第4层分段之外，提供更强的安全隔离。
eBPF的使用使Cilium能够以即使在大规模环境中也具有高度可扩展性。
# 功能概述
## 透明的保护API
clilium能够保护现代应用程序协议，如REST/HTTP、gRPC和Kafka。传统防火墙在第3层和第4层运行。在特定端口上运行的协议要么完全受信任，要么完全被阻止。Cilium能够过滤单个应用程序协议请求，例如：

- 允许使用`GET`方法和path `/public/*`的所有HTTP请求。拒绝所有其他请求。
- 允许`service1`在Kafka主题`topic1`上生产消息，`service2`在`topic1`上消费消息。拒绝所有其他kafka消息。

有关受支持协议的最新列表以及如何使用它的示例，请参阅我们文档中的[7层策略](http://docs.cilium.io/en/stable/policy/#layer-7)部分。
## 基于身份安全的服务到服务通信
现代分布式应用程序依赖于应用程序容器等技术来促进部署的灵活性和按需扩展。这导致在短时间内启动大量应用程序容器。典型的容器防火墙通过过滤源IP地址和目标端口来保护工作负载。这个概念要求在集群中任何地方启动容器时，都要对所有服务器上的防火墙进行操作。
为了避免这种限制规模的情况，Cilium将安全标识分配给共享相同安全策略的应用程序容器组。然后，身份与应用程序容器发出的所有网络数据包相关联，允许在接收节点验证身份。使用键值存储执行安全身份管理。
## 安全访问外部服务
基于标签的安全性是集群内部访问控制的首选工具。为了确保外部服务的安全访问，支持传统的基于CIDR的入口和出口安全策略。这允许将对应用程序容器的访问限制在特定的IP范围内。
## 简单的网络
一个简单的三层网络平面能够跨多个集群连接所有应用程序容器。通过使用主机作用域IP分配器，IP分配保持简单。这意味着每个主机都可以分配IP，而不需要主机之间进行任何协调。
支持以下多节点网络模型：

- Overlay： 跨主机基于封装的虚拟网络。目前，VXLAN和Geneve已被嵌入，但可以启用Linux支持的所有封装格式。何时使用此模式：此模式具有最低的基础架构和集成要求。它几乎适用于任何网络基础设施，因为唯一的要求是主机之间的IP连接，这通常已经给定。
- 原生路由：使用Linux主机的常规路由表。网络需要能够路由应用程序容器的IP地址。何时使用此模式：此模式适用于高级用户，需要了解底层网络基础设施。此模式适用于：
   - 原生ipv6网络
   - 结合云网络路由
   - 已经有路由守护进程
## 负载均衡
Cilium实现了应用程序容器之间和到外部服务的流量的分布式负载平衡，并且能够完全替换kube-proxy等组件。负载平衡在eBPF中使用哈希表实现，允许几乎无限的扩展。
对于南北向负载平衡，Cilium的eBPF实现针对最大性能进行了优化，可以连接到XDP（eXpress Data Path），并且如果不在源主机上执行负载平衡操作，则支持直接服务器返回（DSR）以及一致哈希。
对于东西向类型的负载平衡，Cilium在Linux内核的套接字层（例如在TCP连接时）执行高效的后端转换服务，这样可以避免较低层的NAT操作开销。
## 带宽管理
Cilium通过有效的基于EDT（Earliest Departure Time）的速率限制和eBPF来实现带宽管理，用于出口节点的容器流量。与带宽CNI插件中使用的HTB（Hierarchy Token Bucket）或TBF（Token Bucket Filter）等传统方法相比，这允许显著减少应用程序的传输尾部延迟，并避免在多队列NIC下锁定。
## 监控和故障排查
可观测性和故障排除的能力对于任何分布式系统的运行都至关重要。虽然我们常用`tcpdump`和`ping`之类的工具，虽然它们在我们心中总是有特殊的位置，但我们努力为故障排除提供更好的工具。这包括要提供的工具：

- 元数据事件监控：当数据包被丢弃时，该工具不仅报告数据包的源和目标IP，还提供发送方和接收方的完整标签信息以及许多其他信息。
- prometheus 指标： 通过prometheus导出关键指标，方便与现有仪表盘集成。
- Hubble： 专为cilium设计的可观察性平台。它提供服务依赖关系图、操作监视和警报，以及基于流日志的应用程序和安全可见性。
