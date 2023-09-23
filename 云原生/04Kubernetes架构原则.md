### 什么是云计算

![截屏2023-09-21 21.00.51](assets/1695330051.png)

### 云计算平台的分类

以 Openstack 为典型的虚拟化平台

- 虚拟机构建和业务代码部署分离。

- 可变的基础架构使后续维护风险变大。

 以谷歌 borg 为典型的基于进程的作业调度平台

- 技术的迭代引发 borg 的换代需求。
- 早期的隔离依靠 chroot jail 实现，一些不合理的设计需要在新产品中改进。
  - 对象之间的强依赖 job 和 task 是强包含关系，不利于重组。
  - 所有容器共享 IP，会导致端口冲突，隔离困难等问题。 
  - 为超级用户添加复杂逻辑导致系统过于复杂。

### Kubernetes 架构基础

#### Borg

![截屏2023-09-21 21.01.52](assets/1695330112.png)

#### Google Borg 简介

去读Borg官方文档

特性

- 物理资源利用率高。
- 服务器共享，在进程级别做隔离。
- 应用高可用，故障恢复时间短。
- 调度策略灵活。
- 应用接入和使用方便，提供了完备的 Job 描述语言，服务发现，实时状态监控和诊断工具。

优势

- 对外隐藏底层资源管理和调度、故障处理等。
- 实现应用的高可靠和高可用。
- 足够弹性，支持应用跑在成千上万的机器上。

#### 基本概念

Workload

- prod:在线任务，长 期运行、对延时敏感、 面向终端用户等，比如 Gmail, Google Docs, Web Search 服务等。
- non-prod:离线任 务，也称为批处理任务 (Batch)，比如一些 分布式计算服务等。

Cell

- 一个 Cell 上跑一个集 群管理系统 Borg。
- 通过定义 Cell 可以让Borg 对服务器资源进 行统一抽象，作为用户就无需知道自己的应用
  跑在哪台机器上，也不用关心资源分配、程序安装、依赖管理、健康检查及故障恢复等。

Job 和 Task

- 用户以 Job 的形式提 交应用部署请求。一个 Job 包含一个或多个相 同的 Task，每个 Task 运行相同的应用程序， Task 数量就是应用的 副本数。
- 每个 Job 可以定义属 性、元信息和优先级， 优先级涉及到抢占式调 度过程。

Naming

- Borg 的服务发现通过 BNS ( Borg Name Service)来实现。
- 50.jfoo.ubar.cc.borg .google.com 可 表 示 在一个名为 cc 的 Cell 中由用户 uBar 部署的 一个名为 jFoo 的 Job 下的第50个 Task。

#### Borg 架构

Borgmaster 主进程:

- 处理客户端 RPC 请求，比如创建 Job，查询 Job 等。
- 维护系统组件和服务的状态，比如服务器、Task 等。
- 负责与 Borglet 通信。

- Scheduler 进程:
- 调度策略
  - Worst Fit
  - Best Fit
  - Hybrid 
- 调度优化
  - Score caching: 当服务器或者任务的状态未发生变更或者变更很少时，直接采用缓存数据，避免重复计算。
  - Equivalence classes: 调度同一 Job 下多个相同的 Task 只需计算一次。
  - Relaxed randomization: 引入一些随机性，即每次随机选择一些机器，只要符合需求的服务器数量达到一定值时，就可 以停止计算，无需每次对 Cell 中所有服务器进行feasibility checking。

Borglet:

​	Borglet 是部署在所有服务器上的 Agent，负责接收 Borgmaster 进程的指令。

![截屏2023-09-21 21.05.55](assets/1695330355.png)

#### 应用高可用

- 被抢占的 non-prod 任务放回 pending queue，等待重新调度。
- 多副本应用跨故障域部署。所谓故障域有大有小，比如相同机器、相同机架或相同电源插座等，一挂全挂。
- 对于类似服务器或操作系统升级的维护操作，避免大量服务器同时进行。
- 支持**幂等性**，支持客户端重复操作。
- 当服务器状态变为不可用时，要控制重新调度任务的速率。因为 Borg 无法区分是节点故障还是出现了短暂的 网络分区，如果是后者，静静地等待网络恢复更利于保障服务可用性。
- 当某种“任务 @ 服务器”的组合出现故障时，下次重新调度时需避免这种组合再次出现，因为极大可能会再 次出现相同故障。
- 记录详细的内部信息，便于故障排查和分析。
- 保障应用高可用的关键性设计原则:无论何种原因，即使 Borgmaster 或者 Borglet 挂掉、失联，都不能杀 掉正在运行的服务(Task)。

#### Borg 系统自身高可用

- Borgmaster 组件多副本设计。
- 采用一些简单的和底层(low-level)的工具来部署 Borg 系统实例，避免引入过多的外部依赖。 
- 每个 Cell 的 Borg 均独立部署，避免不同 Borg 系统相互影响。

