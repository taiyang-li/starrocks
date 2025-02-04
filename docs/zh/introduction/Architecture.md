---
displayed_sidebar: Chinese
---

# 架构

StarRocks 的架构简单明了。整个系统仅由两种组件组成：前端和后端。前端节点称为 **FE**。后端节点有两种类型，**BE** 和 **CN** (计算节点)。当使用本地存储数据时，部署 BE；当数据存储在对象存储或 HDFS 时，部署 CN。StarRocks 不依赖任何外部组件，简化了部署和维护。节点可以水平扩展而不影响服务正常运行。此外，StarRocks 具有元数据和服务数据副本机制，提高了数据可靠性，有效防止单点故障 (SPOF)。

StarRocks 兼容 MySQL 协议，支持标准 SQL。用户可以轻松地通过 MySQL 客户端连接到 StarRocks，从而获得即时且有价值的见解。

## 架构选择

StarRocks 支持存算一体 (每个 BE 将其数据存储在本地存储) 和存算分离 (所有数据存储在对象存储或 HDFS 中，每个 CN 仅在本地存储缓存)。您可以根据需要决定数据存储的位置。

![Architecture choices](../assets/architecture_choices.png)

### 存算一体

本地存储为实时查询提供了更低的查询延迟。

作为典型的大规模并行处理 (MPP) 数据库，StarRocks 支持存算一体架构。在这种架构中，BE 负责数据存储和计算。直接访问 BE 本地数据允许本地计算，避免了数据传输和复制，从而提供超快的查询和分析性能。该架构支持多副本数据存储，增强了集群处理高并发查询的能力并确保数据可靠性。非常适合追求最佳查询性能的场景。

![shared-data-arch](../assets/shared-nothing.png)

#### 节点

在存算一体架构中，StarRocks 由两种类型的节点组成：FE 和 BE。

- FE 负责元数据管理和构建执行计划。
- BE 执行查询计划并存储数据。BE 利用本地存储加速查询，并使用多副本机制确保高数据可用性。

##### FE

FE 负责元数据管理、客户端连接管理、查询规划和查询调度。每个 FE 在其内存中存储和维护一份完整的元数据副本，这保证了 FE 之间服务的一致性。FE 可以作为主节点、从节点和观察者工作。从节点可以根据类似 Paxos 的 BDB JE 协议选举主节点。BDB JE 是 Berkeley DB Java Edition 的缩写。

| **FE 角色** | **元数据管理** | **主节点选举**                |
| ----------- | ----------------------- | ---------------------------------- |
| 主节点      | 主 FE 负责读写元数据。从节点和观察者节点只能读取元数据，它们将元数据写请求路由到主 FE。主 FE 更新元数据，然后使用 BDE JE 将元数据更改同步到从节点和观察者节点。只有在元数据更改同步到超过一半的从节点后，数据写入才被认为成功。 | 主 FE 技术上也是一个从节点，是从从节点中选举出来的。要执行主节点选举，集群中必须有超过一半的从节点处于活动状态。当主 FE 发生故障时，从节点将开始另一轮主节点选举。 |
| 从节点    | 从节点只能读取元数据。它们从主 FE 同步和重放日志以更新元数据。 | 从节点参与主节点选举，这需要集群中超过一半的从节点处于活动状态。 |
| 观察者   | 观察者从主 FE 同步和重放日志以更新元数据。     | 观察者主要用于增加集群的查询并发性。观察者不参与主节点选举，因此不会增加集群的主节点选举压力。|

##### BE

BE 负责数据存储和 SQL 执行。

- 数据存储：BE 具有等效的数据存储能力。FE 根据预定义规则将数据分发到 BE。BE 转换摄取的数据，将数据写入所需格式，并为数据生成索引。

- SQL 执行：FE 根据查询的语义将每个 SQL 查询解析为逻辑执行计划，然后将逻辑计划转换为可以在 BE 上执行的物理执行计划。存储目标数据的 BE 执行查询，这消除了数据传输和复制的需求，实现了高查询性能。

### 存算分离

对象存储和 HDFS 提供成本、可靠性和可扩展性优势。除了存储的可扩展性外，CN 节点可以随时添加和删除，无需重新平衡数据，因为存储和计算是分离的。

在存算分离架构中，BE 被“计算节点 (CN)”取代，后者仅负责数据计算任务和缓存热数据。数据存储在低成本且可靠的远端存储系统中，如 Amazon S3、GCP、Azure Blob Storage、MinIO 等。当缓存命中时，查询性能可与存算一体架构相媲美。CN 节点可以根据需要在几秒钟内添加或删除。这种架构降低了存储成本，确保更好的资源隔离，并具有高度的弹性和可扩展性。

存算分离架构与存算一体架构一样简单。它仅由两种类型的节点组成：FE 和 CN。唯一的区别是用户必须配置后端对象存储。

![shared-data-arch](../assets/shared-data.png)

#### 节点

在存算分离架构中，FE 提供的功能与存算一体架构中的相同。

BE 被 CN (计算节点) 取代，存储功能被转移到对象存储或 HDFS。CN 是无状态的计算节点，执行所有 BE 的功能，除了存储数据。

#### 存储

StarRocks 存算分离集群支持两种存储解决方案：对象存储 (例如，AWS S3、Google GCS、Azure Blob Storage 或 MinIO) 和 HDFS。

在存算分离集群中，数据文件格式与存算一体集群 (存储和计算耦合) 保持一致。数据组织成段文件，各种索引技术在云原生表中重复使用，这些表专门用于存算分离集群。

#### 缓存

StarRocks 存算分离集群将数据存储与计算分离，使其能够独立扩展，从而降低成本并提高弹性。然而，这种架构会影响查询性能。

为减小影响，StarRocks 建立了包含内存、本地磁盘和远端存储的多层数据访问系统，以便更好地满足各种业务需求。

对于热数据的查询直接扫描缓存，然后扫描本地磁盘，而冷数据需要从对象存储中加载到本地缓存中以加速后续查询。通过将热数据保持在计算单元附近，StarRocks 实现了真正的高性能计算和性价比高的存储。此外，通过数据预取策略优化了对冷数据的访问，有效消除了查询的性能限制。

在创建表时可以启用缓存。如果启用缓存，数据将同时写入本地磁盘和后端对象存储。在查询过程中，CN 节点首先从本地磁盘读取数据。如果未找到数据，将从后端对象存储中检索，并同时缓存到本地磁盘中。

## 实践学习

- 使用 MinIO 进行对象存储，尝试 [shared-data](../quick_start/shared-data.md)。
- Kubernetes 用户可以使用 [Helm quick start](../quick_start/helm.md) 部署三个 FE 和三个 BE，在使用持久卷的存算一体架构中。
