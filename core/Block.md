# 区块
以太坊在整体上可以看作一个基于交易的状态机：起始于一个创世区块（Genesis）状态，逐笔执行交易直到其转变为某个版本的最终状态。
这里的版本是以区块为单位，所以我们会说某个区块的世界状态。

在以太坊网络中，参与方可以在任意时间，通过任意账户经由任意节点发起一笔交易。
此时，交易尚未生效（写入账本），只是由网络中某些节点传播和暂存。
以太坊是以区块为单位进行记账，需要担任矿工角色的节点将一定数量的交易打包为区块，然后通过挖矿（PoW共识算法）来争夺下一区块的记账（出块）权，最后拥有记账权的矿工将新区块写入本地账本，并同步给其他节点。
节点将区块写入本地账本，包含一系列的操作过程：区块的验证和解析，交易的验证和执行，然后将交易执行后的账户状态数据写入状态数据库，将区块写入账本数据库；也包含一些其他逻辑（比如处理区块分叉）。
我们可以发现，区块在以太坊的设计理念中占据重要的地位，与账本结构、记账粒度、PoW共识算法都密切相关。
  
*在讨论概念细节之前，笔者想强调一个观点：以太坊是区块链技术的一个具体实现方案，而区块链是分布式账本的一种技术范式。有些概念可能是分布式账本或区块链技术自身的，而有些概念可能是以太坊引入的。*  
*比如，分叉并非区块链本身的概念，或者说并非所有的区块链技术都会出现分叉，使用BFT（拜占庭容错）共识算法的区块链技术就不会产生分叉。再者，区块的概念在不同的分布式账本中的地位是不一样的，有的会弱化区块的作用，甚至可能没有区块（以交易为粒度进行记账）。*  
*当然，每种技术都有其适用场景、整体设计理念、技术特性，不能单一的从某个点去评判好与坏。*  
*笔者尽量对各类概念加以区分和标注，更重要的是读者可以认识到这些概念可能属于不同的技术范畴，并加以思考。*

**区块的属性列表：**

|  属性名 | 标识符[^标识符]  | 中文名  | 释义 |
|  ----  | ----  | ----  | ---- |
| header  | B<sub>H</sub>  | 区块头部 | 基本信息和验证信息 |
| transactionsList | B<sub>T</sub>  | 交易列表 | 区块的核心内容 |
| uncleList | B<sub>U</sub>  | 叔父区块头列表 | 祖先区块的兄弟（分叉）块 |

[^标识符]:在以太坊黄皮书中，约定的某些数据的标识符，然后会在相关公式中引用这些数据。

## 区块头部
区块头部包含一些区块相关的基本信息，以及区块生命周期中关键数据的验证信息，每个区块包含一个区块头部（详见BlockHeader.java的解读）。

```java
//Block.java:L54
private BlockHeader header; 
```

打包新区块是一系列的操作过程，仅区块头部的创建过程就可分为3个阶段：

1. **挖矿前：** 
此阶段主要生成区块头部的基本信息，根据父区块、交易列表、叔父块头部列表、当前时间戳等信息，给区块头部的部分属性赋值。

```java
//BlockchainImpl.java:L482，createNewBlock函数部分代码段
final long blockNumber = parent.getNumber() + 1;//当前区块高度等于父区块高度加1，对应黄皮书的40公式

final byte[] extraData = config.getBlockchainConfig().getConfigForBlock(blockNumber).getExtraData(minerExtraData, blockNumber);

Block block = new Block(parent.getHash(),//获取父区块头部哈希值，对应黄皮书的39公式
        EMPTY_LIST_HASH, // uncleHash
        minerCoinbase,
        new byte[0], // log bloom - from tx receipts
        new byte[0], // difficulty computed right after block creation
        blockNumber,
        parent.getGasLimit(), // (add to config ?)
        0,  // gas used - computed after running all transactions
        time,  // block time
        extraData,  // extra data
        new byte[0],  // mixHash (to mine)
        new byte[0],  // nonce   (to mine)
        new byte[0],  // receiptsRoot - computed after running all transactions
        calcTxTrie(txs),    // TransactionsRoot - computed after running all transactions//根据交易列表，计算区块头部的交易树根节点哈希值(txTrieRoot)属性，对应黄皮书的31公式的Ht部分
        new byte[] {0}, // stateRoot - computed after running all tranxsactions
        txs,
        null);  // uncle list

for (BlockHeader uncle : uncles) {//根据叔父块列表，计算区块头部unclesHash属性，对应黄皮书的31公式的部分逻辑
    block.addUncle(uncle);
}

block.getHeader().setDifficulty(ByteUtil.bigIntegerToBytes(block.getHeader().//计算区块头部的难度值，对应黄皮书41、42、43、44、45、46公式
        calcDifficulty(config.getBlockchainConfig(), parent.getHeader())));
```