#### 资源利用率

- 通过将在线任务(prod)和离线任务(non-prod，Batch)混合部署，空闲时，离线任务可以充分利用计 算资源;繁忙时，在线任务通过抢占的方式保证优先得到执行，合理地利用资源。
- 98% 的服务器实现了混部。
- 90% 的服务器中跑了超过 25 个 Task 和 4500 个线程。
- 在一个中等规模的 Cell 里，在线任务和离线任务独立部署比混合部署所需的服务器数量多出约 20%-30%。 可以简单算一笔账，Google 的服务器数量在千万级别，按 20% 算也是百万级别，大概能省下的服务器采 购费用就是百亿级别了，这还不包括省下的机房等基础设施和电费等费用。

#### Brog 调度原理

![截屏2023-09-21 21.07.09](assets/1695330429.png)

#### 隔离性

安全性隔离:

- 早期采用 Chroot jail，后期版本基于 Namespace。 

性能隔离:

- 采用基于 Cgroup 的容器技术实现。

- 在线任务(prod)是延时敏感(latency-sensitive)型的，优先级高，而离线任务(non-prod，

  Batch)优先级低。

- Borg 通过不同优先级之间的抢占式调度来优先保障在线任务的性能，牺牲离线任务。

- Borg 将资源类型分成两类:

  - 可压榨的(compressible)，CPU 是可压榨资源，资源耗尽不会终止进程;
  - 不可压榨的(non-compressible)，内存是不可压榨资源，资源耗尽进程会被终止。

#### 什么是 Kubernetes(K8s)

有状态应用（数据库）如果做自动化运维，很需要成本。

 Kubernetes 是谷歌开源的容器集群管理系统，是 Google 多年大规模容器管理技术 Borg 的开源版本，主要功能包括: 

- 基于容器的应用部署、维护和滚动升级;
- 负载均衡和服务发现;
- 跨机器和跨地区的集群调度;
- 自动伸缩;
- 无状态服务和有状态服务; 
- 插件机制保证扩展性。

![截屏2023-09-21 21.12.26](assets/1695330746.png)

#### 命令式( Imperative)vs 声明式( Declarative)

命令式系统关注 “如何做”

在软件工程领域，命令式系统是写出解决某个问题、 完成某个任务或者达到某个目标的明确步骤。此方法 明确写出系统应该执行某指令，并且期待系统返回期 望结果。

声明式系统关注“做什么”

在软件工程领域，声明式系统指程序代码描述系统应该 做什么而不是怎么做。仅限于描述要达到什么目的，如 何达到目的交给系统。

#### 声明式(Declaritive)系统规范

命令式:

- 我要你做什么，怎么做，请严格按照我说的做。

声明式:

- 我需要你帮我做点事，但是我只告诉你我需要你做什么，不是你应该怎么做。
- 直接声明:我直接告诉你我需要什么。
- 间接声明:我不直接告诉你我的需求，我会把我的需求放在特定的地方，请在方便的时候拿出来处理。

幂等性:

- 状态固定，每次我要你做事，请给我返回相同结果。

面向对象的:

- 把一切抽象成对象。

#### Kubernetes:声明式系统

Kubernetes 的所有管理能力构建在对象抽象的基础上，核心对象包括:

• Node:计算节点的抽象，用来描述计算节点的资源抽象、健康状态等。

• Namespace:资源隔离的基本单位，可以简单理解为文件系统中的目录结构。

• Pod:用来�### 什么是云计算

![截屏2023-09-21 21.00.51](assets/1695330051.png)

### 云计算平台的分类

以 Openstack 为典型的虚拟化平台

- 虚拟机构建和业务代码部署分离。

- 可变的基础架构使后续维护风险变大。

 以谷歌 borg 为典型的基于进程的作业调度平台

- 技术的迭代引发 borg 的换代需求。
- 早期的隔离依靠 chroot jail 实现，一些不合理的设计需要在新产品中改进。
  - 对象之间的强依赖 job 和 task 是强包含关系，不利于重组。
  - 所有容器共享 IP，会导致端口冲突，隔离困难等问题。 
  - 为超级用户添加复杂逻辑导致系统过于复杂。

### Kubernetes 架构基础

#### Borg

![截屏2023-09-21 21.01.52](assets/1695330112.png)

#### Google Borg 简介

去读Borg官方文档

特性

- 物理资源利用率高。
- 服务器共享，在进程级别做隔离。
- 应用高可用，故障恢复时间短。
- 调度策略灵活。
- 应用接入和使用方便，提供了完备的 Job 描述语言，服务发现，实时状态监控和诊断工具。

优势

- 对外隐藏底层资源管理和调度、故障处理等。
- 实现应用的高可靠和高可用。
- 足够弹性，支持应用跑在成千上万的机器上。

