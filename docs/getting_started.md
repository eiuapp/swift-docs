
Getting Started
---------------

updated: 2019-02-14 02:34

Getting Started[¶](#getting-started "Permalink to this headline")
=================================================================

System Requirements[¶](#system-requirements "Permalink to this headline")
-------------------------------------------------------------------------

Swift development currently targets Ubuntu Server 16.04, but should work on most Linux platforms.

Swift开发目前针对Ubuntu Server 16.04，但应该适用于大多数Linux平台。

Swift is written in Python and has these dependencies:

*   Python 2.7
*   rsync 3.0
*   The Python packages listed in [the requirements file](https://github.com/openstack/swift/blob/master/requirements.txt)
*   Testing additionally requires [the test dependencies](https://github.com/openstack/swift/blob/master/test-requirements.txt)
*   Testing requires [these distribution packages](https://github.com/openstack/swift/blob/master/bindep.txt)

Swift是用Python编写的，并具有以下依赖项：

*   Python 2.7
*   rsync 3.0
*   The Python packages listed in [the requirements file](https://github.com/openstack/swift/blob/master/requirements.txt)
*   Testing additionally requires [the test dependencies](https://github.com/openstack/swift/blob/master/test-requirements.txt)
*   Testing requires [these distribution packages](https://github.com/openstack/swift/blob/master/bindep.txt)

There is no current support for Python 3.

目前没有Python 3的支持。

Development[¶](#development "Permalink to this headline")
---------------------------------------------------------

To get started with development with Swift, or to just play around, the following docs will be useful:

*   [Swift All in One](development_saio.html) - Set up a VM with Swift installed
*   [Development Guidelines](development_guidelines.html)
*   [First Contribution to Swift](first_contribution_swift.html)
*   [Associated Projects](associated_projects.html)

要开始使用Swift进行开发，或者只是玩玩，以下文档将非常有用：

* Swift All in One - 设置安装了Swift的VM
* 开发指南
* 对Swift的第一次贡献
* 相关项目

CLI client and SDK library[¶](#cli-client-and-sdk-library "Permalink to this headline")
---------------------------------------------------------------------------------------

There are many clients in the [ecosystem](associated_projects.html#application-bindings). The official CLI and SDK is python-swiftclient.

*   [Source code](https://github.com/openstack/python-swiftclient)
*   [Python Package Index](https://pypi.python.org/pypi/python-swiftclient)

[生态系统](associated_projects.html#application-bindings)中有许多客户端。官方CLI和SDK是python-swiftclient。

*   [Source code](https://github.com/openstack/python-swiftclient)
*   [Python Package Index](https://pypi.python.org/pypi/python-swiftclient)

Production[¶](#production "Permalink to this headline")
-------------------------------------------------------

If you want to set up and configure Swift for a production cluster, the following doc should be useful:

如果要为生产群集设置和配置Swift，则以下文档应该很有用：

*   [Multiple Server Swift Installation](howto_installmultinode.html)

* 多服务器Swift安装

updated: 2019-02-14 02:34
