# 基本定义

## 指令集

目前，EVM提供了140个指令。其中，以太坊黄皮书中定义了135个，包含特定的无效指令`INVALID`；以太坊改进提案（Ethereum Improvement Proposals，下文中简称EIPs）中定义了5个。

### 指令定义

每个指令的定义信息包括：

* **字节码** ：或称指令编号，数值范围0x00 - 0xff。
* **助记符** ：或称指令名称
* **出栈数量**
* **入栈数量**
* **指令含义** ：指令的基本描述、Gas费用、运算逻辑，以及机器状态（栈、内存）的操作

```java
//OpCode.java:L618
private final byte opcode;//字节码
private final int require;//出栈数量
private final Tier tier;//gas费用级别
private final int ret;//入栈数量
private final EnumSet<CallFlags> callFlags;//消息调用类型集合，用于标注CALL类指令，如是否静态调用

//OpCode.java:L636
private OpCode(int op, int require, int ret, Tier tier, CallFlags ... callFlags) ...
```

#### 指定定义示例

下面以加法算术指令`ADD`指令为例，其指令定义为：

* **字节码：** `0x01`
* **出栈数量：** `2`，即`ADD`指令参数，被加数与加数
* **入栈数量：** `1`，即`ADD`指令运算结果，被加数与加数之和
* **Gas费用级别：** `VeryLowTier`，对应gas数量为`3`

```java
//OpCode.java:L45
ADD(0x01, 2, 1, VeryLowTier),
```

### 指令类别

以太坊黄皮书中将指令划分为11个类别，且每类指令的编号的数值范围不同。比如，停止和算术运算类指令编号的数组区间为0x00 - 0x0f（表示为0s）。

**指令类别**及其部分指令：

* 0s：停止和算术运算
  * 停止指令：`STOP`
  * 算术类（部分）：`ADD`（加）、`SUB`（减）、`MUL`（乘）、`DIV`（除）、`MOD`（取模）、`EXP`（指数）
* 10s：比较和按位逻辑运算
  * 比较逻辑运算（部分）：`LT`（小于）、`GT`（大于）、`EQ`（等于）、`ISZERO`（否，结合EQ指令实现不等于）
  * 按位逻辑运算：`AND`（与）、`OR`（或）、`NOT`（非）、`XOR`（异或）
* 20s：SHA3运算，`SHA3`（Keccak-256哈希）
* 30s：环境信息（部分），`ADDRESS`（获取Ia）、`GASPRICE`（获取Ip）、`BALANCE`（获取给定账户余额）
* 40s：区块信息
  * `BLOCKHASH`：获取指定高度的祖先区块的头部哈希
  * `COINBASE`：获取该区块的矿工账户地址
  * `TIMESTAMP`：获取该区块的时间戳
  * `NUMBER`：获取区块的号码
  * `DIFFICULTY`：获得区块的难度
  * `GASLIMIT`：获得区块的gas上限
* 50s：栈，内存，存储和流程操作
  * 栈：`POP`（出栈）
  * 内存：`MLOAD`（从内存中加载字）、`MSTORE`（将字保存到内存）、`MSTORE8`（将字节保存到内存）、`MSIZE`（获取活动内存的大小）
  * 存储：`SLOAD`（从存储加载字）、`SSTORE`（将字保存到存储）
  * 流程控制：`JUMP`（跳转）、`JUMPI`（有条件的跳转）、`JUMPDEST`（μpc加1）、`PC`（获取μpc）、`GAS`（获取μg）
* 60s & 70s: 入栈（Push）操作，`PUSH1` - `PUSH32`
* 80s: 复制（Duplicate）操作，`DUP1` - `DUP16`
* 90s: 替换（Exchange）操作，`SWAP1` - `SWAP16`
* a0s: 日志操作，`LOG0` - `LOG4`
* f0s: 系统操作（部分）
  * `CREATE`：创建合约
  * `CALL`：消息调用合约
  * `CALLCODE`：使用指定帐户的代码对当前帐户进行消息调用
  * `RETURN`：停止执行，返回输出数据
  * `DELEGATECALL`：使用指定帐户的代码对当前帐户进行消息调用，但保留sender和value属性现有的值。
  * `INVALID`：保留的无效指令
  * `SELFDESTRUCT`：销毁合约

此外，还在EIPs中定义了如下指令：

* EIP-145中定义了3个按位逻辑运算（10s）
  * `SHL`：0x1b，左移 (shift left)
  * `SHR`：0x1c，逻辑右移 (logical shift right)
  * `SAR`：0x1d，算术右移 (arithmetic shift right)
* EIP-1014中定义了1个系统操作指令（f0s）
  * `CREATE2`：0xf5，创建指定地址的合约账户
* EIP-1052中定义了1个环境信息获取指令（30s）
  * `EXTCODEHASH`：0x3f，获取合同代码的keccak256哈希

## 执行环境

执行环境数据，表示为`I`，包含属性:

* `Ia`，拥有正在执行的代码的账户地址，一般是指交易的发送方账户_`S(T)`_，在合约调用合约的情况下则不同。
* `Io`，触发这次执行的初始交易的发送者地址，初始交易是指由外部账户发起的交易_`S(T)`_。
* `Ip` ，触发这次执行的初始交易的 gas 价格。
* `Id` ，这次执行的输入数据字节数组；如果执行代理是一个交易，这就是交易数据。
* `Is`，触发这次执行的账户地址；如果执行代理是一个交易，则为交易发送者地址。
* `Iv` ，作为这次执行的一部分传到当前账户的转账金额，以 Wei为单位；如果执行代理是一个交易, 这就是交易的转账金额。
* `Ib`，所要执行的EVM字节码数组。
* `IH`，当前区块的区块头。
* `Ie`，当前消息调用或合约创建的深度（也就是当前已经被执行的 `CALL` 或 `CREATE` 的数量）。
* `Iw`，修改状态的许可。