#### 基本概念

Workload

- prod:在线任务，长 期运行、对延时敏感、 面向终端用户等，比如 Gmail, Google Docs, Web Search 服务等。
- non-prod:离线任 务，也称为批处理任务 (Batch)，比如一些 分布式计算服务等。

Cell

- 一个 Cell 上跑一个集 群管理系统 Borg。
- 通过定义 Cell 可以让Borg 对服务器资源进 行统一抽象，作为用户就无需知道自己的应用
  跑在哪台机器上，也不用关心资源分配、程序安装、依赖管理、健康检查及故障恢复等。

Job 和 Task

- 用户以 Job 的形式提 交应用部署请求。一个 Job 包含一个或多个相 同的 Task，每个 Task 运行相同的应用程序， Task 数量就是应用的 副本数。
- 每个 Job 可以定义属 性、元信息和优先级， 优先级涉及到抢占式调 度过程。

Naming

- Borg 的服务发现通过 BNS ( Borg Name Service)来实现。
- 50.jfoo.ubar.cc.borg .google.com 可 表 示 在一个名为 cc 的 Cell 中由用户 uBar 部署的 一个名为 jFoo 的 Job 下的第50个 Task。

#### Borg 架构

Borgmaster 主进程:

- 处理客户端 RPC 请求，比如创建 Job，查询 Job 等。
- 维护系统组件和服务的状态，比如服务器、Task 等。
- 负责与 Borglet 通信。

- Scheduler 进程:
- 调度策略
  - Worst Fit
  - Best Fit
  - Hybrid 
- 调度优化
  - Score caching: 当服务器或者任务的状态未发生变更或者变更很少时，直接采用缓存数据，避免重复计算。
  - Equivalence classes: 调度同一 Job 下多个相同的 Task 只需计算一次。
  - Relaxed randomization: 引入一些随机性，即每次随机选择一些机器，只要符合需求的服务器数量达到一定值时，就可 以停止计算，无需每次对 Cell 中所有服务器进行feasibility checking。

Borglet:

​	Borglet 是部署在所有服务器上的 Agent，负责接收 Borgmaster 进程的指令。

![截屏2023-09-21 21.05.55](assets/1695330355.png)

#### 应用高可用

- 被抢占的 non-prod 任务放回 pending queue，等待重新调度。
- 多副本应用跨故障域部署。所谓故障域有大有小，比如相同机器、相同机架或相同电源插座等，一挂全挂。
- 对于类似服务器或操作系统升级的维护操作，避免大量服务器同时进行。
- 支持**幂等性**，支持客户端重复操作。
- 当服务器状态变为不可用时，要控制重新调度任务的速率。因为 Borg 无法区分是节点故障还是出现了短暂的 网络分区，如果是后者，静静地等待网络恢复更利于保障服务可用性。
- 当某种“任务 @ 服务器”的组合出现故障时，下次重新调度时需避免这种组合再次出现，因为极大可能会再 次出�)。

#### 控制器的工作流程

![截屏2023-09-21 21.19.46](assets/1695331186.png)

#### Informer 的内部机制

![截屏2023-09-21 21.20.24](assets/1695331224.png)

#### 控制器的协同工作原理

![截屏2023-09-21 21.20.36](assets/1695331236.png)

#### Scheduler

特殊的 Controller，工作原理与其他控制器无差别。

Scheduler 的特殊职责在于监控当前集群所有未调度的 Pod，并且获取当前集群所有节点的健康状况和资源 使用情况，为待调度 Pod 选择最佳计算节点，完成调度。

调度阶段分为:

- Predict:过滤不能满足业务需求的节点，如资源不足、端口冲突等。 
- Priority:按既定要素将满足调度需求的节点评分，选择最佳节点。
- Bind:将计算节点与 Pod 绑定，完成调度。

![截屏2023-09-21 22.11.49](assets/1695334309.png)

#### Kubelet

Kubernetes 的初始化系统(init system)

- 从不同源获取 Pod 清单，并按需求启停 Pod 的核心组件:

  - Pod 清单可从本地文件目录，给定的 HTTPServer 或 Kube-APIServer 等源头获取;

  - Kubelet 将运行时，网络和存储抽象成了 CRI，CNI，CSI。 

- 负责汇报当前节点的资源信息和健康状态;
- 负责 Pod 的健康检查和状态汇报。

![截屏2023-09-21 22.12.52](assets/1695334372.png)

#### Kube-Proxy

- 监控集群中用户发布的服务，并完成负载均衡配置。
- 每个节点的 Kube-Proxy 都会配置相同的负载均衡策略，使得整个集群的服务发现建立在分布式负载 均衡器之上，服务调用无需经过额外的网络跳转(Network Hop)。
- 负载均衡配置基于不同插件实现:
  - userspace。
  - 操作系统网络协议栈不同的 Hooks 点和插件: 
    - iptables;
    - ipvs。

