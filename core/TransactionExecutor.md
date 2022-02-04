# TransactionExecutor

## 交易执行

以太坊在整体上是一个基于交易的状态机，通过交易的执行改变状态。交易执行是以太坊协议中最复杂的部分，其具体逻辑定义为状态转换函数。

> 以太坊黄皮书中，状态转换函数(ϒ)的定义：
>
> $$
> \boldsymbol{\sigma}' = \Upsilon(\boldsymbol{\sigma}, T)
> $$
>
>
>
> 其中T表示交易，σ表示状态，σ'表示交易后的状态。\
> 定义ϒg为交易执行所消耗的gas数量，定义ϒl为交易执行过中产生的日志集合，定义ϒz为交易执行结果的状态码。

交易执行的具体内容包括：

* 交易基础校验：基本格式、交易签名、交易费用（Gas）、转账金额等内容的有效性。
* 合约创建：通过交易创建新的智能合约账户。
* 消息调用：通过交易调用合约账户，执行指定的合约逻辑；也可以通过交易调用外部账户，进行转账或附言。
* 代码执行：通过EVM执行交易中指定的合约代码，用于创建或调用合约。
* 状态修改：根据交易修改状态，进行转账、Gas支付、删除账户状态等操作。

### 交易基础校验

任意交易在执行前都要进行基本的有效性校验。 当然，以太坊会在交易执行的各个阶段针对相关问题进行检查。 比如，合约代码逻辑错误、Gas额度过小等问题，需要代码执行阶段才能发现。

#### 校验交易内容格式

对交易的RLP编码格式，以及每个属性的数据类型和取值范围进行检查。这是一笔交易最基本的校验，会出现在交易生命周期的各个阶段，从交易创建、区块打包、网络传输等，直到目前的交易执行阶段都会进行此项检查。

* 交易RLP格式和数据类型校验：属性数量为9个，依次为nonce(Tn), gasPrice(Tp), gasLimit(Tg), receiveAddress(Tt), value(Tv), data(Ti/Td), signature.v(Tw), signature.r(Tr), signature.s(Ts)

```java
//Transaction.java:L193
public synchronized void rlpParse() {//行193-230：rlpParse，功能函数，对RLP格式的交易内容进行解码，被当前类和InternalTransaction.java引用。
    if (parsed) return;//参与RLP编码的9个交易属性，根据作用划分为两类：交易基本信息(nonce、gasLimit、gasLimit、receiveAddress、value、data)，交易签名(signature.v、signature.r、signature.s)。
    try {//其中，交易签名信息不是必有的，当交易的发送者是合约账户时，是没有交易签名的，此类交易称为内部交易（见InternalTransaction.java解读）。
        RLPList decodedTxList = RLP.decode2(rlpEncoded);
        RLPList transaction = (RLPList) decodedTxList.get(0);

        // Basic verification
        if (transaction.size() > 9 ) throw new RuntimeException("Too many RLP elements");
        for (RLPElement rlpElement : transaction) {
            if (!(rlpElement instanceof RLPItem))
                throw new RuntimeException("Transaction RLP elements shouldn't be lists");
        }

        this.nonce = transaction.get(0).getRLPData();
        this.gasPrice = transaction.get(1).getRLPData();
        this.gasLimit = transaction.get(2).getRLPData();
        this.receiveAddress = transaction.get(3).getRLPData();
        this.value = transaction.get(4).getRLPData();
        this.data = transaction.get(5).getRLPData();
        // only parse signature in case tx is signed
        if (transaction.get(6).getRLPData() != null) {
            byte[] vData =  transaction.get(6).getRLPData();
            BigInteger v = ByteUtil.bytesToBigInteger(vData);
            byte[] r = transaction.get(7).getRLPData();
            byte[] s = transaction.get(8).getRLPData();
            this.chainId = extractChainIdFromRawSignature(v, r, s);
            if (r != null && s != null) {
                this.signature = ECDSASignature.fromComponents(r, s, getRealV(v));
            }
        } else {
            logger.debug("RLP encoded tx is not signed!");
        }
        this.hash = HashUtil.sha3(rlpEncoded);
        this.parsed = true;
    } catch (Exception e) {
        throw new RuntimeException("Error on parsing RLP", e);
    }
}
```

