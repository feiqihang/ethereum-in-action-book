# 区块头部
区块头部包含关联区块（包含链式关系的父区块和树形关系的叔父块）、交易及其执行、账户状态、共识算法等信息。

**区块头部的属性列表：**

|  属性名 | 标识符  | 中文名 | 产生阶段 |
|  ----  | ----  | ----  | ---- |
| parentHash  | H<sub>p</sub>  | 父区块头部哈希 | 挖矿前 |
| ommersHash  | H<sub>o</sub>  | 叔父区块（头部列表）哈希 |  挖矿前 | 
| beneficiary  | H<sub>c</sub> | 区块矿工账户地址 | 挖矿前 |
| stateRoot  | H<sub>r</sub> | 世界状态树的根节点哈希 | 交易执行后 |
| transactionsRoot  | H<sub>t</sub> | 交易树的根节点哈希 | 挖矿前 | 
| receiptsRoot  | H<sub>e</sub> | 交易收据树的根节点哈希 | 交易执行后 | 
| logsBloom  | H<sub>b</sub> | 交易日志Bloom过滤器 | 交易执行后 |
| difficulty  | H<sub>d</sub> | 区块挖矿难度 | 挖矿前 |
| number  | H<sub>i</sub> | 区块高度 | 挖矿前 |
| gasLimit  | H<sub>l</sub> | 区块Gas上限 | 挖矿前 |
| gasUsed  | H<sub>g</sub> | 交易消耗Gas数量 | 交易执行后 |
| timestamp  | H<sub>s</sub> | 区块创建时间 | 挖矿前 |
| extraData  | H<sub>x</sub> | 区块附言 | 挖矿前 |
| mixHash  | H<sub>m</sub> | 工作量证明哈希 | 挖矿后 |
| nonce  | H<sub>n</sub> | 工作量证明随机数 | 挖矿后 |

## 父区块头部哈希
父区块头部的Keccak-256哈希，取值为32字节，与上一区块进行关联，形成链式数据结构。
```java
//BlockHeader.java:L45
/* The SHA3 256-bit hash of the parent block, in its entirety */
private byte[] parentHash;//Hp                                      
```
## 叔父区块哈希
当前区块的叔父块列表的Keccak-256哈希，取值为32字节，与祖先区块的兄弟（叔父）块进行关联，形成树形数据结构。
```java
//BlockHeader.java:L47
/* The SHA3 256-bit hash of the uncles list portion of this block */
private byte[] unclesHash;//Ho
```

在区块打包时（挖矿前），根据区块的叔父区块头部列表，先依次叔父区块头部进行RLP编码（含序列化），再生成整个列表的RLP编码，最后取其Keccak-256哈希。
```java
//BlockchainImpl.java:L500，createNewBlock函数部分代码段
for (BlockHeader uncle : uncles) {//根据叔父块列表，计算区块头部unclesHash属性，对应黄皮书的31公式的部分逻辑
    block.addUncle(uncle);
}
        
//Block.java:L419
public void addUncle(BlockHeader uncle) {//行419-423：addUncle，函数，添加叔父区块并重新计算区块头部的叔父区块哈希值。
    uncleList.add(uncle);//此函数被BlockchainImpl.java引用。
    this.getHeader().setUnclesHash(sha3(getUnclesEncoded()));//计算叔父块（头部）列表的RLP编码的SHA3-256哈希值，作为区块头部的unclesHash属性值，对应黄皮书的31公式的Ho部分
    rlpEncoded = null;
}
```

## 区块矿工账户地址
区块链网络中，矿工提供计算资源将接收到的交易打包为区块，通过共识算法的进行挖矿计算，挖矿成功后以此账户的名义提交区块（含工作量证明），作为接受区块的挖矿奖励和交易费的矿工账户地址（20字节）。
```java
//BlockHeader.java:L49
/* The 160-bit address to which all fees collected from the
 * successful mining of this block be transferred; formally */
private byte[] coinbase;//Hc
```

## 世界状态树的根节点哈希
区块所含交易全部执行完成后，世界状态MPT(Merkle Patricia Tree，以下简称“树”)中的账户状态数据也随之更改，取状态树根节点的Keccak 256位（32字节）哈希值，可看做状态树的版本号。
```java
//BlockHeader.java:L52
/* The SHA3 256-bit hash of the root node of the state trie,
 * after all transactions are executed and finalisations applied */
private byte[] stateRoot;//Hr
```