2. **交易执行后：**
生成区块头部的验证信息：世界状态树的根节点哈希、交易收据树的根节点哈希、交易日志Bloom过滤器、交易消耗Gas数量。

```java
//BlockchainImpl.java:L507，createNewBlock函数部分代码段
Repository track = repository.getSnapshotTo(parent.getStateRoot());//根据父区块的世界状态树的根节点哈希值(Hr)，获取对应版本的世界状态，对应黄皮书的33公式
BlockSummary summary = applyBlock(track, block);
List<TransactionReceipt> receipts = summary.getReceipts();
block.setStateRoot(track.getRoot());//行507-510：计算世界状态树的根节点哈希值，并赋值给区块头部的stateRoot属性，对应黄皮书的31公式的Hr部分和169公式

Bloom logBloom = new Bloom();
for (TransactionReceipt receipt : receipts) {
    logBloom.or(receipt.getBloomFilter());
}
block.getHeader().setLogsBloom(logBloom.getData());//行512-516：根据交易收据列表中的Bloom过滤器信息，计算区块头部的日志Bloom属性，对应黄皮书的31公式的Hb部分
block.getHeader().setGasUsed(receipts.size() > 0 ? receipts.get(receipts.size() - 1).getCumulativeGasLong() : 0);//根据交易收据列表，取最后一个交易收据的Gas累计使用量，对应黄皮书的158公式
block.getHeader().setReceiptsRoot(calcReceiptsTrie(receipts));//根据交易收据列表，计算区块头部的交易收取哈希值，对应黄皮书的31公式的He部分逻辑 
```

3. **挖矿后：**
根据交易执行后的区块头部（以及一些其他信息），通过Ethash PoW共识算法进行挖矿计算，直到挖矿成功（计算结果满足区块头部的难度值）。
然后，将PoW共识算法的相关参数和返回值，分别赋值给区块头部的nonce和mixHash，作为参与过挖矿的（工作量）证明。

```java
//Ethash.java:L343
protected void postProcess(MiningResult result) {
    Pair<byte[], byte[]> pair = hashimotoLight(block.getHeader(), result.nonce);
    block.setNonce(longToBytes(result.nonce));//获取Ethash PoW参数中随机数，赋值给区块头部的nonce属性，对应黄皮书的167公式
    block.setMixHash(pair.getLeft());//获取Ethash PoW函数返回值中的工作量证明哈希，赋值给区块头部的mixHash属性，对应黄皮书的168公式
}
```

## 交易列表
组成当前区块的一些交易，整个区块的核心内容，一个区块中包含多笔交易（详见Transaction.java的解读）。
```java
//Block.java:L57
private List<Transaction> transactionsList = new CopyOnWriteArrayList<>();
```

单个区块可以包含的交易数量，与区块头部的gasLimit属性有关，因为每笔交易的执行都需要消耗一定数量的gas，以太坊通过设定单个区块可以使用的gas数量，来控制区块大小。

*思考：区块的合理大小，应该综合考虑技术架构、网络环境、交易吞吐量目标等因素。通过以太坊黄皮书的第47公式，能看到以太坊的设计支持区块Gas上限值的浮动，可以随着区块高度递增。*

## 叔父区块头列表
祖先区块的兄弟（分叉）块，在给定高度范围内，选择一定数量的叔父区块头部进行打包。
```java
//Block.java:L60
private List<BlockHeader> uncleList = new CopyOnWriteArrayList<>();
```