_注意：通常说的交易是指由外部账户发起和签名的。此外，由合约在执行阶段发起（没有签名信息）的叫作内部交易。上面这段代码中，有针对内部交易的判断和处理逻辑。_

* 数值内容校验：校验nonce、gasLimit、gasPrice、value的取值范围，校验交易签名的有效性。

```java
//Transaction.java:L232
private void validate() {//行232-250：validate，内部功能函数，验证交易内容。对应黄皮书的16（部分）、17公式。
    if (getNonce().length > HASH_LENGTH) throw new RuntimeException("Nonce is not valid");//校验规则1：属性nonce、value、gasPrice、gasLimit、signature.r、signature.s的值都属于N256集合（所有小于2^256的自然数），已知这些属性的数据类型为byte数组，则数组长度应小于32（即常量HASH_LENGTH）才能满足规则，计算方法：byte的取值范围为2^8，属性上限值为(2^8)^32。
    if (receiveAddress != null && receiveAddress.length != 0 && receiveAddress.length != ADDRESS_LENGTH)//校验规则2：属性receiveAddress，数据类型为字节数组，属性值的数组长度可以为0（创建合约交易）或20（消息调用交易）。对应黄皮书的18公式
        throw new RuntimeException("Receive address is not valid");
    if (gasLimit.length > HASH_LENGTH)
        throw new RuntimeException("Gas Limit is not valid");
    if (gasPrice != null && gasPrice.length > HASH_LENGTH)
        throw new RuntimeException("Gas Price is not valid");
    if (value != null  && value.length > HASH_LENGTH)
        throw new RuntimeException("Value is not valid");
    if (getSignature() != null) {
        if (BigIntegers.asUnsignedByteArray(signature.r).length > HASH_LENGTH)
            throw new RuntimeException("Signature R is not valid");
        if (BigIntegers.asUnsignedByteArray(signature.s).length > HASH_LENGTH)
            throw new RuntimeException("Signature S is not valid");
        if (getSender() != null && getSender().length != ADDRESS_LENGTH)
            throw new RuntimeException("Sender is not valid");
    }
}
```

#### 校验交易费用和账户余额

根据以太坊协议（如Gas收费标准）、区块、账户状态等执行环境信息，对交易的nonce、gasLimit，以及交易签名进行检查；并对交易发送方账户的余额进行检查。

* 交易gasLimit：校验交易的固定基础费用是否超过限制

```java
//TransactionExecutor.java:L150，init函数的部分代码
if (txGasLimit.compareTo(BigInteger.valueOf(basicTxCost)) < 0) ...  //校验交易的固定基础费用是否超过限制（交易gasLimit），对应黄皮书的58公式的g0校验逻辑
```

计算交易的固定基础费用：交易创建费用（合约创建另收费）、交易附言费用

```java
//TransactionExecutor.java:L132，init函数的部分代码
basicTxCost = tx.transactionCost(config.getBlockchainConfig(), currentBlock);//计算交易的固定基础费用，对应黄皮书的54、55、56公式

//Transaction.java:L180
public long transactionCost(BlockchainNetConfig config, Block block){//行180-186：transactionCost，函数，用于计算交易的基础费用。需要根据当前区块高度获取对应的区块链配置，不同配置对应的费用计算规则可能有所不同。

    rlpParse();

    return config.getConfigForBlock(block.getNumber()).
            getTransactionCost(this);
}
```

交易创建费用：所有交易的基础费用为21000，合约创建费用为32000。所以，合约创建类交易为53000（21000 + 32000），消息调用类交易为21000。\
交易附言费用：0值字节的单价4，非0值字节的单价68。

```java
//HomesteadConfig.java:L60
public long getTransactionCost(Transaction tx) {
    long nonZeroes = tx.nonZeroDataBytes();
    long zeroVals  = ArrayUtils.getLength(tx.getData()) - nonZeroes;

    return (tx.isContractCreation() ? getGasCost().getTRANSACTION_CREATE_CONTRACT() : getGasCost().getTRANSACTION())//计算交易创建费用，合约创建类交易53000（21000 + 32000），消息调用类交易21000，对应黄皮书的55和56公式
            + zeroVals * getGasCost().getTX_ZERO_DATA() + nonZeroes * getGasCost().getTX_NO_ZERO_DATA();//计算交易附带内容费用，0值字节的单价4，非0值字节的单价68，对应黄皮书的54公式
}
```