基于父区块的世界状态树（即世界状态MPT），执行当前区块的所有交易，将改变交易相关账户的状态。比如，转账操作体现为交易双方账户余额的增减；交易GAS费用表现为交易发送方账户余额的减少，矿工账户余额的增加；合约消息调用交易，可能引起合约存储的变动等情况。

新的世界状态树，以键值对的形式，将账户地址与账户状态存储在以太坊节点的状态数据库中。由于MPT的特性之一：树存储数据的任何变动，都将改变根节点的哈希值，因此MPT根节点哈希值是以太坊进行数据一致性验证的重要手段。世界状态树的根节点哈希值，用于以太坊网络节点验证交易执行后的世界状态树，从而保证全网的状态数据的最终一致性。
```java
//BlockchainImpl.java:L507，createNewBlock函数部分代码段
Repository track = repository.getSnapshotTo(parent.getStateRoot()); //根据父区块的世界状态树的根节点哈希值(Hr)，获取对应版本的世界状态，对应黄皮书的33公式
BlockSummary summary = applyBlock(track, block);
List<TransactionReceipt> receipts = summary.getReceipts();
block.setStateRoot(track.getRoot());    //行507-510：计算世界状态树的根节点哈希值，并赋值给区块头部的stateRoot属性，对应黄皮书的31公式的Hr部分和169公式                                                          
```

## 交易树的根节点哈希
将区块所含交易列表，根据交易的内容和顺序生成的MPT，然后取MPT根节点的Keccak 256位（32字节）哈希值。
```java
//BlockHeader.java:L55
/* The SHA3 256-bit hash of the root node of the trie structure
 * populated with each transaction in the transaction
 * list portion, the trie is populate by [key, val] --> [rlp(index), rlp(tx_recipe)]
 * of the block */
private byte[] txTrieRoot;//Ht
```

在区块打包时，根据区块的交易列表生成交易树，以键值对的形式存储（交易列表）索引号和交易数据，然后获取交易树的根节点哈希。用于以太坊网络节点验证交易内容和顺序是否一致。
```java
//BlockchainImpl.java:L316
public static byte[] calcTxTrie(List<Transaction> transactions) {//根据交易列表，计算区块头部的交易树根节点哈希值(txTrieRoot)属性，对应黄皮书的31公式的Ht部分和32公式

    Trie txsState = new TrieImpl();

    if (transactions == null || transactions.isEmpty())
        return HashUtil.EMPTY_TRIE_HASH;

    for (int i = 0; i < transactions.size(); i++) {
        txsState.put(RLP.encodeInt(i), transactions.get(i).getEncoded());//对应黄皮书的32公式，键值对组合：交易索引的RLP编码值，以及交易的RLP编码值
    }
    return txsState.getRootHash();
}
```

## 交易收据树的根节点哈希
由当前区块中所有交易收据生成的MPT，记录交易收据的内容和顺序，然后取MPT根节点的Keccak 256位（32字节）哈希值。
```java
//BlockHeader.java:L60
/* The SHA3 256-bit hash of the root node of the trie structure
 * populated with each transaction recipe in the transaction recipes
 * list portion, the trie is populate by [key, val] --> [rlp(index), rlp(tx_recipe)]
 * of the block */
private byte[] receiptTrieRoot;//He
```

在区块中的交易全部执行后，根据交易收据列表生成交易收据树，以键值对的形式存储索引号和交易收据，然后获取交易收据树的根节点哈希。以太坊网络节点在执行区块内的所有交易后，可以验证交易的执行状态、日志、Gas使用量是否一致。

```java
//BlockchainImpl.java:L659
public static byte[] calcReceiptsTrie(List<TransactionReceipt> receipts) {//根据交易收据列表，计算交易收据树的根节点哈希值，对应黄皮书的31公式的He部分和32公式
    Trie receiptsTrie = new TrieImpl();

    if (receipts == null || receipts.isEmpty())
        return HashUtil.EMPTY_TRIE_HASH;

    for (int i = 0; i < receipts.size(); i++) {
        receiptsTrie.put(RLP.encodeInt(i), receipts.get(i).getReceiptTrieEncoded());
    }
    return receiptsTrie.getRootHash();
}
```

