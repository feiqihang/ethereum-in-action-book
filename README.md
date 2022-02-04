# 以太坊剖析 


## **以太坊客户端开源项目**
以太坊客户端程序是以太坊协议的具体实现，以太坊协议包含：白皮书、黄皮书、以太坊改进协议（EIP）。以太坊客户端有多种语言的实现版本，托管在Github网站的开源项目列表如下：

|编程语言|项目名称|描述|
| - | - | - |
|Java|<p>ethereumj</p><p>Ethereum-Harmony</p>|<p>Java版以太坊客户端包含EthereumJ(核心包)和Ethereum-Harmony (JSONRPC和图形界面)。</p><p><https://github.com/ethereum/ethereumj></p><p><https://github.com/ether-camp/ethereum-harmony></p>|
|Go|go-ethereum(Geth)|<p>官方的以太坊协议的Go语言实现，提供命令行界面。</p><p><https://github.com/ethereum/go-ethereum></p>|
|Rust|Parity|<p>快速，轻便，强大的以太坊实现。</p><p><https://github.com/paritytech/parity></p>|
|C++|<p>cpp-ethereum</p><p></p>|<p>以太坊协议的C++语言实现。</p><p><https://github.com/ethereum/cpp-ethereum></p>|
|<p>Python</p><p></p>|<p>pyethapp</p><p></p>|<p>Pyethapp是以python为基础的客户端，实现以太坊加密经济状态机。Pyethapp依赖于项目：pyethereum(核心包)和pydevp2p(点对点网络)。</p><p><https://github.com/ethereum/pyethapp></p>|
|JavaScript|<p>ethereumjs-vm</p><p>ethereumjs-blockchain</p><p>ethereumjs-devp2p</p><p>等</p>|<p>EthereumJS社区构建的JavaScript库，实现以太坊核心、协议和APIs，帮助开发人员与以太坊网络进行交互，以及开发应用。</p><p><https://ethereumjs.github.io/></p>|

我们主要介绍以太坊客户端Java版本，包含EthereumJ和Ethereum-Harmony两个子项目。


### **EthereumJ项目**
EthereumJ (全称ethereumj-core)是以太坊协议的纯Java语言实现，包含核心数据结构、加解密、共识算法、P2P通信、以太坊虚拟机(EVM)、存储等功能模块。EthereumJ项目源代码托管地址https://github.com/ethereum/ethereum， 主要功能模块的源代码在ethereumj-core/src/main/java/org/ethereum目录中
主要功能模块的源代码目录列表：

|目录|描述|
| :- | :- |
|src/main/java/org/ethereum的子目录：|
|cli|CLI(command-line interface)的实现。|
|config|程序配置项|
|core|以太坊核心数据结构包。|
|crypto|加密算法实现包。|
|datasource|数据源读写封装|
|db|数据源管理|
|facade|以太坊、区块链、状态数据存储等功能的主要接口|
|json|Json处理工具类|
|listener|事件监听器|
|manager|所有功能的统一管理|
|mine|POW算法实现|
|net|P2P网络协议实现。|
|samples|使用ethereumj的示例。|
|solidity|Solidity编译器|
|sync|网络数据同步|
|trie|梅克尔树实现。|
|util|工具包|
|validator|区块合法性的校验|
|vm|以太坊虚拟机实现|