_上述代码中，计算交易创建费用逻辑，对应黄皮书的55和56公式；计算交易附带内容费用逻辑，对应黄皮书的54公式。_

* 交易gasLimit：校验区块中交易已消耗的gas是否超过限制（区块gasLimit）

```java
//TransactionExecutor.java:L139，init函数的部分代码
BigInteger txGasLimit = new BigInteger(1, tx.getGasLimit());
BigInteger curBlockGasLimit = new BigInteger(1, currentBlock.getGasLimit());

boolean cumulativeGasReached = txGasLimit.add(BigInteger.valueOf(gasUsedInTheBlock)).compareTo(curBlockGasLimit) > 0;
if (cumulativeGasReached) ... //校验区块中交易已消耗的gas是否超过限制（区块gasLimit），对应黄皮书的58公式的Tg校验逻辑
```

* 交易nonce：根据世界状态树存储的发送方账户的nonce信息，校验交易nonce是否一致

```java
//TransactionExecutor.java:L157，init函数的部分代码
BigInteger reqNonce = track.getNonce(tx.getSender());
BigInteger txNonce = toBI(tx.getNonce());
if (isNotEqual(reqNonce, txNonce)) ... //根据世界状态树存储的发送方账户的nonce信息，校验交易nonce是否一致，对应黄皮书的58公式的Tn校验逻辑
```

* 交易签名：校验交易签名的正确性，以及重放攻击（双花问题）检测。

```java
//TransactionExecutor.java:L176，init函数的部分代码
if (!blockchainConfig.acceptTransactionSignature(tx)) ... //校验交易签名的正确性，对应黄皮书的58公式的S(T)部分逻辑

//Eip160HFConfig.java:L77
public boolean acceptTransactionSignature(Transaction tx) {
    if (tx.getSignature() == null) return false;
    // Restoring old logic. Making this through inheritance stinks too much
    if (!tx.getSignature().validateComponents() ||
            tx.getSignature().s.compareTo(SECP256K1N_HALF) > 0) return false;
    return  tx.getChainId() == null || Objects.equals(getChainId(), tx.getChainId());
}
```

_在EIP-155（简单的重放攻击防护）提案中，要求交易签名中除了针对交易内容外，还包扩以太坊网络标识（CHAIN\_ID）信息。从而，避免攻击者将当前网络中产生的已签名交易，在其他网络中再次发送和执行，达到同一笔交易多次执行的非法目的。这也提示我们，在搭建以太坊网络时，应该通过修改参数配置尽量避免网络标识重复，否则EIP-155的保护作用就会降低。以太坊网络标识列表：_

| CHAIN\_ID | Chain(s)                          |
| --------- | --------------------------------- |
| 1         | Ethereum mainnet                  |
| 2         | Morden (disused), Expanse mainnet |
| 3         | Ropsten                           |
| 4         | Rinkeby                           |
| 5         | Goerli                            |
| 42        | Kovan                             |
| 1337      | Geth private chains (default)     |

{% hint style="info" %}
详见EIP-155：https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md
{% endhint %}

* 发送方账户余额：根据世界状态树中的账户状态，校验发送方账户的余额是否足够，以支付交易固定基础费用和转账金额

```java
//TransactionExecutor.java:L165，init函数的部分代码
BigInteger txGasCost = toBI(tx.getGasPrice()).multiply(txGasLimit);
BigInteger totalCost = toBI(tx.getValue()).add(txGasCost);//计算交易的预支付费用，对应黄皮书的57公式
BigInteger senderBalance = track.getBalance(tx.getSender());

if (!isCovers(senderBalance, totalCost)) ... //校验发送方账户的余额是否充足，对应黄皮书的58公式的v0校验逻辑
```

### 合约创建