![截屏2023-09-21 22.13.52](assets/1695334432.png)

#### 推荐的 Add-ons

- kube-dns:负责为整个集群提供 DNS 服务;
- Ingress Controller:为服务提供外网入口;
- MetricsServer:提供资源监控;
- Dashboard:提供 GUI;
- Fluentd-Elasticsearch:提供集群日志采集、存储与查询。

### 了解 kubectl

#### Kubectl 命令和 kubeconfig

- kubectl是一个Kubernetes的命令行工具，它允许 Kubernetes 用户以命令行的方式与 Kubernetes 交 互，其默认读取配置文件 ~/.kube/config。
- kubectl会将接收到的用户请求转化为rest调用以 rest client 的形式与 apiserver 通讯。
- apiserver的地址，用户信息等配置在kubeconfig。

```
apiVersion: v1 clusters:
- cluster:
certificate-authority-data: REDACTED
server: https://127.0.0.1:54729 name: kind-kind
contexts: - context:
cluster: kind-kind
user: kind-kind name: kind-kind
current-context: kind-kind kind:Config
users:
- name: kind-kind
user:
client-certificate-data: REDACTED client-key-data: REDACTED
```

#### kubectl 常用命令

kubectl get po –oyaml -w

kubectl    可查看对象。

-oyaml     输出详细信息为 yaml 格式。

-w watch   该对象的后续变化。

-owide      以详细列表的格式查看对象。

#### Kubectl describe

kubectl describe 展示资源的详细信息和相关 Event。

#### kubectl exec

kubectl exec 提供进入运行容器的通道，可以进入容器进行 debug 操作。

#### kubectl logs

Kubectl logs 可查看 pod 的标准输入(stdout, stderr)，与 tail 用法类似。

### 深入理解 Kubernetes

#### 云计算的传统分类

![截屏2023-09-21 22.16.12](assets/1695334572.png)

#### Kubernetes 生态系统

![截屏2023-09-21 22.16.32](assets/1695334592.png)

#### Kubernetes 设计理念

可扩展性

- 基于CRD的扩展
- 插件化的生态系统

高可用

- 基于 replicaset，statefulset 的 应用高可用
- Kubernetes 组件本身高可用

可移植性

- 多种 host Os 选择
- 多种基础架构的选择
- 多云和混合云

安全

- 基于 TLS 提供服务
- Serviceaccount 和 user
- 基于 Namespace 的隔离
- secret
- Taints，psp， networkpolicy
�相同故障。
- 记录详细的内部信息，便于故障排查和分析。
- 保障应用高可用的关键性设计原则:无论何种原因，即使 Borgmaster 或者 Borglet 挂掉、失联，都不能杀 掉正在运行的服务(Task)。

#### Borg 系统自身高可用

- Borgmaster 组件多副本设计。
- 采用一些简单的和底层(low-level)的工具来部署 Borg 系统实例，避免引入过多的外部依赖。 
- 每个 Cell 的 Borg 均独立部署，避免不同 Borg 系统相互影响。

#### 资源利用率

- 通过将在线任务(prod)和离线任务(non-prod，Batch)混合部署，空闲时，离线任务可以充分利用计 算资源;繁忙时，在线任务通过抢占的方式保证优先得到执行，合理地利用资源。
- 98% 的服务器实现了混部。
- 90% 的服务器中跑了超过 25 个 Task 和 4500 个线程。
- 在一个中等规模的 Cell 里，在线任务和离线任务独立部署比混合部署所需的服务器数量多出约 20%-30%。 可以简单算一笔账，Google 的服务器数量在千万级别，按 20% 算也是百万级别，大概能省下的服务器采 购费用就是百亿级别了，这还不包括省下的机房等基础设施和电费等费用。

#### Brog 调度原理

![截屏2023-09-21 21.07.09](assets/1695330429.png)

#### 隔离性

安全性隔离:

- 早期采用 Chroot jail，后期版本基于 Namespace。 

性能隔离:

- 采用基于 Cgroup 的容器技术实现。

- 在线任务(prod)是延时敏感(latency-sensitive)型的，优先级高，而离线任务(non-prod，

  Batch)优先级低。

- Borg 通过不同优先级之间的抢占式调度来优先保障在线任务的性能，牺牲离线任务。

- Borg 将资源类型分成两类:

  - 可压榨的(compressible)，CPU 是可压榨资源，资源耗尽不会终止进程;
  - 不可压榨的(non-compressible)，内存是不可压榨资源，资源耗尽进程会被终止。

#### 什么是 Kubernetes(K8s)

有状态应用（数据库）如果做自动化运维，很需要成本。

 Kubernetes 是谷歌开源的容器集群管理系统，是 Google 多年大规模容器管理技术 Borg 的开源版本，主要功能包括: 