*思考：交易收据树和世界状态树都是交易执行后产生的，两者都是以太坊节点在交易执行后的验证手段，那么区块头部是否保留其中一个就足够了？*  
*交易收据树侧重于交易执行过程（执行状态也是反应执行过程中是否出现错误），而世界状态树侧重于交易执行结果。如果将交易理解为一道数学题，那么就是解题过程（中间结果）和解题结论（最终结果）的区别。*  
*以太坊是一个分布式账本的技术实现，网络中的交易可能由任何节点打包（区块）和执行，而不是在某个受控制的计算资源中执行。参与方既会关心一笔交易的结果是否正确；也想知道Gas费用是否合理（与标准一致），或者通过日志检查合约逻辑是否正确，等其他执行过程信息。*  
*比如，EIP-658启用后交易收据中会添加状态字段，如果以太坊网络中有的节点启用此EIP，而有些未启用，就会出现不同的以太坊节点执行同一笔交易后，节点间的账户状态数据一致，但是交易收据不一致的情况。此时，仅验证世界状态树的根节点哈希是否无法发现这种情况的。*  
*当然，可能还有其他情况，但说明了世界状态树的根节点哈希，以及交易收据树的根节点哈希都有其存在的必要性。*

## 交易日志Bloom过滤器
区块所含交易收据中的可索引信息（产生日志的地址、日志主题）生成的Bloom（布隆）过滤器，用于日志信息检索。Bloom过滤器的内容是一段2048位（256字节）的数据，多个Bloom可以通过`或位运算`合并为一个。
```java
//BlockHeader.java:L65
/* The Bloom filter composed from indexable information 
 * (logger address and log topics) contained in each log entry 
 * from the receipt of each transaction in the transactions list */
private byte[] logsBloom;//Hb
```

根据交易收据列表，获取每笔交易收据的日志Bloom过滤器，然后合并为一个Bloom过滤器。
```java
//BlockchainImpl.java:L512，createNewBlock函数部分代码段
Bloom logBloom = new Bloom();
for (TransactionReceipt receipt : receipts) {
    logBloom.or(receipt.getBloomFilter());
}
block.getHeader().setLogsBloom(logBloom.getData()); 
```

## 区块挖矿难度
当前区块的挖矿难度，取值范围为自然数（非负整数），与以太坊网络的区块高度、出块速度、分叉情况等信息有关联，可用于调节区块链网络的运行。
```java
//BlockHeader.java:L69
/* A scalar value corresponding to the difficulty level of this block.
 * This can be calculated from the previous block’s difficulty level
 * and the timestamp */
private byte[] difficulty;//Hd
```

区块打包时，在父区块挖矿难度的基础上增加一定的区块周期难度，增加或减少动态调节难度。  
* 区块周期难度，取值范围为非负整数，当区块高度超过20万号块时开始生效（否则为0），以10万个区块为一个周期，周期内的难度值相同，新周期的难度值会指数级(2<sup>n</sup>)增长。  
* 动态调节难度，取值范围为整数，根据出块速度（当前区块与父区块的间隔时间）调节，使其维持在一定水平；另外会根据分叉情况调节，以避免分叉情况。  
*具体的讲，当父区块无叔父块时，当前区块与父区块的时间差临界区间为[9, 18)秒，小于此区间时提高难度，大于此区间时降低难度；当父区块有叔父块时，临界区间为[18,27)，即近期出现过分叉时会加大出块难度，以减小出现分叉的可能性。*
```java
//BlockchainImpl.java:L504，createNewBlock函数部分代码段
block.getHeader().setDifficulty(ByteUtil.bigIntegerToBytes(block.getHeader().//计算区块头部的难度值，对应黄皮书41、42、43、44、45、46公式
        calcDifficulty(config.getBlockchainConfig(), parent.getHeader())));

//ByzantiumConfig.java:L69
public BigInteger calcDifficulty(BlockHeader curBlock, BlockHeader parent) {//对应黄皮书的41公式，计算当前区块难度（启用假区块高度）
    BigInteger pd = parent.getDifficultyBI();
    BigInteger quotient = pd.divide(getConstants().getDIFFICULTY_BOUND_DIVISOR());//对应黄皮书的43公式，计算父区块难度的2048分之一
    BigInteger sign = getCalcDifficultyMultiplier(curBlock, parent);
    BigInteger fromParent = pd.add(quotient.multiply(sign));//基于父区块的叔父块列表和时间戳，根据分叉情况和出块速度动态调整出块难度
    BigInteger difficulty = max(getConstants().getMINIMUM_DIFFICULTY(), fromParent);
    int explosion = getExplosion(curBlock, parent);//行78-82：对应黄皮书的45公式，根据区块高度计算阶梯难度
    if (explosion >= 0) {
        difficulty = max(getConstants().getMINIMUM_DIFFICULTY(), difficulty.add(BigInteger.ONE.shiftLeft(explosion)));
    }
    return difficulty;
}

protected int getExplosion(BlockHeader curBlock, BlockHeader parent) {//对应黄皮书的45公式的部分逻辑，根据区块高度计算对应的周期
    int periodCount = (int) (Math.max(0, curBlock.getNumber() - 3_000_000) / getConstants().getEXP_DIFFICULTY_PERIOD());//部分代码（除号左侧部分）对应黄皮书的46公式，实际区块高度减去300万，以延迟“难度炸弹”（或称为“冰河时期”）。
    return periodCount - 2;
}
```

