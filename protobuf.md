# 背景

最近在公司负责客户端组的事务，包括iOS、Android与前端（其中前端项目与iOS和Android项目关联不大），在项目开发过程中发现，定义好的接口三端（包括服务器）各自都要实现一次，这中间有着重复的人力成本。如果减少这个成本，就需要一个跨语言的协议制定工具，于是便想到了在前公司游戏项目中用到的protobuf。

下面就来简单介绍一下protobuf的应用，以前端为例。

# 什么是protobuf

protobuf是一种灵活高效的独立于语言平台的结构化数据表示方法，与XML相比，protobuf更小更快更简单。你可以自己定义protobuf的数据结构，用编译器生成特定语言的源代码，比如C++，Java，JavaScript等，目前protobuf对主流编程语言都提供了支持，非常方便的进行序列化（格式化）和反序列化（解析）。

特点：
- 平台无关，语言无关
- 二进制，数据自描述
- 提供了完整详细的操作API
- 高性能，比XML要快20-30倍
- 尺寸小，比XML要小3-10倍
- 高可扩展性，前后兼容

# 如何配置protobuf

### 1 安装（Mac）

##### 1.1 安装brew

具体步骤略。

##### 1.2 安装protobuf

使用brew安装protobug，在终端中执行命令

```
brew install protobuf
```
安装完成之后执行命令

```
proton --version
```

可以查看安装的protobuf版本，目前官方最新的版本是3.6.1。

### 2 编写proto文件

在项目的根目录下创建一个proto的文件夹，在里面创建test.proto文件，proto文件内容如下

```protobuf
syntax = "proto3";

message registerC2S {
    string username = 1;
    string password = 2;
}

message loginC2S {
    string username = 11;
    string password = 12;
}
```

具体的语法可以参考[官方介绍](https://developers.google.com/protocol-buffers/docs/proto3)，其中syntax代表当前proto文件的版本，这里使用的是proto3，如果不明文声明syntax的话，默认使用proto2。

### 3 编译成js

##### 3.1 将proto文件编译为js

执行命令

```
protoc --js_out=import_style=commonjs,binary:. proto/test.proto
```

会在proto文件的目录下生成test_bp.js文件。如果是Node.js，这里就可以直接使用了，想要在前端应用，还需要做进一步的处理。

##### 3.2 安装browserify库

使用npm安装库

```json
npm install -g require
npm install -g browserify
npm install google-protobuf
```

##### 3.2 使用browserify生成js文件

在proto目录下面编写一个名为exports的js文件

```javascript
var testProto = require('./test_pb');

module.exports = {
    DataProto: testProto
};
```

然后就可以使用browserify进行打包了，执行命令

```json
browserify ./proto/exports.js > ./proto/bundle.js
```

会在proto目录下生成bundle.js，这也就是最终使用的js文件。

配置protobuf的步骤已经完成，了解之后就可以按照项目的接口需求进行proto文件的编写了。