在以太坊中创建一个智能合约，主要途径是通过外部账户发起一笔合约创建交易，交易的主要内容是合约初始化代码；也可以在消息调用交易中，通过合约的代码逻辑来创建新合约。 相较于发起一笔消息调用交易给他人转账时，我们要知道对方的账户地址，一笔合约创建交易应该如何填写接收方账户呢？其实我们不需要担心这个问题，因为合约创建交易不需要我们填写接收方账户，而且以太坊会自动创建一个新合约账户的地址。

#### 账户状态初始化

* 计算新合约账户地址

根据交易发送账户的地址和nonce属性，计算得到一个20字节的账户地址。

```java
//TransactionExecutor.java:L260，create函数的部分代码
byte[] newContractAddress = tx.getContractAddress();////计算新合约地址，对应黄皮书的77公式

//Transaction.java:L361，获取新创建的合约账户地址
public byte[] getContractAddress() {           //行361-364：getContractAddress，功能函数，获取新创建的合约账户地址（仅合约创建交易有效）。合约账户地址根据交易发送账户的地址和nonce属性计算得出，详见HashUtil.java解读中的calcNewAddr函数。
    if (!isContractCreation()) return null;    //此函数被TransactionExecutor.java、ProgramInvokeFactoryImpl.java等源文件引用。
    return HashUtil.calcNewAddr(this.getSender(), this.getNonce());
}
```

先将交易发送账户的地址和nonce属性进行RLP编码之后，再进行Keccak-256哈希计算，最后取哈希值的最右边20字节（160位）。

```java
//HashUtil.java:L177
public static byte[] calcNewAddr(byte[] addr, byte[] nonce) {//根据合约创建者账户的地址及nonce，计算新合约地址，对应黄皮书的77公式

    byte[] encSender = RLP.encodeElement(addr);
    byte[] encNonce = RLP.encodeBigInteger(new BigInteger(1, nonce));

    return sha3omit12(RLP.encodeList(encSender, encNonce));
}
```

* 账户状态:

新合约账户的交易数量(nonce)被初始定义为1，余额(balance)为交易转账金额，合约存储(storage)为空，合约代码哈希(codeHash)为空字符串的Keccak-256哈希值。同时，交易发送方账户的余额也会扣减转账金额。合约创建交易中的转账金额，也被称为新合约账户的初始捐款(endowment)。

首先，获取新账户在交易之前就有可能拥有的余额。

```java
//TransactionExecutor.java:L270，create函数的部分代码
BigInteger oldBalance = track.getBalance(newContractAddress);//获取新账户在交易之前就有的余额（可以在创世区块中赋予），对应黄皮书的82公式
```

_思考：这可能听上去有些不可思议，这里本来就是要创建一个合约账户，为什么在此之前就可以有余额了呢？_\
_因为创世区块，在以太坊网络的创建阶段，我们通过创世区块中写入初始状态，可以为一些账户地址赋予余额等其他状态。一般情况下，这些账户地址对应的是外部账户，我们可以事先生成外部账户的密钥和地址；但对于合约账户，我们可以通过外部账户地址和预设的nonce，计算出合约账户地址。_\
_虽然此时可能还没有编写合约代码，但是这并不影响通过创世区块为一个合约账户地址事先赋予余额状态。后续，在发送合约创建交易时，也就要求发送方账户的nonce与事先规划一致，从而获取这些预分配金额。_

然后，创建新合约状态：余额等于转账金额加已有余额；在EIP-161（状态树清理）提案中，约定合约账户nonce的初始值为1。并且修改发送方账户状态：余额中扣减转账金额。

```java
//TransactionExecutor.java:L271，create函数的部分代码
cacheTrack.createAccount(tx.getContractAddress());
cacheTrack.addBalance(newContractAddress, oldBalance);//合约初始捐款金额，对应黄皮书79公式的部分逻辑（已有余额v′）
if (blockchainConfig.eip161()) {
    cacheTrack.increaseNonce(newContractAddress);//合约账户的初始nonce为1
}

//TransactionExecutor.java:L295，create函数的部分代码
BigInteger endowment = toBI(tx.getValue());
transfer(cacheTrack, tx.getSender(), newContractAddress, endowment);//合约创建交易的发送方账户向新合约账户转账，对应黄皮书的80、81公式，以及79公式的部分逻辑（转账金额v）
```

#### 合约初始化（代码）执行

