+++
title = "官方中文文档 openstack swift overview policies"
date = 2019-01-21T00:00:00-08:00
lastmod = 2019-01-23T18:26:07-08:00
tags = ["swift", "transilation"]
categories = ["swift"]
draft = false
weight = 3001
+++

<https://docs.openstack.org/swift/queens/overview%5Fpolicies.html>

存储策略

更新时间：2019-01-15 02:30

存储策略允许通过创建多个对象环来为各种目的对集群进行某种程度的分段。存储策略功能在整个代码库中实现，因此它是理解Swift体系结构的重要概念。

如The Rings中所述，Swift使用修改的散列环来确定数据应驻留在集群中的位置。帐户数据库，容器数据库，有一个单独的环; 每个存储策略也有一个对象环。(译者注：有3个ring)每个对象环的行为完全相同，并以相同的方式维护，但是对于策略，不同的设备可以属于不同的环。通过支持多个对象环，Swift允许应用程序和/或部署者基本上将对象存储隔离在单个集群中。有很多原因可能需要这样做：

-   不同级别的持久性：如果提供商想要提供（例如，2x复制和3x复制，但不想维护2个单独的群集），则他们将设置2x和3x复制策略并将节点分配给各自的环。此外，如果提供商想要提供冷存储层，他们可以创建一个擦除编码策略。
-   性能：正如SSD可以用作帐户或数据库环的独占成员一样，也可以创建仅限SSD的对象环，并用于实现低延迟/高性能策略。
-   将节点收集到组中：不同的对象环可能具有不同的物理服务器，因此特定存储策略中的对象始终位于特定的数据中心或地理位置。
-   不同的存储实现：另一个例子是收集使用不同Diskfile的一组节点（例如，Kinetic，GlusterFS）并使用策略将流量直接引导到那些节点。
-   不同的读写关联设置：可以将代理服务器配置为对每个策略使用不同的读写关联选项。有关详细信息，请参阅 [每个策略配置](https://docs.openstack.org/swift/queens/deployment%5Fguide.html#proxy-server-per-policy-config)。

注意

今天，Swift支持两种不同的策略类型：复制和擦除代码。有关详细信息，请参阅 [Erasure Code Support](https://docs.openstack.org/swift/queens/overview%5Ferasure%5Fcode.html)。

另请注意，Diskfile是指后端对象存储插件架构。有关详细信息，请参阅 [Pluggable On-Disk Back-end APIs](https://docs.openstack.org/swift/queens/development%5Fondisk%5Fbackends.html)。


## 容器和策略 {#容器和策略}

策略在容器级别实施。这种方式有许多优点，其中最重要的是它能够轻松地在想要利
用它们的应用程序上make life。它还确保存储策略仍然是Swift的核心功能，与auth实现无
关。帐户/身份验证层未实施策略，是因为它需要Swift部署人员更改正在使用的所有身份验
证系统。每个容器都有一个新的特殊的不可变元数据元素，称为存储策略索引(storage
policy index)。
请注意，在内部，Swift依赖于策略索引(policy indexes)而不是策略名称(policy name)。策略名称的存在是用于人类可读性，并且在代理中转换。创建容器时，支持一个新的可选header以指定策略名称。如果没有指定名称，使用默认策略（如果未定义其他策略，则将Policy-0视为默认策略）。我们将在下一节中介绍default和Policy-0之间的区别。

创建容器时会分配策略。为容器分配策略后，将无法更改该策略（除非将其删除/重新
创建）。对大型数据集的数据放置/移动的影响将使这个任务最好留给应用程序执行。
因此，如果容器具有一个存在的策略（例如3x复制），并且想要将该数据迁移到
Erasure Code策略，则应用程序将创建另一个容器，指定策略其他参数，然后只需将
数据从一个容器移动到另一个。策略适用于每个容器，允许最小的应用程序感知
(minimal application awareness); 一旦使用特定策略创建了容器，则存储在其中的 所有对象将根据该策略完成。如果删除了具有特定名称的容器（要求容器变为空），则可以创建具有相同名称的新容器，而不受先前共享相同名称的已删除容器强制执行的存储策略的任何限制。

容器与策略具有多对一关系，这意味着任意数量的容器都可以共享一个策略。可以使用特定策略的容器数量没有限制。

将环与容器相关联的概念引入了一个有趣的场景：如果同时在网络中断的任一侧使用不同的存储策略创建了2个同名容器，会发生什么？此外，如果将对象放在这些容器中，一大堆容器中，然后网络中断被恢复，会发生什么？好吧，没有特别小心，这将是一个大问题，因为应用程序最终可能会使用错误的环来尝试找到一个对象。幸运的是，这个问题有一个解决方案，一个名为Container Reconciler的守护进程不知疲倦地工作以识别和纠正这种潜在的情况。


## 容器协调器(Container Reconciler) {#容器协调器--container-reconciler}

因为,无法,在分布式最终一致系统中,强制执行容器创建的原子性，所以对象写入错误
的存储策略必须最终由异步守护程序合并到正确的存储策略中。在网络分区期间从对象
写入恢复,导致了使用不同存储策略创建的脑裂容器, 由 swift-container-reconciler守护程序处理。

容器协调程序使用类似于object-expirer的队列。在容器复制期间移动队列。将容器协调器评估的对象排入队列从未被认为是错误的，因为如果对象的位置没有任何问题，则协调器将简单地将其出列。容器协调队列是对象的实际位置的索引日志，对象的实际位置发现了容器的存储策略的差异。

要确定容器的正确存储策略，当容器将状态从已删除更改为重新创建时，必须更新container\_stat表中的status\_changed\_at字段。此事务日志允许容器复制器在复制容器和处理REPLICATE请求时更新正确的存储策略。

由于每个对象写入都是一个单独的分布式事务，因此无法根据给定容器数据库中的整个事务日志确定每个对象写入的存储策略的正确性。因此，容器数据库将始终记录对象写入，而不管基于每个对象行的存储策略。每个容器中的每个存储策略都会跟踪对象字节和计数统计信息，并使用普通对象行合并语义进行协调。

使用常规容器复制在复制期间确保对象行完全持久。在容器复制器将其对象行推送到可
用主节点之后，根据.misplaced\_objects系统帐户下的对象时间戳，将任何放错位置的
对象行,批量加载到容器中。这些行,最初写入本地节点上的切换容器(handoff container)，并且在复制传递结束时，.misplaced\_objects容器将复制到正确的主节点。

容器协调程序按降序处理.misplaced\_objects容器，并在成功协调行所表示的对象时,
重新收集容器。容器协调程序将始终,使用通过经缓存加速后的直接容器HEAD请求,验证排队对象的正确存储策略。

由于假设，在合计下，各个存储节点的故障在大规模上是常见的，因此容器协调器,将
以简单的仲裁多数(a simple quorum majority),进行前进。在故障和rebalance的组合
期间，仲裁(a quorum)可能会提供正确存储策略的不完整记录 - 因此应用对象写入就
可能必须执行多次。由于存储节点和容器数据库,在重新应用对象写入时不会处理
X-Timestamp小于或等于其现有记录的写入，因此它们的时间戳会略微增加。为了将此
增量透明地应用于客户端，已将时间的第二个向量添加到Swift以供内部使用。见 [Timestamp](https://docs.openstack.org/swift/queens/misc.html#swift.common.utils.Timestamp)。

由于协调程序将对象写入应用于正确的存储策略，因此它会清除不再适用于不正确的存储策略的写入，并从.misplaced\_objects容器中删除行。成功处理完所有行后，它会休眠并定期检查容器复制期间要发现的新排队行。


## 默认与'Policy- 0' {#默认与-policy-0}

存储策略是一种通用功能，旨在支持具有相同级别灵活性的新群集和预先存在的群集。
出于这个原因，我们引入了Policy-0, 这个与“default”策略不同的概念。正如您将在我
们开始配置策略时所看到的，每个策略都有一个名称和任意数量的别名（人性化的，可
配置的）以及索引（或简单策略编号）。Swift保留索引0以映射到所有安装中存在的对象环（例如，/etc/swift/object.ring.gz）。您可以将此策略命名为您喜欢的任何名称，如果未定义任何策略，则会将其自身报告为Policy-0，但是您无法更改索引，因为必须始终存在索引为0的策略。

另一个重要概念是默认策略，它可以是群集中的任何策略。默认策略是在未指定存储策
略的情况下发送容器创建请求时自动选择的策略。
[Configuring Policies](https://docs.openstack.org/swift/queens/overview%5Fpolicies.html#configure-policy)介绍了如何设置默认策略。区别Policy-0是微妙的，但非常重要。 Policy-0是Swift在访问预存储策略容器没有策略时使用的 - 在这种情况下，我们不会使用默认值，因为它可能没有与旧容器相同的策略。如果未定义其他策略，Swift将始终选择Policy-0默认值。

换句话说，默认策略意味着“如果没有指定其他内容则使用此策略创建”，并且Policy-0
策略意味着“如果没有容器策略则使用旧策略”,这实际上意味着使用object.ring.gz去查找。

注意

使用基于存储策略的代码，无法创建没有策略的容器。如果没有提供任何内容，Swift
仍将选择默认策略并将其分配给容器。对于在引入存储策略之前创建的容器，将使用旧的Policy-0。


## 弃用策略 {#弃用策略}

有时候不再需要一种策略; 但是，简单地删除策略和相关的环对于现有数据将是有问题的。为了确保资源不会在群集中孤立（留在磁盘上但不再可访问），并且在需要停用策略时向应用程序提供正确的消息传递，则使用弃用(deprecation)的概念。[Configuring Policies](https://docs.openstack.org/swift/queens/overview%5Fpolicies.html#configure-policy)描述了如何弃用策略。

Swift对弃用策略的行为如下：

-   已弃用的策略不会出现在/info中
-   在使用已弃用的策略创建的预先存在的容器上仍允许PUT/GET/DELETE/POST/HEAD
-   尝试使用已弃用的策略创建新容器时，客户端将收到“400 Bad Request”错误
-   客户端仍然可以通过HEAD访问预先存在的容器上的策略统计信息

注意

策略不能同时是默认策略和弃用策略。如果您弃用默认策略，则必须指定新的默认策略。

您还可以使用已弃用的功能部署新策略。如果想在使得新存储策略普遍可用之前测试新
存储策略，则可以在最初将新策略推送到所有节点时弃用该策略。被弃用将使其天生
(render it innate)并且无法使用。要测试它，您需要使用该存储策略创建容器; 这将
需要单个proxy实例（或一组仅在内部可访问的proxy-server），这些proxy实例已经被
一次性配置, 这个配置是带有新策略且未标记为已弃用。使用新存储策略创建容器后，
任何授权使用该容器的客户端都将能够在新存储策略中添加和访问存储在该容器中的数
据。如果满意，你可以推出一个新的swift.conf,它不再将策略标记为已弃用,到所有节点。


## 配置策略 {#配置策略}

注意

有关向 SAIO设置添加策略的分步指南，请参阅 [Adding Storage Policies to an Existing SAIO](https://docs.openstack.org/swift/queens/policies%5Fsaio.html)。

部署者必须充分了解配置策略的语义。配置策略分为三个步骤：

-   编辑/etc/swift/swift.conf文件以定义新策略。
-   创建相应的策略对象环文件。
-   （可选）创建特定于策略的proxy-server配置设置。


### 定义策略 {#定义策略}

每个策略都由/etc/swift/swift.conf文件中的一个section定义。节名称必须是
[storage-policy:<N>]的格式, 所在<N>是策略索引。除了可读性之外，没有理由将策略索引按顺序排列，但强制执行以下规则：

-   如果一个策略具有索引0,且索引0且未定义其他策略，则Swift将创建一个使用索引0的默认策略。
-   策略索引必须是非负整数。
-   策略索引必须是唯一的。

警告

创建和使用策略后，永远不应更改策略索引。更改策略索引可能会导致无法访问数据。

每个策略section包含以下选项：

-   name = <policy\_name> （必需）
    -   策略的主要名称。
    -   策略名称不区分大小写。
    -   策略名称必须仅包含字母，数字或短划线。
    -   策略名称必须是唯一的。
    -   策略名称可以更改。
    -   名称Policy-0只能用于具有索引0的策略。
-   aliases = <policy\_name>[, <policy\_name>, ...] （可选的）
    -   以逗号分隔的策略备用名称列表。
    -   默认值是空列表（即没有别名）。
    -   所有别名必须遵循该name选项的规则。
    -   别名可以添加到列表中或从列表中删除。
    -   如果更改主名称，别名可用于保留对旧主名称的支持。
-   default = [true|false] （可选的）
    -   如果true在客户端未指定策略时将使用此策略。
    -   默认值为false。
    -   通过在所需的策略部分中进行设置，可以随时更改默认策略 。default = true
    -   如果没有将策略声明为缺省策略且未定义其他策略，则将具有索引的策略0设置为缺省策略;
    -   否则，必须将一个策略声明为default。
    -   不能将不推荐使用的策略声明为默认值。
    -   有关详细信息，请参阅默认与'Policy-0'。
-   deprecated = [true|false] （可选的）
    -   如果true此时无法使用此策略创建新容器。
    -   默认值为false。
    -   可以通过将deprecated选项添加到所需的策略部分来弃用任何策略。但是，不推荐使用已弃用的策略作为默认值。因此，由于必须始终存在默认策略，因此必须始终至少有一个未弃用的策略。
    -   有关详细信息，请参阅 [Deprecating Policies](https://docs.openstack.org/swift/queens/overview%5Fpolicies.html#deprecate-policy)。
-   policy\_type = [replication|erasure\_coding] （可选的）
    -   该选项policy\_type用于区分不同的策略类型。
    -   默认值为replication。
    -   定义EC策略时使用该值erasure\_coding。

EC策略类型具有其他必需选项。有关详细信息，请参阅 [Using an Erasure Code Policy](https://docs.openstack.org/swift/queens/overview%5Ferasure%5Fcode.html#using-ec-policy)。

以下是正确配置的swift.conf文件的示例。有关使用此示例配置而去设置一个all-in-one的完整说明，请参阅 [Adding Storage Policies to an Existing SAIO](https://docs.openstack.org/swift/queens/policies%5Fsaio.html)：

```shell
[swift-hash]
# random unique strings that can never change (DO NOT LOSE)
# Use only printable chars (python -c "import string; print(string.printable)")
swift_hash_path_prefix = changeme
swift_hash_path_suffix = changeme

[storage-policy:0]
name = gold
aliases = yellow, orange
policy_type = replication
default = yes

[storage-policy:1]
name = silver
policy_type = replication
deprecated = yes
```


### 创建一个环 {#创建一个环}

一旦swift.conf配置为一个新的策略，必须创建一个新的环。环工具不支持策略名称，
因此在创建新策略的环文件时使用正确的策略索引至关重要。使用
swift-ring-builder创建其他对象环，与传统环是相同的方式,除了在构建文件名称的
单词object后面附加-N，其中N匹配在swift.conf中使用的策略索引。因此，如果是要为索引1策略创建环：

```shell
swift-ring-builder object-1.builder create 10 3 1
```

在swift-ring-builder用于添加设备，rebalance等操作时，继续使用相同的命名约定。
此命名约定也用于pre-policy 存储节点数据字典的模式。

注意

确实可以将相同的驱动器用于多个策略，并且将在后面的部分中介绍如何在磁盘上管理它的详细信息，在设置之前了解此类配置的含义非常重要。确保它真的是你想要做的，在许多情况下它会是，但在其他情况下可能不是。


### Proxy server 配置（可选） {#proxy-server-配置-可选}

与读取和写入关联相关的 [Proxy Server](https://docs.openstack.org/swift/queens/proxy.html#proxy-server)配置选项，可以用单个存储策略来选择性覆盖。
有关详细信息，请参阅 [Per policy configuration](https://docs.openstack.org/swift/queens/deployment%5Fguide.html#proxy-server-per-policy-config)。


## 使用策略 {#使用策略}

使用策略非常简单 - 仅在最初创建容器时指定策略。没有其他API可以更改策略。创建容器可以在没有任何特殊策略信息的情况下完成：

```shell
curl -v -X PUT -H 'X-Auth-Token: <your auth token>' \
     http://127.0.0.1:8080/v1/AUTH_test/myCont0
```

假设我们使用上面的swift.conf示例，这将导致创建与策略名称“gold”关联的容器。它会使用'gold'，因为它被指定为默认值。现在，当我们将一个对象放入这个容器时，它将放置在我们创建的为策略'gold'环的一部分的节点上。

如果我们想明确声明我们想要策略'gold'，那么命令只需要包含一个新的头，如下所示：

```shell
curl -v -X PUT -H 'X-Auth-Token: <your auth token>' \
     -H 'X-Storage-Policy: gold' http://127.0.0.1:8080/v1/AUTH_test/myCont0
```

就是这样！应用程序不需要再次指定策略名称。然而，有一些非法操作：

-   如果指定了无效（错字，不存在）策略：400 Bad Request
-   如果您尝试通过PUT或POST更改策略：409 Conflict

如果您想了解如何使用群集中的存储，只需将HEAD the account即可，您不仅可以看到累积
数量，也可以像之前的一样，看到每个策略统计数据。在下面的示例中，共有3个对象，其中两个在策略'gold'中，另一个在策略'silver'中：

```shell
curl -i -X HEAD -H 'X-Auth-Token: <your auth token>' \
     http://127.0.0.1:8080/v1/AUTH_test
```

并且您的结果将包括（为了便于阅读，删除了一些输出）：

```shell
X-Account-Container-Count: 3
X-Account-Object-Count: 3
X-Account-Bytes-Used: 21
X-Storage-Policy-Gold-Object-Count: 2
X-Storage-Policy-Gold-Bytes-Used: 14
X-Storage-Policy-Silver-Object-Count: 1
X-Storage-Policy-Silver-Bytes-Used: 7
```


## 在引擎盖下 {#在引擎盖下}

现在我们已经解释了一些关于策略是什么以及如何配置/使用它们，让我们来探讨存储
策略如何适用于螺母级别（the nut-n-bolts level）。


### 解析和配置 {#解析和配置}

模块[Storage Policy](https://docs.openstack.org/swift/queens/misc.html#storage-policy)负责解析 swift.conf文件，验证输入以及通过类
[StoragePolicyCollection](https://docs.openstack.org/swift/queens/misc.html#swift.common.storage%5Fpolicy.StoragePolicyCollection) 创建已配置策略的全局集合(a global collection)。该集
合由类[StoragePolicy](https://docs.openstack.org/swift/queens/misc.html#swift.common.storage%5Fpolicy.StoragePolicy)策略组成。集合类包括通过名称或索引获取策略，获取有关策略
等信息的便捷功能。还有一个非常重要的function，[get\_object\_ring()](https://docs.openstack.org/swift/queens/misc.html#swift.common.storage%5Fpolicy.StoragePolicyCollection.get%5Fobject%5Fring)。对象环是类
[StoragePolicy](https://docs.openstack.org/swift/queens/misc.html#swift.common.storage%5Fpolicy.StoragePolicy)的成员，实际上在load\_ring() 方法被调用之前不会实例化。Any caller anywhere in the code base that needs to access an object ring must use the POLICIES global singleton to access the get\_object\_ring() function and provide the policy index which will call load\_ring() if needed; however, when starting request handling services such as the Proxy Server rings are proactively loaded to provide moderate protection against a mis-configuration resulting in a run time error. Swift启动时会实例化全局，并提供修补测试代码的策略的机制。


### 中间件 {#中间件}

中间件可以通过POLICIES全局变量利用策略，并通过导入[get\_container\_info()](https://docs.openstack.org/swift/queens/proxy.html#swift.proxy.controllers.base.get%5Fcontainer%5Finfo)来访问
与相关容器关联的策略索引。通过策略索引，它可以使用 POLICIES singleton来获取正确的环。例如，[List Endpoints](https://docs.openstack.org/swift/queens/middleware.html#list-endpoints)使用刚刚描述的方法识别策略。另一个例子是[Recon](https://docs.openstack.org/swift/queens/middleware.html#recon)，它将报告所有环的md5总和。


### Proxy Server {#proxy-server}

该 [Proxy Server](https://docs.openstack.org/swift/queens/proxy.html#proxy-server)模块在存储策略的任务主要是确保正确的ring作为其成员的元素。在策
略之前，一个对象环将在实例化[Application](https://docs.openstack.org/swift/queens/proxy.html#swift.proxy.server.Application)类时被实例化，并且可以通过init参数被测
试代码覆盖。但是，对于策略，没有init参数，而[Application](https://docs.openstack.org/swift/queens/proxy.html#swift.proxy.server.Application)类依赖于POLICIES global
singleton 来检索,在第一次需要时实例化的,环。因此，替代[Application](https://docs.openstack.org/swift/queens/proxy.html#swift.proxy.server.Application)该类的对象环成员，有一个访问器函数，[get\_object\_ring()](https://docs.openstack.org/swift/queens/proxy.html#swift.proxy.server.Application.get%5Fobject%5Fring)从 POLICIES中获取环。

通常，当代理上运行的任何模块需要对象环时，它首先从缓存的容器信息中获取策略索引。
例外是在容器创建期间，它使用请求头中的策略名称从POLICIES全局变量查找策略索引。
一旦代理确定了策略索引，它就可以使用前面描述的[get\_object\_ring()](https://docs.openstack.org/swift/queens/proxy.html#swift.proxy.server.Application.get%5Fobject%5Fring)方法来访问正确
的环。然后，它有责任通过header X -Backend-Storage-Policy-Index将索引信息而不是
策略名称传递给后端服务器。走另一条路，proxy还会从返回到客户端的标头中删除索引，并确保它们只能看到友好的策略名称。


### On Disk Storage {#on-disk-storage}

每个策略在后端服务器上都有自己的目录，并由其存储策略索引标识。按策略索引组织后
端目录结构有助于跟踪事物，并允许在策略之间共享磁盘，这可能是有意义的也可能没有意义，具体取决于提供者的需求。稍后会详细介绍，但现在请注意以下目录命名约定：

-   /objects 映射到与Policy-0关联的对象
-   /objects-N 映射到存储策略索引#N
-   /async\_pending 映射到Policy-0的异步挂起更新
-   /async\_pending-N 映射到存储策略索引#N的异步挂起更新
-   /tmp 映射到Policy-0的DiskFile临时目录
-   /tmp-N 映射到策略索引#N的DiskFile临时目录
-   /quarantined/objects 映射到Policy-0的隔离目录
-   /quarantined/objects-N 映射到策略索引#N的隔离目录

请注意，这些目录名实际上由特定的Diskfile实现拥有，上面显示的名称由默认的Diskfile使用。


### 对象服务 {#对象服务}

[Object Server](https://docs.openstack.org/swift/queens/object.html#object-server)不参与直接选择存储策略。但是，由于如何为策略设置后端目录结构，
如前所述，对象服务模块确实起作用。当对象服务获取Diskfile时，它会传入策略索
引并保留实际的目录命名/结构机制给Diskfile。通过传入索引，Diskfile正在使用的实例 将确保数据根据其策略正确定位在树中。

出于同样的原因，
[Object Updater](https://docs.openstack.org/swift/queens/object.html#object-updater)也具有策略意识。如前所述，不同的策略使用不同的异步挂起目录
(async pending directories)，因此更新程序需要知道如何正确扫描它们。

[Object Replicator](https://docs.openstack.org/swift/queens/object.html#object-replicator)是策略意识到的是，根据策略，可能需要做大幅度不同的东西，或
者也许不是。例如，处理2x与3x的复制作业的差异是微不足道的;但是，3x和纠删码之
间处理复制的差异绝对不是微不足道的。实际上，术语“replication”实际上并不适合
某些策略，如纠删码;但是，收集和处理工作的大多数框架都很常见。因此，复制器中
的这些功能被用于所有策略，然后对每个策略都需要特定于策略的代码，并在需要时定义策略时添加。

由于同样的原因，ssync功能具有策略感知功能。某些其他模块可能不会受到明显影响，
但Diskfile所拥有的后端目录结构需要策略索引参数。因此，ssync被策略意识,真的
意味着,传递策略索引。见关于SSYNC的[ssync\_sender](https://docs.openstack.org/swift/queens/object.html#module-swift.obj.ssync%5Fsender)和 [ssync\_receiver](https://docs.openstack.org/swift/queens/object.html#module-swift.obj.ssync%5Freceiver)更多信息。

就Diskfile其本身而言，策略感知就是使用提供的策略索引来管理后端结构。换句话
说，获取Diskfile实例的调用者提供策略索引，并且Diskfile工作就是通过此索引
（however it chooses）保持数据分离，以便策略可以共享相同的媒体/节点（如果需
要）。Diskfile包含实现,列出了前面描述的目录结构，但是它在内部Diskfile; 外部
模块无法了解该细节。提供了一种通用功能，用于根据策略索引映射各种目录名称和/
或字符串。例如，Diskfile定义get\_data\_dir(),它建立在一个通用的[get\_policy\_string()](https://docs.openstack.org/swift/queens/misc.html#swift.common.storage%5Fpolicy.get%5Fpolicy%5Fstring)之上,以便为各种用法一致地构建策略感知字符串。


### Container Server {#container-server}

[Container Server](https://docs.openstack.org/swift/queens/container.html#container-server)起着存储策略非常重要的作用，它负责处理在容器中策略的指令，预
防坏事,比如改变策略或选择了错误的策略,在没有其它参数或事物被指定的时候（回忆前面讨论的Policy-0与default）。

[Container Updater](https://docs.openstack.org/swift/queens/container.html#container-updater)是策略感知的，但是它的工作是非常简单的，通过请求头传相应于
[Account Server](https://docs.openstack.org/swift/queens/container.html#container-updater)策略索引。

[Container Backend](https://docs.openstack.org/swift/queens/container.html#container-backend)同时负责修改现有的DB schema，以及确保新的DB创建schema是支持存储策略的。容器模式的“按需”迁移允许Swift在没有停机的情况下进行升级（无论行数如何，sqlite的alter语句都很快）。为了支持滚动升级（和降级），对container\_stat表的不兼容模式更改将对表进行 container\_info，并且该container\_stat表将替换为包含INSTEAD OF UPDATE触发器的视图，该触发器使其行为类似于旧表。

策略索引存储在此处用于报告有关容器的信息以及管理裂脑情景引起的容器及其存储策略之间的差异。此外，在裂脑期间，必须准备容器以跟踪来自多个策略的对象更新，因此对象表还包括 storage\_policy\_index列。Per-policy对象计数和字节在policy\_stat表中使用INSERT和DELETE触发器更新，类似于container\_stat直接更新的pre-policy触发器。

当Container更新container\_info表中的reconciler\_sync\_point条目时，Container
Replicator守护程序将主动迁移旧模式，作为其正常一致性检查过程的一部分 。这确
保了不会遇到任何写入的读取繁重的容器仍将迁移到与存储后策略查询完全兼容，而不
必回退,不必使用旧模式重试查询以向服务容器读取请求。

[Container Sync](https://docs.openstack.org/swift/queens/container.html#container-sync-daemon)功能，仅需要策略感知的，因为它访问对象环。因此，它需要从容器信
息中提取策略索引，并使用它从POLICIES global 中选择适当的对象环。


### Account Server {#account-server}

[Account Server](https://docs.openstack.org/swift/queens/container.html#container-updater)在存储策略的作用实在有限。当对account发出HEAD请求时（请参阅前面提供的示例），帐户服务器提供存储策略索引，并基于每个策略为客户端构建object\_count和byte\_count信息。

由于某些特定于策略的数据库架构更改，account server能够报告每个存储策略对象和
字节数。特定策略的policy\_stat表能基于每个策略的信息维护信息（每个策略一行）, 以与
account\_stat表相同的方式。该account\_stat表仍然用于相同的目的而不会被
policy\_stat替换，它保存总帐户统计数据，而policy\_stat只是分解数据。后端还负责
通过更改DB schema来迁移pre-storage-policy accounts，并在该时间点, 使用当前
account\_stat数据去填充Policy-0 的policy\_stat表。

pre-storage-policy对象和字节数不会随每个对象PUT和DELETE请求一起更新，而是对
account server的容器更新由swift-container-updater异步执行。


### 升级和确认功能 {#升级和确认功能}

升级到具有存储策略支持的Swift版本并不困难，事实上，集群管理员不需要进行任何特
殊的配置更改即可开始。Swift将自动开始使用现有对象环作为default环和Policy-0环。
添加policy 0的声明是完全可选的，在不存在时，给予隐式策略0的名称将是“Policy-0”。让我们出于测试目的，您希望获取已经拥有大量数据的现有集群，并使用存储策略升级到Swift。从那里你想要继续创建一个策略并测试一些东西。你需要做的就是：

1.  将所有Swift节点升级到策略感知版本的Swift
2.  在/etc/swift/swift.conf中定义您的策略
3.  创建相应的对象环
4.  创建容器和对象，并按预期确认其放置位置

有关引导您完成这些步骤的特定示例，请参阅[Adding Storage Policies to an Existing SAIO](https://docs.openstack.org/swift/queens/policies%5Fsaio.html)。

注意

如果从已启用存储策略的Swift版本降级到不支持策略的旧版本，您将无法访问存储
在索引为0的策略之外的任何数据，但这些对象将显示在容器列表中（如果有网络分
区和un-reconciled对象，可能会重复）。在启用其他存储策略以确保为客户提供一
致的API体验之前，对升级的部署执行任何必要的集成测试非常重要。一旦暴露多个存储策略，请勿降级到不支持存储策略的Swift版本。