值得注意的是，在EIP-649（延迟“难度炸弹”和区块奖励减少）提案启用后，计算难度值时使用的是一个假的区块高度（真实高度减去300万）。目的是为以太坊社区争取更多的时间，来开发新的权益证明(PoS)共识算法，而在此之前需要降低区块挖矿难度，以防止因为难度过高导致以太坊网络无法正常出块（进入冰河时期）。  
下面是EIP-649启用前的区块周期难度计算的部分逻辑，可以看到这里用的是真实的区块高度。  
```java
//AbstractConfig.java:L103
protected int getExplosion(BlockHeader curBlock, BlockHeader parent) {
    int periodCount = (int) (curBlock.getNumber() / getConstants().getEXP_DIFFICULTY_PERIOD());
    return periodCount - 2;
}
```

*注：AbstractConfig类中的calcDifficulty函数与ByzantiumConfig类一致，所以这里只摘录了发生变化的getExplosion函数。*

## 区块高度
区块高度，即当前区块的祖先的数量，创世区块的这个数量为0；取值范围为自然数。
```java
//BlockHeader.java:L76
/* A scalar value equal to the number of ancestor blocks.
 * The genesis block has a number of zero */
private long number;//Hi
```

在区块打包时赋值：父区块高度加1。
```java
//BlockchainImpl.java:L478，createNewBlock函数部分代码段
final long blockNumber = parent.getNumber() + 1;//当前区块高度等于父区块高度加1，对应黄皮书的40公式
```

## 区块Gas上限
区块所含交易执行时可用Gas上限值，用于限制区块所含交易的数量和计算复杂度，也可理解为控制区块大小；取值范围为自然数。
```java
//BlockHeader.java:L79
/* A scalar value equal to the current limit of gas expenditure per block */
private byte[] gasLimit;//Hl
```

*注：关于此属性的赋值逻辑，以太坊Java版的实现是：在区块打包时直接取父区块的gasLimit，见BlockchainImpl类的488行。*

以太坊黄皮书中没有约定如何计算区块的gasLimit，但是给定了取值范围：基于父区块gasLimit的增加幅度，以及一个最小阀值。

* 增减幅度不超过父区块gasLimit的1024分之一
```java
//ParentGasLimitRule.java:L43
public boolean validate(BlockHeader header, BlockHeader parent) {//校验区块与父区块的gasLimit变动幅度（增减幅度不超过父区块gasLimit的1024分之一），对应黄皮书的50公式的部分逻辑
    errors.clear();
    BigInteger headerGasLimit = new BigInteger(1, header.getGasLimit());
    BigInteger parentGasLimit = new BigInteger(1, parent.getGasLimit());
    if (headerGasLimit.compareTo(parentGasLimit.multiply(BigInteger.valueOf(GAS_LIMIT_BOUND_DIVISOR - 1)).divide(BigInteger.valueOf(GAS_LIMIT_BOUND_DIVISOR))) < 0 ||
        headerGasLimit.compareTo(parentGasLimit.multiply(BigInteger.valueOf(GAS_LIMIT_BOUND_DIVISOR + 1)).divide(BigInteger.valueOf(GAS_LIMIT_BOUND_DIVISOR))) > 0) {
        errors.add(String.format(
                "#%d: gas limit exceeds parentBlock.getGasLimit() (+-) GAS_LIMIT_BOUND_DIVISOR",
                header.getNumber()
        ));
        return false;
    }
    return true;
}
```

