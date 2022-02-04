## ethereumj项目源代码统计
**项目版本：** 1.8.2

**版本号(Git)：** 17418b7bfe6c522895e84f9c44fe6e17405c5767

**版本URL：<https://github.com/ethereum/ethereumj/releases/tag/1.8.2>**

**主目录源代码统计：** 目录ethereumj-core/src/main/java下的包含446个Java源文件，共73308行代码。定义了568个类（含41个抽象类），61个接口，32个枚举，2个注解(Annotation)。

|类型|行数|占比|
| :- | :- | :- |
|代码|45273|61.76%|
|注释|16574|22.61%|
|空白|11461|15.63%|

整个EthereumJ项目主要源文件目录层次图如下：
```
ethereumj-core/src/main/java/org/ethereum/ (52 文件夹, 446 文件)
├── ConcatKDFBytesGenerator.java
├── Start.java
├── cli
│   └── CLIInterface.java
├── config
│   ├── BlockchainConfig.java (等9个Java文件)
│   ├── blockchain
│   │   └── AbstractConfig.java (等12个Java文件)
│   └── net
│       └── BaseNetConfig.java (等6个Java文件)
├── core
│   ├── AccountState.java (等26个Java文件)
│   └── genesis
│       └── GenesisConfig.java (等2个Java文件)
├── crypto
│   ├── ECIESCoder.java (等5个Java文件)
│   ├── cryptohash
│   │   └── Digest.java (等4个Java文件)
│   ├── jce
│   │   └── ECAlgorithmParameters.java (等5个Java文件)
│   └── zksnark
│       └── BN128.java (等11个Java文件)
├── datasource
│   ├── AbstractCachedSource.java (等31个Java文件)
│   ├── inmem
│   │   ├── HashMapDB.java
│   │   └── HashMapDBSimple.java
│   ├── leveldb
│   │   └── LevelDbDataSource.java
│   └── rocksdb
│       └── RocksDbDataSource.java
├── db
│   ├── AbstractBlockstore.java (等14个Java文件)
│   ├── index
│   │   ├── ArrayListIndex.java
│   │   └── Index.java
│   ├── migrate
│   │   └── MigrateHeaderSourceTotalDiff.java
│   └── prune
│       └── Chain.java (等3个Java文件)
├── facade
│   └──Blockchain.java (等6个Java文件)
├── json
│   ├── EtherObjectMapper.java
│   └── JSONHelper.java
├── listener
│   └── BlockReplay.java (等8个Java文件)
├── manager
│   ├── AdminInfo.java
│   ├── BlockLoader.java
│   └── WorldManager.java
├── mine
│   └── AnyFuture.java (等10个Java文件)
├── net
│   ├── MessageQueue.java
│   ├── MessageRoundtrip.java
│   ├── client
│   │   ├── Capability.java
│   │   ├── ConfigCapabilities.java
│   │   └── PeerClient.java
│   ├── dht
│   │   ├── Bucket.java
│   │   ├── DHTUtils.java
│   │   └── Peer.java
│   ├── eth
│   │   ├── EthVersion.java
│   │   ├── handler
│   │   │   └── Eth.java (等7个Java文件)
│   │   └── message
│   │       └── BlockBodiesMessage.java (等15个Java文件)
│   ├── message
│   │   └── Message.java (等3个Java文件)
│   ├── p2p
│   │   └── DisconnectMessage.java (等10个Java文件)
│   ├── rlpx
│   │   ├── AuthInitiateMessage.java (等20个Java文件)
│   │   └── discover
│   │       ├── DiscoverListener.java (等11个Java文件)
│   │       └── table
│   │           └── DistanceComparator.java (等5个Java文件)
│   ├── server
│   │   └── Channel.java (等5个Java文件)
│   ├── shh
│   │   └── BloomFilter.java (等13个Java文件)
│   ├── submit
│   │   ├── TransactionExecutor.java
│   │   ├── TransactionTask.java
│   │   └── WalletTransaction.java
│   └── swarm
│       ├── Chunk.java (等15个Java文件)
│       └── bzz
│           └── BzzHandler.java (等9个Java文件)
├── samples
│   └── BasicSample.java (等14个Java文件)
├── solidity
│   ├── Abi.java
│   ├── SolidityType.java
│   └── compiler
│       └──CompilationResult.java (等5个Java文件)
├── sync
│   └── BlockBodiesDownloader.java (等12个Java文件)
├── trie
│   └── CollectFullSetOfNodes.java (等7个Java文件)
├── util
│   ├── ALock.java (等22个Java文件)
│   └── blockchain 
│       └── Contract.java (等10个Java文件)
├── validator
│   └──AbstractValidationRule.java (等17个Java文件)
└── vm
    ├── CallCreate.java (等9个Java文件)
    ├── program
    │   ├── InternalTransaction.java (等6个Java文件)
    │   ├── invoke
    │   │    └── ProgramInvoke.java (等4个Java文件)
    │   └── listener
    │       └── CompositeProgramListener.java (等4个Java文件)
    └── trace
        └── Op.java (等4个Java文件)
```
