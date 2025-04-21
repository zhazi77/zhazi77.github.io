---
draft: true
date: 
  created: 2025-04-19
categories:
  - Reading
tags:
  - Distributed Systems
  - Redis
authors:
  - zhazi
---

# 翻译：Scale with Redis Cluster

> 使用 Redis 集群进行水平扩展

!!! info "文献"

    - [Scale with Redis Cluster](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/#learn-more)


Redis 通过一种称为 Redis 集群的部署拓扑结构(1)实现横向扩展。本主题将指导你如何在生产环境中搭建、测试并运行 Redis Cluster。你将从终端用户的角度了解 Redis Cluster 在可用性和一致性方面的特性。
{ .annotate }

1. :nerd::point_up:: **部署拓扑结构（Deployment Topology）**是指在分布式系统或应用程序部署过程中，用于描述系统组件如何分布和连接的架构设计。它涵盖了硬件、软件、网络资源等的布局，以及它们之间的交互方式。

如果你计划在生产环境中部署 Redis Cluster，或希望更深入了解 Redis Cluster 的内部工作原理，请参考 [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)。若想了解 Redis Enterprise 如何实现扩展，请参阅[Linear Scaling with Redis Enterprise](https://redis.com/redis-enterprise/technology/linear-scaling-redis-enterprise/?_gl=1*15ds9b9*_gcl_au*MTM0OTI2MDczNy4xNzQyNjIyMTU5)。

## Redis Cluster 101

Redis 集群提供了一种运行 Redis 实例的方法，其可以将数据自动分片到多个 Redis 节点上。Redis 集群还在出现网络分区时提供一定程度的可用性——也就是说，在部分节点故障或无法通信的情况下，集群仍能继续运行。然而，当发生更大范围的故障（例如多数主节点不可用）时，整个集群将变得不可用。

因此，使用 Redis 集群，你可以：

- 在多个节点之间自动拆分数据集。
- 当一部分节点遇到故障或无法与集群的其余部分通信时，请继续作。(分区容错性)

### Redis Cluster TCP ports

每个 Redis 集群节点都需要开放两个 TCP 端口：一个是用于服务客户端的 Redis TCP 端口，例如 6379，另一个是称为**集群总线端口（cluster bus port）**的端口。默认情况下，集群总线端口是在数据端口的基础上加上 10000（例如 16379）；不过，你也可以在配置中通过 `cluster-port` 参数来自定义该端口。

**集群总线（cluster bus）**是一个（节点之间）点对点的通信通道，采用的是一种二进制协议，由于其带宽占用少、处理开销低，更适合用于节点间的信息交换。节点通过集群总线(1)进行故障检测、配置更新、故障转移授权等操作。客户端不应尝试与集群总线端口通信，而应始终使用 Redis 命令端口。然而，你仍需确保在防火墙中打开这两个端口，否则 Redis 集群节点将无法相互通信。
{ .annotate }

1. :nerd::point_up:: 为了完成各自的任务，所有集群节点通过一个称为 **Redis 集群总线（Redis Cluster Bus）**的 TCP 总线和二进制协议进行连接。集群中的每个节点都通过集群总线与其他所有节点相连接。节点使用一种 gossip 协议来传播集群信息，以发现新的节点，发送 ping 包以确保其他节点运行正常，以及发送用于标识特定状态的集群消息。集群总线还用于在集群中传播 Pub/Sub 消息，并在用户发起请求时协调手动故障转移（手动故障转移是由系统管理员手动触发的，而不是由 Redis Cluster 的故障检测机制自动发起的）。

为了使 Redis Cluster 正常运行，每个节点都需要满足以下条件：

- 客户端通信端口（通常为 6379）必须开放，用于与客户端通信，需对所有需要访问集群的客户端开放，同时也需对其他集群节点开放，因为这些节点会通过该端口进行键迁移操作；
- 集群总线端口必须对所有其他集群节点可达；

如果未同时开放这两个 TCP 端口，集群将无法正常工作。

## Redis Cluster data sharding

Redis 集群并不使用一致性哈希，而是采用另一种分片方式，其中每个键在概念上属于一个称为**哈希槽（hash slot）**的部分。

在 Redis 集群中共有 16384 个哈希槽，计算某个键对应的哈希槽的方法是：对该键进行 CRC16 运算，然后对 16384 取模。

在 Redis 集群中，每个节点负责一部分哈希槽。例如，你可能有一个包含 3 个节点的集群，其中：

- 节点 A 负责哈希槽 0 到 5500；
- 节点 B 负责哈希槽 5501 到 11000；
- 节点 C 负责哈希槽 11001 到 16383。

这种机制使得添加或移除集群节点变得非常简单。例如，如果想添加一个新的节点 D，只需将部分哈希槽从 A、B、C 迁移到 D 即可。类似地，如果想从集群中移除节点 A，只需将它负责的哈希槽迁移到 B 和 C，当节点 A 的哈希槽都迁出后，就可以将它从集群中完全移除。

将哈希槽从一个节点迁移到另一个节点的过程中无需中断任何操作，因此，添加或移除节点，或调整某个节点所持有的哈希槽比例，都不需要停机。

Redis 集群支持多键操作，只要在一次命令执行（或整个事务，或 Lua 脚本执行）中所涉及的所有键都属于同一个哈希槽即可。用户可以通过一种称为**哈希标签（hash tags）**的功能，强制多个键被映射到同一个哈希槽中。

哈希标签在 Redis 集群规范中有详细说明，其核心原理是：如果一个键中包含一对大括号 `{}`，则只有括号中的内容会被用于计算哈希值。例如，键 `user:{123}:profile` 和 `user:{123}:account` 因为具有相同的哈希标签 `{123}`，所以它们一定会被映射到同一个哈希槽中。这样，你就可以在一次多键操作中同时操作这两个键。

### Redis Cluster master-replica model

为了在部分主节点发生故障或无法与大多数节点通信的情况下仍保持可用性，Redis 集群使用了**主从模型（master-replica model）**：每个哈希槽除了由主节点负责外，还可以有 1 个或多个副本节点（即 N 个副本，其中包括 1 个主节点和 N-1 个副本节点）。

以上面提到的 Redis 集群（节点 A、B、C）为例，如果节点 B 发生故障，整个集群将无法继续运行，因为哈希槽范围 5501–11000 无法被访问。

为了解决这个问题，在集群创建时（或之后的某个时间点），我们可以为每个主节点添加一个副本节点，从而形成一个包含 A、B、C（主节点）和 A1、B1、C1（对应的副本节点）的完整集群结构。这样，即使节点 B 故障，系统也能继续正常运行。

当节点 B 的副本节点 B1 侦测到 B 故障后，集群会将 B1 提升为新的主节点，从而维持集群的正常运行。

但需要注意的是，如果 B 和 B1 同时故障，Redis Cluster 将无法继续工作。

### Redis Cluster consistency guarantees

Redis 集群不保证**强一致性（strong consistency）**。这意味着在某些情况下，Redis 集群可能会丢失那些已经向客户端确认成功的写入操作。

Redis 集群可能丢失写入的第一个原因是它采用**异步复制**机制。也就是说，在写操作过程中，会发生如下流程：

1. 客户端向主节点 B 发起写请求；
2. 主节点 B 向客户端返回 OK，表示写入成功；
3. 主节点 B 将该写操作异步地传播给其副本节点 B1、B2 和 B3。

如你所见，主节点 B 在向客户端确认写入成功之前，并不会等待来自 B1、B2、B3 的副本确认，这是因为等待副本确认会带来 Redis 无法接受的高延迟。因此，如果客户端向 B 写入数据，B 向客户端返回成功，但随后在将写入同步到副本之前崩溃了，此时某个未收到该写入的副本可能会被提升为新的主节点，从而导致该写入永久丢失。

这种情况与大多数数据库设置为每秒将数据刷新到磁盘一次的行为非常类似，因此你可能已经在使用传统数据库系统（非分布式系统）时遇到过类似问题。类似地，如果希望提升一致性，可以要求数据库在向客户端确认写入成功之前先将数据刷新到磁盘，但这通常会带来极大的性能损失。在 Redis 集群中，这种做法就相当于采用**同步复制**机制。

本质上，这是一个在**性能**和**一致性**之间进行权衡的问题。

Redis 集群在确实需要时也支持**同步写入**，通过 `WAIT` 命令来实现。使用该机制可以大幅降低写入丢失的可能性。但是，请注意，即使采用了同步复制，Redis 集群也**并不提供强一致性**：在一些复杂的故障场景下，仍有可能将未接收到写入的副本选举为主节点。

还有另一种比较典型的场景也可能导致 Redis Cluster 丢失写入，那就是**网络分区（network partition）**，并且客户端与包括至少一个主节点在内的少数节点处于同一分区中。

举例来说，假设我们有一个由 6 个节点组成的集群：A、B、C 是主节点，A1、B1、C1 是各自对应的副本节点。同时存在一个客户端 Z1。

当网络发生分区时，可能出现以下情况：一侧分区包含 A、C、A1、B1、C1，另一侧分区只包含 B 和客户端 Z1。

这时，Z1 仍然可以向 B 写入数据，而 B 也会接受这些写入。如果网络分区很快恢复，整个集群可以正常继续运行。但如果分区持续时间较长，导致多数节点的一侧将 B 的副本 B1 提升为新的主节点，那么在此期间 Z1 向 B 写入的数据就会**永久丢失**。

!!! tip "注意"

    Z1 能够向 B 发送写操作的数量有一个最大窗口：如果经过了足够长的时间，使得分区中的多数侧能够选举出一个副本作为主节点，那么少数侧中的每一个主节点都将停止接受写操作。

这个足够长的时间被称为**节点超时时间（node timeout）**，是 Redis 集群的一个关键配置选项。

当节点超时时间到期后，主节点会被视为故障节点，可被它的某个副本取代。同样，若主节点在超时时间内无法感知到大多数其他主节点的存在，则该节点将进入错误状态并停止接受写请求。

## Redis Cluster configuration parameters

我们即将创建一个示例集群部署。不过在那之前，让我们先了解一下 `redis.conf` 文件中与 Redis Cluster 相关的一些配置参数：

<div class="annotate" markdown>
- **cluster-enabled <yes/no>**：如果设置为 `yes`，则启用该 Redis 实例的集群功能。否则，该实例将以普通的单机模式启动。

- **cluster-config-file <filename>**：尽管这个选项的名称中带有 "config"，但它实际上不是供用户手动编辑的配置文件，而是 Redis Cluster 节点在每次集群状态发生变化时自动持久化其状态信息的文件。这个文件中记录了集群中其他节点的信息、节点状态、持久化变量等内容。该文件通常会在接收到某些集群消息后被重写并刷新到磁盘。

- **cluster-node-timeout <毫秒数>**：这是允许一个 Redis Cluster 节点不可达的最长时间。如果一个主节点在这个时间内都无法被访问，它将被其副本节点**故障转移（failover）**(1)。这个参数还控制其他重要行为，比如：当某个节点在该时间内无法联系到大多数主节点时，它将停止接受客户端请求。

- **cluster-slave-validity-factor <因子>**：如果设置为 0，副本节点将始终认为自己是有效的，并在主节点不可用时始终尝试进行故障转移，无论主从连接中断了多长时间。如果该值为正数，最大断开时间将由 `cluster-node-timeout` 乘以该因子计算得到。如果副本节点与主节点的连接中断超过该时间限制，它将不再尝试进行故障转移。例如，如果 `cluster-node-timeout` 设置为 5 秒，而有效性因子设置为 10，那么断开连接超过 50 秒的副本将不会尝试故障转移。需要注意的是，非零值可能会导致在主节点故障时，集群无法完成故障转移，从而导致整个集群不可用，直到原主节点重新加入集群。

> 为便于理解，有修改

- **cluster-migration-barrier <数量>**： 当一个主节点（目标主节点）没有任何副本时，要将一个其他主节点（源主节点）的副本迁移到这个主节点时，源主节点必须保持连接的最小副本数量。有关副本迁移的更多信息，请参阅本教程中的相关部分。

- **cluster-require-full-coverage <yes/no>**：如果设置为 `yes`（默认值），当集群中某些哈希槽没有被任何节点覆盖时，集群将停止接受写请求。如果设置为 `no`，即使只有部分键空间可以被处理，集群仍将继续提供服务。

- **cluster-allow-reads-when-down <yes/no>**：如果设置为 `no`（默认值），当集群处于失败状态（比如节点无法达成主节点多数，或者集群没有被完全覆盖）时，该节点将停止提供所有服务，以避免返回可能不一致的数据。如果设置为 `yes`，即使在集群失败状态下也允许读取操作。这对于优先考虑读可用性（而不太在意写一致性）的应用场景是有用的，比如使用只有一个或两个分片的 Redis Cluster，可以在主节点宕机且无法自动故障转移时继续提供读服务。
</div>

1. :nerd::point_up:: 故障转移是指当主节点（Master）因故障无法提供服务时，系统自动选择一个从节点（Slave）作为新的主节点，并重新分配数据和服务的过程。这一机制可以确保系统的高可用性和数据一致性。在集群模式下，集群节点之间会相互监控，当某个节点故障时，其他节点会检测到。此时，故障节点的数据（槽位）会被迁移到其他节点,集群会重新平衡数据，并继续提供服务。

## Create and use a Redis Cluster

要创建和使用Redis集群，请按照以下步骤操作：

<style>
.md-typeset ul li {
    margin-top: 0.2em;    /* 顶部间距 */
    margin-bottom: 0.2em; /* 底部间距 */
}
</style>

- [创建 Redis 集群](#create-a-redis-cluster)
- [与集群交互](#interact-with-the-cluster)
- [使用 redis-rb-cluster 编写示例应用](#write-an-example-app-with-redis-rb-cluster)
- [重新分片集群](#reshard-the-cluster)
- [一个更有趣的示例应用](#a-more-interesting-example-application)
- [测试故障转移](#test-the-failover)
- [手动故障转移](#manual-failover)
- [添加新节点](#add-a-new-node)
- [移除节点](#remove-a-node)
- [副本迁移](#replica-migration)
- [升级 Redis 集群中的节点](#upgrade-nodes-in-a-redis-cluster)
- [迁移到 Redis 集群](#migrate-to-redis-cluster)
- [创建 Redis 集群](#create-a-redis-cluster)

不过在那之前，你需要先熟悉创建集群的要求。

### Requirements to create a Redis Cluster

要创建一个 Redis 集群，首先需要启动几个处于集群模式下的空 Redis 实例。

至少需要在 redis.conf 文件中设置以下指令：

``` conf title="redis.conf"
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

要启用集群模式，请将`cluster-enabled`指令设置为`yes`。每个实例还包含一个文件的路径，该文件存储了此节点的配置，默认情况下为`nodes.conf`。这个文件永远不会被人工修改；它仅在启动时由 Redis 集群实例生成，并在需要时更新。

注意，一个能正常工作的最小集群必须包含至少三个主节点。对于生产部署，我们强烈建议使用六节点集群，其中包含三个主节点和三个副本节点。

你可以通过以下方式在本地测试：在任一目录下创建以实例端口号命名的子目录。

例如：

```bash
mkdir cluster-test
cd cluster-test
mkdir 7000 7001 7002 7003 7004 7005
```
然后在每个目录（7000 到 7005）内创建一个 `redis.conf` 文件。配置文件可以直接使用上面的简单示例模板，但记得根据目录名称修改端口号（比如 `7000` 要改成对应目录的端口号）。  

你可以像这样启动每个实例（每个实例运行在单独的终端标签页中）：

```bash
cd 7000
redis-server ./redis.conf
```

从日志中你会看到，每个节点都会给自己分配一个新的 ID：

```console
[82462] 26 Nov 11:56:55.329 * No cluster configuration found, I'm 97a3a64667477371c4479320d683e4c8db5858b1
```
这个 ID 会被该实例永久使用，以确保它在集群中拥有唯一的名称。每个节点都通过这个 ID（而不是 IP 或端口）来识别其他节点。IP 地址和端口可能会变，但节点的唯一标识符在整个生命周期内都不会改变。我们把这个标识符简称为 **节点 ID（Node ID）**。

### Create a Redis Cluster {#create-a-redis-cluster}
现在我们已经运行了多个 Redis 实例，接下来需要通过向节点写入有效的配置来创建集群。

你可以手动配置并逐个启动实例，也可以使用 `create-cluster` 脚本。我们先来看看如何手动操作。

要创建集群，请运行以下命令：

``` bash
redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 \
127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
--cluster-replicas 1
```

这里使用的命令是 `create`，因为我们要创建一个新集群。选项 `--cluster-replicas 1` 表示我们希望为每个主节点分配一个副本节点。

其余参数是用于创建新集群的实例地址列表。

`redis-cli` 会提供一个配置方案。输入 `yes` 接受提议的配置后，集群将开始配置并建立连接，这意味着各个实例会开始相互通信。如果一切顺利，你最终会看到类似这样的消息：

```console
[OK] All 16384 slots covered
```

这表明集群中至少有一个主节点正在处理 16384 个可用哈希槽中的每一个。

如果你不想按照上述方式手动配置并逐个启动实例来创建 Redis 集群，还有一个更简单的方案（不过这样你就学不到那么多操作细节了）。

在 Redis 发行版的 `utils/create-cluster` 目录下，有一个名为 `create-cluster` 的 bash 脚本（与所在目录同名）。要快速创建一个包含 3 个主节点和 3 个副本节点的 6 节点集群，只需执行以下命令：

```bash linenums="1"
create-cluster start
create-cluster create
```
在第二步中，当 `redis-cli` 工具提示你确认集群布局时，请回复 `yes`。  

现在你可以与集群进行交互了（默认情况下第一个节点会运行在 30001 端口）。完成后，使用以下命令停止集群：  

```bash linenums="3"
create-cluster stop
```  

更多关于脚本运行方式的详细信息，请阅读该目录下的 README 文件。

### Interact with the Cluster {#interact-with-the-cluster}
要连接 Redis 集群，你需要使用支持集群模式的 Redis 客户端。请根据所选客户端文档确认其集群支持情况。  

也可以使用 `redis-cli` 命令行工具测试 Redis 集群：  

```console
$ redis-cli -c -p 7000
redis 127.0.0.1:7000> set foo bar
-> Redirected to slot [12182] located at 127.0.0.1:7002
OK
redis 127.0.0.1:7002> set hello world
-> Redirected to slot [866] located at 127.0.0.1:7000
OK
redis 127.0.0.1:7000> get foo
-> Redirected to slot [12182] located at 127.0.0.1:7002
"bar"
redis 127.0.0.1:7002> get hello
-> Redirected to slot [866] located at 127.0.0.1:7000
"world"
```  

!!! tip "注意"

    如果使用脚本创建集群，节点默认会从 30001 端口开始监听（具体端口号可能有所不同）。

`redis-cli` 的集群支持较为基础，始终依赖集群节点的重定向机制引导客户端连接正确节点。专业的客户端会缓存哈希槽与节点地址的映射关系，直接建立与目标节点的连接。该映射仅在集群配置变化时刷新，例如发生故障转移，或管理员通过增删节点调整集群布局时。

### Write an Example App with redis-rb-cluster {#write-an-example-app-with-redis-rb-cluster}

在继续演示如何操作 Redis 集群（例如进行故障转移或重新分片）之前，我们需要创建示例应用或至少理解 Redis 集群客户端交互的基本语义。  

通过这种方式，我们既能运行示例，又可以模拟节点故障或启动重新分片，观察 Redis 集群在真实场景下的行为。若没有数据写入集群，单纯观察其行为意义不大。  
本节通过两个示例说明[redis-rb-cluster](https://github.com/antirez/redis-rb-cluster) 的基本用法。第一个示例是 redis-rb-cluster 发行版中的 `example.rb` 文件：  

```bash linenums="1" title="example.rb" hl_lines="14 18-26 28-37"
require './cluster'

if ARGV.length != 2
    startup_nodes = [
        {:host => "127.0.0.1", :port => 7000},
        {:host => "127.0.0.1", :port => 7001}
    ]
else
    startup_nodes = [
        {:host => ARGV[0], :port => ARGV[1].to_i}
    ]
end

rc = RedisCluster.new(startup_nodes,32,:timeout => 0.1)

last = false

while not last
    begin
        last = rc.get("__last__")
        last = 0 if !last
    rescue => e
        puts "error #{e.to_s}"
        sleep 1
    end
end

((last.to_i+1)..1000000000).each{|x|
    begin
        rc.set("foo#{x}",x)
        puts rc.get("foo#{x}")
        rc.set("__last__",x)
    rescue => e
        puts "error #{e.to_s}"
    end
    sleep 0.1
}
```
这个程序看起来比通常需要的更复杂，因为它被设计成在屏幕上显示错误而不是直接抛出异常退出，所以每个集群操作都被包裹在 `begin rescue` 代码块中。

第 14 行是程序中第一个值得关注的地方。它创建了 Redis 集群对象，参数包括：启动节点列表（*startup_nodes*）、允许对不同节点建立的最大连接数，以及操作超时判定时间。

启动节点不需要包含集群所有节点，关键是要确保至少有一个节点可达。请注意 redis-rb-cluster 在成功连接第一个节点后就会立即更新这个启动节点列表，这是所有专业客户端都应具备的行为。

现在我们将Redis集群对象实例存储在变量 `rc` 中，之后就可以像使用普通 Redis 对象实例一样操作它。

18 到 26 行演示了这一点：当重启示例时，我们不希望从 `foo0` 重新开始，所以将计数器存储在 Redis 内部。这段代码专门用于读取该计数器，若不存在则将其初始化为零。

注意这里使用了 while 循环，因为即使集群宕机返回错误，我们也希望不断重试。普通应用不需要如此谨慎。

28 到 37 行开始了主循环，要么执行键值设置，要么显示错误信息。

注意循环末尾的 `sleep` 调用。在测试中若想尽快写入集群可以移除这个 `sleep` （当然这只是个没有真正并发的忙循环，最佳情况下通常能达到每秒 1 万次操作）。

通常情况下我们会放慢写入速度，这是为了让示例应用程序更便于人类观察和理解。

启动应用程序会产生以下输出：

```console
ruby ./example.rb
1
2
3
4
5
6
7
8
9
^C (I stopped the program here)
```
### Reshard the Cluster {#reshard-the-cluster}

现在我们可以尝试进行集群的**重新分片（resharding）**了。在操作期间，请保持 `example.rb` 程序持续运行，以便观察重新分片是否会对正在运行的程序产生影响。此外，您也可以考虑注释掉 `sleep` 调用，以便在重新分片过程中产生更高的写入负载。  

重新分片的核心操作是将哈希槽（hash slots）从一组节点迁移到另一组节点。与集群创建类似，这一过程可以通过 `redis-cli` 工具完成。  

要启动重新分片，只需输入以下命令：

```bash
redis-cli --cluster reshard 127.0.0.1:7000
```

你只需指定一个节点即可，redis-cli 会自动发现其他节点。  

目前 redis-cli 仅支持在管理员协助下进行重新分片，无法直接指定“将此节点的 5% 槽迁移到另一个节点”（不过实现这一功能应该很简单）。因此，它会以提问的方式开始。第一个问题是你想迁移多少哈希槽:

```console
How many slots do you want to move (from 1 to 16384)?
```

我们可以尝试迁移 1000 个哈希槽。如果示例程序仍在运行且未包含 `sleep` 调用，那么这些槽中应该已经包含了相当数量的键。  

接下来，redis-cli 需要知道迁移的目标节点，即接收这些哈希槽的节点。这里我们选择第一个主节点 `127.0.0.1:7000`，但需要指定该实例的 **Node ID**。虽然 redis-cli 之前输出的节点列表中已包含此信息，但如果需要，我们始终可以通过以下命令查询任意节点的 ID：

```console
$ redis-cli -p 7000 cluster nodes | grep myself
97a3a64667477371c4479320d683e4c8db5858b1 :0 myself,master - 0 0 0 connected 0-5460
```
OK，我的目标节点是 `97a3a64667477371c4479320d683e4c8db5858b1`。  

接下来，系统会询问你想从哪些节点转移这些哈希槽。我将输入 `all`，表示从所有其他主节点中各取一部分槽。  

在最终确认后，你会看到 redis-cli 针对每个待迁移槽的提示信息，并且每实际迁移一个键时会打印一个点作为进度标识。  

在重新分片过程中，你的示例程序应该能持续正常运行而不受影响。如果需要，你甚至可以多次停止并重启程序来验证这一点。  

重新分片完成后，可以通过以下命令检查集群的健康状态：

``` console
redis-cli --cluster check 127.0.0.1:7000
```
所有槽位仍会像往常一样被覆盖，但这次主节点 `127.0.0.1:7000` 将拥有更多的哈希槽，大约为 6461 个。  

重新分片操作也可以自动执行，无需以交互方式手动输入参数。通过以下命令行即可实现：

``` console
redis-cli --cluster reshard <host>:<port> --cluster-from <node-id> --cluster-to <node-id> --cluster-slots <number of slots> --cluster-yes
```

这样就能构建一些自动化流程（如果您需要频繁进行重新分片的话）。不过目前 redis-cli 还无法自动检查集群节点间的键分布情况，并智能地按需迁移槽位来实现集群再平衡。该功能将在未来版本中添加。  

`--cluster-yes` 选项会让集群管理器自动对操作提示回答 `yes`，从而以非交互模式运行。需要注意的是，您也可以通过设置 `REDISCLI_CLUSTER_YES` 环境变量来启用这一功能。

### A More Interesting Example Application {#a-more-interesting-example-application}
我们之前编写的示例程序并不完善。它只是简单地写入集群，甚至不会校验写入内容是否正确。  

从我们的角度来看，接收写入的集群可能每次都将键 `foo` 的值设为 `42`，而我们可能完全察觉不到异常。  

因此在 redis-rb-cluster 代码库中，有一个更有趣的应用程序 `consistency-test.rb`。它默认使用 1000 个计数器，通过发送 `INCR` 命令来递增这些计数器。  

但这个程序不仅仅是写入数据，还做了两件事：  

- 当使用 `INCR` 更新计数器时，程序会记录这次写入操作  
- 每次写入前，程序还会随机读取一个计数器，检查其值是否符合预期（与内存中记录的值进行对比）  

这意味着这个程序是一个简单的**一致性检查器（consistency checker）**，能够判断集群是否丢失了某些写入，或者是否接受了未收到确认的写入。第一种情况我们会发现计数器的值比记录的要小，而第二种情况值会比记录的更大。  

运行 `consistency-test` 程序每秒会输出一行日志：

```console
$ ruby consistency-test.rb
925 R (0 err) | 925 W (0 err) |
5030 R (0 err) | 5030 W (0 err) |
9261 R (0 err) | 9261 W (0 err) |
13517 R (0 err) | 13517 W (0 err) |
17780 R (0 err) | 17780 W (0 err) |
22025 R (0 err) | 22025 W (0 err) |
25818 R (0 err) | 25818 W (0 err) |
```

日志行会显示已执行的**读取次数**和**写入次数**，以及**错误数量**（因系统不可用而被拒绝的查询）。  

如果发现数据不一致，输出中会追加新的提示行。例如，当程序正在运行时，如果我手动重置某个计数器，会出现以下情况：

```console
$ redis-cli -h 127.0.0.1 -p 7000 set key_217 0
OK

(in the other tab I see...)

94774 R (0 err) | 94774 W (0 err) |
98821 R (0 err) | 98821 W (0 err) |
102886 R (0 err) | 102886 W (0 err) | 114 lost |
107046 R (0 err) | 107046 W (0 err) | 114 lost |
```

当我将计数器重置为 0 时，其实际值应为 114，因此程序会报告 114 次写入丢失（即集群未记录的 INCR 命令）。  

这个测试程序能更有趣地验证集群行为，因此我们将用它来测试 Redis 集群的故障转移功能。

### Test the Failover {#test-the-failover}

要触发故障转移，最简单的方式（也是分布式系统中最基础的故障场景）就是让其中一个进程崩溃——在我们的场景中即终止一个主节点。

!!! tip "注意"

    在此测试过程中，你应该开启一个（终端）标签页运行着一致性测试应用程序。

我们可以通过以下命令识别主节点并使其崩溃：

```console
$ redis-cli -p 7000 cluster nodes | grep master
3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 127.0.0.1:7001 master - 0 1385482984082 0 connected 5960-10921
2938205e12de373867bf38f1ca29d31d0ddb3e46 127.0.0.1:7002 master - 0 1385482983582 0 connected 11423-16383
97a3a64667477371c4479320d683e4c8db5858b1 :0 myself,master - 0 0 0 connected 0-5959 10922-11422
```

OK，7000、7001 和 7002 都是主节点。现在我们用 `DEBUG SEGFAULT` 命令让节点 7002 崩溃：

```console
$ redis-cli -p 7002 debug segfault
Error: Server closed the connection
```

现在我们可以查看一致性测试的输出，看看它报告了什么。

```console
18849 R (0 err) | 18849 W (0 err) |
23151 R (0 err) | 23151 W (0 err) |
27302 R (0 err) | 27302 W (0 err) |

... many error warnings here ...

29659 R (578 err) | 29660 W (577 err) |
33749 R (578 err) | 33750 W (577 err) |
37918 R (578 err) | 37919 W (577 err) |
42077 R (578 err) | 42078 W (577 err) |
```

可以看到，在故障转移过程中，系统有 578 次读取和 577 次写入未能执行，但数据库中没有产生任何不一致。这可能听起来有些意外，因为在本教程第一部分我们说过，Redis 集群在故障转移期间可能会丢失写入，因为它使用的是异步复制。但我们没有说明的是，这种情况实际上不太容易发生，因为 Redis 几乎会同时向客户端发送响应和向副本节点发送复制命令，所以出现数据丢失的时间窗口非常小。不过虽然很难触发，但这并非完全不可能，因此这并不会改变 Redis 集群提供的一致性保证。

现在我们可以检查故障转移后的集群状态（注意，在此期间我已经重启了之前崩溃的实例，使其作为副本重新加入集群）：

```console
$ redis-cli -p 7000 cluster nodes
3fc783611028b1707fd65345e763befb36454d73 127.0.0.1:7004 slave 3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 0 1385503418521 0 connected
a211e242fc6b22a9427fed61285e85892fa04e08 127.0.0.1:7003 slave 97a3a64667477371c4479320d683e4c8db5858b1 0 1385503419023 0 connected
97a3a64667477371c4479320d683e4c8db5858b1 :0 myself,master - 0 0 0 connected 0-5959 10922-11422
3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 127.0.0.1:7005 master - 0 1385503419023 3 connected 11423-16383
3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 127.0.0.1:7001 master - 0 1385503417005 0 connected 5960-10921
2938205e12de373867bf38f1ca29d31d0ddb3e46 127.0.0.1:7002 slave 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 0 1385503418016 3 connected
```
现在主节点运行在 7000、7001 和 7005 端口上。原先作为主节点运行在 7002 端口的 Redis 实例，现在成为了 7005 的副本。

`CLUSTER NODES` 命令的输出看起来可能很复杂，但其实非常简单，由以下几个部分组成：

<div class="annotate" markdown>
- 节点ID （Node ID）
- IP 地址:端口号 （ip:port）
- 状态标识：master（主节点）、replica（副本节点，旧版本称为 **slave**）、myself（当前节点）、fail（故障节点）等
- 如果是副本节点，会显示其主节点的 ID
- 最后一次待应答的 `PING` 命令的等待时间
- 最后一次收到 `PONG` 回复的时间戳
- 该节点的配置版本号(1)（详见集群规范）
- 与该节点的连接状态
- 负责的哈希槽(2)...
</div>

1. 原文用词为 epoch，推测是版本号的意思，因为配置会自动更新，所以会有多个版本的配置。
2. 似乎只有主节点显示

### Manual Failover {#manual-failover}
有时候，在不影响主节点正常运行的情况下强制进行故障转移是有用的。例如，为了升级某个主节点的 Redis 进程，可以将其故障转移为副本节点，从而最大限度地减少对可用性的影响。  

Redis 集群支持通过 `CLUSTER FAILOVER` 命令执行手动故障转移，该命令必须在目标主节点的某个副本节点上执行。  

手动故障转移是特殊的，相比因主节点真正故障而触发的故障转移，它更加安全。这种方式能确保在切换过程中不会丢失数据——只有当系统确认新主节点已处理完来自旧主节点的所有复制流后，才会将客户端从原主节点切换到新主节点。  

执行手动故障转移时，你会在副本日志中看到如下信息：

```console
# Manual failover user request accepted.
# Received replication offset for paused master manual failover: 347540
# All master replication stream processed, manual failover can start.
# Start of election delayed for 0 milliseconds (rank #0, offset 347540).
# Starting a failover election for epoch 7545.
# Failover election won: I'm the new master.
```

基本上，连接到正在进行故障转移的主节点的客户端会被暂时阻塞。与此同时，主节点会将自己的复制偏移量发送给副本节点，副本节点会等待自身达到该偏移量。当副本节点的复制偏移量达到目标值时，故障转移流程正式启动，原主节点会收到配置切换的通知。当原主节点上的客户端解除阻塞后，它们会被重定向到新的主节点。

!!! tip "注意"

    若要将一个副本提升为主节点，首先必须确保集群中大多数主节点已将该副本识别为合法副本。否则，该副本将无法在故障转移选举中胜出。如果该副本是刚刚加入集群的新节点（参见 [Add a new node as a replica](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/#add-a-new-node-as-a-replica)），可能需要等待一段时间再发送 `CLUSTER FAILOVER` 命令，以确保集群中的主节点能够感知到新副本的存在。

### Add a New Node {#add-a-new-node}
### Remove a Node {#remove-a-node}
### Replica Migration {#replica-migration}
### Upgrade Nodes in a Redis Cluster {#upgrade-nodes-in-a-redis-cluster}
### Migrate to Redis Cluster {#migrate-to-redis-cluster}