可用gas扣除固定基础费用后，通过EVM（详见VM.java的解读）执行交易中的合约初始化逻辑，获取合约代码。

```java
//TransactionExecutor.java:L281，create函数的部分代码
ProgramInvoke programInvoke = programInvokeFactory.createProgramInvoke(tx, currentBlock, cacheTrack, blockStore);

this.vm = new VM(config);
this.program = new Program(tx.getData(), programInvoke, tx, config).withCommonConfig(commonConfig);//构建合约创建交易执行模型，对应黄皮书的83公式（等号右侧部分）。其中，第一个参数为合约创建交易的Ib（所要执行的机器代码字节数组），取交易附带数据（合约初始化的EVM字节码），对应黄皮书的90公式。

//TransactionExecutor.java:L309，go函数的部分代码
program.spendGas(tx.transactionCost(config.getBlockchainConfig(), currentBlock), "TRANSACTION COST");//可用gas扣除固定基础费用，对应黄皮书的63公式

//TransactionExecutor.java:L312，go函数的部分代码
vm.play(program);//调用代码执行函数，对应黄皮书的83、121、123公式，119公式的otherwise部分

result = program.getResult();//获取交易执行后的累积子状态
m_endGas = toBI(tx.getGasLimit()).subtract(toBI(program.getResult().getGasUsed()));//计算交易执行后的剩余可用gas数量
```

#### 合约代码存储

计算合约代码的存储费用（与合约初始化代码长度成正比）,并在可用的剩余gas减去合约代码的存储费用，然后将合约代码存储到世界状态树中对应的账户状态。

```java
//TransactionExecutor.java:L318
int returnDataGasValue = getLength(program.getResult().getHReturn()) *//计算合约代码的存储费用（与合约初始化代码长度成正比），对应黄皮书的93公式
        blockchainConfig.getGasCost().getCREATE_DATA();
        
//TransactionExecutor.java:L336
m_endGas = m_endGas.subtract(BigInteger.valueOf(returnDataGasValue));//可用的剩余gas减去合约代码的存储费用，对应黄皮书的94公式的otherwise部分逻辑
cacheTrack.saveCode(tx.getContractAddress(), result.getHReturn());//存储合约账户代码，对应黄皮书的95公式的otherwise部分逻辑逻辑
```

### 消息调用

以太坊协议中的消息调用，一般是指通过外部账户发起消息调用交易，来调用合约的代码逻辑；或者在调用合约的过程中，又通过消息调用指令来触发其他合约代码的执行。关于合约账户，除了通过合约创建章节中的方式来创建合约，以太坊还自带了预编译合约。\
_**注意：在实际应用中，以太坊中很多交易只是完成外部账户之间的转账和附言，根本不涉及智能合约，自然也没有代码要执行，但笔者也把它们归为消息调用交易的范畴。**_

#### 执行前的状态修改

根据交易指定的转账金额，由发送方账户向接收方账户转账。

```java
//TransactionExecutor.java:L253
BigInteger endowment = toBI(tx.getValue());
transfer(cacheTrack, tx.getSender(), targetAddress, endowment);//发送方账户向接收方账户转账，对应黄皮书的99、100、101、102、103、104、105公式
```

#### 预编译合约

根据消息调用的接收方账户地址，查找对应的预编译合约（如果有）。

```java
//TransactionExecutor.java:L209
byte[] targetAddress = tx.getReceiveAddress();
precompiledContract = PrecompiledContracts.getContractForAddress(new DataWord(targetAddress), blockchainConfig);//根据消息调用的接收方账户地址，查找对应的预编译合约（如果有）

//PrecompiledContracts.java:L58
public static PrecompiledContract getContractForAddress(DataWord address, BlockchainConfig config) {//根据指定地址获取对应的预编译合约，对应黄皮书的119公式
    if (address == null) return identity;
    if (address.equals(ecRecoverAddr)) return ecRecover;
    if (address.equals(sha256Addr)) return sha256;
    if (address.equals(ripempd160Addr)) return ripempd160;
    if (address.equals(identityAddr)) return identity;
    // Byzantium precompiles
    if (address.equals(modExpAddr) && config.eip198()) return modExp;
    if (address.equals(altBN128AddAddr) && config.eip213()) return altBN128Add;
    if (address.equals(altBN128MulAddr) && config.eip213()) return altBN128Mul;
    if (address.equals(altBN128PairingAddr) && config.eip212()) return altBN128Pairing;
    return null;
}
```