- 基于容器的应用部署、维护和滚动升级;
- 负载均衡和服务发现;
- 跨机器和跨地区的集群调度;
- 自动伸缩;
- 无状态服务和有状态服务; 
- 插件机制保证扩展性。

![截屏2023-09-21 21.12.26](assets/1695330746.png)

#### 命令式( Imperative)vs 声明式( Declarative)

命令式系统关注 “如何做”

在软件工程领域，命令式系统是写出解决某个问题、 完成某个任务或者达到某个目标的明确步骤。此方法 明确写出系统应该执行某指令，并且期待系统返回期 望结果。

声明式系统关注“做什么”

在软件工程领域，声明式系统指程序代码描述系统应该 做什么而不是怎么做。仅限于描述要达到什么目的，如 何达到目的交给系统。

#### 声明式(Declaritive)系统规范

命令式:

- 我要你做什么，怎么做，请严格按照我说的做。

声明式:

- 我需要你帮我做点事，但是我只告诉你我需要你做什么，不是你应该怎么做。
- 直接声明:我直接告诉你我需要什么。
- 间接声明:我不直接告诉你我的需求，我会把我的需求放在特定的地方，请在方便的时候拿出来处理。

幂等性:

- 状态固定，每次我要你做事，请给我返回相同结果。

面向对象的:

- 把一切抽象成对象。

#### Kubernetes:声明式系统

Kubernetes 的所有管理能力构建在对象抽象的基础上，核心对象包括:

• Node:计算节点的抽象，用来描述计算节点的资源抽象、健康状态等。

• Namespace:资源隔离的基本单位，可以简单理解为文件系统中的目录结构。

• Pod:用来�)。

#### 控制器的工作流程

![截屏2023-09-21 21.19.46](assets/1695331186.png)

#### Informer 的内部机制

![截屏2023-09-21 21.20.24](assets/1695331224.png)

#### 控如网络分区或服务重启等);

- 所有组件都应该在内存中保持所需要的状态，APIServer 将状态写入 etcd 存储，而其 他组件则通过 APIServer 更新并监听所有的变化;

- 优先使用事件监听而不是轮询。

#### 引导(Bootstrapping)原则

- Self-hosting 是目标。

- 减少依赖，特别是稳态运行的依赖。 

- 通过分层的原则管理依赖。

- 循环依赖问题的原则:

  - 同时还接受其他方式的数据输入(比如本地文件等)，这样在其他服务不可 用时还可以手动配置引导服务;

  - 状态应该是可恢复或可重新发现的;

  - 支持简单的启动临时实例来创建稳态运行所需要的状态，使用分布式锁或文件锁等来协调不同状态的切换(通常称为 pivoting 技术);

  - 自动重启异常退出的服务，比如副本或者进程管理器等。

#### 课后练习 4.1

用 Kubeadm 安装 Kubernetes 集群。

### 核心技术概念和 API 对象

API 对象是 Kubernetes 集群中的管理操作单元。

Kubernetes 集群系统每支持一项新功能，引入一项新技术，一定会新引入对应的 API 对象，支持对该功能的管理操 作。

每个 API 对象都有四大类属性: 

- TypeMeta
- MetaData
- Spec
- Status

#### TypeMeta

Kubernetes对象的最基本定义，它通过引入GKV(Group，Kind，Version)模型定义了一个对象的类型。

1. Group

Kubernetes 定义了非常多的对象，如何将这些对象进行归类是一门学问，将对象依据其功能范围归入不同的分组， 比如把支撑最基本功能的对象归入 core 组，把与应用部署有关的对象归入 apps 组，会使这些对象的可维护性和可 理解性更高。

2. Kind

定义一个对象的基本类型，比如 Node、Pod、Deployment 等。

3. Version

社区每个季度会推出一个 Kubernetes 版本，随着 Kubernetes 版本的演进，对象从创建之初到能够完全生产化就 绪的版本是不断变化的。与软件版本类似，通常社区提出一个模型定义以后，随着该对象不断成熟，其版本可能会 从 v1alpha1 到 v1alpha2，或者到 v1beta1，最终变成生产就绪版本 v1。

#### Metadata

Metadata 中有两个最重要的属性:Namespace和Name，分别定义了对象的Namespace 归属及名字，这两个属性唯一定义了某个对象实例。

1. Label

顾名思义就是给对象打标签，一个对象可以有任意对标签，其存在形式是键值对。 Label 定义了对象的可识别属性，Kubernetes API 支持以 Label 作为过滤条件 查询对象。

2. Annotation

Annotation 与 Label 一样用键值对来定义，但 Annotation 是作为属性扩展， 更多面向于系统管理员和开发人员，因此需要像其他属性一样做合理归类。

#### Label

- Label是识别Kubernetes对象的标签，以key/value的方式附加到对象上。