**注：目前主要基于ethereumj 1.8.2版本分析，代码统计情况详见 [ethereumj项目源代码统计](https://github.com/feiqihang/EthereumInAction/blob/main/ethereumj.md)**

### **Ethereum-Harmony项目**
Ethereum-Harmony基于ethereumj-core提供了JSON-RPC模块和区块链浏览器模块。其中，JSON-RPC模块提供基于HTTP通信提供符合JSON-RPC 2.0标准的远程调用服务，可以与以太坊客户端进行交互；区块链浏览器模块提供节点基本信息、区块信息、对等节点监控、系统日志、终端命令行、钱包（含账户私钥管理）等功能。

Ethereum-Harmony项目源代码的托管地址https://github.com/ether-camp/ethereum-harmony ，主要功能模块的源代码在ethereum-harmony/src/main/java/com/ethercamp/harmony目录中。

主要功能模块的源代码目录列表：

|目录|描述|
| :- | :- |
|根目录下的Gradle脚本文件：|
|build.gradle|gradle构建脚本|
|gradlew|gradle包装器(wrapper)脚本|
|src/main/java/com/ethercamp/harmony的子目录：|
|config|配置模块，包含JSONRPC配置、以太坊核心配置、WebSocket配置|
|desktop|桌面程序（区块链浏览器）模块|
|jsonrpc|远程调用（JSONRPC）模块，包含JSONRPC接口定义和实现，以及交易和交易收据的查询结果数据结构定义|
|keystore|账户私钥管理模块，按照以太坊密钥存储格式实现账户私钥的生成、加密、存储、加载等功能。|
|model|区块链浏览器数据结构模块，定义区块链浏览器中用到的数据结构，包含操作状态、链信息、区块信息、合约信息、初始化信息、物理环境信息、JSONRPC接口信息、挖矿信息、对等节点信息、账户信息等。|
|service|区块链浏览器逻辑层，包含合约服务、钱包服务、区块链信息查询服务、监控服务、节点服务等。|
|util|工具类模块，包含异常管理、错误编码字典、TrustSSL等工具。|
|web|区块链浏览器控制层，处理和响应前端页面发送的http请求。拦截JSON-RPC请求并更新使用情况统计信息。|
|src/main/resources/static的子目录：|
|images|区块链浏览器展示层的图片资源|
|js|区块链浏览器展示层的JavaScript脚本，包含浏览器页面逻辑和依赖的第三方类库|
|pages|区块链浏览器展示层的HTML页面文件|
|styles|区块链浏览器展示层的CSS样式文件|


## 基本概念
我们先简单了解一下以太坊的几个基本概念，后续会深入代码详细讲解这些内容。

- 世界状态(World State)

以太坊在整体上可以看做一个基于交易(transaction-based)的状态机：从创世状态开始，随着逐步执行交易，而将其转变为某个最终状态。其中，状态是指以太坊的世界状态，即账户地址和账户状态的映射，在黄皮书中的符号为σ（Sigma，西格玛）。世界状态没有直接存储在区块链上，而是维护在一个MPT树上，然后存储在状态数据库。

- MPT（梅克尔帕特里夏树）

MPT是Merkle Patricia Tree(Trie)的简称，提供加密验证的数据结构，用来映射256位二进制摘要和任意长度的二进制数据的关系，通常实现为数据库以存储映射关系。

- RLP（递归长度前缀）编码

RLP是Recursive Length Prefix的简称，这是一种用于编码任意结构化二进制数据（字节数组）的序列化方法。

- 价值单位

为了激励网络中的计算，需要一种商定的转移价值方法。为了解决这个问题，以太坊设计了一种内置的货币：以太币(Ether)，也称ETH，有时也用古英语中的Ð表示。以太币最小的单位是Wei(伟)，所有货币值都以Wei的整数倍来记录。以太币的单位列表：

|倍数|单位|
| - | - |
|1|Wei(伟)|
|1012|Szabo(萨博)|
|1015|Finney(芬尼)|
|1018|Ether(以太)|

目前，任何在以太币的上下文中（比如货币、余额或者支付）所使用的价值类型，都应以Wei为单位来计算。

- Gas费用

为了避免网络滥用及回避由于图灵完整性而带来的一些问题，在以太坊中所有的可编程计算都需要收费。费用表以 gas(详见附录)为单位指定。因此，任何指定的可编程计算（包括创建合同，进行消息调用，利用和访问账户存储、在虚拟机上执行操作）都具有普遍认可的gas成本。 

每一个交易都要预先指定gas的数量(gasLimit)和价格(gasPirce)，需要通过交易发送方的账户来支付gas费用。如果账户余额不足，则交易会被视为无效交易。之所以将其命名为gasLimit，在交易结束时未使用的gas会自动返还到交易发送方账户。

一般而言，交易的未返还gas会作为报酬转移到打包此笔交易的区块的矿工账户。交易发送方可以自由指定gas价格，然而矿工也可以任意根据gas价格来筛选交易。交易发送方指定gas价格越高，则被矿工选择打包的机率就越大。因此，交易者需要在更低gas价格和更快交易打包速度之间进行权衡。 

