# Kubernetes入门

## 一. Google Borg

[borg论文地址][https://storage.googleapis.com/pub-tools-public-publication-data/pdf/43438.pdf]

### 1. 特性

+ 物理资源利用率高
+ 服务器共享, 进程级别隔离
+ 应用高可用, 故障恢复时间短
+ 调度策略灵活
+ 应用接入和使用方便, 提供了完备的 Job 语言描述, 服务发现, 实时状态监控和诊断工具

### 2. 优势

+ 对外隐藏底层资源管理和调度,故障处理等
+ 实现应用的高可靠和高可用
+ 足够弹性,支持应用跑在成千上万的机器上

### 3. 基本概念

+ Workload
  + prod(production) : 在线任务, 长期运行,对延时敏感, 面向终端用户, 比如 Gmail, Google docs, Web Search 服务等
  + non-prod: 离线任务, 也成批处理任务(Batch), 比如分布式计算服务等
+ Cell(集群)
  + 一个 Cell 上跑一个集群管理系统 Borg
  + 通过定义 Cell 可以让 Borg 对服务器资源进行统一抽象, 作为用户就无需知道自己的应用跑在那台机器上,也不用关心资源分配,程序安装,依赖管理,健康检查及故障恢复
+ Job 和 Task
  + 用户以 Job 的形式提交应用部署请求. 一个 Job 包含一个或多个相同的 Task, 每个 Task 运行相同的应用程序, Task 数量就是应用的副本数.
  + 每个 Job 可以定义属性, 元信息和优先级, 优先级涉及到抢占式调度过程
+ Naming
  + Borg 的服务发现通过 BNS(Borg Name Service)来实现
  + 50.jfoo.ubar.cc.borg.google.com, 表示一个名为 cc 的 Cell 中, 由用户 uBar 部署的一个名为 JFoo 的 Job 下的第 50 个 Task

### 4. 架构

![image-20211115142712380](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211115142712380.png)

+ Borgmaster 主进程
  + 处理客户端 RPC 请求,比如创建 Job,查询 Job
  + 维护系统组件和服务的状态, 比如服务器, Task 等
  + 负责与 Borglet 通信
+ Scheduler 进程:
  + 调度策略
    + Worst Fit
    + Best Fit
    + Hybrid
  + 调度优化
    + Score caching: 当服务器或任务的状态未发生变更,或者变更很少时, 直接采用缓存数据, 避免重复计算
    + Equivalence classes: 调度同一 Job 下多个相同的 Task 只需计算一次.
    + Relaxed randomization: 引入随机性, 每次随机选择一些机器, 只要符合需求的服务器数量到达一定值时就可以停止计算, 无需每次对 Cell 中所有的服务器进行 feasibility checking.
+ Borglet:
  + 是部署在所有服务器上的 Agent, 负责接收 Borgmaster 进程的指令

### 5. 应用高可用

+ 被抢占的 non-prod 任务放回 pending queue, 从新等待调度
+ **多副本应用跨故障域部署**. 其中故障域有大有小,比如相同机器, 相同供电线路, 相同区域等一挂全挂等
+ 对于类似服务器/操作系统升级的维护操作,避免大量服务器同时进行
+ 支持**幂等性**, 支持客户端重复操作
+ `当服务器状态变为不可用时, 要控制重新调度任务的速率`. 因为 Borg 无法区分节点是故障还是短暂的网络分区, 如果是网络分区, 静静等待网络恢复,更利于保障服务可用性
+ 当某种"任务@服务器"的组合出现故障时, 下次重新调度应避免这种组合再次出现, 因为极大可能会再次出现相同故障.
+ **记录详细的内部信息, 便于故障排查和分析**
+ **保障应用高可用的关键性设计原则**:无论何种原因, 即使 Borgmaster 或者 Borglet 挂掉/失联, 都不能 kill 掉正在运行的服务(Task). 



### 6. Borg 高可用

+ Borgmaster 组件多副本设计
+ 采用一些简单的和底层(low-level)的工具来部署 Borg 系统实例, 避免引入过多外部依赖
+ 每个 Cell 的 Borg 均独立部署,避免不同 Borg 系统相互影响

### 7.资源利用率

+ 通过将 prod 和 no-prod 混合部署, 空闲时间, no-prod 任务可以充分利用系统资源, 繁忙时, prod 通过抢占的方式保证优先得到执行,合理利用资源
+ 98%的服务器实现了混合部署
+ 90%的服务器中跑过了 25 个 Task 和 4500 个线程
+ 在一个中等规模的 Cell 里, prod 好 no-prod 任务独立部署,比混合部署所需的服务器资源数量多出约 20%-30%.

### 8.Borg 调度原理

![image-20211115164131695](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211115164131695.png)

在 task 启动 300 秒后, 进行资源回收工作, 把保留资源设置为: 实际使用资源 + 安全资源, 每过几秒后重新计算一次

### 9. 隔离性

+ 安全性隔离
  + 早期使用 Chroot jail, 后期使用 Namespace
+ 性能隔离
  + 基于 Cgroup 的容器技术实现
  + prod 是延时敏感型的(latency-sensitive), 优先级高, non-prod优先级低
  + Borg 通过不同优先级之间的抢占式调度来优先保障在线任务的性能, 牺牲离线任务
  + Borg 将资源类型分成两类:
    + 可压榨的(compressible), CPU 是可压榨资源, 资源耗尽不会终止进程
    + 不可压榨的(non-compressible), 内存是不可压榨的,资源耗尽进程会被终止

## 二. Kubernetes

![image-20211115164955287](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211115164955287.png)

### 1. 概念:

+ Kubernetes 是 Google 开源的容器集群管理系统, 是 Google多年大规模容器管理技术 Borg 的开源版本

### 2. 主要功能:

+ 基于容器的应用部署/维护和滚动升级
+ 负载均衡和服务发现
+ 跨机器和跨地区的集群调度
+ 自动伸缩
+ 无状态服务和有状态服务
+ 插件机制保证扩展性

### 3. 核心对象

+ Node: 计算节点的抽象, 用来描述计算节点的资源抽象/健康状态等
+ Namespace: 资源隔离的基本单位, 可以简单理解为文件系统中的目录结构
+ Pod: 用来描述应用实例, 包括镜像地址/资源需求等.Kubernetes 中最核心的对象,是打通应用和基础架构的秘密武器
+ Service: 服务如何将应用发布成服务, 本质是负载均衡和域名服务的声明

### 4. 架构

![image-20211115173802152](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211115173802152.png)

### 5. 主要组件

![image-20211115174514863](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211115174514863.png)

### 6. Kubernetes 的主节点(Master Node)

+ API Server
  + 是 Kubernetes 控制面板中唯一带有用户可访问 Api 以及用户可交互的组件
  + API server 会暴露一个 Restful 的 Kubernetes API 并使用 json 格式的清单文件(manifest files)
+ Cluster Data Store
  + 使用 etcd 作为数据存储
  + 是一个高可用的分布式键值存储
+ Controller Manager
  + 运行着所有处理集群日常任务的控制器
  + 包括节点控制器(node), 副本控制器(replicate), 端点(endpoint)控制器以及服务账户等
+ Scheduler
  + 监听新建的 pods(一组或一个容器)并将其分配给 node
+ Kubelet
  + 负责调度对应Node的 Pod 的生命周期管理
  + 执行任务并将 Pod 状态报告给Master 节点
  + 通过 CRI管理容器
  + 定期执行被请求的容器的健康探测程序
+ Kube-proxy
  + 负责 Node 的网络,在主机上维护网络规则并执行连接转发
  + 负责对正在服务的 pods 进行负载均衡

### 7. etcd

![image-20211115182731856](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211115182731856.png)

**概念**: 基于 Raft 算法开发的 KV 存储, 可用于服务发现/配置共享/一致性保障如(数据库选主, 分布式锁等)

**功能** :

+ 基本的 key-value 存储
+ 监听机制, 长连接监听,代替轮询
+ key 的过期及续约机制, 用于监控和服务发现
+ 原子 CAS 和 CAD, 用于分布式锁和 leader 选举

### 8. APIServer

![image-20211115214600985](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211115214600985.png)

+ 提供集群管理的 Rest API 接口
  + 认证 Authentication
  + 授权 Authorization
  + 准入 Admission (Mutating & Valiating)
+ 提供其他模块之间的数据交互和通讯(其他模块通过 APIServer 查询或修改数据,只有 APIServer 才直接操作 etcd)
+ APIServer 提供 etcd 数据缓存以减少集群对 etcd 的访问

![image-20211115214815859](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211115214815859.png)

### 9. Controller Manager

+ 是集群的大脑,确保整个集群动起来
+ 确保 Kubernetes 遵循声明式系统规范,确保系统的真实状态(actual state)与用户定义的期望状态(Desired State)一致
+ 是多个控制器的组合, 每个 Controller 事实上都是一个 control loop, 负责监听其管控的对象, 当对象发生变更时完成配置
+ Controller 配置失败通畅会触发自动重试, 整个集群会在控制器不断重试的机制下确保最终一致性

+ controller 内部机制

  ![image-20211115220321519](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211115220321519.png)

​	![image-20211115220626717](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211115220626717.png)

![image-20211115220912698](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211115220912698.png)

### 10. Scheduler

![image-20211116151354275](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211116151354275.png)

+ 特殊的 Controller, 工作原理与其他控制器无差别

+ 监控当前集群所有未调度的 Pod, 并获取当前集群所有节点的健康状况和资源使用状况,为待调度 Pod 选择计算最佳计算节点,完成调度
+ 调度阶段
  + Predict: 过滤不满足业务需求的 node
  + Priority: 按既定要素将满足调度需求的节点平分,选择最佳节点
  + Bind: 将计算节点与 Pod 绑定,完成调度

### 11.Kubelet

![image-20211116152533271](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211116152533271.png)

+ Kubernetes 的初始化系统(init system)
+ 从不同源获取 Pod 清单, 并按需求启停 Pod 的核心组件
  + Pod 清单可以从本地文件目录, 给定的 HTTPServer 或者 Kube-APIServer 等源获取
  + Kubelet 将 Runtime 抽象为 CRI, CNI, CSI
+ 负责汇报当前节点的资源信息和健康状态
+ 负责 Pod 的健康检查和状态汇报

### 12. Kube-Proxy

![image-20211116153028972](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211116153028972.png)

+ 监控集群中用户发布的服务, 并完成负载均衡
+ 每个节点的 Kube-proxy 都会配置相同的负载均衡策略, 似的整个集群的服务发现建立在分布式负载均衡器上, 服务调用无需经过额外的网络跳转(Network Hop)
+ 负载均衡配置基于不同插件实现:
  + userspace
  + 操作系统网络协议栈不同的 Hooks 点和插件
    + iptables
    + ipvs

### 13. 常用插件 add-ons

+ kube-dns: 负责整个集群提供 DNS 服务
+ Ingress Controller: 为服务提供外网入口
+ MetricsServer: 提供资源监控
+ Dashboard: 提供 GUI
+ Fluentd-Elasticsearch: 提供集群日志采集/存储/与查询