区块打包时，矿工可以在叔父块中按照规则选取一定数量的叔父块，将其区块头打包到区块中。具体选取方法是：
1. **祖先区块范围：** 
找到当前区块的最近6代的祖先区块，包含父区块、祖父区块、曾祖区块，以及再往前的3代。
2. **叔父区块范围：** 
寻找上述祖先区块的兄弟块，即与祖先区块的高度相同，且有共同的父区块。  
在区块链网络中，叔父块代表过去一段时间出现过分叉，反应了有多个矿工节点在竞争同一区块的记账权，且在相差无几的时间内挖矿成功。而这些同一高度的区块，会按照规则选择其中一个写入账本（包含的交易生效），而其他的作为区块链的新分叉区块（包含的交易不生效）。
3. **选择要打包的叔父块：**
在上诉叔父区块范围内选择2个，将其区块头部打包至新区块。  
关于被选择的叔父块，虽然其包含的交易依然无法生效，无法收取交易费用（Gas），但是可以获得一定比例（与区块高度成正比）的挖矿奖励。
这是以太坊对矿工利益的维护，降低矿工对分叉区块的顾虑，提高其参与以太坊网络建设的积极性。
```java
//BlockMiner.java:L207
protected List<BlockHeader> getUncles(Block mineBest) {
    List<BlockHeader> ret = new ArrayList<>();
    long miningNum = mineBest.getNumber() + 1;
    Block mineChain = mineBest;
    long limitNum = max(0, miningNum - UNCLE_GENERATION_LIMIT);
    Set<ByteArrayWrapper> ancestors = BlockchainImpl.getAncestors(blockStore, mineBest, UNCLE_GENERATION_LIMIT + 1, true);
    Set<ByteArrayWrapper> knownUncles = ((BlockchainImpl)blockchain).getUsedUncles(blockStore, mineBest, true);
    knownUncles.addAll(ancestors);
    knownUncles.add(new ByteArrayWrapper(mineBest.getHash()));
    if (blockStore instanceof IndexedBlockStore) {
        outer:
        while (mineChain.getNumber() > limitNum) {
            List<Block> genBlocks = ((IndexedBlockStore) blockStore).getBlocksByNumber(mineChain.getNumber());
            if (genBlocks.size() > 1) {
                for (Block uncleCandidate : genBlocks) {
                    if (!knownUncles.contains(new ByteArrayWrapper(uncleCandidate.getHash())) &&
                            ancestors.contains(new ByteArrayWrapper(blockStore.getBlockByHash(uncleCandidate.getParentHash()).getHash()))) {
                        ret.add(uncleCandidate.getHeader());
                        if (ret.size() >= UNCLE_LIST_LIMIT) {
                            break outer;
                        }
                    }
                }
            }
            mineChain = blockStore.getBlockByHash(mineChain.getParentHash());
        }
    } else {
        logger.warn("BlockStore is not instance of IndexedBlockStore: miner can't include uncles");
    }
    return ret;
}
```

*思考：通过分析上述代码，发现在选择要打包的2个叔父块时，其逻辑是优先选择较高的叔父块，即从父区块开始往前查找。这样做的特点是，可以最大化当前区块的叔父块奖励总额，对整个矿工群体是好的。如果想让自己的收益最大化，在现有逻辑上加个条件，来判断叔父块矿工账户是否属于自己的，这段代码怎么改呢？*

## 区块序列化
区块的RLP序列化的属性顺序：B<sub>H</sub>, B<sub>T</sub>, B<sub>U</sub>
```java
//Block.java:L425
public byte[] getEncoded() {//行425-436：getEncoded，函数，获取区块内容的RLP编码字节数组，参与RLP编码的区块属性依次为：header（含nonce）、transactionsList、uncleList。
    if (rlpEncoded == null) {//此函数被当前类和BlockchainImpl.java、IndexedBlockStore.java、BlockMiner.java、SyncManager.java等源文件引用。
        byte[] header = this.header.getEncoded();

            List<byte[]> block = getBodyElements();//获取区块体：交易列表、叔父块列表
        block.add(0, header);//区块序列化顺序：区块头、区块列表、叔父块列表
        byte[][] elements = block.toArray(new byte[block.size()][]);

            this.rlpEncoded = RLP.encodeList(elements);//行426-433：区块的RLP序列化的属性顺序：区块头部、交易列表、叔父块列表，对应黄皮书的35、36公式
    }
    return rlpEncoded;
}
```