* GasLimit的最小阀值为5000
```java
//GasLimitRule.java:L44
public ValidationResult validate(BlockHeader header) {//检验区块头部GasLimit的最小阀值，对应黄皮书的47公式的部分逻辑
    if (new BigInteger(1, header.getGasLimit()).compareTo(BigInteger.valueOf(MIN_GAS_LIMIT)) < 0) { //这里MIN_GAS_LIMIT为5000
        return fault("header.getGasLimit() < MIN_GAS_LIMIT");
    }
    return Success;
}
```

## 交易消耗Gas数量
区块所含交易执行后消耗Gas的累计值，取值范围为自然数。
```java
//BlockHeader.java:L81
/* A scalar value equal to the total gas used in transactions in this block */
private long gasUsed;//Hg
```

当区块中的所有交易执行后，根据交易收据列表，取最后一个交易收据的Gas累计使用量。
```java
//BlockchainImpl.java:L517，createNewBlock函数部分代码段
block.getHeader().setGasUsed(receipts.size() > 0 ? receipts.get(receipts.size() - 1).getCumulativeGasLong() : 0);//根据交易收据列表，取最后一个交易收据的Gas累计使用量，对应黄皮书的158公式
```

## 区块创建时间
当前区块初始化时的Unix时间戳，即从1970年1月1日0时0分0秒（UTC）起至现在的总秒数（不考虑闰秒），取值范围为小于2<sup>256</sup>的自然数。
```java
/* A scalar value equal to the reasonable output of Unix's time()
 * at this block's inception */
private long timestamp;//Hs
```
在区块打包时，取当前以太坊节点的系统时间戳。由于以太坊是一个分布式网络环境，各个节点的系统时间可能是不同步的，为了避免这种情况导致区块创建时间的混乱，以太坊黄皮书中约定：当前区块的创建时间必须大于父区块的创建时间。
```java
//BlockchainImpl.java:L469
public synchronized Block createNewBlock(Block parent, List<Transaction> txs, List<BlockHeader> uncles) {
    long time = System.currentTimeMillis() / 1000;
    // adjust time to parent block this may happen due to system clocks difference
    if (parent.getTimestamp() >= time) time = parent.getTimestamp() + 1;//保证区块的出块时间大于父区块，对应黄皮书的48公式

    return createNewBlock(parent, txs, uncles, time);
}
```

## 区块附言
区块附言是一段与当前区块相关的不超过32个字节的数据。
```java
//BlockHeader.java:L87
/* An arbitrary byte array containing data relevant to this block.
 * With the exception of the genesis block, this must be 32 bytes or fewer */
private byte[] extraData;//Hx
```

*注：在以太坊主链(MainNet)中，常被作为一种宣传途径，如标注当前区块来自哪个矿池。*

## 工作量证明哈希
Ethash PoW共识算法的返回值之一，根据挖矿前的区块头部（以及nonce）、Ethash数据集生成的哈希值，取值为32字节（256位）。可作为工作量证明，与nonce一起证明当前区块已经承载了足够的计算量。
```java
//BlockHeader.java:L85
private byte[] mixHash;//Hm
```

在区块挖矿成功后，获取Ethash PoW函数返回值中的工作量证明哈希，赋值给区块头部的mixHash属性。
```java
//Ethash.java:L343，MineTask子类的函数
protected void postProcess(MiningResult result) {
    Pair<byte[], byte[]> pair = hashimotoLight(block.getHeader(), result.nonce);
    block.setNonce(longToBytes(result.nonce));  
    block.setMixHash(pair.getLeft());//获取Ethash PoW函数返回值中的工作量证明哈希，赋值给区块头部的mixHash属性，对应黄皮书的168公式
}
```

## 工作量证明随机数
作为Ethash PoW算法的参数之一，挖矿过程中尝试不同nonce值，直到计算结果满足难度值要求（即挖矿成功），取值为8字节（64位），即数值区间为[0, 2<sup>64</sup>)。可以作为工作量证明，与mixHash一起于证明当前区块已经承载了足够的计算量。
```java
//BlockHeader.java:L92
private byte[] nonce;//Hn
```
*注：目前nonce属性的注释有误，作者已经提交PR(pull request)并被采纳，详见https://github.com/ethereum/ethereumj/pull/1281*
在区块挖矿成功后，将获取Ethash PoW参数中随机数，赋值给区块头部的nonce属性。
```java
//Ethash.java:L343，MineTask子类的函数
protected void postProcess(MiningResult result) {
    Pair<byte[], byte[]> pair = hashimotoLight(block.getHeader(), result.nonce);
    block.setNonce(longToBytes(result.nonce));//获取Ethash PoW参数中随机数，赋值给区块头部的nonce属性，对应黄皮书的167公式
    block.setMixHash(pair.getLeft());   
}
```

