# 猿辅导工程

猿辅导是一家在线教育公司，目前公司的产品涵盖学龄前（斑马 AI 课）、K12（猿辅导）、少儿编程（猿编程）以及教育类的相关应用（猿题库、小猿搜题、小猿口算），我们希望用科技去改变教育。

在猿辅导背后，是一个庞大的工程体系去支撑整个教育系统。这个教育系统除了面向几亿名终端用户以外，还负责支撑几万人的辅导老师、教研、主讲老师团队的日常工作。这个工程系统是一部庞大的在线教育机器，涉及的领域包括机器学习、音视频处理、实时计算、互动课堂等多个技术方向。我们使用了微服务架构去实现这样庞大的业务系统，成千上万个微服务被部署在多个核心机房中。

## 工程上的挑战

我们的主要用户群体均为付费用户，对于系统的稳定性要求会更加严格。辅导老师、主讲老师和教研老师依赖我们的系统去传递知识、传递快乐，学生依赖我们的系统获得成长，短暂的服务不可用都有可能给大量的家庭带来影响。我们的业务规模每年以数倍增速进行扩张，系统设计之初的冗余很快就会被高增速打满。可用性、可扩展性是我们在系统设计上最优先考虑的要素。

为此，团队需要不停地检视我们的工程体系，发现其中潜在的问题，做好准备去迎接未来的挑战。

## 猿辅导的技术栈

//TBD (大概介绍下思路，自下向上，Platform -> Service -> Gateway -> Client，以及 Infra Service)

### 基础平台

//TBD (做一个 Summary，画个图说明下，主要是 IDC, Network -> Service Container, Storage)

#### IDC

猿辅导的机房设施分为核心机房与边缘机房，其中核心机房内部署了大部分的存储、应用服务。核心机房主要部署在国内的主要一线城市内，除了自建的 IDC 机房以外，我们也会使用[阿里云]和[腾讯云]的相关云主机来补充弹性能力。核心机房之间一般都通过多条网络专线连接，核心机房本身也可以作为中转节点以确保网络的高可用能力。

同时，我们也使用了阿里云和腾讯云的边缘计算能力，在众多国内二三级城市以及北美的部分城市内部署了大量的边缘机房，边缘机房则作为直播系统的一部分，部署直播边缘节点，为用户提供低延迟的音视频流转发服务。

#### 存储设施

我们的持久化数据存储以关系型存储为主，因此 [MySQL] 是公司内使用范围最广的数据库。我们自己维护了若干个大规模的 MySQL 集群，分布在各个核心机房内。此外，对于一些特殊场合，我们也使用了阿里云的相关关系型存储产品作为补充，包括 [RDS]、[DRDS] 和 [PolarDB]。

在性能要求极高的场景下，我们优先使用 [Redis] 作为分布式内存缓存数据库。和 MySQL 类似，工程团队在每个核心机房内都有自维护的 Redis 集群，大部分自维护的集群都使用 Redis Sentinel 作为高可用方案。阿里云上 Redis 集群也同时被团队所使用。同时，有部分早期的服务也在使用 [Memcached] 作为分布式内存数据库。

[Hbase] 和 [ElasticSearch] 在一些 OLAP 场景下也会被使用，例如教研系统的数据分析、用户搜索等等。

#### 应用服务编排

在猿辅导早期，我们直接使用物理机进行服务部署。很快地我们遇到了这种部署模式的瓶颈，因此我们基于 [Docker] 容器运行时，自己实现了一套容器编排服务。现在，我们大部分的应用服务都使用自建的这套 Docker 编排服务进行部署，它主要帮助我们去关联容器与物理机，去维护容器的生命周期，暴露一套统一的接口使研发可以自助进行构建、部署操作。

在 2021 年的春晚活动前期，这套容器编排引擎已经不再很好地满足大规模扩、缩容的场景了，我们的研发团队在这次活动中小规模尝试了 [Kubernetes]。Kubernetes 很好地支持了春晚活动业务，因此在活动后，猿辅导开始大规模迁移应用服务编排引擎到 Kubernetes。

我们为 Kubernetes 实现了一些相关的工程设施，例如流水线引擎 Pipelix、Kubernetes API 代理 Plume、CI/CD 平台 Phoenix。同时我们也使用 Argocd 来管理非标服务的编排需求。