- key最长不能超过63字节，value可以为空，也可以是不超过253字节的字符串。

- Label不提供唯一性，并且实际上经常是很多对象(如Pods)都使用相同的label来标志具 体的应用。

- Label定义好后其他对象可以使用LabelSelector来选择一组相同label的对象

- LabelSelector支持以下几种方式:

  - 等式，如 app=nginx 和 env!=production;

  - 集合，如 env in (production, qa);

  - 多个 label(它们之间是 AND 关系)，如 app=nginx,env=test。

![截屏2023-09-21 22.28.06](assets/1695335286.png)

#### Annotations

• Annotations 是 key/value 形式附加于对象的注解。

• 不同于 Labels 用于标志和选择对象，Annotations 则是用来记录一些附加信息，用来辅助应用部署、安 全策略以及调度策略等。

• 比如 deployment 使用 annotations 来记录 rolling update 的状态。

#### Metadata

3. Finalizer

Finalizer 本质上是一个资源锁，Kubernetes 在接收某对象的删除请求时，会检 查 Finalizer 是否为空，如果不为空则只对其做逻辑删除，即只会更新对象中的 metada制器的协同工作原理

![截屏2023-09-21 21.20.36](assets/1695331236.png)

#### Scheduler

特殊的 Controller，工作原理与其他控制器无差别。

Scheduler 的特殊职责在于监控当前集群所有未调度的 Pod，并且获取当前集群所有节点的健康状况和资源 使用情况，为待调度 Pod 选择最佳计算节点，完成调度。

调度阶段分为:

- Predict:过滤不能满足业务需求的节点，如资源不足、端口冲突等。 
- Priority:按既定要素将满足调度需求的节点评分，选择最佳节点。
- Bind:将计算节点与 Pod 绑定，完成调度。

![截屏2023-09-21 22.11.49](assets/1695334309.png)

#### Kubelet

Kubernetes 的初始化系统(init system)

- 从不同源获取 Pod 清单，并按需求启停 Pod 的核心组件:

  - Pod 清单可从本地文件目录，给定的 HTTPServer 或 Kube-APIServer 等源头获取;

  - Kubelet 将运行时，网络和存储抽象成了 CRI，CNI，CSI。 

- 负责汇报当前节点的资源信息和健康状态;
- 负责 Pod 的健康检查和状态汇报。

![截屏2023-09-21 22.12.52](assets/1695334372.png)

#### Kube-Proxy

- 监控集群中用户发布的服务，并完成负载均衡配置。
- 每个节点的 Kube-Proxy 都会配置相同的负载均衡策略，使得整个集群的服务发现建立在分布式负载 均衡器之上，服务调用无需经过额外的网络跳转(Network Hop)。
- 负载均衡配置基于不同插件实现:
  - userspace。
  - 操作系统网络协议栈不同的 Hooks 点和插件: 
    - iptables;
    - ipvs。

![截屏2023-09-21 22.13.52](assets/1695334432.png)

#### 推荐的 Add-ons

- kube-dns:负责为整个集群提供 DNS 服务;
- Ingress Controller:为服务提供外网入口;
- MetricsServer:提供资源监控;
- Dashboard:提供 GUI;
- Fluentd-Elasticsearch:提供集群日志采集、存储与查询。

### 了解 kubectl

#### Kubectl 命令和 kubeconfig

- kubectl是一个Kubernetes的命令行工具，它允许 Kubernetes 用户以命令行的方式与 Kubernetes 交 互，其默认读取配置文件 ~/.kube/config。
- kubectl会将接收到的用户请求转化为rest调用以 rest client 的形式与 apiserver 通讯。
- apiserver的地址，用户信息等配置在kubeconfig。

```
apiVersion: v1 clusters:
- cluster:
certificate-authority-data: REDACTED
server: https://127.0.0.1:54729 name: kind-kind
contexts: - context:
cluster: kind-kind
user: kind-kind name: kind-kind
current-context: kind-kind kind:Config
users:
- name: kind-kind
user:
client-certificate-data: REDACTED client-key-data: REDACTED
```

#### kubectl 常用命令

kubectl get po –oyaml -w

kubectl    可查看对象。

-oyaml     输出详细信息为 yaml 格式。

-w watch   该对象的后续变化。

-owide      以详细列表的格式查看对象。

#### Kubectl describe

kubectl describe 展示资源的详细信息和相关 Event。

#### kubectl exec

kubectl exec 提供进入运行容器的通道，可以进入容器进行 debug 操作。

#### kubectl logs

Kubectl logs 可查看 pod 的标准输入(stdout, stderr)，与 tail 用法类似。

### 深入理解 Kubernetes

#### 云计算的传统分类

![截屏2023-09-21 22.16.12](assets/1695334572.png)

#### Kubernetes 生态系统

