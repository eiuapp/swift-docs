OpenStack Docs: Access Control Lists (ACLs)                   
 
https://docs.openstack.org/swift/queens/overview_acl.html
 

Access Control Lists (ACLs)
---------------------------

updated: 2019-01-15 02:30

Access Control Lists (ACLs)[¶](#access-control-lists-acls "Permalink to this headline")
=======================================================================================

通常，要创建，读取和修改容器和对象，您必须在与该帐户关联的项目中具有相应的角色，即您必须是该帐户的所有者。但是，所有者可以使用访问控制列表（ACL）授予对其他用户的访问权限。

有两种类型的ACL：

*   [Container ACLs](#container-acls). 这些是在容器上指定的，仅适用于该容器和容器中的对象。
*   [Account ACLs](#account-acls). 这些在帐户级别指定，并应用于帐户中的所有容器和对象。

Container ACLs[¶](#container-acls "Permalink to this headline")
---------------------------------------------------------------

容器ACL存储在X-Container-Write和X-Container-Read 元数据中。ACL的范围仅限于设置元数据的容器和容器中的对象。此外：

*   X-Container-Write授予对容器内对象执行PUT，POST和DELETE操作的能力。它不授予对容器本身执行POST或DELETE操作的能力。某些ACL元素还授予对容器执行HEAD或GET操作的能力。
*   X-Container-Read授予对容器内对象执行GET和HEAD操作的能力。某些ACL元素还授予对容器本身执行HEAD或GET操作的能力。但是，容器ACL不允许访问特权元数据（例如X-Container-Sync-Key）。

容器ACL使用“V1”ACL语法，该语法是逗号分隔的元素字符串，如以下示例所示：

```
.r:*,.rlistings,7ec59e87c6584c348b563254aae4c221:*
```

元素之间可能出现空格，如以下示例所示：

```
.r : *, .rlistings, 7ec59e87c6584c348b563254aae4c221:*
```

但是，这些空格将从存储在X-Container-Write和X-Container-Read元数据中的值中删除 。另外，.r:字符串可以写为.referrer:，但存储为.r:。

虽然所有auth系统使用相同的语法，但由于不同auth系统使用的不同概念，某些元素的含义不同，如以下部分所述：

*   [Common ACL Elements](#acl-common-elements)
*   [Keystone Auth ACL Elements](#acl-keystone-elements)
*   [TempAuth ACL Elements](#acl-tempauth-elements)

### Common ACL Elements[¶](#common-acl-elements "Permalink to this headline")

下表描述了Keystone auth和TempAuth都支持的ACL元素。这些元素只能与`X-Container-Read`一起使用（除了`.rlistings`, 与`X-Container-Write`使用时会发生错误）：
  

| 元件(元素)   |     描述      |   
|:----------|:-------------| 
| `.r:*` | 任何用户都可以访问对象。请求中不需要令牌。 |  
| `.r:<referrer>`    |  引用者被授予访问对象的权限。referrer 由Referer 请求中的请求标头标识。无需令牌。 | 
| `.r:-<referrer>`     | 支持此语法（在引用者前面加上“-”）。但是，如果另一个元素（例如`.r:*`）授予访问权限，它不会拒绝访问。   |     
| `.rlistings`     |   任何用户都可以对容器执行HEAD或GET操作，前提是用户对对象也具有读取权限（例如，也有`.r:*` 或者`.r:<referrer>`。不需要令牌。  |    


### Keystone Auth ACL Elements[¶](#keystone-auth-acl-elements "Permalink to this headline")


下表描述了仅由Keystone身份验证支持的ACL的元素。Keystone auth还支持Common ACL Elements中描述的元素。

请求中必须包含一个令牌才能使这些ACL元素生效。
  
| 元件(元素)   |     描述      |   
|:----------|:-------------| 
| `<project-id>:<user-id>` | 指定的用户，提供了作用于项目的令牌，包含在请求中，被授予访问权限。使用X-Container-Read时也可以访问容器。 |  
| `<project-id>:*`  |  在指定Keystone项目中具有角色的任何用户都具有访问权限。必须在请求中包含作用于项目的token。使用X-Container-Read时也可以访问容器。 | 
| `*:<user-id>`    | 指定的用户具有访问权限。必须在请求中包含用户的token（作用于任何项目）。使用X-Container-Read时也可以访问容器。 |     
| `*:*`   |   任何用户都有权限。使用X-Container-Read时也可以访问容器。所述`*:*`元件不同于`.r:*` 元件，因为 `*:*`需要在该请求包括有效的token, 而`.r:*` 并不需要的token。此外， `.r:*`不授予对容器列表的访问权限。  |    
|`<role_name>` | 具有指定角色名称的用户在其中存储容器的项目被授予访问权限。作用域必须包含在项目范围内的用户令牌。使用`X-Container-Read`时也可以访问容器 。 |     
 
注意

必须不再使用Keystone项目（租户）或用户 _names_（即`<project-name>:<user-name`）， 因为在Keystone中引入domains，_names_ 不是全局唯一的。您应该使用用户和项目 _ids_ 。

为了向后兼容，当可以确定 被授权者项目，被授权者用户和被访问的项目要么尚未在domain中（例如，X-Auth-Token已通过Keystone V2 API获得），或者要么都在默认域中，而旧帐户将迁移到该域, 时，将使用keystoneauth授予ACLs using names。

### TempAuth ACL Elements[¶](#tempauth-acl-elements "Permalink to this headline")

下表描述了仅由TempAuth支持的ACL的元素。TempAuth auth同样支持Common ACL Elements中描述的元素。
  

| 元件(元素)   |     描述      |   
|:----------|:-------------| 
| `<user-name>` | 指定的用户被授予访问权限。不支持通配符（“*”）字符。来自用户的令牌必须包含在请求中。 |

Container ACL Examples[¶](#container-acl-examples "Permalink to this headline")
-------------------------------------------------------------------------------

可以通过向容器URL发送包含X-Container-Write 和/或 X-Container-Read请求头的PUT或POST请求设置容器ACL。以下示例使用swift命令行客户端，该客户端支持通过其`--write-acl`和`--read-acl`选项设置的这些请求头。

### Example: Public Container[¶](#example-public-container "Permalink to this headline")

以下内容允许任何人列出www容器中的对象并下载对象。用户不需要在其请求中包含令牌。此ACL通常称为使容器“公开”。与[StaticWeb](https://docs.openstack.org/swift/queens/middleware.html#staticweb)一起使用时非常有用：

```shell
swift post www --read-acl ".r:*,.rlistings"
```

### Example: Shared Writable Container[¶](#example-shared-writable-container "Permalink to this headline")

以下内容允许任何人上传或下载对象。但是，要下载对象，必须知道对象的确切名称，因为用户无法列出容器中的对象。用户必须在上载请求中包含Keystone令牌。但是，它不需要作用于与容器关联的项目：

```shell
swift post www \--read\-acl ".r:\*" \--write\-acl "\*:\*"
```

### Example: Sharing a Container with Project Members[¶](#example-sharing-a-container-with-project-members "Permalink to this headline")

以下内容允许`77b8f82565f14814bece56e50c4c240f` 项目的任何成员上传和下载对象或列出www容器的内容。`77b8f82565f14814bece56e50c4c240f` 必须在请求中包含作用于项目的token：

```shell
swift post www \--read\-acl "77b8f82565f14814bece56e50c4c240f:\*" \\
               \--write\-acl "77b8f82565f14814bece56e50c4c240f:\*"
```

### Example: Sharing a Container with Users having a specified Role[¶](#example-sharing-a-container-with-users-having-a-specified-role "Permalink to this headline")

以下内容允许在`www`容器的项目中分配了`my_read_access_role`的任何用户 下载对象或列出`www`容器的内容。作用于项目的用户令牌必须包含在下载或列表请求中：
```shell
swift post www \--read\-acl "my\_read\_access\_role"
```
### Example: Allowing a Referrer Domain to Download Objects[¶](#example-allowing-a-referrer-domain-to-download-objects "Permalink to this headline")


以下内容允许来自`example.com`域的任何请求访问容器中的对象：

```shell
swift post www \--read\-acl ".r:.example.com"
```

However, the request from the user **must** contain the appropriate Referer header as shown in this example request:
但是，来自用户的请求**必须**包含相应的 Referer 标头，如下示例所示：

```shell
curl -i $publicURL/www/document --head -H "Referer: http://www.example.com/index.html"
```

注意

该Referer的头被包含在许多浏览器的请求。但是，由于在Referer头中很容易创建具有任何所需值的请求 ，因此referrer ACL的安全性非常低。


Account ACLs[¶](#account-acls "Permalink to this headline")
-----------------------------------------------------------

注意

Keystone auth目前不支持帐户ACL

X-Account-Access-Control报头用于指定帐户级的ACL，通过身份验证系统的特定格式。这些标题只有帐户所有者可见且可设置（`swift_owner`为true）。帐户ACL的行为取决于身份验证系统。对于TempAuth，如果经过身份验证的用户具有ACL中列出的组的成员身份，则允许该用户访问该ACL的访问级别。

帐户ACL使用“V2”ACL语法，该语法是一个JSON字典，其密钥名为“admin”，“read-write”和“read-only”。（注意区分大小写）.一个`X-Account-Access-Control`报头值的例子是这样的，其中`a`，`b`和`c`是用户名：

```
{"admin":["a","b"],"read-only":["c"]}
```
Keys 可能不存在（如上例所示）.

生成ACL字符串的推荐方法如下：

```
from swift.common.middleware.acl import format_acl
acl_data = { 'admin': ['alice'], 'read-write': ['bob', 'carol'] }
acl_string = format_acl(version=2, acl_dict=acl_data)
```

使用该`format_acl()`方法将确保JSON编码为ASCII（例如，对于Unicode,使用'u1234'）。虽然允许人工发送包含X-Account-Access-Control标头的curl命令 ，但由于可能存在人为错误，因此在执行此操作时应谨慎。

Within the JSON dictionary stored in `X-Account-Access-Control`, the keys have the following meanings:
在JSON字典中存储`X-Account-Access-Control`，Keys 具有以下含义：
  

| 访问权限   |     描述      |   
|:----------|:-------------| 
| read-only | 这些身份可以读取帐户中的 _everything_ （privileged headers除外）。具体来说，具有只读帐户访问权限的用户可以获取帐户中的容器列表，列出任何容器的内容，检索任何对象，以及查看任何容器或任何对象或任何帐户的（non-privileged）标头。 |
| read-write | 这些身份可以读取或写入（或创建）任何容器。具有读写帐户访问权限的用户，可以创建新容器，设置任何非特权容器标头，覆盖对象，删除容器等。读写用户不能设置帐户标头（或对帐户执行任何PUT/POST/DELETE请求）。 |
| admin | 这些身份具有“swift_owner”权限。具有帐户管理员访问权限的用户可以执行帐户所有者可以执行的任何操作，包括设置帐户标头和任何特权标头 - 从而授予对其他用户的只读，读写或管理员访问权限。 |

有关详细信息，请参阅[`swift.common.middleware.tempauth`](https://docs.openstack.org/swift/queens/middleware.html#module-swift.common.middleware.tempauth)。

有关ACL格式的详细信息，请参阅[`swift.common.middleware.acl`](https://docs.openstack.org/swift/queens/misc.html#module-swift.common.middleware.acl)。

updated: 2019-01-15 02:30
