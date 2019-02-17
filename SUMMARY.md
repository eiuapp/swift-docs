# Summary

## 序言

- [前言](README.md)

## 环境配置

- [swift 2.17 环境配置](docs/env.md)

## index

*   [Index](docs/index.md)
*   [Getting Started](getting_started.html)

Overview and Concepts[¶](#overview-and-concepts "Permalink to this headline")
-----------------------------------------------------------------------------

*   [Object Storage API overview](api/object_api_v1_overview.html)
*   [Swift Architectural Overview](overview_architecture.html)
*   [The Rings](docs/overview_rings.md)
*   [Storage Policies](docs/overview_policies.md)
*   [The Account Reaper](docs/overview_reaper.md)
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

Developer Documentation[¶](#developer-documentation "Permalink to this headline")
---------------------------------------------------------------------------------

*   [Development Guidelines](development_guidelines.html)
*   [SAIO - Swift All In One](development_saio.html)
*   [First Contribution to Swift](first_contribution_swift.html)
*   [Adding Storage Policies to an Existing SAIO](policies_saio.html)
*   [Auth Server and Middleware](development_auth.html)
*   [Middleware and Metadata](development_middleware.html)
*   [Pluggable On-Disk Back-end APIs](development_ondisk_backends.html)

Administrator Documentation[¶](#administrator-documentation "Permalink to this headline")
-----------------------------------------------------------------------------------------

*   [Instructions for a Multiple Server Swift Installation](howto_installmultinode.html)
*   [Deployment Guide](deployment_guide.html)
*   [Apache Deployment Guide](apache_deployment_guide.html)
*   [Administrator’s Guide](admin_guide.html)
*   [Dedicated replication network](replication_network.html)
*   [Logs](logs.html)
*   [Swift Ops Runbook](ops_runbook/index.html)
*   [OpenStack Swift Administrator Guide](admin/index.html)
*   [Object Storage Install Guide](install/index.html)

Object Storage v1 REST API Documentation[¶](#object-storage-v1-rest-api-documentation "Permalink to this headline")
-------------------------------------------------------------------------------------------------------------------

See [Complete Reference for the Object Storage REST API](http://developer.openstack.org/api-ref/object-storage/)

The following provides supporting information for the REST API:

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

OpenStack End User Guide[¶](#openstack-end-user-guide "Permalink to this headline")
-----------------------------------------------------------------------------------

The [OpenStack End User Guide](http://docs.openstack.org/user-guide) has additional information on using Swift. See the [Manage objects and containers](http://docs.openstack.org/user-guide/managing-openstack-object-storage-with-swift-cli.html) section.

Source Documentation[¶](#source-documentation "Permalink to this headline")
---------------------------------------------------------------------------

*   [Partitioned Consistent Hash Ring](ring.html)
    *   [Ring](ring.html#ring)
    *   [Ring Builder](ring.html#module-swift.common.ring.builder)
    *   [Composite Ring Builder](ring.html#module-swift.common.ring.composite_builder)
*   [Proxy](proxy.html)
    *   [Proxy Controllers](proxy.html#proxy-controllers)
    *   [Proxy Server](proxy.html#module-swift.proxy.server)
*   [Account](account.html)
    *   [Account Auditor](account.html#module-swift.account.auditor)
    *   [Account Backend](account.html#module-swift.account.backend)
    *   [Account Reaper](account.html#module-swift.account.reaper)
    *   [Account Server](account.html#module-swift.account.server)
*   [Container](container.html)
    *   [Container Auditor](container.html#module-swift.container.auditor)
    *   [Container Backend](container.html#module-swift.container.backend)
    *   [Container Server](container.html#module-swift.container.server)
    *   [Container Reconciler](container.html#module-swift.container.reconciler)
    *   [Container Replicator](container.html#module-swift.container.replicator)
    *   [Container Sync](container.html#module-swift.container.sync)
    *   [Container Updater](container.html#module-swift.container.updater)
*   [Account DB and Container DB](db.html)
    *   [DB](db.html#db)
    *   [DB replicator](db.html#module-swift.common.db_replicator)
*   [Object](object.html)
    *   [Object Auditor](object.html#module-swift.obj.auditor)
    *   [Object Backend](object.html#module-swift.obj.diskfile)
    *   [Object Replicator](object.html#module-swift.obj.replicator)
    *   [Object Reconstructor](object.html#module-swift.obj.reconstructor)
    *   [Object Server](object.html#module-swift.obj.server)
    *   [Object Updater](object.html#module-swift.obj.updater)
*   [Misc](misc.html)
    *   [ACLs](misc.html#acls)
    *   [Buffered HTTP](misc.html#module-swift.common.bufferedhttp)
    *   [Constraints](misc.html#constraints)
    *   [Container Sync Realms](misc.html#module-swift.common.container_sync_realms)
    *   [Direct Client](misc.html#module-swift.common.direct_client)
    *   [Exceptions](misc.html#exceptions)
    *   [Internal Client](misc.html#module-swift.common.internal_client)
    *   [Manager](misc.html#module-swift.common.manager)
    *   [MemCacheD](misc.html#module-swift.common.memcached)
    *   [Request Helpers](misc.html#module-swift.common.request_helpers)
    *   [Swob](misc.html#swob)
    *   [Utils](misc.html#utils)
    *   [WSGI](misc.html#wsgi)
    *   [Storage Policy](misc.html#module-swift.common.storage_policy)
*   [Middleware](middleware.html)
    *   [Account Quotas](middleware.html#module-swift.common.middleware.account_quotas)
    *   [Bulk Operations (Delete and Archive Auto Extraction)](middleware.html#module-swift.common.middleware.bulk)
    *   [CatchErrors](middleware.html#module-swift.common.middleware.catch_errors)
    *   [CNAME Lookup](middleware.html#module-swift.common.middleware.cname_lookup)
    *   [Container Quotas](middleware.html#module-swift.common.middleware.container_quotas)
    *   [Container Sync Middleware](middleware.html#module-swift.common.middleware.container_sync)
    *   [Cross Domain Policies](middleware.html#module-swift.common.middleware.crossdomain)
    *   [Discoverability](middleware.html#discoverability)
    *   [Domain Remap](middleware.html#module-swift.common.middleware.domain_remap)
    *   [Dynamic Large Objects](middleware.html#dynamic-large-objects)
    *   [Encryption](middleware.html#encryption)
    *   [FormPost](middleware.html#formpost)
    *   [GateKeeper](middleware.html#gatekeeper)
    *   [Healthcheck](middleware.html#healthcheck)
    *   [Keymaster](middleware.html#keymaster)
    *   [KeystoneAuth](middleware.html#keystoneauth)
    *   [List Endpoints](middleware.html#module-swift.common.middleware.list_endpoints)
    *   [Memcache](middleware.html#module-swift.common.middleware.memcache)
    *   [Name Check (Forbidden Character Filter)](middleware.html#module-swift.common.middleware.name_check)
    *   [Object Versioning](middleware.html#module-swift.common.middleware.versioned_writes)
    *   [Proxy Logging](middleware.html#module-swift.common.middleware.proxy_logging)
    *   [Ratelimit](middleware.html#module-swift.common.middleware.ratelimit)
    *   [Recon](middleware.html#recon)
    *   [Server Side Copy](middleware.html#module-swift.common.middleware.copy)
    *   [Static Large Objects](middleware.html#static-large-objects)
    *   [StaticWeb](middleware.html#staticweb)
    *   [Symlink](middleware.html#symlink)
    *   [TempAuth](middleware.html#module-swift.common.middleware.tempauth)
    *   [TempURL](middleware.html#tempurl)
    *   [XProfile](middleware.html#module-swift.common.middleware.xprofile)

Indices and tables[¶](#indices-and-tables "Permalink to this headline")
-----------------------------------------------------------------------

*   [Index](genindex.html)
*   [Module Index](py-modindex.html)
*   [Search Page](search.html)


## 未来

- [我的swift探险之旅](https://b.qqbb.app/tags/swift/)
- [Swift Handbook](https://eiuapp/swift-handbook/)

## 相关资源

- [swift技术工具与资源](docs/tech_resource.md)