![截屏2023-09-21 22.16.32](assets/1695334592.png)

#### Kubernetes 设计理念

可扩展性

- 基于CRD的扩展
- 插件化的生态系统

高可用

- 基于 replicaset，statefulset 的 应用高可用
- Kubernetes 组件本身高可用

可移植性

- 多种 host Os 选择
- 多种基础架构的选择
- 多云和混合云

安全

- 基于 TLS 提供服务
- Serviceaccount 和 user
- 基于 Namespace 的隔离
- secret
- Taints，psp， networkpolicy
如网络分区或服务重启等);

- 所有组件都应该在内存中保持所需要的状态，APIServer 将状态写入 etcd 存储，而其 他组件则通过 APIServer 更新并监听所有的变化;

- 优先使用事件监听而不是轮询。

#### 引导(Bootstrapping)原则

- Self-hosting 是目标。

-启动完成，如果在 failureThreshold*periodSeconds 周期内未就绪，则会应用进程会被重启。

2. 探活方式:

- Exec
- TCP socket
- HTTP

#### 健康检查 spec

```
apiVersion: extensions/v1beta1 kind: Deployment
metadata:
labels: app: nginx
name: nginx-default spec:
replicas: 3 selector:
matchLabels: app: nginx
template: metadata:
labels: app: nginx
```

```
spec: containers:
- image: nginx imagePullPolicy: Always name: http
resources: {} terminationMessagePath:
/dev/termination-log terminationMessagePolicy: File resources:
limits:
cpu: "500m" memory: "128Mi"
```

```
livenessProbe: httpGet: path: /
port: 80 initialDelaySeconds: 15 timeoutSeconds: 1
readinessProbe: httpGet:
path: /ping
port: 80 initialDelaySeconds: 5 timeoutSeconds: 1
```

#### ConfigMap

- ConfigMap 用来将非机密性的数据保存到键值对中。
- 使用时， Pods 可以将其用作环境变量、命令行参数或者存储卷中的配置文件。
- ConfigMap 将环境配置信息和 容器镜像解耦，便于应用配置的修改。

#### 密钥对象(Secret)

- Secret 是用来保存和传递密码、密钥、认证凭证这些敏感信息的对象。
- 使用 Secret 的好处是可以避免把敏感信息明文写在配置文件里。
- Kubernetes 集群中配置和使用服务不可避免的要用到各种敏感信息实现登录、认 证等功能，例如访问 AWS 存储的用户名密码。
- 为了避免将类似的敏感信息明文写在所有需要使用的配置文件中，可以将这些信息 存入一个 Secret 对象，而在配置文件中通过 Secret 对象引用这些敏感信息。
- 这种方式的好处包括:意图明确，避免重复，减少暴漏机会。

#### 用户(User Account)& 服务帐户(Service Account)

- 顾名思义，用户帐户为人提供账户标识，而服务账户为计算机进程和 Kubernetes 集群中运行的 Pod 提供 账户标识。

- 用户帐户和服务帐户的一个区别是作用范围:

  - 用户帐户对应的是人的身份，人的身份与服务的 Namespace 无关，所以用户账户是跨

    Namespace 的;

  - 而服务帐户对应的是一个运行中程序的身份，与特定 Namespace 是相关的。

#### Service

Service 是应用服务的抽象，通过 labels 为应用提供负载均衡和服务发现。匹 配 labels 的 Pod IP 和端口列表组成 endpoints，由 Kube-proxy 负责将服务 IP 负载均衡到这些 endpoints 上。

每个 Service 都会自动分配一个 cluster IP(仅在集群内部可访问的虚拟地址) 和 DNS 名，其他容器可以通过该地址或 DNS 来访问服务，而不需要了解后端 容器的运行。

![截屏2023-09-21 22.33.10](assets/1695335590.png)

#### Service Spec

```
apiVersion: v1 kind: Service metadata:
name: nginx spec:
ports:
- port: 8078 # the port that this service should serve on
name: http
# the container on each pod to connect to, can be a name # (e.g. 'www') or a number (e.g. 80)
targetPort: 80
protocol: TCP
selector: app: nginx
```

#### 副本集(Replica Set)

- Pod 只是单个应用实例的抽象，要构建高可用应用，通常需要构建多个同样的副本，提供同一个服务。
- Kubernetes 为此抽象出副本集 ReplicaSet，其允许用户定义 Pod 的副本数，每一个 Pod 都会被当作一 个无状态的成员进行管理，Kubernetes 保证总是有用户期望的数量的 Pod 正常运行。
- 当某个副本宕机以后，控制器将会创建一个新的副本。
- 当因业务负载发生变更而需要调整扩缩容时，可以方便地调整副本数量。

![截屏2023-09-21 22.33.33](assets/1695335613.png)

#### 部署(Deployment)