### 应用服务

#### 站内流量调度

目前猿辅导有两套注册中心，分别是 [ZooKeeper] 和 [Nacos]。ZooKeeper 是比较早期的技术选型，在经历业务的高速增长后，ZooKeeper 的写能力遇到了瓶颈，因此我们在 2020 年将注册中心替换成了 Nacos 以支持更好的集群扩展。同时我们还自己实现了一套 Naming Agent，代理所有的注册、发现请求，为应用服务屏蔽了 Nacos 的实现细节，为将来更换注册中心预留了接口。

应用服务之间使用 [Thrift] 协议相互进行通信，为了携带 meta 信息，我们使用 THeaderTransport 作为 Thrift 传输层协议。

#### 任务调度

定时任务被广泛应用于各个业务系统中，例如当用户下订单后，定时检查订单是否超时；在最终一致性事务中负责最终状态检查；在系统中巡检潜在的数据不一致风险等等。因为历史原因，目前系统中有三套定时任务实践。

首先我们通过自定义的 Docker 编排引擎，实现了 job 容器的持久化运行，在这个容器中通过 crontab 来调度任务。但这种方案在物理机故障时不能很好地处理容灾问题，所以我们在业务服务中直接使用 Spring 的 @Cron 注解进行任务调度，同时使用 ZooKeeper 进行 Leader 选举确保同一时刻只有一个任务在执行。随着系统的进一步复杂化，原先零星的定时任务成了系统设计中的一个常见模式，我们引入了 [xxl-job] 作为分布式任务调度引擎。

#### 消息队列

在公司早期，我们使用过 [RabbitMQ] 作为消息队列服务，但很快地我们遇到了 RabbitMQ 不能抗挤压以及可扩展性上的问题。就像一开始我们提到的，可用性与可扩展性是团队考虑技术选型的第一要素，为此我们迁移了大部分的消息队列到 Kafka 和 RocketMQ 上。

同时，我们使用 MySQL 以及 tutor-on-tick-message 服务实现了延迟消息能力，解决了 RocketMQ 延迟消息天数的限制。

#### 开发与部署

//TBD (代码仓库、代码搜索、发布系统、CI 系统、包管理、配置管理、装机)

#### 开发语言

//TBD (Java、Spring Boot、Monorepo、Node.js、Python、C++)

### 可见性

//TBD (介绍下可见性工程的体系，目标)

#### 日志

我们使用 [Fluentd] 去各个基础设施上进行日志采集，并将日志写入[阿里云 SLS]以提供搜索、分析能力。此外，日志也会持久化落盘在 [Hbase] 上。

//TBD (需要跟大数据的同学聊一下把这部分写的更详细一些)

#### 指标分析

随着业务规模扩大和复杂度逐渐上升，也为了适应基础设施云原生化发展，我们正在将指标体系从 [OpenFalcon] 切换到 [Prometheus]。当前业务可以使用 Prometheus SDK 埋点，通过 PromQL 查询和通过 [Grafana] 可视化自己的业务数据大盘。

同时也因为庞大的业务规模和 Prometheus 的数据模型特点，我们遇到了部分指标维度爆炸的问题，这极大影响了 Prometheus 集群的性能和可用性。我们正在积极尝试 [VictoriaMetrics]、[M3] 等分布式存储方案，也在探索进一步增强 TSDB 的各维度线性拓展的能力。

#### 链路分析

//TBD (阿里云 Tracing，Skywalking)

#### 报警

//TBD (AlertManager, Cicada)

### 测试

//TBD (持续测试平台、流量复制、流量比对)

### 入站流量

猿辅导的大部分入站流量都是 HTTPS 请求，这些请求在经过 DNS 解析后，会通过阿里云、腾讯云的相关 ELB 产品在 4 层进行负载均衡。接下来请求会进入 [Nginx] 集群在 7 层进行流量路由，我们同时也使用了 Nginx 的 Upstream 负载均衡能力使流量均匀地分发到每组应用服务的不同实例上。

