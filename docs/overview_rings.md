+++
title = "官方中文文档 openstack swift overview rings"
date = 2019-01-21T00:00:00-08:00
lastmod = 2019-01-22T02:35:17-08:00
tags = ["swift", "transilation"]
categories = ["swift"]
draft = false
weight = 3001
+++

<https://docs.openstack.org/swift/queens/overview_ring.html>

OpenStack Docs: The Rings                  

The Rings
---------

updated: 2019-01-15 02:30

The Rings[¶](#the-rings "Permalink to this headline")
=====================================================

The rings determine where data should reside in the cluster. There is a separate ring for account databases, container databases, and individual object storage policies but each ring works in the same way. These rings are externally managed. The server processes themselves do not modify the rings; they are instead given new rings modified by other tools.

环确定数据应驻留在集群中的位置。帐户数据库，容器数据库和单个对象存储策略有一个单独的环，但每个环以相同的方式工作。这些环是外部管理的。服务进程本身不会修改环; 相反，它们被赋予了由其他工具修改的新环。(译注：ring不能被修改，只能产生新ring)

The ring uses a configurable number of bits from the MD5 hash of an item’s path as a partition index that designates the device(s) on which that item should be stored. The number of bits kept from the hash is known as the partition power, and 2 to the partition power indicates the partition count. Partitioning the full MD5 hash ring allows the cluster components to process resources in batches. This ends up either more efficient or at least less complex than working with each item separately or the entire cluster all at once.

（译注：关于下面2段文字，先看[这里](https://b.qqbb.app/post/analysis-openstack-swift-overview-ring/)，可能效果要好很多。我第一次看的时候，全蒙）

环使用来自项目路径的MD5哈希的可配置位数作为分区索引（partition index），该分区索引指定应该存储该项目的设备。从散列中保留的比特数称为分区功率（partition power），2的分区功率次方表示分区总数（partition count）。对完整MD5哈希环进行分区允许群集组件批量处理资源。与单独处理每个项目或同时处理整个集群相比，这最终会更有效或至少更简单。(译注：比如[initial-rings](https://docs.openstack.org/swift/queens/install/initial-rings.html#create-account-ring)时，你配置的是`swift-ring-builder account.builder create 10 3 1`, 则partition power是10，partition count是1024=2**10)

Another configurable value is the replica count, which indicates how many devices to assign for each partition in the ring. By having multiple devices responsible for each partition, the cluster can recover from drive or network failures.

另一个可配置的值是副本数(译注：同理，此配置副本数是3)，它指示为环中的每个分区分配的设备数量。通过让多个设备负责每个分区，群集可以从驱动器或网络故障中恢复。

Devices are added to the ring to describe the capacity available for partition replica assignments. Devices are placed into failure domains consisting of region, zone, and server. Regions can be used to describe geographical systems characterized by lower bandwidth or higher latency between machines in different regions. Many rings will consist of only a single region. Zones can be used to group devices based on physical locations, power separations, network separations, or any other attribute that would lessen multiple replicas being unavailable at the same time.

将设备添加到环中以描述可用于分区副本分配的容量。设备被置于由region, zone, 和 server组成的故障域中。Regions可用于描述以不同区域中的机器之间的较低带宽或较高等待时间为特征的地理系统。许多环只有一个region。Zones可用于根据物理位置，电源分离，网络分离或任何其他属性来对设备进行分组，这些属性可以减少同一时间多个副本都不可用的情形发生。

Devices are given a weight which describes the relative storage capacity contributed by the device in comparison to other devices.

给予设备权重，该权重描述了设备与其他设备相比所贡献的相对存储容量。

When building a ring, replicas for each partition will be assigned to devices according to the devices’ weights. Additionally, each replica of a partition will preferentially be assigned to a device whose failure domain does not already have a replica for that partition. Only a single replica of a partition may be assigned to each device - you must have at least as many devices as replicas.

构建环时，将根据设备的权重为每个分区分配副本。此外，分区的每个副本将优先分配给其故障域尚未具有该分区的副本的设备。只能为每个设备分配一个分区的单个副本 - 您必须至少拥有与副本一样多的设备。（译注：也就是设备数量最少为3）

Ring Builder[¶](#ring-builder "Permalink to this headline")
-----------------------------------------------------------

The rings are built and managed manually by a utility called the ring-builder. The ring-builder assigns partitions to devices and writes an optimized structure to a gzipped, serialized file on disk for shipping out to the servers. The server processes check the modification time of the file occasionally and reload their in-memory copies of the ring structure as needed. Because of how the ring-builder manages changes to the ring, using a slightly older ring usually just means that for a subset of the partitions the device for one of the replicas will be incorrect, which can be easily worked around.

环由称为环构建器的实用程序手动构建和管理。环构建器将分区分配给设备，并将优化的结构的gzip压缩序列化文件写入磁盘上，以便传送到servers。服务进程偶尔检查文件的修改时间，并根据需要重新加载环结构的内存副本。由于环构建器管理环的变化，使用稍微较旧的环通常只意味着对于一个分区的子集，其中一个副本的设备将是不正确的，这可以很容易地解决。

The ring-builder also keeps a separate builder file which includes the ring information as well as additional data required to build future rings. It is very important to keep multiple backup copies of these builder files. One option is to copy the builder files out to every server while copying the ring files themselves. Another is to upload the builder files into the cluster itself. Complete loss of a builder file will mean creating a new ring from scratch, nearly all partitions will end up assigned to different devices, and therefore nearly all data stored will have to be replicated to new locations. So, recovery from a builder file loss is possible, but data will definitely be unreachable for an extended time.

环构建器还保留单独的构建器文件，其中包括环信息以及构建未来环所需的其他数据。保留这些构建器文件的多个备份副本非常重要。一种选择是将构建器文件复制到每个服务器的时候，同时复制环文件。另一种方法是将构建器文件上载到群集本身。完全丢失构建器文件意味着从头开始创建新环，几乎所有分区最终都将分配给不同的设备，因此几乎所有存储的数据都必须复制到新位置。因此，可以从构建器文件丢失中恢复，但数据肯定会在较长时间内无法访问。

Ring Data Structure[¶](#ring-data-structure "Permalink to this headline")
-------------------------------------------------------------------------

The ring data structure consists of three top level fields: a list of devices in the cluster, a list of lists of device ids indicating partition to device assignments, and an integer indicating the number of bits to shift an MD5 hash to calculate the partition for the hash.

环数据结构由三个顶级字段组成：集群中的设备列表，指出分区分配到设备的设备ID列表（译注：下称“分区分配列表”），以及一个整数。这个整数，用于，计算分区哈希过程中的一个MD5哈希的移位位数(译注：源码中是_part_shift，配置示例就是：22=32-10)。

（译注：根据下文知道，这三个字段名，在源码中，分别为：`devs`，`_replica2part2dev_id`，`_part_shift`）

### List of Devices[¶](#list-of-devices "Permalink to this headline")

The list of devices is known internally to the Ring class as `devs`. Each item in the list of devices is a dictionary with the following keys:

Ring类内部已知设备列表用`devs`表示。设备列表中的每个项目都是包含以下键的字典：（译注：示例可看[这里](/post/analysis-openstack-swift-overview-ring/) ）

|   |          |    |
|:----------|:-------------:|:------|
|id	|integer|	The index into the list of devices.|
|zone|	integer|	The zone in which the device resides.|
|region|	integer|	The region in which the zone resides.|
|weight	|float|	The relative weight of the device in comparison to other devices. This usually corresponds directly to the amount of disk space the device has compared to other devices. For instance a device with 1 terabyte of space might have a weight of 100.0 and another device with 2 terabytes of space might have a weight of 200.0. This weight can also be used to bring back into balance a device that has ended up with more or less data than desired over time. A good average weight of 100.0 allows flexibility in lowering the weight later if necessary.|
|ip	|string|	The IP address or hostname of the server containing the device.|
|port|	int|	The TCP port on which the server process listens to serve requests for the device.|
|device	|string|	The on-disk name of the device on the server. For example: sdb1|
|meta	|string|	A general-use field for storing additional information for the device. This information isn’t used directly by the server processes, but can be useful in debugging. For example, the date and time of installation and hardware manufacturer could be stored here.|


|   |          |    |
|:----------|:-------------:|:------|
|id	|integer|	索引到设备列表.|
|zone|	integer|	The zone in which the device resides.|
|region|	integer|	The region in which the zone resides.|
|weight	|float|	与其他设备相比，设备的相对重量。这通常直接对应于设备与其他设备相比的磁盘空间量。例如，具有1TB空间的设备可能具有100.0的权重，而具有2TB空间的另一设备可具有200.0的权重。这个权重还可以用来恢复一个设备，该设备最终会有比预期更多或更少的数据。良好的平均重量100.0可以在必要时灵活地降低重量。|
|ip	|string|	包含设备的服务器的IP地址或主机名。|
|port|	int|	服务器进程侦听的TCP端口为设备提供请求。|
|device	|string|	服务器上设备的磁盘名称。例如：sdb1|
|meta	|string|	用于存储设备的附加信息的通用字段。服务器进程不直接使用此信息，但在调试时可能很有用。例如，安装的日期和时间以及硬件制造商可以存储在此处。|


Note

The list of devices may contain holes, or indexes set to `None`, for devices that have been removed from the cluster. However, device ids are reused. Device ids are reused to avoid potentially running out of device id slots when there are available slots (from prior removal of devices). A consequence of this device id reuse is that the device id (integer value) does not necessarily correspond with the chronology of when the device was added to the ring. Also, some devices may be temporarily disabled by setting their weight to `0.0`. To obtain a list of active devices (for uptime polling, for example) the Python code would look like:

```python
devices = list(self._iter_devs())
```

注意

对于已从群集中删除的设备，设备列表可能包含holes或索引设置为`None`。但是，设备ID可以重复使用。重新使用设备ID以避免在有可用插槽（从先前移除设备）时可能耗尽设备ID插槽。此设备ID重用的结果是设备ID（整数值）不一定与设备添加到环中的时间顺序相对应。此外，可以通过将其权重设置为 `0.0`以暂时禁用某些设备。要获取活动设备列表（例如，用于正常运行时间轮询），Python代码将如下所示：

```python
devices = list(self._iter_devs())
```

### Partition Assignment List[¶](#partition-assignment-list "Permalink to this headline")

The partition assignment list is known internally to the Ring class as `_replica2part2dev_id`. This is a list of `array('H')`s, one for each replica. Each `array('H')` has a length equal to the partition count for the ring. Each integer in the `array('H')` is an index into the above list of devices.

分区分配列表在Ring类内部表示为 `_replica2part2dev_id`。这是`array('H')`的列表（list），每个副本一个。每个`array('H')`的长度等于环的分区数（译注：安装示例为：1024=2**10）。其中的`array('H')`上的每个整数都是上述的设备列表的索引（译注：也就是上面 List of Devices 小节中的表中的ID字段值）。

So, to create a list of device dictionaries assigned to a partition, the Python code would look like:

```python
devices = [self.devs[part2dev_id[partition]]
           for part2dev_id in self._replica2part2dev_id]
```

因此，创建分配给分区的设备列表的字典，Python代码将如下所示：

```python
devices = [self.devs[part2dev_id[partition]]
           for part2dev_id in self._replica2part2dev_id]
```

`array('H')` is used for memory conservation as there may be millions of partitions.

`array('H')` 用于内存保护，因为可能有数百万个分区。

### Partition Shift Value[¶](#partition-shift-value "Permalink to this headline")

The partition shift value is known internally to the Ring class as `_part_shift`. This value is used to shift an MD5 hash of an item’s path to calculate the partition on which the data for that item should reside. Only the top four bytes of the hash are used in this process. For example, to compute the partition for the path `/account/container/object`, the Python code might look like:

```python
objhash = md5('/account/container/object').digest()
partition = struct.unpack_from('>I', objhash)[0] >> self._part_shift
```

分区移位值在Ring类内部为 `_part_shift`。此值用于移动项目路径的MD5哈希值，以计算该项目的数据应驻留在的分区。在此过程中仅使用散列的前四个字节。例如，要计算路径的分区`/account/container/object`，Python代码可能如下所示：

```python
objhash = md5('/account/container/object').digest()
partition = struct.unpack_from('>I', objhash)[0] >> self._part_shift
```

For a ring generated with partition power `P`, the partition shift value is `32 - P`.

对于使用分区功率`P`生成的环，分区移位值为`32 - P` 。

### Fractional Replicas[¶](#fractional-replicas "Permalink to this headline")

A ring is not restricted to having an integer number of replicas. In order to support the gradual changing of replica counts, the ring is able to have a real number of replicas.

环不限于具有整数个副本。为了支持副本计数的逐渐变化，环能够具有实数值的副本。

When the number of replicas is not an integer, the last element of `_replica2part2dev_id` will have a length that is less than the partition count for the ring. This means that some partitions will have more replicas than others. For example, if a ring has `3.25` replicas, then 25% of its partitions will have four replicas, while the remaining 75% will have just three.

当副本的数量不是整数时，`_replica2part2dev_id` 最后一个元素的长度将小于环的分区数。这意味着某些分区将具有比其他分区更多的副本。例如，如果一个环有3.25副本，那么其25％的分区将有四个副本，而剩下的75％将只有三个副本。

（译注：3.25 = 4*0.25 + 3*0.75）

### Dispersion[¶](#dispersion "Permalink to this headline")

With each rebalance, the ring builder calculates a dispersion metric. This is the percentage of partitions in the ring that have too many replicas within a particular failure domain.

对于每个重新平衡，环构建器计算分散度。这是环中具有特定故障域内太多副本的分区的百分比。

For example, if you have three servers in a cluster but two replicas for a partition get placed onto the same server, that partition will count towards the dispersion metric.

例如，如果群集中有三台服务器，但分区的两个副本放置在同一服务器上，则该分区将计入分散度量标准。

（译注：因为这个分区，并不分散）

A lower dispersion value is better, and the value can be used to find the proper value for “overload”.

较低的分散度更好，并且该值可用于找到“overload”的适当值。

### Overload[¶](#overload "Permalink to this headline")

The ring builder tries to keep replicas as far apart as possible while still respecting device weights. When it can’t do both, the overload factor determines what happens. Each device may take some extra fraction of its desired partitions to allow for replica dispersion; once that extra fraction is exhausted, replicas will be placed closer together than is optimal for durability.

环构建器尝试尽可能远离replicas，同时仍然尊重设备weights。如果不能同时做到这两点，overload决定了会发生什么。每个设备可以占用其所需分区的一些额外部分以允许复制品分散; 一旦额外的部分耗尽，复制品将被放置在一起，而不是最佳的耐久性。

Essentially, the overload factor lets the operator trade off replica dispersion (durability) against device balance (uniform disk usage).

从本质上讲，过载因子可以让操作员在设备平衡（统一磁盘使用率）之间权衡复制分散（耐久性）。

The default overload factor is `0`, so device weights will be strictly followed.

默认的过载因子是`0`，因此将严格遵循设备权重。

With an overload factor of `0.1`, each device will accept 10% more partitions than it otherwise would, but only if needed to maintain dispersion.

在过载因子的情况下`0.1`，每个设备将接受比其他情况多10％的分区，但仅在需要保持分散时。

Example: Consider a 3-node cluster of machines with equal-size disks; let node A have 12 disks, node B have 12 disks, and node C have only 11 disks. Let the ring have an overload factor of `0.1` (10%).

示例：考虑具有相等大小磁盘的3节点计算机群集; 让节点A有12个磁盘，节点B有12个磁盘，节点C只有11个磁盘。让环的过载系数为0.1（10％）。

Without the overload, some partitions would end up with replicas only on nodes A and B. However, with the overload, every device is willing to accept up to 10% more partitions for the sake of dispersion. The missing disk in C means there is one disk’s worth of partitions that would like to spread across the remaining 11 disks, which gives each disk in C an extra 9.09% load. Since this is less than the 10% overload, there is one replica of each partition on each node.

如果没有过载，一些分区最终只会在节点A和B上出现副本。但是，由于过载，每个设备都愿意为了分散而接受多达10％的分区。C中丢失的磁盘意味着有一个磁盘的分区值要分布在剩余的11个磁盘上，这为C中的每个磁盘提供了额外的9.09％负载。由于这小于10％的过载，每个节点上的每个分区都有一个副本。

However, this does mean that the disks in node C will have more data on them than the disks in nodes A and B. If 80% full is the warning threshold for the cluster, node C’s disks will reach 80% full while A and B’s disks are only 72.7% full.

但是，这确实意味着节点C中的磁盘将拥有比节点A和B中的磁盘更多的数据。如果80％已满是群集的警告阈值，则节点C的磁盘将达到80％满，而A和B的磁盘只有72.7％已满。

Partition & Replica Terminology[¶](#partition-replica-terminology "Permalink to this headline")
-----------------------------------------------------------------------------------------------

All descriptions of consistent hashing describe the process of breaking the keyspace up into multiple ranges (vnodes, buckets, etc.) - many more than the number of “nodes” to which keys in the keyspace must be assigned. Swift calls these ranges partitions - they are partitions of the total keyspace.

所有关于一致性散列的描述都描述了将键空间（keyspace）分解为多个范围（vnode，buckets等）的过程 - 比必须分配键空间中的键的“节点”的数量多得多。Swift调用这些范围分区 - 它们是总键空间的分区。

Each partition will have multiple replicas. Every replica of each partition must be assigned to a device in the ring. When describing a specific replica of a partition (like when it’s assigned a device) it is described as a part-replica in that it is a specific replica of the specific partition. A single device will likely be assigned different replicas from many partitions, but it may not be assigned multiple replicas of a single partition.

每个分区都有多个副本。必须将每个分区的每个副本分配给环中的设备。在描述分区的特定副本时（例如，当它被分配设备时），它被描述为 部分副本(part-replica)，因为它是特定分区的特定副本。单个设备可能会从许多分区分配不同的副本，但可能不会为单个分区分配多个副本。

(译注：这个是肯定的。因为，创造出 partition的目的，就是为了让单个设备存放多个partition。而单个partition都有一个唯一的 partition index，同时：在xfs文件系统中，同一个文件夹下，不允许有2个同名文件夹，所以单个设备不会为单个分区分配多个副本。在上面提过的[示例中](https://b.qqbb.app/post/analysis-openstack-swift-overview-ring/)，在`/src/node/sdb/objects/`下，不能同时有多个partiton index为`119`的文件夹)

The total number of partitions in a ring is calculated as `2 ** <part-power>`. The total number of part-replicas in a ring is calculated as `<replica-count> * 2 ** <part-power>`.

环中分区的总数计算为`2 ** <part-power>`。环中部分副本的总数计算为`<replica-count> * 2 ** <part-power>` 。（译注：安装示例，分区总数:`2**10`，part-replicas总数为 `3*2**10`)

When considering a device’s weight it is useful to describe the number of part-replicas it would like to be assigned. A single device, regardless of weight, will never hold more than `2 ** <part-power>` part-replicas because it can not have more than one replica of any partition assigned. The number of part-replicas a device can take by weights is calculated as its parts-wanted. The true number of part-replicas assigned to a device can be compared to its parts-wanted similarly to a calculation of percentage error - this deviation in the observed result from the idealized target is called a device’s balance.

在考虑设备的权重时，描述它希望分配的部分副本的数量是有用的。无论weight如何，单个设备永远不会超过`2 ** <part-power>`个部分副本，因为它不能分配任何一个分区的多个副本给同一个设备。设备可以按weights计算的部分副本的数量按其parts-wanted计算。分配给设备的部分副本的真实数量可以与其parts-wanted进行比较，类似于计算百分比误差 - 观察结果与理想目标的偏差称为设备平衡（balance）。

When considering a device’s failure domain it is useful to describe the number of part-replicas it would like to be assigned. The number of part-replicas wanted in a failure domain of a tier is the sum of the part-replicas wanted in the failure domains of its sub-tier. However, collectively when the total number of part-replicas in a failure domain exceeds or is equal to `2 ** <part-power>` it is most obvious that it’s no longer sufficient to consider only the number of total part-replicas, but rather the fraction of each replica’s partitions. Consider for example a ring with 3 replicas and 3 servers: while dispersion requires that each server hold only ⅓ of the total part-replicas, placement is additionally constrained to require `1.0` replica of _each_ partition per server. It would not be sufficient to satisfy dispersion if two devices on one of the servers each held a replica of a single partition, while another server held none. By considering a decimal fraction of one replica’s worth of partitions in a failure domain we can derive the total part-replicas wanted in a failure domain (`1.0 * 2 ** <part-power>`). Additionally we infer more about which part-replicas must go in the failure domain. Consider a ring with three replicas and two zones, each with two servers (four servers total). The three replicas worth of partitions will be assigned into two failure domains at the zone tier. Each zone must hold more than one replica of some partitions. We represent this improper fraction of a replica’s worth of partitions in decimal form as `1.5` (`3.0 / 2`). This tells us not only the _number_ of total partitions (`1.5 * 2 ** <part-power>`) but also that _each_ partition must have at least one replica in this failure domain (in fact `0.5` of the partitions will have 2 replicas). Within each zone the two servers will hold `0.75` of a replica’s worth of partitions - this is equal both to “the fraction of a replica’s worth of partitions assigned to each zone (`1.5`) divided evenly among the number of failure domains in its sub-tier (2 servers in each zone, i.e. `1.5 / 2`)” but _also_ “the total number of replicas (`3.0`) divided evenly among the total number of failure domains in the server tier (2 servers × 2 zones = 4, i.e. `3.0 / 4`)”. It is useful to consider that each server in this ring will hold only `0.75` of a replica’s worth of partitions which tells that any server should have at most one replica of a given partition assigned. In the interests of brevity, some variable names will often refer to the concept representing the fraction of a replica’s worth of partitions in decimal form as _replicanths_ - this is meant to invoke connotations similar to ordinal numbers as applied to fractions, but generalized to a replica instead of a four\*th\* or a fif\*th\*. The “n” was probably thrown in because of Blade Runner.

在考虑设备的故障域时，描述它希望分配的部分副本的数量是有用的。在层的故障域中需要的部分副本的数量是其子层的故障域中所需的部分副本的总和。但是，当故障域中的部分副本的总数超过或等于`2 ** <part-power>`时，最明显的是，仅考虑总部分副本的数量，而不是仅考虑每个副本的分区的分数，这已经不够了。例如，考虑一个具有3个副本和3个服务器的环：虽然分散度要求每个服务器仅保留总部分副本中的1/3，但是，另外，placement必须限制为 _each_ 服务器需要`1.0`个副本。如果其中一个服务器上的两个设备都拥有单个分区的副本，而另一个服务器没有保留，则是不够满足分散度的。通过考虑故障域中一个副本的分区的小数部分，我们可以导出故障域中所需的总部分副本(`1.0 * 2 ** <part-power>`)。此外，我们更多地推断出哪些部分副本必须在故障域中进行。考虑一个带有三个副本和两个zones的环，每个zones有两个服务器（总共四个服务器）。三个副本的分区将分配到zones层的两个故障域。每个zones必须包含某些分区的多于1个副本。We represent this improper fraction of a replica’s worth of partitions in decimal form as `1.5` (`3.0 / 2`). 这就告诉我们，不仅总分区数量（`1.5 * 2 ** <part-power>`），而且 _each_ 分区在该故障域中必须至少有一个副本（其实`0.5` of the partitions将有2个副本）。Within each zone the two servers will hold `0.75` of a replica’s worth of partitions - this is equal both to “the fraction of a replica’s worth of partitions assigned to each zone (`1.5`) divided evenly among the number of failure domains in its sub-tier (2 servers in each zone, i.e. `1.5 / 2`)” but _also_ “the total number of replicas (`3.0`) divided evenly among the total number of failure domains in the server tier (2 servers × 2 zones = 4, i.e. `3.0 / 4`)”.考虑到此环中的每个服务器仅0.75包含`0.75`副本的分区，这有助于告知任何服务器最多只能分配给定分区一个副本。为了简洁起见，一些变量名称通常将表示replica的十进制形式的分区值的概念称为 _replicanths_ - 这意味着invoke connotations，类似于应用于分数的有序数，但是副本一般化为1，而不是 a four*th* or a fif*th*。The “n” was probably thrown in because of Blade Runner.

（译注：讲真的，上面这一段，不知道在讲啥）

Building the Ring[¶](#building-the-ring "Permalink to this headline")
---------------------------------------------------------------------

First the ring builder calculates the replicanths wanted at each tier in the ring’s topology based on weight.

首先，环构建器根据权重计算环的拓扑中每层所需的replicanths。(译注：也就是计算每层应该有的 replica的理论值)

Then the ring builder calculates the replicanths wanted at each tier in the ring’s topology based on dispersion.

然后，环构建器根据dispersion计算环拓扑中每层所需的replicanths。

Then the ring builder calculates the maximum deviation on a single device between its weighted replicanths and wanted replicanths.

然后，环构建器计算单个设备在其weighted replicanth和wanted replicanth之间的最大偏差。

Next we interpolate between the two replicanth values (weighted & wanted) at each tier using the specified overload (up to the maximum required overload). It’s a linear interpolation, similar to solving for a point on a line between two points - we calculate the slope across the max required overload and then calculate the intersection of the line with the desired overload. This becomes the target.

接下来，我们使用指定的overload（up to the maximum required overload）在每层的两个replicanth值（weighted & wanted）之间进行插值。这是一个线性插值，类似于求解两点之间的直线上的点 - 我们计算最大所需过载的斜率，然后计算该线与所需过载的交点。This becomes the target.

From the target we calculate the minimum and maximum number of replicas any partition may have in a tier. This becomes the replica-plan.

从目标，我们计算任何分区在层中可能具有的最小和最大副本数。这成为副本计划（replica-plan）。

Finally, we calculate the number of partitions that should ideally be assigned to each device based the replica-plan.

最后，我们根据副本计划计算理想情况下应分配给每个设备的分区数。

On initial balance (i.e., the first time partitions are placed to generate a ring) we must assign each replica of each partition to the device that desires the most partitions excluding any devices that already have their maximum number of replicas of that partition assigned to some parent tier of that device’s failure domain.

在初始平衡initial balance（即，第一次放置分区以生成环）时，我们必须将每个分区的每个副本分配给需要最多分区的设备，不包括已分配给某些分区的最大分区副本数的故障域的父层的设备。

When building a new ring based on an old ring, the desired number of partitions each device wants is recalculated from the current replica-plan. Next the partitions to be reassigned are gathered up. Any removed devices have all their assigned partitions unassigned and added to the gathered list. Any partition replicas that (due to the addition of new devices) can be spread out for better durability are unassigned and added to the gathered list. Any devices that have more partitions than they now desire have random partitions unassigned from them and added to the gathered list. Lastly, the gathered partitions are then reassigned to devices using a similar method as in the initial assignment described above.

在基于旧环建立新环时，从当前副本计划重新计算每个设备所需的所需分区数。接下来，将收集要重新分配的分区。任何已删除的设备都会将所有已分配的分区取消分配并添加到gathered list中。任何分区副本（由于添加了新设备）可以分散以获得更好的持久性，这些副本是未分配的并添加到收集列表中。任何具有比现在更多分区的设备，都会从它们中取消分配随机分区，并添加到收集列表中。最后，使用与上述初始分配中类似的方法将收集的分区重新分配给设备。

（译注：简单说，就是通过各种信息，收集设备，用于等会rebalance的时候分配parition）

Whenever a partition has a replica reassigned, the time of the reassignment is recorded. This is taken into account when gathering partitions to reassign so that no partition is moved twice in a configurable amount of time. This configurable amount of time is known internally to the RingBuilder class as `min_part_hours`. This restriction is ignored for replicas of partitions on devices that have been removed, as device removal should only happens on device failure and there’s no choice but to make a reassignment.

每当分区重新分配副本时，都会记录重新分配的时间。在收集分区以重新分配时会考虑这一点，以便在可配置的时间内不会移动分区两次。这个可配置的时间量在RingBuilder类内部是 `min_part_hours`。对于已删除的设备上的分区副本，将忽略此限制，因为设备删除应仅在设备故障时发生，并且除了进行重新分配之外，别无选择。

The above processes don’t always perfectly rebalance a ring due to the random nature of gathering partitions for reassignment. To help reach a more balanced ring, the rebalance process is repeated a fixed number of times until the replica-plan is fulfilled or unable to be fulfilled (indicating we probably can’t get perfect balance due to too many partitions recently moved).

由于收集分区用于重新分配的随机性质，上述过程并不总是perfectly rebalance a ring。为了帮助达到更平衡的环，rebalance过程重复固定次数，直到replica-plan实现或无法实现（表明由于最近移动的分区太多，我们可能无法get perfect balance）。

Composite Rings[¶](#module-swift.common.ring.composite_builder "Permalink to this headline")
--------------------------------------------------------------------------------------------

A standard ring built using the [ring-builder](#ring-builder) will attempt to randomly disperse replicas or erasure-coded fragments across failure domains, but does not provide any guarantees such as placing at least one replica of every partition into each region. Composite rings are intended to provide operators with greater control over the dispersion of object replicas or fragments across a cluster, in particular when there is a desire to have strict guarantees that some replicas or fragments are placed in certain failure domains. This is particularly important for policies with duplicated erasure-coded fragments.

使用[ring-builder](#ring-builder)构建的标准环将尝试跨越故障域随机分散副本或擦除编码的片段，但不提供任何保证，例如将每个分区的至少一个副本放置到每个region中。复合环（Composite rings）旨在为操作员提供对跨群集的对象replicas或fragments的分散的更大控制，特别是当希望严格保证某些复制品或片段放置在某些故障域中时。这对于具有重复的擦除编码片段的策略尤其重要。

A composite ring comprises two or more component rings that are combined to form a single ring with a replica count equal to the sum of replica counts from the component rings. The component rings are built independently, using distinct devices in distinct regions, which means that the dispersion of replicas between the components can be guaranteed. The `composite_builder` utilities may then be used to combine components into a composite ring.

复合环包括两个或更多个组件环，这些组件环被组合以形成单个环，其复制计数等于来自组件环的复制计数的总和。组件环是独立构建的，使用不同区域中的不同设备，这意味着可以保证组件之间的复制品的分散。composite_builder 然后可以使用这些实用程序将组件组合成复合环。

For example, consider a normal ring `ring0` with replica count of 4 and devices in two regions `r1` and `r2`. Despite the best efforts of the ring-builder, it is possible for there to be three replicas of a particular partition placed in one region and only one replica placed in the other region. For example:

例如，考虑一个正常的环ring0为4副本计数和在两个区域的设备r1和r2。尽管环构建器做了最大努力，但是有可能在一个区域中放置三个特定分区的复制品，而在另一个区域中只放置一个复制品。例如：

part\_n \-> r1z1h110/sdb r1z2h12/sdb r1z3h13/sdb r2z1h21/sdb

Now consider two normal rings each with replica count of 2: `ring1` has devices in only `r1`; `ring2` has devices in only `r2`. When these rings are combined into a composite ring then every partition is guaranteed to be mapped to two devices in each of `r1` and `r2`, for example:

part\_n \-> r1z1h10/sdb r1z2h20/sdb  r2z1h21/sdb r2z2h22/sdb
          |\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_|  |\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_|
                     |                        |
                   ring1                    ring2

The dispersion of partition replicas across failure domains within each of the two component rings may change as they are modified and rebalanced, but the dispersion of replicas between the two regions is guaranteed by the use of a composite ring.

For rings to be formed into a composite they must satisfy the following requirements:

*   All component rings must have the same part power (and therefore number of partitions)
*   All component rings must have an integer replica count
*   Each region may only be used in one component ring
*   Each device may only be used in one component ring

Under the hood, the composite ring has a `_replica2part2dev_id` table that is the union of the tables from the component rings. Whenever the component rings are rebalanced, the composite ring must be rebuilt. There is no dynamic rebuilding of the composite ring.

Note

The order in which component rings are combined into a composite ring is very significant because it determines the order in which the Ring.get\_part\_nodes() method will provide primary nodes for the composite ring and consequently the node indexes assigned to the primary nodes. For an erasure-coded policy, inadvertent changes to the primary node indexes could result in large amounts of data movement due to fragments being moved to their new correct primary.

The `id` of each component RingBuilder is therefore stored in metadata of the composite and used to check for the component ordering when the same composite ring is re-composed. RingBuilder `id`s are normally assigned when a RingBuilder instance is first saved. Older RingBuilder instances loaded from file may not have an `id` assigned and will need to be saved before they can be used as components of a composite ring. This can be achieved by, for example:

swift\-ring\-builder <builder\-file\> rebalance \--force

Ring Builder Analyzer[¶](#module-swift.cli.ring_builder_analyzer "Permalink to this headline")
----------------------------------------------------------------------------------------------

This is a tool for analyzing how well the ring builder performs its job in a particular scenario. It is intended to help developers quantify any improvements or regressions in the ring builder; it is probably not useful to others.

The ring builder analyzer takes a scenario file containing some initial parameters for a ring builder plus a certain number of rounds. In each round, some modifications are made to the builder, e.g. add a device, remove a device, change a device’s weight. Then, the builder is repeatedly rebalanced until it settles down. Data about that round is printed, and the next round begins.

Scenarios are specified in JSON. Example scenario for a gradual device addition:

{
    "part\_power": 12,
    "replicas": 3,
    "overload": 0.1,
    "random\_seed": 203488,

    "rounds": \[
        \[
            \["add", "r1z2-10.20.30.40:6200/sda", 8000\],
            \["add", "r1z2-10.20.30.40:6200/sdb", 8000\],
            \["add", "r1z2-10.20.30.40:6200/sdc", 8000\],
            \["add", "r1z2-10.20.30.40:6200/sdd", 8000\],

            \["add", "r1z2-10.20.30.41:6200/sda", 8000\],
            \["add", "r1z2-10.20.30.41:6200/sdb", 8000\],
            \["add", "r1z2-10.20.30.41:6200/sdc", 8000\],
            \["add", "r1z2-10.20.30.41:6200/sdd", 8000\],

            \["add", "r1z2-10.20.30.43:6200/sda", 8000\],
            \["add", "r1z2-10.20.30.43:6200/sdb", 8000\],
            \["add", "r1z2-10.20.30.43:6200/sdc", 8000\],
            \["add", "r1z2-10.20.30.43:6200/sdd", 8000\],

            \["add", "r1z2-10.20.30.44:6200/sda", 8000\],
            \["add", "r1z2-10.20.30.44:6200/sdb", 8000\],
            \["add", "r1z2-10.20.30.44:6200/sdc", 8000\]
        \], \[
            \["add", "r1z2-10.20.30.44:6200/sdd", 1000\]
        \], \[
            \["set\_weight", 15, 2000\]
        \], \[
            \["remove", 3\],
            \["set\_weight", 15, 3000\]
        \], \[
            \["set\_weight", 15, 4000\]
        \], \[
            \["set\_weight", 15, 5000\]
        \], \[
            \["set\_weight", 15, 6000\]
        \], \[
            \["set\_weight", 15, 7000\]
        \], \[
            \["set\_weight", 15, 8000\]
        \]\]
}

History[¶](#history "Permalink to this headline")
-------------------------------------------------

The ring code went through many iterations before arriving at what it is now and while it has largely been stable, the algorithm has seen a few tweaks or perhaps even fundamentally changed as new ideas emerge. This section will try to describe the previous ideas attempted and attempt to explain why they were discarded.

A “live ring” option was considered where each server could maintain its own copy of the ring and the servers would use a gossip protocol to communicate the changes they made. This was discarded as too complex and error prone to code correctly in the project timespan available. One bug could easily gossip bad data out to the entire cluster and be difficult to recover from. Having an externally managed ring simplifies the process, allows full validation of data before it’s shipped out to the servers, and guarantees each server is using a ring from the same timeline. It also means that the servers themselves aren’t spending a lot of resources maintaining rings.

A couple of “ring server” options were considered. One was where all ring lookups would be done by calling a service on a separate server or set of servers, but this was discarded due to the latency involved. Another was much like the current process but where servers could submit change requests to the ring server to have a new ring built and shipped back out to the servers. This was discarded due to project time constraints and because ring changes are currently infrequent enough that manual control was sufficient. However, lack of quick automatic ring changes did mean that other components of the system had to be coded to handle devices being unavailable for a period of hours until someone could manually update the ring.

The current ring process has each replica of a partition independently assigned to a device. A version of the ring that used a third of the memory was tried, where the first replica of a partition was directly assigned and the other two were determined by “walking” the ring until finding additional devices in other zones. This was discarded due to the loss of control over how many replicas for a given partition moved at once. Keeping each replica independent allows for moving only one partition replica within a given time window (except due to device failures). Using the additional memory was deemed a good trade-off for moving data around the cluster much less often.

Another ring design was tried where the partition to device assignments weren’t stored in a big list in memory but instead each device was assigned a set of hashes, or anchors. The partition would be determined from the data item’s hash and the nearest device anchors would determine where the replicas should be stored. However, to get reasonable distribution of data each device had to have a lot of anchors and walking through those anchors to find replicas started to add up. In the end, the memory savings wasn’t that great and more processing power was used, so the idea was discarded.

A completely non-partitioned ring was also tried but discarded as the partitioning helps many other components of the system, especially replication. Replication can be attempted and retried in a partition batch with the other replicas rather than each data item independently attempted and retried. Hashes of directory structures can be calculated and compared with other replicas to reduce directory walking and network traffic.

Partitioning and independently assigning partition replicas also allowed for the best-balanced cluster. The best of the other strategies tended to give ±10% variance on device balance with devices of equal weight and ±15% with devices of varying weights. The current strategy allows us to get ±3% and ±8% respectively.

Various hashing algorithms were tried. SHA offers better security, but the ring doesn’t need to be cryptographically secure and SHA is slower. Murmur was much faster, but MD5 was built-in and hash computation is a small percentage of the overall request handling time. In all, once it was decided the servers wouldn’t be maintaining the rings themselves anyway and only doing hash lookups, MD5 was chosen for its general availability, good distribution, and adequate speed.

The placement algorithm has seen a number of behavioral changes for unbalanceable rings. The ring builder wants to keep replicas as far apart as possible while still respecting device weights. In most cases, the ring builder can achieve both, but sometimes they conflict. At first, the behavior was to keep the replicas far apart and ignore device weight, but that made it impossible to gradually go from one region to two, or from two to three. Then it was changed to favor device weight over dispersion, but that wasn’t so good for rings that were close to balanceable, like 3 machines with 60TB, 60TB, and 57TB of disk space; operators were expecting one replica per machine, but didn’t always get it. After that, overload was added to the ring builder so that operators could choose a balance between dispersion and device weights. In time the overload concept was improved and made more accurate.

For more background on consistent hashing rings, please see [Building a Consistent Hashing Ring](ring_background.html).

updated: 2019-01-15 02:30

 [![Creative Commons Attribution 3.0 License](_static/images/docs/license.png)](https://creativecommons.org/licenses/by/3.0/) 

Except where otherwise noted, this document is licensed under [Creative Commons Attribution 3.0 License](https://creativecommons.org/licenses/by/3.0/). See all [OpenStack Legal Documents](http://www.openstack.org/legal).

[found an error? report a bug](#) [questions?](http://ask.openstack.org)

OpenStack Documentation

*   Guides
*   [Install Guides](http://docs.openstack.org/index.html#install-guides)
*   [User Guides](http://docs.openstack.org/index.html#user-guides)
*   [Configuration Guides](http://docs.openstack.org/index.html#configuration-guides)
*   [Operations and Administration Guides](http://docs.openstack.org/index.html#ops-and-admin-guides)
*   [API Guides](http://docs.openstack.org/index.html#api-guides)
*   [Contributor Guides](http://docs.openstack.org/index.html#contributor-guides)
*   Languages
*   [Deutsch (German)](http://docs.openstack.org/de/)
*   [Français (French)](http://docs.openstack.org/fr/)
*   [Bahasa Indonesia (Indonesian)](http://docs.openstack.org/id/)
*   [Italiano (Italian)](http://docs.openstack.org/it/)
*   [日本語 (Japanese)](http://docs.openstack.org/ja/)
*   [한국어 (Korean)](http://docs.openstack.org/ko_KR/)
*   [Português (Portuguese)](http://docs.openstack.org/pt_BR/)
*   [Türkçe (Türkiye)](http://docs.openstack.org/tr_TR/)
*   [简体中文 (Simplified Chinese)](http://docs.openstack.org/zh_CN/)

[

#### swift 2.17.1.dev19

](index.html)

*   [Getting Started](getting_started.html)

*   [Object Storage API overview](api/object_api_v1_overview.html)
*   [Swift Architectural Overview](overview_architecture.html)
*   [The Rings](#)
    *   [Ring Builder](#ring-builder)
    *   [Ring Data Structure](#ring-data-structure)
    *   [Partition & Replica Terminology](#partition-replica-terminology)
    *   [Building the Ring](#building-the-ring)
    *   [Composite Rings](#module-swift.common.ring.composite_builder)
    *   [Ring Builder Analyzer](#module-swift.cli.ring_builder_analyzer)
    *   [History](#history)
*   [Storage Policies](overview_policies.html)
*   [The Account Reaper](overview_reaper.html)
*   [The Auth System](overview_auth.html)
*   [Access Control Lists (ACLs)](overview_acl.html)
*   [Replication](overview_replication.html)
*   [Rate Limiting](ratelimit.html)
*   [Large Object Support](overview_large_objects.html)
*   [Object Versioning](overview_object_versioning.html)
*   [Global Clusters](overview_global_cluster.html)
*   [Container to Container Synchronization](overview_container_sync.html)
*   [Expiring Object Support](overview_expiring_objects.html)
*   [CORS](cors.html)
*   [Cross-domain Policy File](crossdomain.html)
*   [Erasure Code Support](overview_erasure_code.html)
*   [Object Encryption](overview_encryption.html)
*   [Using Swift as Backing Store for Service Data](overview_backing_store.html)
*   [Building a Consistent Hashing Ring](ring_background.html)
*   [Modifying Ring Partition Power](ring_partpower.html)
*   [Associated Projects](associated_projects.html)

*   [Development Guidelines](development_guidelines.html)
*   [SAIO - Swift All In One](development_saio.html)
*   [First Contribution to Swift](first_contribution_swift.html)
*   [Adding Storage Policies to an Existing SAIO](policies_saio.html)
*   [Auth Server and Middleware](development_auth.html)
*   [Middleware and Metadata](development_middleware.html)
*   [Pluggable On-Disk Back-end APIs](development_ondisk_backends.html)

*   [Instructions for a Multiple Server Swift Installation](howto_installmultinode.html)
*   [Deployment Guide](deployment_guide.html)
*   [Apache Deployment Guide](apache_deployment_guide.html)
*   [Administrator’s Guide](admin_guide.html)
*   [Dedicated replication network](replication_network.html)
*   [Logs](logs.html)
*   [Swift Ops Runbook](ops_runbook/index.html)
*   [OpenStack Swift Administrator Guide](admin/index.html)
*   [Object Storage Install Guide](install/index.html)

*   [Object Storage API overview](api/object_api_v1_overview.html)
*   [Discoverability](api/discoverability.html)
*   [Authentication](api/authentication.html)
*   [Container quotas](api/container_quotas.html)
*   [Object versioning](api/object_versioning.html)
*   [Large objects](api/large_objects.html)
*   [Temporary URL middleware](api/temporary_url_middleware.html)
*   [Form POST middleware](api/form_post_middleware.html)
*   [Use Content-Encoding metadata](api/use_content-encoding_metadata.html)
*   [Use the Content-Disposition metadata](api/use_the_content-disposition_metadata.html)

*   [Partitioned Consistent Hash Ring](ring.html)
*   [Proxy](proxy.html)
*   [Account](account.html)
*   [Container](container.html)
*   [Account DB and Container DB](db.html)
*   [Object](object.html)
*   [Misc](misc.html)
*   [Middleware](middleware.html)

#### Page Contents

*   [The Rings](#)
    *   [Ring Builder](#ring-builder)
    *   [Ring Data Structure](#ring-data-structure)
        *   [List of Devices](#list-of-devices)
        *   [Partition Assignment List](#partition-assignment-list)
        *   [Partition Shift Value](#partition-shift-value)
        *   [Fractional Replicas](#fractional-replicas)
        *   [Dispersion](#dispersion)
        *   [Overload](#overload)
    *   [Partition & Replica Terminology](#partition-replica-terminology)
    *   [Building the Ring](#building-the-ring)
    *   [Composite Rings](#module-swift.common.ring.composite_builder)
    *   [Ring Builder Analyzer](#module-swift.cli.ring_builder_analyzer)
    *   [History](#history)

### OpenStack

*   [Projects](http://openstack.org/projects/)
*   [OpenStack Security](http://openstack.org/projects/openstack-security/)
*   [Common Questions](http://openstack.org/projects/openstack-faq/)
*   [Blog](http://openstack.org/blog/)
*   [News](http://openstack.org/news/)

### Community

*   [User Groups](http://openstack.org/community/)
*   [Events](http://openstack.org/community/events/)
*   [Jobs](http://openstack.org/community/jobs/)
*   [Companies](http://openstack.org/foundation/companies/)
*   [Contribute](http://docs.openstack.org/infra/manual/developers.html)

### Documentation

*   [OpenStack Manuals](http://docs.openstack.org)
*   [Getting Started](http://openstack.org/software/start/)
*   [API Documentation](http://developer.openstack.org)
*   [Wiki](https://wiki.openstack.org)

### Branding & Legal

*   [Logos & Guidelines](http://openstack.org/brand/)
*   [Trademark Policy](http://openstack.org/brand/openstack-trademark-policy/)
*   [Privacy Policy](http://openstack.org/privacy/)
*   [OpenStack CLA](https://wiki.openstack.org/wiki/How_To_Contribute#Contributor_License_Agreement)

### Stay In Touch

[](https://twitter.com/OpenStack)[](https://www.facebook.com/openstack)[](https://www.linkedin.com/company/openstack)[](https://www.youtube.com/user/OpenStackFoundation)

The OpenStack project is provided under the [Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0). Openstack.org is powered by [Rackspace Cloud Computing](http://rackspace.com).

var DOCUMENTATION\_OPTIONS = { URL\_ROOT: './', VERSION: '2.17.1.dev19', COLLAPSE\_INDEX: false, FILE\_SUFFIX: '.html', SOURCELINK\_SUFFIX: '.txt', HAS\_SOURCE: true }; /\* build a description of this page including SHA, source location on git repo, build time and the project's launchpad bug tag. Set the HREF of the bug buttons \*/ var lineFeed = "%0A"; var gitURL = "Source: Can't derive source file URL"; /\* there have been cases where "pagename" wasn't set; better check for it \*/ /\* "giturl" is the URL of the source file on Git and is auto-generated by openstackdocstheme. "pagename" is a standard sphinx parameter containing the name of the source file, without extension. \*/ var sourceFile = "overview\_ring" + ".rst"; gitURL = "Source: https://git.openstack.org/cgit/openstack/swift/tree/doc/source" + "/" + sourceFile; /\* gitsha, project and bug\_tag rely on variables in conf.py \*/ var gitSha = "SHA: 4ea52f3a950b5db1bb0549de322e252471218401"; var bugProject = "swift"; var bugTitle = "The Rings in swift"; var fieldTags = ""; var useStoryboard = ""; /\* "last\_updated" is the build date and time. It relies on the conf.py variable "html\_last\_updated\_fmt", which should include year/month/day as well as hours and minutes \*/ var buildstring = "Release: 2.17.1.dev19 on 2019-01-15 02:30"; var fieldComment = encodeURI(buildstring) + lineFeed + encodeURI(gitSha) + lineFeed + encodeURI(gitURL) ; logABug(bugTitle, bugProject, fieldComment, fieldTags); $(document).ready(function(){ $.ajax({ context: this, dataType : "html", url : "https://docs.openstack.org/queens/badge.html", success : function(results) { $('#deprecated-badge-container').html(results); } }); });