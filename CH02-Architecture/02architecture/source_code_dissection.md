mongodb的源码树肢解
==================
### 1 源码下载
mongodb的源码目前整个托管在github上，你可以直接从上面[下载](https://github.com/mongodb/mongo)，可以直接点击下载zip包，也可以通过git工具下载。因为本系列分析文章基于mongodb v3.4版本进行，所以在打开github的项目页面后，你需要选择v3.4的branch之后再下载。

	git clone https://github.com/mongodb/mongo.git

下载下来之后，我们再接着看具体的目录组织结构。

### 2 源码目录介绍
#### 2.1 源码根目录
mongodb的根目录下面主要有以下文件夹：

##### [buildscripts]
该目录下主要是mongodb编译打包相关的代码实现，看着其中还有smoke测试的相关代码. 目前mongodb整体采用scons工具体系进行编译，关于scons可点击[这里](http://www.scons.org/)了解。

其中一个值得一提的文件是errorcodes.py文件，通过它可以编译生成mongodb源码中使用到的错误码。

##### [debian]
看名字像是debian系统相关的文件，这里暂不分析。

##### [docs]
该目录目前主要包括mongodb的一些介绍性文档。

##### [etc]
该目录主要相关配置文件。

##### [jstests]
该目录下主要包括基于javascipt编写的mongodb功能测试的源码。

##### [rpm]
主要包括生成rpm包的一些spec文件

##### [site_scons]
主要是scons的辅助工具

##### [src]
该目录下包括mongodb的核心实现源码，我们的源码分析也主要针对该目录进行。

#### 2.2 src源码
该目录下进一步细分为以下两个文件夹：
##### [mongo]
此目录下的内容为mongodb核心功能的源码实现，它的组织方式和mongodb实现的功能有密切的关系，可以说基本是按功能划分的。在这里，我参考数据库的功能划分，整理出mongodb的功能模块架构图，如下所示：

![模块架构图](https://github.com/JevonQ/deepdiveintomongodb/blob/master/pics/02archtecture/mongodb_modular_architecture.png)

基于以上模块架构图，可以很容易理解mongo目录下的代码组织。

##### [公共组件]
* [base]
* [bson]
* [util]
* [logger]
* [platform]
* [rpc]
* [stdx]
* [installer]
* [executor]

##### [管理工具]
* [gotools]
* [shell]
* [tools]
* [scripting]

##### [测试系统]
* [unittest]
* [dbtest]

##### [shardServer/configServer]
* [db]

##### [mongos]
* [s]

其中shardServer和configServer共用代码，再加上Mongos基本构成了mongodb的核心组件。可以再将功能详细细分为:

* I/O 子系统
* catalog 管理
* 副本集
* sharding机制

我们后面的源码分析也主要围绕这些功能展开。

##### [third_party]
此目录下主要包含了mongodb所使用的第三方的开源代码，主要包括asio、boost、gperftools、scons、valgrind、yaml-cpp等，这里就不过多赘述了。