在今年基础架构的一个主要改进方向上，我们拆分了这一层 Nginx 的路由转发能力与负载均衡能力，让这一层 Nginx 只负责路由转发。同时使用 Kubernetes Ingress Controller 集群做 Kubernetes 内 HTTP 流量的负载均衡。这个架构一方面简化了每一层的复杂度，使得路由组件可以单独演进；另一方面也对 Docker 集群向 Kubernetes 迁移提供了支持。

### 客户端

//TBD (介绍下客户端体系)

#### Web

//TBD (介绍下 Web 的相关应用项目、场景)

##### 语言

//TBD (TypeScript、JavaScript、Node.js、Angular、Vue、React、SPA、SSR、Electron)

##### 构建

//TBD (构建系统、发布、CDN、NPM)

#### iOS

//TBD (介绍下 iOS 的相关应用项目、场景)

##### 语言

//TBD (OC、Swift)

##### Library

//TBD

##### 存储

//TBD

##### 构建

//TBD (构建系统、包发布系统、制品管理)

#### Android
以下功能拿斑马 App 举例。
- 斑马是公司对于学龄前在线教育的战略布局产品，主要用户群体是学龄前 2-8 岁儿童。
 
##### 语言
目前 App 内 java 使用场景较少，主要开发语言为 kotlin。
效率、安全场景需要写一些 C 代码，为了优化工程效率，有时候会写一些 python 脚本。
 
##### 混合容器
除支持 Native 开发外，App 目前还支持 Flutter、Flutter For Web、H5 的混合栈开发形式，由 Native 封装基础端能力后供混合栈调用，各个业务只需要根据业务特性选择不同的开发语言开发业务即可。
由于 App 是混合架构，页面的启动、返回以及参数的携带情况需要尽可能的考虑全面，基础路由能力已经支持完全，目前页面混合路由正在逐步完善中。
 
##### 组件化
拿斑马 APP 来举例，经过两个 Q 的努力，目前来看我们逻辑层面从上到下依次拆分成了四个层次：

壳工程：除了一些全局配置和主 Activity 之外，不包含任何业务代码。
业务组件：顶层业务，每个组件表示一条完整的业务线，彼此之间互相独立。
基础服务：支撑上层业务组件运行的基础业务服务。
基础能力（三方库）：完全业务无关的基础库。
 
各层次职责清晰独立，层次内各模块逐步解耦，后续大的模块都会有自己的版本，足够独立后，业务线可以独立发版，随时升级或者回滚。
 
##### DevOps on Android
**基于 Jenkins 的自动构建系统**：主要目标是解决自动代码检测、打包构建、分发相关的问题。代码检测使用 Python 开发一个框架，可以方便得增减检查项并将结果反馈到 Gerrit 平台。
**Crash 监测**：使用了 Bugly 和 Sentry（自建） 两个平台。基于平台做了 crash 率统计等信息通知的机器人。
**日志框架**：本地日志使用 Xlog + Timber，可以通过长链接触发回传，或者用户主动回传；实时日志采用阿里云的 SLS 服务。
 

#### Hybrid

//TBD

##### Flutter

//TBD

##### React Native

//TBD

[MySQL]: https://www.mysql.com
[Redis]: https://redis.io
[Kubernetes]: https://kubernetes.io
[Hbase]: https://hbase.apache.org
[ElasticSearch]: https://www.elastic.co
[Memcached]: https://memcached.org
[ZooKeeper]: https://zookeeper.apache.org
[Nacos]: https://nacos.io
[Thrift]: https://thrift.apache.org
[Docker]: https://www.docker.com
[xxl-job]: https://www.xuxueli.com/xxl-job
[RabbitMQ]: https://www.rabbitmq.com
[Kafka]: https://kafka.apache.org
[RocketMQ]: https://rocketmq.apache.org
[Fluentd]: https://www.fluentd.org
[Nginx]: https://www.nginx.com
[RDS]: https://cn.aliyun.com/product/rds
[DRDS]: https://cn.aliyun.com/product/drds
[PolarDB]: https://www.aliyun.com/product/polardb
[阿里云]: https://www.aliyun.com
[腾讯云]: https://cloud.tencent.com
[OpenFalcon]: http://open-falcon.org
[Prometheus]: https://prometheus.io
[M3]: https://github.com/m3db/m3
[VictoriaMetrics]: https://victoriametrics.com
[Grafana]: https://grafana.com
