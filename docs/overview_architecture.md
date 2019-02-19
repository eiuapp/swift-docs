
Swift Architectural Overview
----------------------------

updated: 2019-02-14 02:34

Swift Architectural Overview[¶](#swift-architectural-overview "Permalink to this headline")
===========================================================================================

Proxy Server[¶](#proxy-server "Permalink to this headline")
-----------------------------------------------------------

The Proxy Server is responsible for tying together the rest of the Swift architecture. For each request, it will look up the location of the account, container, or object in the ring (see below) and route the request accordingly. For Erasure Code type policies, the Proxy Server is also responsible for encoding and decoding object data. See [Erasure Code Support](overview_erasure_code.html) for complete information on Erasure Code support. The public API is also exposed through the Proxy Server.

代理服务器(Proxy Server)负责将Swift架构的其余部分捆绑在一起。对于每个请求，它将查找环中的帐户，容器或对象的位置（参见下文）并相应地路由请求。对于Erasure Code类型策略，Proxy Server还负责编码和解码对象数据。有关Erasure Code支持的完整信息，请参阅[Erasure Code Support](overview_erasure_code.html)。公共API也通过代理服务器公开。

A large number of failures are also handled in the Proxy Server. For example, if a server is unavailable for an object PUT, it will ask the ring for a handoff server and route there instead.

代理服务器中也处理了大量故障。例如，如果服务器不可用于对象PUT操作，它将向环请求切换服务器并在那里路由。

When objects are streamed to or from an object server, they are streamed directly through the proxy server to or from the user – the proxy server does not spool them.

当对象流式传输到对象服务器或从对象服务器流式传出时，它们直接通过代理服务器流入或流出用户 - 代理服务器不会对它们进行假脱机（spool）。

The Ring[¶](#the-ring "Permalink to this headline")
---------------------------------------------------

A ring represents a mapping between the names of entities stored on disk and their physical location. There are separate rings for accounts, containers, and one object ring per storage policy. When other components need to perform any operation on an object, container, or account, they need to interact with the appropriate ring to determine its location in the cluster.

环表示存储在磁盘上的实体名称与其物理位置之间的映射。每个存储策略都有一个单独的帐户，容器和对象环。当其他组件需要对 对象，容器或帐户 执行任何操作时，他们需要与相应的环进行交互以确定其在群集中的位置。

The Ring maintains this mapping using zones, devices, partitions, and replicas. Each partition in the ring is replicated, by default, 3 times across the cluster, and the locations for a partition are stored in the mapping maintained by the ring. The ring is also responsible for determining which devices are used for handoff in failure scenarios.

Ring使用zones，设备，分区和副本来维护此映射。默认情况下，环中的每个分区都会在群集中复制3次（译注：复制成3份，总共3份），并且分区的位置将存储在由环维护的映射中。环还负责确定在故障情况下哪些设备用于切换。

The replicas of each partition will be isolated onto as many distinct regions, zones, servers and devices as the capacity of these failure domains allow. If there are less failure domains at a given tier than replicas of the partition assigned within a tier (e.g. a 3 replica cluster with 2 servers), or the available capacity across the failure domains within a tier are not well balanced it will not be possible to achieve both even capacity distribution (balance) as well as complete isolation of replicas across failure domains (dispersion). When this occurs the ring management tools will display a warning so that the operator can evaluate the cluster topology.

每个分区的副本将被隔离到尽可能多的不同regions，zones，服务器和设备上，因为这些故障域的容量允许（译注：这个地方翻译得拗口，其实就是“由于故障区域的允许的容量”）。如果给定层上的故障域数量少于层内分配的分区的副本数量（例如，具有2个服务器的3个副本群集），或者层内故障域的可用容量未得到很好的平衡，则无法实现均匀的容量分配（平衡）以及跨故障域（分散度dispersion）完全隔离replicas。发生这种情况时，环管理工具将显示警告，以便操作员可以评估集群拓扑。

Data is evenly distributed across the capacity available in the cluster as described by the devices weight. Weights can be used to balance the distribution of partitions on drives across the cluster. This can be useful, for example, when different sized drives are used in a cluster. Device weights can also be used when adding or removing capacity or failure domains to control how many partitions are reassigned during a rebalance to be moved as soon as replication bandwidth allows.

数据均匀分布在群集中可用的容量中，如设备权重（weight）所述。权重可用于平衡群集中驱动器上的分区分布。例如，当在群集中使用不同大小的驱动器时，这可能很有用。在添加或删除容量或故障域时，还可以使用设备权重来控制在复制带宽允许的情况下在重新平衡期间重新分配的分区数。

Note

Prior to Swift 2.1.0 it was not possible to restrict partition movement by device weight when adding new failure domains, and would allow extremely unbalanced rings. The greedy dispersion algorithm is now subject to the constraints of the physical capacity in the system, but can be adjusted with-in reason via the overload option. Artificially unbalancing the partition assignment without respect to capacity can introduce unexpected full devices when a given failure domain does not physically support its share of the used capacity in the tier.

注意

在Swift 2.1.0之前，在添加新的故障域时，不可能通过设备权重限制分区移动，并且允许极不平衡的环。贪婪的分散度算法（greedy dispersion algorithm）现在受制于系统中物理容量的约束，但可以通过过载选项进行合理调整。在给定的故障域不能物理上支持其在层中使用的容量的份额时，人为地不平衡分区分配而不考虑容量会引入意外的完整设备（unexpected full devices）。

When partitions need to be moved around (for example if a device is added to the cluster), the ring ensures that a minimum number of partitions are moved at a time, and only one replica of a partition is moved at a time.

当需要移动分区时（例如，如果将设备添加到群集中），环确保一次移动最少数量的分区，并且一次只移动一个分区的一个副本。

The ring is used by the Proxy server and several background processes (like replication). See [The Rings](overview_ring.html) for complete information on the ring.

代理服务器和几个后台进程（如replication）使用环。有关戒指的完整信息，请参阅[The Rings](overview_ring.html)。

Storage Policies[¶](#storage-policies "Permalink to this headline")
-------------------------------------------------------------------

Storage Policies provide a way for object storage providers to differentiate service levels, features and behaviors of a Swift deployment. Each Storage Policy configured in Swift is exposed to the client via an abstract name. Each device in the system is assigned to one or more Storage Policies. This is accomplished through the use of multiple object rings, where each Storage Policy has an independent object ring, which may include a subset of hardware implementing a particular differentiation.

存储策略为对象存储提供者（object storage providers）提供了一种区分Swift部署的服务级别，功能和行为的方法。Swift中配置的每个存储策略都通过抽象名称公开给客户端。系统中的每个设备都分配给一个或多个存储策略。这是通过使用多个对象环来实现的，其中每个存储策略具有独立的对象环，其可以包括实现特定区分的硬件子集。

（译注：每一个 device 都至少有一个存储策略。每一个存储策略都对应于一个对象环（不是容器环，不是账户环），但是，后文知道，这个存储策略更多的是与容器环相关联，在创建容器时，定义好存储策略，然后容器内的对象根据该定义策略来存储。）

For example, one might have the default policy with 3x replication, and create a second policy which, when applied to new containers only uses 2x replication. Another might add SSDs to a set of storage nodes and create a performance tier storage policy for certain containers to have their objects stored there. Yet another might be the use of Erasure Coding to define a cold-storage tier.

例如，可以使用具有3x复制的默认策略，并创建第二个策略，当应用于新容器时，仅使用2x复制。另一个例子，可能会将SSD添加到一组存储节点，并为某些容器创建性能层存储策略，以将其对象存储在那里。另一个例子，可能是使用Erasure Coding来定义冷存储层。

This mapping is then exposed on a per-container basis, where each container can be assigned a specific storage policy when it is created, which remains in effect for the lifetime of the container. Applications require minimal awareness of storage policies to use them; once a container has been created with a specific policy, all objects stored in it will be done so in accordance with that policy.

然后，基于每个容器公开此映射，其中每个容器在创建时都可以分配特定的存储策略，该策略在容器的生命周期内保持有效。应用程序需要很少的存储策略意识才能使用它们; 一旦使用特定策略创建了容器，则存储在其中的所有对象将根据该策略完成。

The Storage Policies feature is implemented throughout the entire code base so it is an important concept in understanding Swift architecture.

存储策略功能在整个代码库中实现，因此它是理解Swift体系结构的重要概念。

See [Storage Policies](overview_policies.html) for complete information on storage policies.

有关存储策略的完整信息，请参阅存储策略。

Object Server[¶](#object-server "Permalink to this headline")
-------------------------------------------------------------

The Object Server is a very simple blob storage server that can store, retrieve and delete objects stored on local devices. Objects are stored as binary files on the filesystem with metadata stored in the file’s extended attributes (xattrs). This requires that the underlying filesystem choice for object servers support xattrs on files. Some filesystems, like ext3, have xattrs turned off by default.

对象服务（Object Server）是一个非常简单的blob存储服务器，可以存储，检索和删除存储在本地设备上的对象。对象作为二进制文件存储在文件系统中，元数据存储在文件的扩展属性（xattrs）中（译注：所以，当你在找到了对象后，仅通过`ls -l`这样的命令是看不到对象元数据的）。这要求对象服务器的基础文件系统选择支持文件上的xattrs。某些文件系统（如ext3）默认关闭xattrs。

Each object is stored using a path derived from the object name’s hash and the operation’s timestamp. Last write always wins, and ensures that the latest object version will be served. A deletion is also treated as a version of the file (a 0 byte file ending with “.ts”, which stands for tombstone). This ensures that deleted files are replicated correctly and older versions don’t magically reappear .

使用从对象名称的hash和操作的时间戳派生的路径存储每个对象。上次写入总是获胜，并确保将提供最新的对象版本（译注：这里只是确保提供最新，不是一定提供最新，这也是最终一致性的体现。如，写一个文件后，并不一定马上（比如下一秒钟），就得到这个最新的写入数据，但是，5分钟后，就一定能得到这个最新的写入数据）。删除也被视为文件的一个版本（一个以“.ts”结尾的0字节文件，代表墓碑tombstone）。这可确保正确复制已删除的文件，旧版本不会神奇地重新出现 due to failure scenarios。

Container Server[¶](#container-server "Permalink to this headline")
-------------------------------------------------------------------

The Container Server’s primary job is to handle listings of objects. It doesn’t know where those object’s are, just what objects are in a specific container. The listings are stored as sqlite database files, and replicated across the cluster similar to how objects are. Statistics are also tracked that include the total number of objects, and total storage usage for that container.

Container Server的主要工作是处理对象列表。它不知道那些对象在哪里，只知道特定容器中的对象。列表存储为sqlite数据库文件，并在集群中复制，类似于对象的方式。还会跟踪统计信息，其中包括对象总数以及该容器的总存储使用情况。

Account Server[¶](#account-server "Permalink to this headline")
---------------------------------------------------------------

The Account Server is very similar to the Container Server, excepting that it is responsible for listings of containers rather than objects.

Account Server与Container服务非常相似，不同之处在于它负责容器而不是对象的列表。

Replication[¶](#replication "Permalink to this headline")
---------------------------------------------------------

Replication is designed to keep the system in a consistent state in the face of temporary error conditions like network outages or drive failures.

复制旨在使系统在面临网络中断或驱动器故障等临时错误情况时保持一致状态。

The replication processes compare local data with each remote copy to ensure they all contain the latest version. Object replication uses a hash list to quickly compare subsections of each partition, and container and account replication use a combination of hashes and shared high water marks.

复制过程将本地数据与每个远程副本进行比较，以确保它们都包含最新版本。对象复制使用哈希列表快速比较每个分区的子部分，容器和帐户复制使用哈希和共享高水位标记的组合。

Replication updates are push based. For object replication, updating is just a matter of rsyncing files to the peer. Account and container replication push missing records over HTTP or rsync whole database files.

复制更新是基于推送的。对于对象复制，更新只是将文件发送到对等方的事件。帐户和容器复制通过HTTP或rsync推送整个数据库文件丢失的记录。

The replicator also ensures that data is removed from the system. When an item (object, container, or account) is deleted, a tombstone is set as the latest version of the item. The replicator will see the tombstone and ensure that the item is removed from the entire system.

复制器(replicator)还确保从系统中删除数据。删除项目（对象，容器或帐户）时，会将逻辑删除设置为项目的最新版本。复制器将看到墓碑，并确保从整个系统中删除该项目。

See [Replication](overview_replication.html) for complete information on replication.

有关复制的完整信息，请参见[Replication](overview_replication.html)。

Reconstruction[¶](#reconstruction "Permalink to this headline")
---------------------------------------------------------------

The reconstructor is used by Erasure Code policies and is analogous to the replicator for Replication type policies. See [Erasure Code Support](overview_erasure_code.html) for complete information on both Erasure Code support as well as the reconstructor.

重构器（reconstructor）由Erasure Code策略使用，类似于复制类型策略的复制器。有关Erasure Code支持 以及重建器的完整信息，请参阅[Erasure Code Support](overview_erasure_code.html)。

Updaters[¶](#updaters "Permalink to this headline")
---------------------------------------------------

There are times when container or account data can not be immediately updated. This usually occurs during failure scenarios or periods of high load. If an update fails, the update is queued locally on the filesystem, and the updater will process the failed updates. This is where an eventual consistency window will most likely come in to play. For example, suppose a container server is under load and a new object is put in to the system. The object will be immediately available for reads as soon as the proxy server responds to the client with success. However, the container server did not update the object listing, and so the update would be queued for a later update. Container listings, therefore, may not immediately contain the object.

有时无法立即更新容器或帐户数据。这通常发生在故障情况或高负载期间。如果更新失败，则更新将在文件系统上本地排队，更新程序将处理失败的更新。这是最有可能进入最终一致性窗口（an eventual consistency window）的地方。例如，假设容器服务器处于负载状态，并且新对象被放入系统。一旦代理服务器成功响应客户端，该对象将立即可用于读取。但是，容器服务器未更新对象列表，因此更新将排队等待以后的更新。因此，容器列表可能不会立即包含该对象。

In practice, the consistency window is only as large as the frequency at which the updater runs and may not even be noticed as the proxy server will route listing requests to the first container server which responds. The server under load may not be the one that serves subsequent listing requests – one of the other two replicas may handle the listing.

实际上，一致性窗口仅与更新程序运行的频率一样大，甚至可能没有被注意到，因为代理服务器将列表请求路由到响应的第一个容器服务器。加载的服务器可能不是服务后续列表请求的服务器 - 另外两个副本中的一个可以处理列表。

Auditors[¶](#auditors "Permalink to this headline")
---------------------------------------------------

Auditors crawl the local server checking the integrity of the objects, containers, and accounts. If corruption is found (in the case of bit rot, for example), the file is quarantined, and replication will replace the bad file from another replica. If other errors are found they are logged (for example, an object’s listing can’t be found on any container server it should be).

审计员抓取本地服务器，检查对象，容器和帐户的完整性。如果发现损坏（例如，在发生损坏的情况下），则会隔离该文件，并且将从另一个副本复制替换该错误文件。如果发现其他错误，则会记录它们（例如，在任何容器服务器上都找不到对象的列表）。

updated: 2019-02-14 02:34