- 部署表示用户对 Kubernetes 集群的一次更新操作。
- 部署是一个比 RS 应用模式更广的 API 对象，可以是创建一个新的服务，更新一个新的� 减少依赖，特别是稳态运行的依赖。 

- 通过分层的原则管理依赖。

- 循环依赖问题的原则:

  - 同时还接受其他方式的数据输入(比如本地文件等)，这样在其他服务不可 用时还可以手动配置引导服务;

  - 状态应该是可恢复或可重新发现的;

  - 支持简单的启动临时实例来创建稳态运行所需要的状态，使用分布式锁或文件锁等来协调不同状态的切换(通常称为 pivoting 技术);

  - 自动重启异常退出的服务，比如副本或者进程管理器等。

#### 课后练习 4.1

用 Kubeadm 安装 Kubernetes 集群。

### 核心技术概念和 API 对象

API 对象是 Kubernetes 集群中的管理操作单元。

Kubernetes 集群系统每支持一项新功能，引入一项新技术，一定会新引入对应的 API 对象，支持对该功能的管理操 作。

每个 API 对象都有四大类属性: 

- TypeMeta
- MetaData
- Spec
- Status

#### TypeMeta

Kubernetes对象的最基本定义，它通过引入GKV(Group，Kind，Version)模型定义了一个对象的类型。

1. Group

Kubernetes 定义了非常多的对象，如何将这些对象进行归类是一门学问，将对象依据其功能范围归入不同的分组， 比如把支撑最基本功能的对象归入 core 组，把与应用部署有关的对象归入 apps 组，会使这些对象的可维护性和可 理解性更高。

2. Kind

定义一个对象的基本类型，比如 Node、Pod、Deployment 等。

3. Version

社区每个季度会推出一个 Kubernetes 版本，随着 Kubernetes 版本的演进，对象从创建之初到能够完全生产化就 绪的版本是不断变化的。与软件版本类似，通常社区提出一个模型定义以后，随着该对象不断成熟，其版本可能会 从 v1alpha1 到 v1alpha2，或者到 v1beta1，最终变成生产就绪版本 v1。

#### Metadata

Metadata 中有两个最重要的属性:Namespace和Name，分别定义了对象的Namespace 归属及名字，这两个属性唯一定义了某个对象实例。

1. Label

顾名思义就是给对象打标签，一个对象可以有任意对标签，其存在形式是键值对。 Label 定义了对象的可识别属性，Kubernetes API 支持以 Label 作为过滤条件 查询对象。

2. Annotation

Annotation 与 Label 一样用键值对来定义，但 Annotation 是作为属性扩展， 更多面向于系统管理员和开发人员，因此需要像其他属性一样做合理归类。

#### Label

- Label是识别Kubernetes对象的标签，以key/value的方式附加到对象上。

- key最长不能超过63字节，value可以为空，也可以是不超过253字节的字符串。

- Label不提供唯一性，并且实际上经常是很多对象(如Pods)都使用相同的label来标志具 体的应用。

- Label定义好后其他对象可以使用LabelSelector来选择一组相同label的对象

- LabelSelector支持以下几种方式:

  - 等式，如 app=nginx 和 env!=production;

  - 集合，如 env in (production, qa);

  - 多个 label(它们之间是 AND 关系)，如 app=nginx,env=test。

![截屏2023-09-21 22.28.06](assets/1695335286.png)

#### Annotations

• Annotations 是 key/value 形式附加于对象的注解。

• 不同于 Labels 用于标志和选择对象，Annotations 则是用来记录一些附加信息，用来辅助应用部署、安 全策略以及调度策略等。

• 比如 deployment 使用 annotations 来记录 rolling update 的状态。

#### Metadata

3. Finalizer

Finalizer 本质上是一个资源锁，Kubernetes 在接收某对象的删除请求时，会检 查 Finalizer 是否为空，如果不为空则只对其做逻辑删除，即只会更新对象中的 metada启动完成，如果在 failureThreshold*periodSeconds 周期内未就绪，则会应用进程会被重启。

2. 探活方式:

- Exec
- TCP socket
- HTTP

#### 健康检查 spec

```
apiVersion: extensions/v1beta1 kind: Deployment
metadata:
labels: app: nginx
name: nginx-default spec:
replicas: 3 selector:
matchLabels: app: nginployment。
- 要求 Envoy 的启动配置从外部的配置文件 Mount 进 Pod。
- 进入 Pod 查看 Envoy 进程和配置。
- 更改配置的监听端口并测试访问入口的变化。
- 通过非级联删除的方法逐个删除对象。
nt。
- 要求 Envoy 的启动配置从外部的配置文件 Mount 进 Pod。
- 进入 Pod 查看 Envoy 进程和配置。
- 更改配置的监听端口并测试访问入口的变化。
- 通过非级联删除的方法逐个删除对象。