> 这里我们来回顾一下**交易数据与待执行代码的对照关系：**
>
> * 消息调用交易
>   * `Ib`：即交易的被调用合约`Tt`，存储在世界状态树中的合约代码`σ[Tt]c`
>   * `Id`：交易的消息调用输入数据`Td`，存储在交易附言(Transaction.data)属性。
> * 合约创建交易
>   * `Ib`：即交易的合约初始化代码`Ti`，也存储在交易附言(Transaction.data)属性。
>   * `Id`：此类交易无输入数据

## EVM状态

EVM状态，也称机器状态（Machine State），表示为`μ`。

* 可用Gas

可用gas，表示为`μg`，根据交易gas上限和已用gas数量计算：gasLimit - gasUsed。

```java
//ProgramResult.java:L40
private long gasUsed;//已用gas数量

//Program.java:L745，计算剩余可用gas，对应黄皮书的125公式
public long getGasLong() {
    return invoke.getGasLong() - getResult().getGasUsed();
}
```

* 程序计数器

程序计数器，表示为`μpc`，初始值为0。

```java
//Program.java:L98
private int pc; //程序计数器，初始值为0，对应黄皮书的126公式
```

* 内存

内存，表示为`μm`，内存模型是基于字寻址(word-addressed)的字节数组，所有内存中的数据都会初始化为0。

```java
//Program.java:L89
private Memory memory;

//Memory.java:L33
public class Memory ... {//内存模型是简单地基于字寻址(word-addressed) 的字节数组

    private static final int CHUNK_SIZE = 1024;//内存块大小，内存按块进行扩展(1 chunk = 32 word = 1024 byte)
    private static final int WORD_SIZE = 32;//字的大小(1 word = 32 byte)，对应黄皮书的9.1章节中的约定

    private List<byte[]> chunks = new LinkedList<>();//内存块数据列表
    ...
}
```

* 内存已激活字数

内存已激活字数，表示为μi，初始值为0。

```java
//Memory.java:L39
private int softSize; //μi，内存已激活的字数，初始值为0，对应黄皮书
```

* 栈

栈，表示为`μs`，其中栈中数据项的大小（即字长）是256位，最大深度为1024，初始值为空序列。

```java
//Program.java:L88
private Stack stack;

//Stack.java:L24
public class Stack extends java.util.Stack<DataWord> ... //栈的字大小为256位，对应黄皮书9.1章节的约定
```

* 结果数据

主要是指消息调用指令的结果输出数据，表示为`μo`，初始值为空序列。

```java
//Program.java:L91
private byte[] returnDataBuffer;//μo，机器状态中的指令输出，对应黄皮书的130公式
```

## 交易子状态

交易执行过程中会产生一些特定的信息，记录交易相关的账户、日志等内容，也称为交易子状态或累积子状态。

* 自销毁账户集合

合约代码执行过程中，通过`SELFDESTRUCT`指令销毁账户的地址的集合，表示为`As`，初始值为空集合。

```java
//ProgramResult.java:L45
private Set<DataWord> deleteAccounts;//自毁账户集合(As)：一组应该在交易完成后被删除的账户
```

* 接触账户集合

交易执行过程中所接触账户的地址的集合，表示为`At`，初始值为空集合。此集合的特征是交易执行过程中账户状态发生过变化，包含新创建的合约账户、消息调用接收方账户、销毁账户余额的继承账户等，交易执行完成时将删除其中的空账户。

```java
//ProgramResult.java:L46
private ByteArraySet touchedAccounts = new ByteArraySet(); //接触账户集合(At)，其中的空账户可以在交易结束时删除
```

* 日志集合

合约代码执行中，输出的一系列带有主题和数据内容的日志，并且记录了输出日志的账户地址，表示为`Al`，初始值为空。其中，日志的所属账户地址和主题可以作为检索条件，通过区块头部和交易收据中的Bloom过滤器，实现区块、交易二级索引的信息查询。 合约开发者通过日志，可以获取合约执行的详细状态，方便合约的上层应用实现一些类似于异步通知或回调功能，为应用最终用户提供更加友好的体验。

```java
//ProgramResult.java:L48
private List<LogInfo> logInfoList;//日志集合(Al)：这是一些列针对VM代码执行的归档的、可索引的‘检查点’，允许以太坊世界的外部旁观者(例如去中心化应用的前端)来简单地跟踪合约调用。
```

* 返还Gas数量

以太坊为了鼓励合约开发者尽可能少的占用计算资源，对于释放已占用资源的操作给予一定的gas补贴以抵扣部分交易费用，如通过`SSTORE`指令释放合约存储资源，通过`SELFDESTRUCT`指令清空指定账户的状态数据。合约代码执行中，将根据约定计算返还gas数量，表示为`Ar`，初始值为0。

```java
//ProgramResult.java:L49
private long futureRefund = 0;//返还gas数量(Ar)：与SSTORE、SUICIDE（黄皮书中为SELFDESTRUCT）指令相关
```