*看到这里，可能有的读者已经被PoW（工作量证明）、nonce（随机数）等概念搞得不知所措了，后续章节中我们会详细探讨，这里先简单了解一下。PoW共识算法，一种比较形象的解释是：当前新的区块产生后，系统会出一道数学题，谁先解题完成就获得下一区块的记账权。*  
***这是一道什么样的数学题呢？***  
*以太坊采用的是Ethash PoW共识算法：*  
(n, m) = PoW(H<sub><del>n</del></sub>, H<sub>n</sub>, **d**)  
*且计算结果满足规则：*  
n <= (2^256) / H<sub>d</sub>  
*其中，Ethash PoW算法的3个参数中，有2个是已知的：*  
* *H<sub><del>n</del></sub>，挖矿前的区块头部，即不含nonce和mixHash的区块头部。*  
* **d**，数据集，根据区块高度按照一定规则生成的1GB（2<sup>30</sup>个字节）大小的数据，且以每隔3万个区块更新一次。*

***那么，未知的参数H<sub>n</sub>就是解题的关键了，怎么获得呢？***  
*根据nonce的定义，我们只知道其取值范围是[0, 2<sup>64</sup>)，有可能是其中的任意数值，获取正确数值的途径只有不断尝试，然后将计算结果与目标难度进行比对，直到满足要求。*  
*当然，我们能做的就是想办法提高nonce的命中率，可以尝试随机取数，也可以按一定顺序取数；可以用CPU资源，也可以用GPU资源，甚至专用设备；可以用采用集中式计算，也可以分布式计算，等其他途径。*

## 区块头部序列化
区块头部的RLP序列化的属性顺序：Hp,Ho,Hc,Hr,Ht,He,Hb,Hd,Hi,Hl,Hg,Hs,Hx,Hm,Hn

需要注意的是，在挖矿之前mixHash和nonce属性还没有，所以此时Hm、Hn不参与序列化。
```java
//BlockHeader.java:L308
public byte[] getEncoded(boolean withNonce) {
    byte[] parentHash = RLP.encodeElement(this.parentHash);

    byte[] unclesHash = RLP.encodeElement(this.unclesHash);
    byte[] coinbase = RLP.encodeElement(this.coinbase);

    byte[] stateRoot = RLP.encodeElement(this.stateRoot);

    if (txTrieRoot == null) this.txTrieRoot = EMPTY_TRIE_HASH;
    byte[] txTrieRoot = RLP.encodeElement(this.txTrieRoot);

    if (receiptTrieRoot == null) this.receiptTrieRoot = EMPTY_TRIE_HASH;
    byte[] receiptTrieRoot = RLP.encodeElement(this.receiptTrieRoot);

    byte[] logsBloom = RLP.encodeElement(this.logsBloom);
    byte[] difficulty = RLP.encodeBigInteger(new BigInteger(1, this.difficulty));
    byte[] number = RLP.encodeBigInteger(BigInteger.valueOf(this.number));
    byte[] gasLimit = RLP.encodeElement(this.gasLimit);
    byte[] gasUsed = RLP.encodeBigInteger(BigInteger.valueOf(this.gasUsed));
    byte[] timestamp = RLP.encodeBigInteger(BigInteger.valueOf(this.timestamp));

    byte[] extraData = RLP.encodeElement(this.extraData);
    if (withNonce) {
        byte[] mixHash = RLP.encodeElement(this.mixHash);
        byte[] nonce = RLP.encodeElement(this.nonce);
        return RLP.encodeList(parentHash, unclesHash, coinbase,//区块头部的RLP序列化的属性顺序：Hp,Ho,Hc,Hr,Ht,He,Hb,Hd,Hi,Hl,Hg,Hs,Hx,Hm,Hn，对应黄皮书的34公式
                stateRoot, txTrieRoot, receiptTrieRoot, logsBloom, difficulty, number,
                gasLimit, gasUsed, timestamp, extraData, mixHash, nonce);
    } else {
        return RLP.encodeList(parentHash, unclesHash, coinbase,//挖矿前的区块头部，作为Ethash PoW函数的参数，参与RLP序列化的属性不含Hm、Hn，对应黄皮书的49公式
                stateRoot, txTrieRoot, receiptTrieRoot, logsBloom, difficulty, number,
                gasLimit, gasUsed, timestamp, extraData);
    }
}
```
