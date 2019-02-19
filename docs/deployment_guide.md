 OpenStack Docs: Deployment Guide                      

https://docs.openstack.org/swift/latest/deployment_guide.html

Deployment Guide
----------------

this page last updated: 2018-10-29 14:49:48

Deployment Guide[¶](#deployment-guide "Permalink to this headline")
===================================================================

Hardware Considerations[¶](#hardware-considerations "Permalink to this headline")
---------------------------------------------------------------------------------

Swift is designed to run on commodity hardware. At Rackspace, our storage servers are currently running fairly generic 4U servers with 24 2T SATA drives and 8 cores of processing power. RAID on the storage drives is not required and not recommended. Swift’s disk usage pattern is the worst case possible for RAID, and performance degrades very quickly using RAID 5 or 6.

Deployment Options[¶](#deployment-options "Permalink to this headline")
-----------------------------------------------------------------------

The Swift services run completely autonomously, which provides for a lot of flexibility when architecting the hardware deployment for Swift. The 4 main services are:

1.  Proxy Services
2.  Object Services
3.  Container Services
4.  Account Services

The Proxy Services are more CPU and network I/O intensive. If you are using 10g networking to the proxy, or are terminating SSL traffic at the proxy, greater CPU power will be required.

The Object, Container, and Account Services (Storage Services) are more disk and network I/O intensive.

The easiest deployment is to install all services on each server. There is nothing wrong with doing this, as it scales each service out horizontally.

At Rackspace, we put the Proxy Services on their own servers and all of the Storage Services on the same server. This allows us to send 10g networking to the proxy and 1g to the storage servers, and keep load balancing to the proxies more manageable. Storage Services scale out horizontally as storage servers are added, and we can scale overall API throughput by adding more Proxies.

If you need more throughput to either Account or Container Services, they may each be deployed to their own servers. For example you might use faster (but more expensive) SAS or even SSD drives to get faster disk I/O to the databases.

A high-availability (HA) deployment of Swift requires that multiple proxy servers are deployed and requests are load-balanced between them. Each proxy server instance is stateless and able to respond to requests for the entire cluster.

Load balancing and network design is left as an exercise to the reader, but this is a very important part of the cluster, so time should be spent designing the network for a Swift cluster.

Web Front End Options[¶](#web-front-end-options "Permalink to this headline")
-----------------------------------------------------------------------------

Swift comes with an integral web front end. However, it can also be deployed as a request processor of an Apache2 using mod\_wsgi as described in [Apache Deployment Guide](apache_deployment_guide.html).

Preparing the Ring[¶](#preparing-the-ring "Permalink to this headline")
-----------------------------------------------------------------------

第一步是确定将在环中的分区数。我们建议每个驱动器至少有100个分区，以确保在驱动器上均匀分布。一个好的起点可能是找出群集将包含的最大驱动器数，然后乘以100，然后向上舍入到最接近的2的幂。

例如，假设我们正在构建一个不超过5,000个驱动器的集群。这意味着我们将拥有总共500,000个分区，这非常接近2^19，向上舍入。

保持分区数量较小（相对）也是一个好主意。分区越多，复制器和其他后端作业必须完成的工作越多，并且环在进程中消耗的内存越多。目标是在最小rings和最大cluster sizes之间找到良好的平衡。

下一步是确定要存储数据的副本数。目前建议使用3（因为这是唯一经过测试的值）。数字越大，使用的存储空间越多，但丢失数据的可能性就越小。

确定群集应具有的zones也很重要。建议至少从5个zones开始。您可以从更少的开始，但我们的测试表明，当发生故障时，至少有五个zones是最佳的。我们还建议尝试将zones配置为尽可能高的级别，以尽可能多地创建隔离。需要考虑的一些事情包括：物理位置，电源可用性和网络连接。例如，在小型集群中，您可能决定按机柜分割zones，每个机柜都有自己的电源和网络连接。zone概念非常抽象，因此可以随意使用它以最佳方式将数据与故障隔离开来。每个zone都存在于一个region中。

region也是一个抽象概念，可用于区分地理上分离的区域，也用于同一数据中心内。Regions and zones需要是正整数。

您现在可以开始构建环：

```shell
swift-ring-builder <builder_file> create <part_power> <replicas> <min_part_hours>
```

这将启动环构建过程，创建带有2^<part_power>分区的<builder_file>。<min_part_hours>是特定分区可以连续移动之前的小时数（24是一个很好的值）。

可以将设备添加到环中：

```shell
swift-ring-builder <builder_file> add r<region>z<zone>-<ip>:<port>/<device_name>_<meta> <weight>
```

这将向环添加设备，其中<builder_file>是先前创建的构建器文件的名称，<region>是zone所在region的编号，<zone>是此设备所在zone的编号，<ip>是设备所在服务器的IP地址，<port>是运行服务器的端口号，<device_name>是服务器上设备的名称（例如：sdb1）， <meta>是设备的元数据串（可选），<weight>是浮点权重，用于确定设备相对于群集中其余设备的分区数量的相对值（一个好的起点是100.0 x 设备上的 TB 数）。在群集中的添加进的每个设备将初始化。

将所有设备添加到环中后，运行：

```shell
swift-ring-builder <builder_file> rebalance
```

这将partitions分配到环中的设备。在进行rebalance之前，每当对环进行更改时，进行所有必需的更改，这一点非常重要。这将确保环保持尽可能平衡，并尽可能少地移动分区。

应该执行上述过程，为每个存储服务（帐户，容器和对象）创建一个ring。在将来对环进行更改时将需要builder文件，因此保留和备份这些文件非常重要。应将生成的.tar.gz环文件推送到群集中的所有服务器。有关构建环的更多信息，运行不带选项的`swift-ring-builder`将显示帮助文本，包含可用命令和可选项。有关环内部工作原理的更多信息，请参阅[Ring Overview](https://docs.openstack.org/swift/latest/overview_ring.html)。

Running object-servers Per Disk[¶](#running-object-servers-per-disk "Permalink to this headline")
-------------------------------------------------------------------------------------------------

The lack of true asynchronous file I/O on Linux leaves the object-server workers vulnerable to misbehaving disks. Because any object-server worker can service a request for any disk, and a slow I/O request blocks the eventlet hub, a single slow disk can impair an entire storage node. This also prevents object servers from fully utilizing all their disks during heavy load.

Another way to get full I/O isolation is to give each disk on a storage node a different port in the storage policy rings. Then set the [servers\_per\_port](#object-server-default-options) option in the object-server config. NOTE: while the purpose of this config setting is to run one or more object-server worker processes per _disk_, the implementation just runs object-servers per unique port of local devices in the rings. The deployer must combine this option with appropriately-configured rings to benefit from this feature.

Here’s an example (abbreviated) old-style ring (2 node cluster with 2 disks each):

Devices:    id  region  zone      ip address  port  replication ip  replication port      name
             0       1     1       1.1.0.1    6200       1.1.0.1                6200      d1
             1       1     1       1.1.0.1    6200       1.1.0.1                6200      d2
             2       1     2       1.1.0.2    6200       1.1.0.2                6200      d3
             3       1     2       1.1.0.2    6200       1.1.0.2                6200      d4

And here’s the same ring set up for servers\_per\_port:

Devices:    id  region  zone      ip address  port  replication ip  replication port      name
             0       1     1       1.1.0.1    6200       1.1.0.1                6200      d1
             1       1     1       1.1.0.1    6201       1.1.0.1                6201      d2
             2       1     2       1.1.0.2    6200       1.1.0.2                6200      d3
             3       1     2       1.1.0.2    6201       1.1.0.2                6201      d4

When migrating from normal to servers\_per\_port, perform these steps in order:

> 1.  Upgrade Swift code to a version capable of doing servers\_per\_port.
> 2.  Enable servers\_per\_port with a > 0 value
> 3.  Restart swift-object-server processes with a SIGHUP. At this point, you will have the servers\_per\_port number of swift-object-server processes serving all requests for all disks on each node. This preserves availability, but you should perform the next step as quickly as possible.
> 4.  Push out new rings that actually have different ports per disk on each server. One of the ports in the new ring should be the same as the port used in the old ring (“6200” in the example above). This will cover existing proxy-server processes who haven’t loaded the new ring yet. They can still talk to any storage node regardless of whether or not that storage node has loaded the ring and started object-server processes on the new ports.

If you do not run a separate object-server for replication, then this setting must be available to the object-replicator and object-reconstructor (i.e. appear in the \[DEFAULT\] config section).

General Service Configuration[¶](#general-service-configuration "Permalink to this headline")
---------------------------------------------------------------------------------------------

Most Swift services fall into two categories. Swift’s wsgi servers and background daemons.

For more information specific to the configuration of Swift’s wsgi servers with paste deploy see [General Server Configuration](#general-server-configuration).

Configuration for servers and daemons can be expressed together in the same file for each type of server, or separately. If a required section for the service trying to start is missing there will be an error. The sections not used by the service are ignored.

Consider the example of an object storage node. By convention, configuration for the object-server, object-updater, object-replicator, object-auditor, and object-reconstructor exist in a single file `/etc/swift/object-server.conf`:

\[DEFAULT\]
reclaim\_age \= 604800

\[pipeline:main\]
pipeline \= object\-server

\[app:object\-server\]
use \= egg:swift#object

\[object\-replicator\]

\[object\-updater\]

\[object\-auditor\]

Swift services expect a configuration path as the first argument:

$ swift-object-auditor
Usage: swift-object-auditor CONFIG \[options\]

Error: missing config path argument

If you omit the object-auditor section this file could not be used as the configuration path when starting the `swift-object-auditor` daemon:

$ swift-object-auditor /etc/swift/object-server.conf
Unable to find object-auditor config section in /etc/swift/object-server.conf

If the configuration path is a directory instead of a file all of the files in the directory with the file extension “.conf” will be combined to generate the configuration object which is delivered to the Swift service. This is referred to generally as “directory based configuration”.

Directory based configuration leverages ConfigParser’s native multi-file support. Files ending in “.conf” in the given directory are parsed in lexicographical order. Filenames starting with ‘.’ are ignored. A mixture of file and directory configuration paths is not supported - if the configuration path is a file only that file will be parsed.

The Swift service management tool `swift-init` has adopted the convention of looking for `/etc/swift/{type}-server.conf.d/` if the file `/etc/swift/{type}-server.conf` file does not exist.

When using directory based configuration, if the same option under the same section appears more than once in different files, the last value parsed is said to override previous occurrences. You can ensure proper override precedence by prefixing the files in the configuration directory with numerical values.:

/etc/swift/
    default.base
    object\-server.conf.d/
        000\_default.conf \-> ../default.base
        001\_default\-override.conf
        010\_server.conf
        020\_replicator.conf
        030\_updater.conf
        040\_auditor.conf

You can inspect the resulting combined configuration object using the `swift-config` command line tool

General Server Configuration[¶](#general-server-configuration "Permalink to this headline")
-------------------------------------------------------------------------------------------

Swift uses paste.deploy ([http://pythonpaste.org/deploy/](http://pythonpaste.org/deploy/)) to manage server configurations.

Default configuration options are set in the \[DEFAULT\] section, and any options specified there can be overridden in any of the other sections BUT ONLY BY USING THE SYNTAX `set option_name = value`. This is the unfortunate way paste.deploy works and I’ll try to explain it in full.

First, here’s an example paste.deploy configuration file:

\[DEFAULT\]
name1 \= globalvalue
name2 \= globalvalue
name3 \= globalvalue
set name4 \= globalvalue

\[pipeline:main\]
pipeline \= myapp

\[app:myapp\]
use \= egg:mypkg#myapp
name2 \= localvalue
set name3 \= localvalue
set name5 \= localvalue
name6 \= localvalue

The resulting configuration that myapp receives is:

global {'\_\_file\_\_': '/etc/mypkg/wsgi.conf', 'here': '/etc/mypkg',
        'name1': 'globalvalue',
        'name2': 'globalvalue',
        'name3': 'localvalue',
        'name4': 'globalvalue',
        'name5': 'localvalue',
        'set name4': 'globalvalue'}
local {'name6': 'localvalue'}

So, name1 got the global value which is fine since it’s only in the DEFAULT section anyway.

name2 got the global value from DEFAULT even though it appears to be overridden in the app:myapp subsection. This is just the unfortunate way paste.deploy works (at least at the time of this writing.)

name3 got the local value from the app:myapp subsection because it is using the special paste.deploy syntax of `set option_name = value`. So, if you want a default value for most app/filters but want to override it in one subsection, this is how you do it.

name4 got the global value from DEFAULT since it’s only in that section anyway. But, since we used the `set` syntax in the DEFAULT section even though we shouldn’t, notice we also got a `set name4` variable. Weird, but probably not harmful.

name5 got the local value from the app:myapp subsection since it’s only there anyway, but notice that it is in the global configuration and not the local configuration. This is because we used the `set` syntax to set the value. Again, weird, but not harmful since Swift just treats the two sets of configuration values as one set anyway.

name6 got the local value from app:myapp subsection since it’s only there, and since we didn’t use the `set` syntax, it’s only in the local configuration and not the global one. Though, as indicated above, there is no special distinction with Swift.

That’s quite an explanation for something that should be so much simpler, but it might be important to know how paste.deploy interprets configuration files. The main rule to remember when working with Swift configuration files is:

Note

Use the `set option_name = value` syntax in subsections if the option is also set in the `[DEFAULT]` section. Don’t get in the habit of always using the `set` syntax or you’ll probably mess up your non-paste.deploy configuration files.

Common configuration[¶](#common-configuration "Permalink to this headline")
---------------------------------------------------------------------------

An example of common configuration file can be found at etc/swift.conf-sample

The following configuration options are available:

  

Option

Default

Description

max\_header\_size

8192

max\_header\_size is the max number of bytes in the utf8 encoding of each header. Using 8192 as default because eventlet use 8192 as max size of header line. This value may need to be increased when using identity v3 API tokens including more than 7 catalog entries. See also include\_service\_catalog in proxy-server.conf-sample (documented in overview\_auth.rst).

extra\_header\_count

0

By default the maximum number of allowed headers depends on the number of max allowed metadata settings plus a default value of 32 for regular http headers. If for some reason this is not enough (custom middleware for example) it can be increased with the extra\_header\_count constraint.

Object Server Configuration[¶](#object-server-configuration "Permalink to this headline")
-----------------------------------------------------------------------------------------

An Example Object Server configuration can be found at etc/object-server.conf-sample in the source code repository.

The following configuration sections are available:

*   [\[DEFAULT\]](#object-server-default-options)
*   [\[object-server\]](#object-server)
*   [\[object-replicator\]](#object-replicator)
*   [\[object-reconstructor\]](#object-reconstructor)
*   [\[object-updater\]](#object-updater)
*   [\[object-auditor\]](#object-auditor)

### \[DEFAULT\][¶](#default "Permalink to this headline")

  

Option

Default

Description

swift\_dir

/etc/swift

Swift configuration directory

devices

/srv/node

Parent directory of where devices are mounted

mount\_check

true

Whether or not check if the devices are mounted to prevent accidentally writing to the root device

bind\_ip

0.0.0.0

IP Address for server to bind to

bind\_port

6200

Port for server to bind to

keep\_idle

600

Value to set for socket TCP\_KEEPIDLE

bind\_timeout

30

Seconds to attempt bind before giving up

backlog

4096

Maximum number of allowed pending connections

workers

auto

Override the number of pre-forked workers that will accept connections. If set it should be an integer, zero means no fork. If unset, it will try to default to the number of effective cpu cores and fallback to one. Increasing the number of workers helps slow filesystem operations in one request from negatively impacting other requests, but only the [servers\_per\_port](#server-per-port-configuration) option provides complete I/O isolation with no measurable overhead.

servers\_per\_port

0

If each disk in each storage policy ring has unique port numbers for its “ip” value, you can use this setting to have each object-server worker only service requests for the single disk matching the port in the ring. The value of this setting determines how many worker processes run for each port (disk) in the ring. If you have 24 disks per server, and this setting is 4, then each storage node will have 1 + (24 \* 4) = 97 total object-server processes running. This gives complete I/O isolation, drastically reducing the impact of slow disks on storage node performance. The object-replicator and object-reconstructor need to see this setting too, so it must be in the \[DEFAULT\] section. See [Running object-servers Per Disk](#server-per-port-configuration).

max\_clients

1024

Maximum number of clients one worker can process simultaneously (it will actually accept(2) N + 1). Setting this to one (1) will only handle one request at a time, without accepting another request concurrently.

disable\_fallocate

false

Disable “fast fail” fallocate checks if the underlying filesystem does not support it.

log\_name

swift

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_address

/dev/log

Logging directory

log\_max\_line\_length

0

Caps the length of log lines to the value given; no limit if set to 0, the default.

log\_custom\_handlers

None

Comma-separated list of functions to call to setup custom log handlers.

log\_udp\_host

 

Override log\_address

log\_udp\_port

514

UDP log port

log\_statsd\_host

None

Enables StatsD logging; IPv4/IPv6 address or a hostname. If a hostname resolves to an IPv4 and IPv6 address, the IPv4 address will be used.

log\_statsd\_port

8125

 

log\_statsd\_default\_sample\_rate

1.0

 

log\_statsd\_sample\_rate\_factor

1.0

 

log\_statsd\_metric\_prefix

 

 

eventlet\_debug

false

If true, turn on debug logging for eventlet

fallocate\_reserve

1%

You can set fallocate\_reserve to the number of bytes or percentage of disk space you’d like fallocate to reserve, whether there is space for the given file size or not. Percentage will be used if the value ends with a ‘%’. This is useful for systems that behave badly when they completely run out of space; you can make the services pretend they’re out of space early.

conn\_timeout

0.5

Time to wait while attempting to connect to another backend node.

node\_timeout

3

Time to wait while sending each chunk of data to another backend node.

client\_timeout

60

Time to wait while receiving each chunk of data from a client or another backend node

network\_chunk\_size

65536

Size of chunks to read/write over the network

disk\_chunk\_size

65536

Size of chunks to read/write to disk

container\_update\_timeout

1

Time to wait while sending a container update on object update.

reclaim\_age

604800

Time elapsed in seconds before the tombstone file representing a deleted object can be reclaimed. This is the maximum window for your consistency engine. If a node that was disconnected from the cluster because of a fault is reintroduced into the cluster after this window without having its data purged it will result in dark data. This setting should be consistent across all object services.

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

### \[object-server\][¶](#object-server "Permalink to this headline")

  

Option

Default

Description

use

 

paste.deploy entry point for the object server. For most cases, this should be egg:swift#object.

set log\_name

object-server

Label used when logging

set log\_facility

LOG\_LOCAL0

Syslog log facility

set log\_level

INFO

Logging level

set log\_requests

True

Whether or not to log each request

set log\_address

/dev/log

Logging directory

user

swift

User to run as

max\_upload\_time

86400

Maximum time allowed to upload an object

slow

0

If > 0, Minimum time in seconds for a PUT or DELETE request to complete. This is only useful to simulate slow devices during testing and development.

mb\_per\_sync

512

On PUT requests, sync file every n MB

keep\_cache\_size

5242880

Largest object size to keep in buffer cache

keep\_cache\_private

false

Allow non-public objects to stay in kernel’s buffer cache

allowed\_headers

Content-Disposition, Content-Encoding, X-Delete-At, X-Object-Manifest, X-Static-Large-Object Cache-Control, Content-Language, Expires, X-Robots-Tag

Comma separated list of headers that can be set in metadata on an object. This list is in addition to X-Object-Meta-\* headers and cannot include Content-Type, etag, Content-Length, or deleted

auto\_create\_account\_prefix

.

Prefix used when automatically creating accounts.

replication\_server

 

Configure parameter for creating specific server. To handle all verbs, including replication verbs, do not specify “replication\_server” (this is the default). To only handle replication, set to a True value (e.g. “True” or “1”). To handle only non-replication verbs, set to “False”. Unless you have a separate replication network, you should not specify any value for “replication\_server”.

replication\_concurrency

4

Set to restrict the number of concurrent incoming SSYNC requests; set to 0 for unlimited

replication\_concurrency\_per\_device

1

Set to restrict the number of concurrent incoming SSYNC requests per device; set to 0 for unlimited requests per devices. This can help control I/O to each device. This does not override replication\_concurrency described above, so you may need to adjust both parameters depending on your hardware or network capacity.

replication\_lock\_timeout

15

Number of seconds to wait for an existing replication device lock before giving up.

replication\_failure\_threshold

100

The number of subrequest failures before the replication\_failure\_ratio is checked

replication\_failure\_ratio

1.0

If the value of failures / successes of SSYNC subrequests exceeds this ratio, the overall SSYNC request will be aborted

splice

no

Use splice() for zero-copy object GETs. This requires Linux kernel version 3.0 or greater. If you set “splice = yes” but the kernel does not support it, error messages will appear in the object server logs at startup, but your object servers should continue to function.

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

eventlet\_tpool\_num\_threads

auto

The number of threads in eventlet’s thread pool. Most IO will occur in the object server’s main thread, but certain “heavy” IO operations will occur in separate IO threads, managed by eventlet. The default value is auto, whose actual value is dependent on the servers\_per\_port value. If servers\_per\_port is zero then it uses eventlet’s default (currently 20 threads). If the servers\_per\_port is nonzero then it’ll only use 1 thread per process. This value can be overridden with an integer value.

### \[object-replicator\][¶](#object-replicator "Permalink to this headline")

  

Option

Default

Description

log\_name

object-replicator

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_address

/dev/log

Logging directory

daemonize

yes

Whether or not to run replication as a daemon

interval

30

Time in seconds to wait between replication passes

concurrency

1

Number of replication jobs to run per worker process

replicator\_workers

0

Number of worker processes to use. No matter how big this number is, at most one worker per disk will be used. The default value of 0 means no forking; all work is done in the main process.

sync\_method

rsync

The sync method to use; default is rsync but you can use ssync to try the EXPERIMENTAL all-swift-code-no-rsync-callouts method. Once ssync is verified as or better than, rsync, we plan to deprecate rsync so we can move on with more features for replication.

rsync\_timeout

900

Max duration of a partition rsync

rsync\_bwlimit

0

Bandwidth limit for rsync in kB/s. 0 means unlimited.

rsync\_io\_timeout

30

Timeout value sent to rsync –timeout and –contimeout options

rsync\_compress

no

Allow rsync to compress data which is transmitted to destination node during sync. However, this is applicable only when destination node is in a different region than the local one. NOTE: Objects that are already compressed (for example: .tar.gz, .mp3) might slow down the syncing process.

stats\_interval

300

Interval in seconds between logging replication statistics

handoffs\_first

false

If set to True, partitions that are not supposed to be on the node will be replicated first. The default setting should not be changed, except for extreme situations.

handoff\_delete

auto

By default handoff partitions will be removed when it has successfully replicated to all the canonical nodes. If set to an integer n, it will remove the partition if it is successfully replicated to n nodes. The default setting should not be changed, except for extreme situations.

node\_timeout

DEFAULT or 10

Request timeout to external services. This uses what’s set here, or what’s set in the DEFAULT section, or 10 (though other sections use 3 as the final default).

http\_timeout

60

Max duration of an http request. This is for REPLICATE finalization calls and so should be longer than node\_timeout.

lockup\_timeout

1800

Attempts to kill all workers if nothing replicates for lockup\_timeout seconds

rsync\_module

{replication\_ip}::object

Format of the rsync module where the replicator will send data. The configuration value can include some variables that will be extracted from the ring. Variables must follow the format {NAME} where NAME is one of: ip, port, replication\_ip, replication\_port, region, zone, device, meta. See etc/rsyncd.conf-sample for some examples.

rsync\_error\_log\_line\_length

0

Limits how long rsync error log lines are

ring\_check\_interval

15

Interval for checking new ring file

recon\_cache\_path

/var/cache/swift

Path to recon cache

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

### \[object-reconstructor\][¶](#object-reconstructor "Permalink to this headline")

  

Option

Default

Description

log\_name

object-reconstructor

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_address

/dev/log

Logging directory

daemonize

yes

Whether or not to run reconstruction as a daemon

interval

30

Time in seconds to wait between reconstruction passes

reconstructor\_workers

0

Maximum number of worker processes to spawn. Each worker will handle a subset of devices. Devices will be assigned evenly among the workers so that workers cycle at similar intervals (which can lead to fewer workers than requested). You can not have more workers than devices. If you have no devices only a single worker is spawned.

concurrency

1

Number of reconstruction threads to spawn per reconstructor process.

stats\_interval

300

Interval in seconds between logging reconstruction statistics

handoffs\_only

false

The handoffs\_only mode option is for special case emergency situations during rebalance such as disk full in the cluster. This option SHOULD NOT BE CHANGED, except for extreme situations. When handoffs\_only mode is enabled the reconstructor will _only_ revert fragments from handoff nodes to primary nodes and will not sync primary nodes with neighboring primary nodes. This will force the reconstructor to sync and delete handoffs’ fragments more quickly and minimize the time of the rebalance by limiting the number of rebuilds. The handoffs\_only option is only for temporary use and should be disabled as soon as the emergency situation has been resolved.

node\_timeout

DEFAULT or 10

Request timeout to external services. The value used is the value set in this section, or the value set in the DEFAULT section, or 10.

http\_timeout

60

Max duration of an http request. This is for REPLICATE finalization calls and so should be longer than node\_timeout.

lockup\_timeout

1800

Attempts to kill all threads if no fragment has been reconstructed for lockup\_timeout seconds.

ring\_check\_interval

15

Interval for checking new ring file

recon\_cache\_path

/var/cache/swift

Path to recon cache

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

### \[object-updater\][¶](#object-updater "Permalink to this headline")

  

Option

Default

Description

log\_name

object-updater

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_address

/dev/log

Logging directory

interval

300

Minimum time for a pass to take

updater\_workers

1

Number of worker processes

concurrency

8

Number of updates to run concurrently in each worker process

node\_timeout

DEFAULT or 10

Request timeout to external services. This uses what’s set here, or what’s set in the DEFAULT section, or 10 (though other sections use 3 as the final default).

objects\_per\_second

50

Maximum objects updated per second. Should be tuned according to individual system specs. 0 is unlimited.

slowdown

0.01

Time in seconds to wait between objects. Deprecated in favor of objects\_per\_second.

report\_interval

300

Interval in seconds between logging statistics about the current update pass.

recon\_cache\_path

/var/cache/swift

Path to recon cache

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

### \[object-auditor\][¶](#object-auditor "Permalink to this headline")

  

Option

Default

Description

log\_name

object-auditor

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_address

/dev/log

Logging directory

log\_time

3600

Frequency of status logs in seconds.

interval

30

Time in seconds to wait between auditor passes

disk\_chunk\_size

65536

Size of chunks read during auditing

files\_per\_second

20

Maximum files audited per second per auditor process. Should be tuned according to individual system specs. 0 is unlimited.

bytes\_per\_second

10000000

Maximum bytes audited per second per auditor process. Should be tuned according to individual system specs. 0 is unlimited.

concurrency

1

The number of parallel processes to use for checksum auditing.

zero\_byte\_files\_per\_second

50

 

object\_size\_stats

 

 

recon\_cache\_path

/var/cache/swift

Path to recon cache

rsync\_tempfile\_timeout

auto

Time elapsed in seconds before rsync tempfiles will be unlinked. Config value of “auto” try to use object-replicator’s rsync\_timeout + 900 or fallback to 86400 (1 day).

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

Container Server Configuration[¶](#container-server-configuration "Permalink to this headline")
-----------------------------------------------------------------------------------------------

An example Container Server configuration can be found at etc/container-server.conf-sample in the source code repository.

The following configuration sections are available:

*   [\[DEFAULT\]](#container-server-default-options)
*   [\[container-server\]](#container-server)
*   [\[container-replicator\]](#container-replicator)
*   [\[container-updater\]](#container-updater)
*   [\[container-auditor\]](#container-auditor)

### \[DEFAULT\][¶](#container-server-default-options "Permalink to this headline")

  

Option

Default

Description

swift\_dir

/etc/swift

Swift configuration directory

devices

/srv/node

Parent directory of where devices are mounted

mount\_check

true

Whether or not check if the devices are mounted to prevent accidentally writing to the root device

bind\_ip

0.0.0.0

IP Address for server to bind to

bind\_port

6201

Port for server to bind to

keep\_idle

600

Value to set for socket TCP\_KEEPIDLE

bind\_timeout

30

Seconds to attempt bind before giving up

backlog

4096

Maximum number of allowed pending connections

workers

auto

Override the number of pre-forked workers that will accept connections. If set it should be an integer, zero means no fork. If unset, it will try to default to the number of effective cpu cores and fallback to one. Increasing the number of workers may reduce the possibility of slow file system operations in one request from negatively impacting other requests. See [General Service Tuning](#general-service-tuning).

max\_clients

1024

Maximum number of clients one worker can process simultaneously (it will actually accept(2) N + 1). Setting this to one (1) will only handle one request at a time, without accepting another request concurrently.

user

swift

User to run as

disable\_fallocate

false

Disable “fast fail” fallocate checks if the underlying filesystem does not support it.

log\_name

swift

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_address

/dev/log

Logging directory

log\_max\_line\_length

0

Caps the length of log lines to the value given; no limit if set to 0, the default.

log\_custom\_handlers

None

Comma-separated list of functions to call to setup custom log handlers.

log\_udp\_host

 

Override log\_address

log\_udp\_port

514

UDP log port

log\_statsd\_host

None

Enables StatsD logging; IPv4/IPv6 address or a hostname. If a hostname resolves to an IPv4 and IPv6 address, the IPv4 address will be used.

log\_statsd\_port

8125

 

log\_statsd\_default\_sample\_rate

1.0

 

log\_statsd\_sample\_rate\_factor

1.0

 

log\_statsd\_metric\_prefix

 

 

eventlet\_debug

false

If true, turn on debug logging for eventlet

fallocate\_reserve

1%

You can set fallocate\_reserve to the number of bytes or percentage of disk space you’d like fallocate to reserve, whether there is space for the given file size or not. Percentage will be used if the value ends with a ‘%’. This is useful for systems that behave badly when they completely run out of space; you can make the services pretend they’re out of space early.

db\_preallocation

off

If you don’t mind the extra disk space usage in overhead, you can turn this on to preallocate disk space with SQLite databases to decrease fragmentation.

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

### \[container-server\][¶](#container-server "Permalink to this headline")

  

Option

Default

Description

use

 

paste.deploy entry point for the container server. For most cases, this should be egg:swift#container.

set log\_name

container-server

Label used when logging

set log\_facility

LOG\_LOCAL0

Syslog log facility

set log\_level

INFO

Logging level

set log\_requests

True

Whether or not to log each request

set log\_address

/dev/log

Logging directory

node\_timeout

3

Request timeout to external services

conn\_timeout

0.5

Connection timeout to external services

allow\_versions

false

Enable/Disable object versioning feature

auto\_create\_account\_prefix

.

Prefix used when automatically

replication\_server

 

Configure parameter for creating specific server. To handle all verbs, including replication verbs, do not specify “replication\_server” (this is the default). To only handle replication, set to a True value (e.g. “True” or “1”). To handle only non-replication verbs, set to “False”. Unless you have a separate replication network, you should not specify any value for “replication\_server”.

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

### \[container-replicator\][¶](#container-replicator "Permalink to this headline")

  

Option

Default

Description

log\_name

container-replicator

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_address

/dev/log

Logging directory

per\_diff

1000

Maximum number of database rows that will be sync’d in a single HTTP replication request. Databases with less than or equal to this number of differing rows will always be sync’d using an HTTP replication request rather than using rsync.

max\_diffs

100

Maximum number of HTTP replication requests attempted on each replication pass for any one container. This caps how long the replicator will spend trying to sync a given database per pass so the other databases don’t get starved.

concurrency

8

Number of replication workers to spawn

interval

30

Time in seconds to wait between replication passes

databases\_per\_second

50

Maximum databases to process per second. Should be tuned according to individual system specs. 0 is unlimited.

node\_timeout

10

Request timeout to external services

conn\_timeout

0.5

Connection timeout to external services

reclaim\_age

604800

Time elapsed in seconds before a container can be reclaimed

rsync\_module

{replication\_ip}::container

Format of the rsync module where the replicator will send data. The configuration value can include some variables that will be extracted from the ring. Variables must follow the format {NAME} where NAME is one of: ip, port, replication\_ip, replication\_port, region, zone, device, meta. See etc/rsyncd.conf-sample for some examples.

rsync\_compress

no

Allow rsync to compress data which is transmitted to destination node during sync. However, this is applicable only when destination node is in a different region than the local one. NOTE: Objects that are already compressed (for example: .tar.gz, mp3) might slow down the syncing process.

recon\_cache\_path

/var/cache/swift

Path to recon cache

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

### \[container-updater\][¶](#container-updater "Permalink to this headline")

  

Option

Default

Description

log\_name

container-updater

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_address

/dev/log

Logging directory

interval

300

Minimum time for a pass to take

concurrency

4

Number of updater workers to spawn

node\_timeout

3

Request timeout to external services

conn\_timeout

0.5

Connection timeout to external services

containers\_per\_second

50

Maximum containers updated per second. Should be tuned according to individual system specs. 0 is unlimited.

slowdown

0.01

Time in seconds to wait between containers. Deprecated in favor of containers\_per\_second.

account\_suppression\_time

60

Seconds to suppress updating an account that has generated an error (timeout, not yet found, etc.)

recon\_cache\_path

/var/cache/swift

Path to recon cache

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

### \[container-auditor\][¶](#container-auditor "Permalink to this headline")

  

Option

Default

Description

log\_name

container-auditor

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_address

/dev/log

Logging directory

interval

1800

Minimum time for a pass to take

containers\_per\_second

200

Maximum containers audited per second. Should be tuned according to individual system specs. 0 is unlimited.

recon\_cache\_path

/var/cache/swift

Path to recon cache

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

Account Server Configuration[¶](#account-server-configuration "Permalink to this headline")
-------------------------------------------------------------------------------------------

An example Account Server configuration can be found at etc/account-server.conf-sample in the source code repository.

The following configuration sections are available:

*   [\[DEFAULT\]](#account-server-default-options)
*   [\[account-server\]](#account-server)
*   [\[account-replicator\]](#account-replicator)
*   [\[account-auditor\]](#account-auditor)
*   [\[account-reaper\]](#account-reaper)

### \[DEFAULT\][¶](#account-server-default-options "Permalink to this headline")

  

Option

Default

Description

swift\_dir

/etc/swift

Swift configuration directory

devices

/srv/node

Parent directory or where devices are mounted

mount\_check

true

Whether or not check if the devices are mounted to prevent accidentally writing to the root device

bind\_ip

0.0.0.0

IP Address for server to bind to

bind\_port

6202

Port for server to bind to

keep\_idle

600

Value to set for socket TCP\_KEEPIDLE

bind\_timeout

30

Seconds to attempt bind before giving up

backlog

4096

Maximum number of allowed pending connections

workers

auto

Override the number of pre-forked workers that will accept connections. If set it should be an integer, zero means no fork. If unset, it will try to default to the number of effective cpu cores and fallback to one. Increasing the number of workers may reduce the possibility of slow file system operations in one request from negatively impacting other requests. See [General Service Tuning](#general-service-tuning).

max\_clients

1024

Maximum number of clients one worker can process simultaneously (it will actually accept(2) N + 1). Setting this to one (1) will only handle one request at a time, without accepting another request concurrently.

user

swift

User to run as

db\_preallocation

off

If you don’t mind the extra disk space usage in overhead, you can turn this on to preallocate disk space with SQLite databases to decrease fragmentation.

disable\_fallocate

false

Disable “fast fail” fallocate checks if the underlying filesystem does not support it.

log\_name

swift

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_address

/dev/log

Logging directory

log\_max\_line\_length

0

Caps the length of log lines to the value given; no limit if set to 0, the default.

log\_custom\_handlers

None

Comma-separated list of functions to call to setup custom log handlers.

log\_udp\_host

 

Override log\_address

log\_udp\_port

514

UDP log port

log\_statsd\_host

None

Enables StatsD logging; IPv4/IPv6 address or a hostname. If a hostname resolves to an IPv4 and IPv6 address, the IPv4 address will be used.

log\_statsd\_port

8125

 

log\_statsd\_default\_sample\_rate

1.0

 

log\_statsd\_sample\_rate\_factor

1.0

 

log\_statsd\_metric\_prefix

 

 

eventlet\_debug

false

If true, turn on debug logging for eventlet

fallocate\_reserve

1%

You can set fallocate\_reserve to the number of bytes or percentage of disk space you’d like fallocate to reserve, whether there is space for the given file size or not. Percentage will be used if the value ends with a ‘%’. This is useful for systems that behave badly when they completely run out of space; you can make the services pretend they’re out of space early.

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

### \[account-server\][¶](#account-server "Permalink to this headline")

  

Option

Default

Description

use

 

Entry point for paste.deploy for the account server. For most cases, this should be egg:swift#account.

set log\_name

account-server

Label used when logging

set log\_facility

LOG\_LOCAL0

Syslog log facility

set log\_level

INFO

Logging level

set log\_requests

True

Whether or not to log each request

set log\_address

/dev/log

Logging directory

auto\_create\_account\_prefix

.

Prefix used when automatically creating accounts.

replication\_server

 

Configure parameter for creating specific server. To handle all verbs, including replication verbs, do not specify “replication\_server” (this is the default). To only handle replication, set to a True value (e.g. “True” or “1”). To handle only non-replication verbs, set to “False”. Unless you have a separate replication network, you should not specify any value for “replication\_server”.

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

### \[account-replicator\][¶](#account-replicator "Permalink to this headline")

  

Option

Default

Description

log\_name

account-replicator

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_address

/dev/log

Logging directory

per\_diff

1000

Maximum number of database rows that will be sync’d in a single HTTP replication request. Databases with less than or equal to this number of differing rows will always be sync’d using an HTTP replication request rather than using rsync.

max\_diffs

100

Maximum number of HTTP replication requests attempted on each replication pass for any one container. This caps how long the replicator will spend trying to sync a given database per pass so the other databases don’t get starved.

concurrency

8

Number of replication workers to spawn

interval

30

Time in seconds to wait between replication passes

databases\_per\_second

50

Maximum databases to process per second. Should be tuned according to individual system specs. 0 is unlimited.

node\_timeout

10

Request timeout to external services

conn\_timeout

0.5

Connection timeout to external services

reclaim\_age

604800

Time elapsed in seconds before an account can be reclaimed

rsync\_module

{replication\_ip}::account

Format of the rsync module where the replicator will send data. The configuration value can include some variables that will be extracted from the ring. Variables must follow the format {NAME} where NAME is one of: ip, port, replication\_ip, replication\_port, region, zone, device, meta. See etc/rsyncd.conf-sample for some examples.

rsync\_compress

no

Allow rsync to compress data which is transmitted to destination node during sync. However, this is applicable only when destination node is in a different region than the local one. NOTE: Objects that are already compressed (for example: .tar.gz, mp3) might slow down the syncing process.

recon\_cache\_path

/var/cache/swift

Path to recon cache

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

### \[account-auditor\][¶](#account-auditor "Permalink to this headline")

  

Option

Default

Description

log\_name

account-auditor

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_address

/dev/log

Logging directory

interval

1800

Minimum time for a pass to take

accounts\_per\_second

200

Maximum accounts audited per second. Should be tuned according to individual system specs. 0 is unlimited.

recon\_cache\_path

/var/cache/swift

Path to recon cache

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

### \[account-reaper\][¶](#account-reaper "Permalink to this headline")

  

Option

Default

Description

log\_name

account-reaper

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_address

/dev/log

Logging directory

concurrency

25

Number of replication workers to spawn

interval

3600

Minimum time for a pass to take

node\_timeout

10

Request timeout to external services

conn\_timeout

0.5

Connection timeout to external services

delay\_reaping

0

Normally, the reaper begins deleting account information for deleted accounts immediately; you can set this to delay its work however. The value is in seconds, 2592000 = 30 days, for example. The sum of this value and the container-updater `interval` should be less than the account-replicator `reclaim_age`. This ensures that once the account-reaper has deleted a container there is sufficient time for the container-updater to report to the account before the account DB is removed.

reap\_warn\_after

2892000

If the account fails to be be reaped due to a persistent error, the account reaper will log a message such as: Account <name> has not been reaped since <date> You can search logs for this message if space is not being reclaimed after you delete account(s). This is in addition to any time requested by delay\_reaping.

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

Proxy Server Configuration[¶](#proxy-server-configuration "Permalink to this headline")
---------------------------------------------------------------------------------------

An example Proxy Server configuration can be found at etc/proxy-server.conf-sample in the source code repository.

The following configuration sections are available:

*   [\[DEFAULT\]](#proxy-server-default-options)
*   [\[proxy-server\]](#proxy-server)
*   Individual sections for [Proxy middlewares](#proxy-middlewares)

### \[DEFAULT\][¶](#proxy-server-default-options "Permalink to this headline")

  

Option

Default

Description

bind\_ip

0.0.0.0

IP Address for server to bind to

bind\_port

80

Port for server to bind to

keep\_idle

600

Value to set for socket TCP\_KEEPIDLE

bind\_timeout

30

Seconds to attempt bind before giving up

backlog

4096

Maximum number of allowed pending connections

swift\_dir

/etc/swift

Swift configuration directory

workers

auto

Override the number of pre-forked workers that will accept connections. If set it should be an integer, zero means no fork. If unset, it will try to default to the number of effective cpu cores and fallback to one. See [General Service Tuning](#general-service-tuning).

max\_clients

1024

Maximum number of clients one worker can process simultaneously (it will actually accept(2) N + 1). Setting this to one (1) will only handle one request at a time, without accepting another request concurrently.

user

swift

User to run as

cert\_file

 

Path to the ssl .crt. This should be enabled for testing purposes only.

key\_file

 

Path to the ssl .key. This should be enabled for testing purposes only.

cors\_allow\_origin

 

List of origin hosts that are allowed for CORS requests in addition to what the container has set.

strict\_cors\_mode

True

If True (default) then CORS requests are only allowed if their Origin header matches an allowed origin. Otherwise, any Origin is allowed.

cors\_expose\_headers

 

This is a list of headers that are included in the header Access-Control-Expose-Headers in addition to what the container has set.

client\_timeout

60

 

trans\_id\_suffix

 

This optional suffix (default is empty) that would be appended to the swift transaction id allows one to easily figure out from which cluster that X-Trans-Id belongs to. This is very useful when one is managing more than one swift cluster.

log\_name

swift

Label used when logging

log\_facility

LOG\_LOCAL0

Syslog log facility

log\_level

INFO

Logging level

log\_headers

False

 

log\_address

/dev/log

Logging directory

log\_max\_line\_length

0

Caps the length of log lines to the value given; no limit if set to 0, the default.

log\_custom\_handlers

None

Comma separated list of functions to call to setup custom log handlers.

log\_udp\_host

 

Override log\_address

log\_udp\_port

514

UDP log port

log\_statsd\_host

None

Enables StatsD logging; IPv4/IPv6 address or a hostname. If a hostname resolves to an IPv4 and IPv6 address, the IPv4 address will be used.

log\_statsd\_port

8125

 

log\_statsd\_default\_sample\_rate

1.0

 

log\_statsd\_sample\_rate\_factor

1.0

 

log\_statsd\_metric\_prefix

 

 

eventlet\_debug

false

If true, turn on debug logging for eventlet

expose\_info

true

Enables exposing configuration settings via HTTP GET /info.

admin\_key

 

Key to use for admin calls that are HMAC signed. Default is empty, which will disable admin calls to /info.

disallowed\_sections

swift.valid\_api\_versions

Allows the ability to withhold sections from showing up in the public calls to /info. You can withhold subsections by separating the dict level with a “.”.

expiring\_objects\_container\_divisor

86400

 

expiring\_objects\_account\_name

expiring\_objects

 

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort) and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

### \[proxy-server\][¶](#proxy-server "Permalink to this headline")

  

Option

Default

Description

use

 

Entry point for paste.deploy for the proxy server. For most cases, this should be egg:swift#proxy.

set log\_name

proxy-server

Label used when logging

set log\_facility

LOG\_LOCAL0

Syslog log facility

set log\_level

INFO

Log level

set log\_headers

True

If True, log headers in each request

set log\_handoffs

True

If True, the proxy will log whenever it has to failover to a handoff node

recheck\_account\_existence

60

Cache timeout in seconds to send memcached for account existence

recheck\_container\_existence

60

Cache timeout in seconds to send memcached for container existence

object\_chunk\_size

65536

Chunk size to read from object servers

client\_chunk\_size

65536

Chunk size to read from clients

memcache\_servers

127.0.0.1:11211

Comma separated list of memcached servers ip:port or \[ipv6addr\]:port

memcache\_max\_connections

2

Max number of connections to each memcached server per worker

node\_timeout

10

Request timeout to external services

recoverable\_node\_timeout

node\_timeout

Request timeout to external services for requests that, on failure, can be recovered from. For example, object GET.

client\_timeout

60

Timeout to read one chunk from a client

conn\_timeout

0.5

Connection timeout to external services

error\_suppression\_interval

60

Time in seconds that must elapse since the last error for a node to be considered no longer error limited

error\_suppression\_limit

10

Error count to consider a node error limited

allow\_account\_management

false

Whether account PUTs and DELETEs are even callable

account\_autocreate

false

If set to ‘true’ authorized accounts that do not yet exist within the Swift cluster will be automatically created.

max\_containers\_per\_account

0

If set to a positive value, trying to create a container when the account already has at least this maximum containers will result in a 403 Forbidden. Note: This is a soft limit, meaning a user might exceed the cap for recheck\_account\_existence before the 403s kick in.

max\_containers\_whitelist

 

This is a comma separated list of account names that ignore the max\_containers\_per\_account cap.

rate\_limit\_after\_segment

10

Rate limit the download of large object segments after this segment is downloaded.

rate\_limit\_segments\_per\_sec

1

Rate limit large object downloads at this rate.

request\_node\_count

2 \* replicas

Set to the number of nodes to contact for a normal request. You can use ‘\* replicas’ at the end to have it use the number given times the number of replicas for the ring being used for the request.

swift\_owner\_headers

<see the sample conf file for the list of default headers>

These are the headers whose values will only be shown to swift\_owners. The exact definition of a swift\_owner is up to the auth system in use, but usually indicates administrative responsibilities.

sorting\_method

shuffle

Storage nodes can be chosen at random (shuffle), by using timing measurements (timing), or by using an explicit match (affinity). Using timing measurements may allow for lower overall latency, while using affinity allows for finer control. In both the timing and affinity cases, equally-sorting nodes are still randomly chosen to spread load. This option may be overridden in a per-policy configuration section.

timing\_expiry

300

If the “timing” sorting\_method is used, the timings will only be valid for the number of seconds configured by timing\_expiry.

concurrent\_gets

off

Use replica count number of threads concurrently during a GET/HEAD and return with the first successful response. In the EC case, this parameter only affects an EC HEAD as an EC GET behaves differently.

concurrency\_timeout

conn\_timeout

This parameter controls how long to wait before firing off the next concurrent\_get thread. A value of 0 would we fully concurrent, any other number will stagger the firing of the threads. This number should be between 0 and node\_timeout. The default is conn\_timeout (0.5).

nice\_priority

None

Scheduling priority of server processes. Niceness values range from -20 (most favorable to the process) to 19 (least favorable to the process). The default does not modify priority.

ionice\_class

None

I/O scheduling class of server processes. I/O niceness class values are IOPRIO\_CLASS\_RT (realtime), IOPRIO\_CLASS\_BE (best-effort), and IOPRIO\_CLASS\_IDLE (idle). The default does not modify class and priority. Linux supports io scheduling priorities and classes since 2.6.13 with the CFQ io scheduler. Work only with ionice\_priority.

ionice\_priority

None

I/O scheduling priority of server processes. I/O niceness priority is a number which goes from 0 to 7. The higher the value, the lower the I/O priority of the process. Work only with ionice\_class. Ignored if IOPRIO\_CLASS\_IDLE is set.

read\_affinity

None

Specifies which backend servers to prefer on reads; used in conjunction with the sorting\_method option being set to ‘affinity’. Format is a comma separated list of affinity descriptors of the form <selection>=<priority>. The <selection> may be r<N> for selecting nodes in region N or r<N>z<M> for selecting nodes in region N, zone M. The <priority> value should be a whole number that represents the priority to be given to the selection; lower numbers are higher priority. Default is empty, meaning no preference. This option may be overridden in a per-policy configuration section.

write\_affinity

None

Specifies which backend servers to prefer on writes. Format is a comma separated list of affinity descriptors of the form r<N> for region N or r<N>z<M> for region N, zone M. Default is empty, meaning no preference. This option may be overridden in a per-policy configuration section.

write\_affinity\_node\_count

2 \* replicas

The number of local (as governed by the write\_affinity setting) nodes to attempt to contact first on writes, before any non-local ones. The value should be an integer number, or use ‘\* replicas’ at the end to have it use the number given times the number of replicas for the ring being used for the request. This option may be overridden in a per-policy configuration section.

write\_affinity\_handoff\_delete\_count

auto

The number of local (as governed by the write\_affinity setting) handoff nodes to attempt to contact on deletion, in addition to primary nodes. Example: in geographically distributed deployment, If replicas=3, sometimes there may be 1 primary node and 2 local handoff nodes in one region holding the object after uploading but before object replicated to the appropriate locations in other regions. In this case, include these handoff nodes to send request when deleting object could help make correct decision for the response. The default value ‘auto’ means Swift will calculate the number automatically, the default value is (replicas - len(local\_primary\_nodes)). This option may be overridden in a per-policy configuration section.

### Per policy configuration[¶](#per-policy-configuration "Permalink to this headline")

Some proxy-server configuration options may be overridden for individual [Storage Policies](overview_policies.html) by including per-policy config section(s). These options are:

*   `sorting_method`
*   `read_affinity`
*   `write_affinity`
*   `write_affinity_node_count`
*   `write_affinity_handoff_delete_count`

The per-policy config section name must be of the form:

\[proxy\-server:policy:<policy index\>\]

Note

The per-policy config section name should refer to the policy index, not the policy name.

Note

The first part of proxy-server config section name must match the name of the proxy-server config section. This is typically `proxy-server` as shown above, but if different then the names of any per-policy config sections must be changed accordingly.

The value of an option specified in a per-policy section will override any value given in the proxy-server section for that policy only. Otherwise the value of these options will be that specified in the proxy-server section.

For example, the following section provides policy-specific options for a policy with index `3`:

\[proxy\-server:policy:3\]
sorting\_method \= affinity
read\_affinity \= r2\=1
write\_affinity \= r2
write\_affinity\_node\_count \= 1 \* replicas
write\_affinity\_handoff\_delete\_count \= 2

Note

It is recommended that per-policy config options are _not_ included in the `[DEFAULT]` section. If they are then the following behavior applies.

Per-policy config sections will inherit options in the `[DEFAULT]` section of the config file, and any such inheritance will take precedence over inheriting options from the proxy-server config section.

Per-policy config section options will override options in the `[DEFAULT]` section. Unlike the behavior described under [General Server Configuration](#id2) for paste-deploy `filter` and `app` sections, the `set` keyword is not required for options to override in per-policy config sections.

For example, given the following settings in a config file:

\[DEFAULT\]
sorting\_method \= affinity
read\_affinity \= r0\=100
write\_affinity \= r0

\[app:proxy\-server\]
use \= egg:swift#proxy
\# use of set keyword here overrides \[DEFAULT\] option
set read\_affinity \= r1\=100
\# without set keyword, \[DEFAULT\] option overrides in a paste-deploy section
write\_affinity \= r1

\[proxy\-server:policy:0\]
sorting\_method \= affinity
\# set keyword not required here to override \[DEFAULT\] option
write\_affinity \= r1

would result in policy with index `0` having settings:

*   `read_affinity = r0=100` (inherited from the `[DEFAULT]` section)
*   `write_affinity = r1` (specified in the policy 0 section)

and any other policy would have the default settings of:

*   `read_affinity = r1=100` (set in the proxy-server section)
*   `write_affinity = r0` (inherited from the `[DEFAULT]` section)

### Proxy Middlewares[¶](#proxy-middlewares "Permalink to this headline")

Many features in Swift are implemented as middleware in the proxy-server pipeline. See [Middleware](middleware.html) and the `proxy-server.conf-sample` file for more information. In particular, the use of some type of [authentication and authorization middleware](overview_auth.html) is highly recommended.

Memcached Considerations[¶](#memcached-considerations "Permalink to this headline")
-----------------------------------------------------------------------------------

Several of the Services rely on Memcached for caching certain types of lookups, such as auth tokens, and container/account existence. Swift does not do any caching of actual object data. Memcached should be able to run on any servers that have available RAM and CPU. At Rackspace, we run Memcached on the proxy servers. The memcache\_servers config option in the proxy-server.conf should contain all memcached servers.

System Time[¶](#system-time "Permalink to this headline")
---------------------------------------------------------

Time may be relative but it is relatively important for Swift! Swift uses timestamps to determine which is the most recent version of an object. It is very important for the system time on each server in the cluster to by synced as closely as possible (more so for the proxy server, but in general it is a good idea for all the servers). At Rackspace, we use NTP with a local NTP server to ensure that the system times are as close as possible. This should also be monitored to ensure that the times do not vary too much.

General Service Tuning[¶](#general-service-tuning "Permalink to this headline")
-------------------------------------------------------------------------------

Most services support either a worker or concurrency value in the settings. This allows the services to make effective use of the cores available. A good starting point to set the concurrency level for the proxy and storage services to 2 times the number of cores available. If more than one service is sharing a server, then some experimentation may be needed to find the best balance.

At Rackspace, our Proxy servers have dual quad core processors, giving us 8 cores. Our testing has shown 16 workers to be a pretty good balance when saturating a 10g network and gives good CPU utilization.

Our Storage server processes all run together on the same servers. These servers have dual quad core processors, for 8 cores total. We run the Account, Container, and Object servers with 8 workers each. Most of the background jobs are run at a concurrency of 1, with the exception of the replicators which are run at a concurrency of 2.

The max\_clients parameter can be used to adjust the number of client requests an individual worker accepts for processing. The fewer requests being processed at one time, the less likely a request that consumes the worker’s CPU time, or blocks in the OS, will negatively impact other requests. The more requests being processed at one time, the more likely one worker can utilize network and disk capacity.

On systems that have more cores, and more memory, where one can afford to run more workers, raising the number of workers and lowering the maximum number of clients serviced per worker can lessen the impact of CPU intensive or stalled requests.

The nice\_priority parameter can be used to set program scheduling priority. The ionice\_class and ionice\_priority parameters can be used to set I/O scheduling class and priority on the systems that use an I/O scheduler that supports I/O priorities. As at kernel 2.6.17 the only such scheduler is the Completely Fair Queuing (CFQ) I/O scheduler. If you run your Storage servers all together on the same servers, you can slow down the auditors or prioritize object-server I/O via these parameters (but probably do not need to change it on the proxy). It is a new feature and the best practices are still being developed. On some systems it may be required to run the daemons as root. For more info also see setpriority(2) and ioprio\_set(2).

The above configuration setting should be taken as suggestions and testing of configuration settings should be done to ensure best utilization of CPU, network connectivity, and disk I/O.

Filesystem Considerations[¶](#filesystem-considerations "Permalink to this headline")
-------------------------------------------------------------------------------------

Swift is designed to be mostly filesystem agnostic–the only requirement being that the filesystem supports extended attributes (xattrs). After thorough testing with our use cases and hardware configurations, XFS was the best all-around choice. If you decide to use a filesystem other than XFS, we highly recommend thorough testing.

For distros with more recent kernels (for example Ubuntu 12.04 Precise), we recommend using the default settings (including the default inode size of 256 bytes) when creating the file system:

mkfs.xfs /dev/sda1

In the last couple of years, XFS has made great improvements in how inodes are allocated and used. Using the default inode size no longer has an impact on performance.

For distros with older kernels (for example Ubuntu 10.04 Lucid), some settings can dramatically impact performance. We recommend the following when creating the file system:

mkfs.xfs \-i size\=1024 /dev/sda1

Setting the inode size is important, as XFS stores xattr data in the inode. If the metadata is too large to fit in the inode, a new extent is created, which can cause quite a performance problem. Upping the inode size to 1024 bytes provides enough room to write the default metadata, plus a little headroom.

The following example mount options are recommended when using XFS:

mount \-t xfs \-o noatime,nodiratime,nobarrier,logbufs\=8 /dev/sda1 /srv/node/sda

We do not recommend running Swift on RAID, but if you are using RAID it is also important to make sure that the proper sunit and swidth settings get set so that XFS can make most efficient use of the RAID array.

For a standard Swift install, all data drives are mounted directly under `/srv/node` (as can be seen in the above example of mounting `/dev/sda1` as `/srv/node/sda`). If you choose to mount the drives in another directory, be sure to set the devices config option in all of the server configs to point to the correct directory.

The mount points for each drive in `/srv/node/` should be owned by the root user almost exclusively (`root:root 755`). This is required to prevent rsync from syncing files into the root drive in the event a drive is unmounted.

Swift uses system calls to reserve space for new objects being written into the system. If your filesystem does not support fallocate() or posix\_fallocate(), be sure to set the disable\_fallocate = true config parameter in account, container, and object server configs.

Most current Linux distributions ship with a default installation of updatedb. This tool runs periodically and updates the file name database that is used by the GNU locate tool. However, including Swift object and container database files is most likely not required and the periodic update affects the performance quite a bit. To disable the inclusion of these files add the path where Swift stores its data to the setting PRUNEPATHS in /etc/updatedb.conf:

PRUNEPATHS\="... /tmp ... /var/spool ... /srv/node"

General System Tuning[¶](#general-system-tuning "Permalink to this headline")
-----------------------------------------------------------------------------

Rackspace currently runs Swift on Ubuntu Server 10.04, and the following changes have been found to be useful for our use cases.

The following settings should be in /etc/sysctl.conf:

\# disable TIME\_WAIT.. wait..
net.ipv4.tcp\_tw\_recycle\=1
net.ipv4.tcp\_tw\_reuse\=1

\# disable syn cookies
net.ipv4.tcp\_syncookies \= 0

\# double amount of allowed conntrack
net.ipv4.netfilter.ip\_conntrack\_max \= 262144

To load the updated sysctl settings, run `sudo sysctl -p`

A note about changing the TIME\_WAIT values. By default the OS will hold a port open for 60 seconds to ensure that any remaining packets can be received. During high usage, and with the number of connections that are created, it is easy to run out of ports. We can change this since we are in control of the network. If you are not in control of the network, or do not expect high loads, then you may not want to adjust those values.

Logging Considerations[¶](#logging-considerations "Permalink to this headline")
-------------------------------------------------------------------------------

Swift is set up to log directly to syslog. Every service can be configured with the log\_facility option to set the syslog log facility destination. We recommended using syslog-ng to route the logs to specific log files locally on the server and also to remote log collecting servers. Additionally, custom log handlers can be used via the custom\_log\_handlers setting.

this page last updated: 2018-10-29 14:49:48
