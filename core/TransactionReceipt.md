# 交易收据
每笔交易执行后，会根据执行过程中的一些关键信息生成交易收据。
这些信息包括交易执行产生的日志，区块内交易执行所消耗Gas累计量，交易执行是否成功的状态码等信息。

通过交易收据，可以了解交易执行过程是否符合以太坊协议（如Gas收费标准），可以通过日志观察交易执行细节（如用于合约开发过程中的故障诊断），也可以为智能合约的上层应用的开发提供便利（如查看交易状态、订阅合约事件）。

**交易收据的属性列表：**

|  属性名 | 标识符 | 中文名 | 释义 |
|  ----  | ----  | ----  | ---- |
| gasUsed  | R<sub>u</sub>  | 累计Gas使用量 | 区块内交易消耗的Gas数量之和 |
| bloomFilter  | R<sub>b</sub>  | 日志Bloom过滤器 | 用于日志信息检索 |
| logInfoList  | R<sub>l</sub>  | 交易日志集合 | 交易执行过程中创建日志的集合 |
| postTxState  | R<sub>z</sub>  | 交易状态码 | 1表示成功，0表示失败；EIP-658提案中加入的属性 |

## 累计Gas使用量
当前交易执行后，区块内交易消耗的Gas数量之和，取值范围是正整数。
```java
//TransactionReceipt.java:L50
private byte[] cumulativeGas = EMPTY_BYTE_ARRAY;//Ru
```
在交易执行过后，通过当前交易gas上限减去剩余可用gas，计算当前交易的gas使用量，然后区块内其他已执行交易的gas使用累计值相加，计算出累计Gas使用量并赋值给交易收据。
```java
//TransactionExecutor.java:L482，getReceipt函数的部分代码
long totalGasUsed = gasUsedInTheBlock + getGasUsed();
receipt.setCumulativeGas(totalGasUsed);//赋值交易收据的累计Gas使用量，对应黄皮书的171公式

//TransactionExecutor.java:L502
public long getGasUsed() {
    return toBI(tx.getGasLimit()).subtract(m_endGas).longValue();
}
```

## 交易日志集合
交易执行过程中产生的一系列的日志（详见LogInfo.java的解读）。这些日志的内容是在合约代码中定义的，然后在EVM执行这些代码时产生。
```java
//TransactionReceipt.java:L52
private List<LogInfo> logInfoList = new ArrayList<>();//交易执行过程中创建的日志集合(Rl)
```
交易执行后，在交易子状态（详见ProgramResult.java的解读）中获取日志集合，并赋值给交易收据。
```java
//TransactionExecutor.java:L436，finalization函数的部分代码
logs = result.getLogInfoList();//获取日志集合，用于构建交易收据

//TransactionExecutor.java:L485，getReceipt函数的部分代码
receipt.setLogInfoList(getVMLogs());//赋值交易收据的日志集合，对应黄皮书的172公式

//TransactionExecutor.java:L494
public List<LogInfo> getVMLogs() {
    return logs;
}
```

## 日志Bloom过滤器
根据交易日志集合生成的Bloom过滤器，内容是一段256字节（2048位）的数据，可以用于判断日志集合中是否包含某些账户地址或某些主题的日志信息。
```java
//TransactionReceipt.java:L51
private Bloom bloomFilter = new Bloom();//日志信息构成的Bloom过滤器(Rb)
```
根据交易日志集合生成的日志Bloom过滤器，多个Bloom可以通过`或位运算 (|)`进行合并，详见Bloom.java的解读。
```java
//TransactionReceipt.java:L256，setLogInfoList函数的部分代码
for (LogInfo loginfo : logInfoList) {
    bloomFilter.or(loginfo.getBloom());
}
```

## 交易状态码
交易状态码是一个整数，1表示成功，0表示失败。交易状态码是在EIP-658提案中加入交易收据的属性，用来判断交易执行是否成功。
```java
//TransactionReceipt.java:L49
private byte[] postTxState = EMPTY_BYTE_ARRAY;//交易状态码(Rz)，取字节数组的第一个元素作为交易状态

//TransactionReceipt.java:L213
public void setTxStatus(boolean success) {
    this.postTxState = success ? new byte[]{1} : new byte[0];
    rlpEncoded = null;
}
```
执行中如果发生异常导致程序退出，则状态为0；正常执行完成则为1。
```java
//BlockchainImpl.java:L892，applyBlock函数的部分代码
if (blockchainConfig.eip658()) {
    receipt.setTxStatus(receipt.isSuccessful());//设置交易状态（根据交易收据的error属性判断），对应黄皮书的173公式
} ...
```

## 交易收据序列化
交易收据的RLP序列化的属性顺序：R<sub>z</sub>, R<sub>u</sub>, R<sub>b</sub>, R<sub>l</sub>。在交易收据MPT的生成阶段，会进行交易收据的序列化操作。
```java
//TransactionReceipt.java:L201
RLP.encodeList(postTxStateRLP, cumulativeGasRLP, bloomRLP, logInfoListRLP) ... //交易收据的RLP序列化，对应黄皮书的21公式
```