> 这是8个所谓的“预编译”合约，它们作为最初架构中的一部分，后续可能会变成原生扩展。合约账户地址从1到8，分别是椭圆曲线公钥恢复函数、SHA2 256 位哈希方案、RIPEMD 160位哈希方案、标识函数、任意精度的模幂运算、椭圆曲线加法、椭圆曲线纯量乘法和椭圆曲线配对检查。\
> 详见以太坊黄皮书附录E：Precompiled Contracts

以太坊黄皮书中提到的“后续可能会变成原生扩展”，在Java版以太坊中就以原生的方式，通过Java语言实现的这些预编译智能合约，而不是通过EVM字节码。

```java
//PrecompiledContracts.java:L40，对应以太坊黄皮书附录E
private static final ECRecover ecRecover = new ECRecover();//椭圆曲线公钥恢复函数(elliptic curve public key recovery function)
private static final Sha256 sha256 = new Sha256();//SHA2 256位哈希方案(SHA2 256-bit hash scheme)
private static final Ripempd160 ripempd160 = new Ripempd160();//RIPEMD 160位哈希方案(RIPEMD 160-bit hash scheme)
private static final Identity identity = new Identity();//标识函数(identity function)
private static final ModExp modExp = new ModExp();//任意精度的模幂运算(ar-bitrary precision modular exponentiation)
private static final BN128Addition altBN128Add = new BN128Addition();//椭圆曲线加法(elliptic curve addition)
private static final BN128Multiplication altBN128Mul = new BN128Multiplication();//椭圆曲线标量乘法(elliptic curve scalar multiplication)
private static final BN128Pairing altBN128Pairing = new BN128Pairing(); //椭圆曲线配对检查(elliptic curve pairing check respectively)

private static final DataWord ecRecoverAddr =       new DataWord("0000000000000000000000000000000000000000000000000000000000000001");
private static final DataWord sha256Addr =          new DataWord("0000000000000000000000000000000000000000000000000000000000000002");
private static final DataWord ripempd160Addr =      new DataWord("0000000000000000000000000000000000000000000000000000000000000003");
private static final DataWord identityAddr =        new DataWord("0000000000000000000000000000000000000000000000000000000000000004");
private static final DataWord modExpAddr =          new DataWord("0000000000000000000000000000000000000000000000000000000000000005");
private static final DataWord altBN128AddAddr =     new DataWord("0000000000000000000000000000000000000000000000000000000000000006");
private static final DataWord altBN128MulAddr =     new DataWord("0000000000000000000000000000000000000000000000000000000000000007");
private static final DataWord altBN128PairingAddr = new DataWord("0000000000000000000000000000000000000000000000000000000000000008");
```

但是，这些Java版以太坊中原生支持的预编译合约，是通过JVM执行的；而其他合约则以EVM字节码的形式通过EVM执行。

```java
//TransactionExecutor.java:L229，call函数的部分代码
Pair<Boolean, byte[]> out = precompiledContract.execute(tx.getData());
```

#### 消息调用代码执行

根据接收方账户地址获取合约代码，即消息调用交易的所要执行的EVM字节码，然后通过EVM执行。

```java
//TransactionExecutor.java:L240，call函数的部分代码
byte[] code = track.getCode(targetAddress);//根据接收方账户地址获取合约代码，作为消息调用交易的Ib（所要执行的机器代码字节数组），对应黄皮书的120公式

//TransactionExecutor.java:L245，call函数的部分代码
ProgramInvoke programInvoke =
        programInvokeFactory.createProgramInvoke(tx, currentBlock, cacheTrack, blockStore);
        
this.vm = new VM(config);
this.program = new Program(track.getCodeHash(targetAddress), code, programInvoke, tx, config).withCommonConfig(commonConfig);//构建消息调用交易执行模型，对应黄皮书的108公式（等号右侧部分）

//TransactionExecutor.java:L309，go函数的部分代码
program.spendGas(tx.transactionCost(config.getBlockchainConfig(), currentBlock), "TRANSACTION COST");//可用gas扣除固定基础费用，对应黄皮书的63公式

//TransactionExecutor.java:L312，go函数的部分代码
vm.play(program);//调用代码执行函数，对应黄皮书的83、121、123公式，119公式的otherwise部分
```

