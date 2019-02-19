Object Storage API overview
---------------------------

updated: 2019-02-14 02:34

Object Storage API overview[¶](#object-storage-api-overview "Permalink to this headline")
=========================================================================================

OpenStack Object Storage is a highly available, distributed, eventually consistent object/blob store. You create, modify, and get objects and metadata by using the Object Storage API, which is implemented as a set of Representational State Transfer (REST) web services.

OpenStack Object Storage是一个高可用，分布式，最终一致性的对象/blob存储。您可以使用Object Storage API创建，修改和获取对象和元数据，该API是作为一组Representational State Transfer（REST）Web服务实现的。

For an introduction to OpenStack Object Storage, see the [OpenStack Swift Administrator Guide](../admin/index.html).

有关OpenStack对象存储的介绍，请参阅“[OpenStack Swift管理员指南](../admin/index.html)”。

You use the HTTPS (SSL) protocol to interact with Object Storage, and you use standard HTTP calls to perform API operations. You can also use language-specific APIs, which use the RESTful API, that make it easier for you to integrate into your applications.

您使用HTTPS（SSL）协议与对象存储进行交互，并使用标准HTTP调用来执行API操作。您还可以使用特定语言的API，这些API使用RESTful API，使您可以更轻松地集成到应用程序中。

To assert your right to access and change data in an account, you identify yourself to Object Storage by using an authentication token. To get a token, you present your credentials to an authentication service. The authentication service returns a token and the URL for the account. Depending on which authentication service that you use, the URL for the account appears in:

*   **OpenStack Identity Service**. The URL is defined in the service catalog.
*   **Tempauth**. The URL is provided in the `X-Storage-Url` response header.

要维护您有权去访问和更改帐户中的数据，您可以使用身份验证令牌(authentication token)向对象存储标识自己。要获取令牌，请将凭据提供给身份验证服务。身份验证服务返回令牌和帐户的URL。根据您使用的身份验证服务，该帐户的URL显示在：

*   **OpenStack身份服务**。URL在服务目录中定义。
*   **Tempauth**。URL在X-Storage-Url响应头中提供。

In both cases, the URL is the full URL and includes the account resource.

在这两种情况下，URL都是完整的URL并包含帐户资源。

The Object Storage API supports the standard, non-serialized response format, which is the default, and both JSON and XML serialized response formats.

Object Storage API支持标准的非序列化响应格式，这是默认格式，以及JSON和XML序列化响应格式。

The Object Storage system organizes data in a hierarchy, as follows:

对象存储系统按层次结构组织数据，如下所示：

*   **Account**. Represents the top-level of the hierarchy.
    
    Your service provider creates your account and you own all resources in that account. The account defines a namespace for containers. A container might have the same name in two different accounts.
    
    In the OpenStack environment, _account_ is synonymous with a project or tenant.

*   **帐户**。表示层次结构的顶级。

    您的服务提供商会创建您的帐户，并拥有该帐户中的所有资源。该帐户定义容器(container)的命名空间。容器在两个不同的帐户中可能具有相同的名称。

    在OpenStack环境中，_帐户_与_项目_或_租户_同义。

*   **Container**. Defines a namespace for objects. An object with the same name in two different containers represents two different objects. You can create any number of containers within an account.
    
    In addition to containing objects, you can also use the container to control access to objects by using an access control list (ACL). You cannot store an ACL with individual objects.
    
    In addition, you configure and control many other features, such as object versioning, at the container level.
    
    You can bulk-delete up to 10,000 containers in a single request.
    
    You can set a storage policy on a container with predefined names and definitions from your cloud provider.

*   **容器**。定义对象(object)的命名空间。两个不同容器中具有相同名称的对象表示两个不同的对象。您可以在帐户中创建任意数量的容器。

    除了包含对象之外，您还可以使用容器通过使用访问控制列表（ACL）来控制对对象的访问。您不能使用单个对象存储ACL。

    此外，您还可以在容器级别配置和控制许多其他功能，例如对象版本控制。

    您可以在一个请求中批量删除多达10,000个容器。

    您可以在具有云提供商的预定义名称和定义的容器上设置存储策略。
    
*   **Object**. Stores data content, such as documents, images, and so on. You can also store custom metadata with an object.
    
    With the Object Storage API, you can:
    
    *   Store an unlimited number of objects. Each object can be as large as 5 GB, which is the default. You can configure the maximum object size.
    *   Upload and store objects of any size with large object creation.
    *   Use cross-origin resource sharing to manage object security.
    *   Compress files using content-encoding metadata.
    *   Override browser behavior for an object using content-disposition metadata.
    *   Schedule objects for deletion.
    *   Bulk-delete up to 10,000 objects in a single request.
    *   Auto-extract archive files.
    *   Generate a URL that provides time-limited **GET** access to an object.
    *   Upload objects directly to the Object Storage system from a browser by using form **POST** middleware.
    *   Create symbolic links to other objects.

*   **对象**。存储数据内容，例如文档，图像等。您还可以使用对象存储自定义元数据。

    使用Object Storage API，您可以：

    *   Store an unlimited number of objects. Each object can be as large as 5 GB, which is the default. You can configure the maximum object size.
    *   Upload and store objects of any size with large object creation.
    *   Use cross-origin resource sharing to manage object security.
    *   Compress files using content-encoding metadata.
    *   Override browser behavior for an object using content-disposition metadata.
    *   Schedule objects for deletion.
    *   Bulk-delete up to 10,000 objects in a single request.
    *   Auto-extract archive files.
    *   Generate a URL that provides time-limited **GET** access to an object.
    *   Upload objects directly to the Object Storage system from a browser by using form **POST** middleware.
    *   Create symbolic links to other objects.

The account, container, and object hierarchy affects the way you interact with the Object Storage API.

帐户，容器和对象层次结构会影响您与Object Storage API交互的方式。

Specifically, the resource path reflects this structure and has this format:

```
/v1/{account}/{container}/{object}
```

具体来说，资源路径反映了这种结构，并具有以下格式：

```
/v1/{account}/{container}/{object}
```

For example, for the `flowers/rose.jpg` object in the `images` container in the `12345678912345` account, the resource path is:

```
/v1/12345678912345/images/flowers/rose.jpg
```

例如，对于帐户`12345678912345`中 容器 `images`中的`flowers/rose.jpg`对象，资源路径为：

```
/v1/12345678912345/images/flowers/rose.jpg
```

Notice that the object name contains the `/` character. This slash does not indicate that Object Storage has a sub-hierarchy called `flowers` because containers do not store objects in actual sub-folders. However, the inclusion of `/` or a similar convention inside object names enables you to create pseudo-hierarchical folders and directories.

请注意，对象名称包含该`/`字符。此斜杠不表示对象存储具有调用的子层次结构， `flowers`因为容器不会将对象存储在实际的子文件夹中。但是，`/`在对象名称中包含或类似的约定使您可以创建伪分层文件夹和目录。

For example, if the endpoint for Object Storage is `objects.mycloud.com`, the returned URL is `https://objects.mycloud.com/v1/12345678912345`.

例如，如果对象存储的端点(endpoint)是 `objects.mycloud.com`，则返回的URL是 `https://objects.mycloud.com/v1/12345678912345`。

To access a container, append the container name to the resource path.

要访问容器，请将容器名称附加到资源路径。

To access an object, append the container and the object name to the path.

要访问对象，请将容器和对象名称附加到路径。

If you have a large number of containers or objects, you can use query parameters to page through large lists of containers or objects. Use the `marker`, `limit`, and `end_marker` query parameters to control how many items are returned in a list and where the list starts or ends. If you want to page through in reverse order, you can use the query parameter `reverse`, noting that your marker and end_markers should be switched when applied to a reverse listing. I.e, for a list of objects `[a, b, c, d, e]` the non-reversed could be:

```
/v1/{account}/{container}/?marker=a&end_marker=d
b
c
```

如果您有大量容器或对象，则可以使用查询参数来浏览大型容器或对象列表。使用 `marker`，`limit`和`end_marker`查询参数来控制多少列表中的项目，并在列表开始或结束时返回。如果要以相反的顺序翻页，可以使用查询参数`reverse`，注意应用于反向列表时应切换标记和`end_markers`,即，对于非反转的对象列表 可以是：`[a, b, c, d, e]`。

```
/v1/{account}/{container}/?marker=a&end_marker=d
b
c
```

However, when reversed marker and end_marker are applied to a reversed list:

```
/v1/{account}/{container}/?marker=d&end\_marker=a&reverse=on
c
b
```

但是，当反向标记和end_marker应用于反转列表时：

```
/v1/{account}/{container}/?marker=d&end\_marker=a&reverse=on
c
b
```

Object Storage HTTP requests have the following default constraints. Your service provider might use different default values.

|Item	|Maximum value|	Notes|
|----------|-----------------------------|-------------------|
|Number of HTTP headers|	90	 ||
|Length of HTTP headers|	4096 bytes	 ||
|Length per HTTP request line|	8192 bytes	 ||
|Length of HTTP request|	5 GB	|| 
|Length of container names|	256 bytes|	Cannot contain the / character.|
|Length of object names	|1024 bytes	|By default, there are no character restrictions.|

对象存储HTTP请求具有以下默认约束。您的服务提供商可能使用不同的默认值。

|项目	|最大价值	|笔记|
|----------|-----------------------------|-------------------|
|HTTP标头数量|	90	 |
|HTTP标头的长度|	4096字节|	| 
|每个HTTP请求行的长度|	8192个字节|	| 
|HTTP请求的长度|	5 GB|	| 
|容器名称的长度|	256个字节|	不能包含该/字符。|
|对象名称的长度|	1024字节|	默认情况下，没有字符限制。|  


You must UTF-8-encode and then URL-encode container and object names before you call the API binding. If you use an API binding that performs the URL-encoding for you, do not URL-encode the names before you call the API binding. Otherwise, you double-encode these names. Check the length restrictions against the URL-encoded string.

在调用API绑定之前，必须使用UTF-8编码然后对容器和对象名称进行URL编码。如果您使用为您执行URL编码的API绑定，请不要在调用API绑定之前对名称进行URL编码。否则，您将对这些名称进行双重编码。检查URL编码字符串的长度限制。

The API Reference describes the operations that you can perform with the Object Storage API:

*   [Storage accounts](https://developer.openstack.org/api-ref/object-store/index.html#accounts): Use to perform account-level tasks.
    
    Lists containers for a specified account. Creates, updates, and deletes account metadata. Shows account metadata.
    
*   [Storage containers](https://developer.openstack.org/api-ref/object-store/index.html#containers): Use to perform container-level tasks.
    
    Lists objects in a specified container. Creates, shows details for, and deletes containers. Creates, updates, shows, and deletes container metadata.
    
*   [Storage objects](https://developer.openstack.org/api-ref/object-store/index.html#objects): Use to perform object-level tasks.
    
    Creates, replaces, shows details for, and deletes objects. Copies objects with another object with a new or different name. Updates object metadata.
    

API Reference描述了可以使用Object Storage API执行的操作：

*   [存储帐户](https://developer.openstack.org/api-ref/object-store/index.html#accounts)：用于执行帐户级任务。

    列出指定帐户的容器。创建，更新和删除帐户元数据。显示帐户元数据。

*   [存储容器](https://developer.openstack.org/api-ref/object-store/index.html#containers)：用于执行容器级任务。

    列出指定容器中的对象。创建，显示容器的详细信息和删除容器。创建，更新，显示和删除容器元数据。

*   [存储对象](https://developer.openstack.org/api-ref/object-store/index.html#objects)：用于执行对象级任务。

    创建，替换，显示和删除对象的详细信息。使用具有新名称或不同名称的另一个对象复制对象。更新对象元数据。

updated: 2019-02-14 02:34