### 状态转换

#### 交易费用结算

* Gas预支付

在EVM执行代码之前，已经按照交易指定的Gas数量上限和Gas单价，预先在交易发起方账户中扣除相应金额。

```java
BigInteger txGasLimit = toBI(tx.getGasLimit());
BigInteger txGasCost = toBI(tx.getGasPrice()).multiply(txGasLimit);
track.addBalance(tx.getSender(), txGasCost.negate());//修改发送方账户余额状态：按照交易指定的gasLimit和gasPrice扣除费用，对应黄皮书的59、60公式
```

* 剩余Gas返还至交易发起方

通过EVM执行交易指定的代码之后，获取EVM机器状态中的剩余可用Gas数量，以及交易子状态中的返还Gas数量（即费用补贴，且不可超过已用Gas数量的50%），两者数量相加即要返还的Gas总量，根据交易指定的Gas单价将相应金额返还至交易发起方账户。

```java
//TransactionExecutor.java:L314，go函数的部分代码
result = program.getResult();//获取交易执行后的累积子状态
m_endGas = toBI(tx.getGasLimit()).subtract(toBI(program.getResult().getGasUsed()));//计算交易执行后的剩余可用gas数量

//TransactionExecutor.java:L399，finalization函数的部分代码
long gasRefund = Math.min(result.getFutureRefund(), getGasUsed() / 2); //计算有效的gas返还数量：根据费用补贴和交易已用Gas，计算费用返还金额(refund)，对应黄皮书的65公式

//TransactionExecutor.java:L427
track.addBalance(tx.getSender(), summary.getLeftover().add(summary.getRefund()));//返还剩余Gas，增加发送者账户余额：剩余Gas、费用返还金额(refund)，对应黄皮书的67公式
```

* 已用Gas结算至矿工

根据交易Gas单价和扣除补贴后的已用Gas数量，计算交易费用并发放至矿工账户。

```java
//TransactionExecutor.java:L431
track.addBalance(coinbase, summary.getFee());//矿工收取交易费，增加矿工账户余额

//TransactionExecutionSummary.java:L243
public BigInteger getFee() {
    if (!parsed) rlpParse();
    return calcCost(gasLimit.subtract(gasLeftover.add(gasRefund)));//计算交易费：gasUsed减去refund金额，即自销毁合约账户的返还金额用于抵扣交易gas费用，最多可以抵扣gas费用的50%，对应黄皮书的68公式
}
```

#### 世界状态树清理

* 删除自销毁账户

根据交易执行后子状态中的自销毁账户集合，在世界状态树中删除账户地址对应的状态数据。

```java
//TransactionExecutor.java:L438
for (DataWord address : result.getDeleteAccounts()) {//删除自销毁合约账户的状态，对应黄皮书的71公式
    track.delete(address.getLast20Bytes());
}
```

* 删除空账户

启用EIP-161（状态树清理）提案后，根据交易子状态中的接触账户集合，将其中的空账户状态在世界状态树中清除。

```java
if (blockchainConfig.eip161()) {//删除交易接触账户列表中的空账户的状态，对应黄皮书的72公式和EIP-161
    for (byte[] acctAddr : touchedAccounts) {
        AccountState state = track.getAccountState(acctAddr);
        if (state != null && state.isEmpty()) {
            track.delete(acctAddr);
        }
    }
}
```

### 交易收据

根据费用结算、交易子状态、交易执行异常等信息，生成交易收据：

* Gas使用量：结算给矿工账户的Gas数量，即补贴后的已用Gas数量。
* 日志集合：交易子状态中的日志集合。
* 日志Bloom过滤器：根据日志集合生成的Bloom过滤器，可通过日志产生地址和日志主题进行检索，用于查找日志信息。
* 交易状态码：通过EVM执行交易相关代码时，是否出现异常情况导致代码执行失败的标识，`0`代表失败，`1`代表成功。
