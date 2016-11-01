---
layout: post
title:  HDFS BlockToken异常分析
date:   2016-11-01 08:30:00 +0800
categories: HDFS
---

近期HDFS出现了一次BlockToken异常，导致故障期间，某组NameNode托管的所有Block均验证失败，读写服务不可用，在此记录下故障原因和分析过程。

## HDFS BlockToken原理

### 背景

早期的HDFS只能在客户端访问NameNode时做访问授权验证，但DataNode端没有任何措施，故客户端可以在知道BlockID后，直接访问DataNode，对Block进行随意读写，带来很大安全隐患。故社区引入BlockToken的feature（[HADOOP-4359](https://issues.apache.org/jira/browse/HADOOP-4359)），客户端在访问DataNode读写Block时，需先进行Token验证，验证成功后方可进行后续操作。

### 验证流程

1. 客户端在读写文件时先访问NameNode
2. NameNode将文件对应的Block和生成的Token一并返回客户端
3. 客户端带着NameNode返回的Token去访问Block对应的DataNode
4. DataNode对Token进行验证
   1. 检查Token所带的userId、blockPoolId、blockId、是否过期、accessMode
   2. DataNode由Token计算password，检查与Token自带password（由NameNode计算）是否一致
5. Token验证成功，客户端方可进行后续操作

Token的结构如下：

~~~java
Token:
- byte[] identifier // BlockTokenIdentifier对象序列化
- byte[] password

BlockTokenIdentifier:
- long expiryDate
- int keyId
- String userId
- String blockPoolId
- long blockId
- EnumSet<AccessMode> modes
~~~

NameNode端生成Token代码调用过程：

~~~java
NameNodeRpcServer.getBlockLocations() -> BlockManager.createLocatedBlocks() -> BlockManager.setBlockToken() -> BlockTokenSecretManager.generateToken() -> BlockTokenSecretManager.createPassword()
~~~

DataNode端代验证Token代码调用过程：

~~~java
DataXeceiver.readBlock() -> DataXeceiver.checkAccess() -> BlockTokenSecretManager.checkAccess() -> BlockTokenSecretManager.retrievePassword()
~~~

### 关于BlockKey

NameNode默认每隔10小时会更新一次BlockKey（dfs.block.access.key.update.interval），并分发给所有DataNode。在NameNode端和DataNode端计算password时都需要BlockKey，故NameNode与DataNode必须保持相同的BlockKey，才能验证通过。

NameNode内部会有一个Monitor线程去定期更新BlockKey，并在DataNode汇报心跳时，通过发送DNA_ACCESSKEYUPDATE命令，告知DataNode更新BlockKey

详见代码：org.apache.hadoop.hdfs.server.blockmanagement.HeartbeatManager.Monitor

## 故障分析过程

### 故障现象

客户端所有的读写请求失败，DataNode端对每次读写请求报如下异常：

~~~
org.apache.hadoop.security.token.SecretManager$InvalidToken: Can't re-compute password for block_token_identifier (expiryDate=1477730061461, keyId=989107790, userId=zhutg, blockPoolId=BP-715213703-10.141.46.46-1418959337587, blockId=2447256509, access modes=[READ]), since the required block key (keyID=989107790) doesn't exist.
        at org.apache.hadoop.hdfs.security.token.block.BlockTokenSecretManager.retrievePassword(BlockTokenSecretManager.java:382)
        at org.apache.hadoop.hdfs.security.token.block.BlockTokenSecretManager.checkAccess(BlockTokenSecretManager.java:302)
        at org.apache.hadoop.hdfs.security.token.block.BlockPoolTokenSecretManager.checkAccess(BlockPoolTokenSecretManager.java:97)
        at org.apache.hadoop.hdfs.server.datanode.DataXceiver.checkAccess(DataXceiver.java:1183)
        at org.apache.hadoop.hdfs.server.datanode.DataXceiver.readBlock(DataXceiver.java:466)
        at org.apache.hadoop.hdfs.protocol.datatransfer.Receiver.opReadBlock(Receiver.java:111)
        at org.apache.hadoop.hdfs.protocol.datatransfer.Receiver.processOp(Receiver.java:69)
        at org.apache.hadoop.hdfs.server.datanode.DataXceiver.run(DataXceiver.java:226)
        at java.lang.Thread.run(Thread.java:745)
~~~

### 故障分析

#### DataNode端分析

通过DataNode端日志可见原因为客户端请求DataNode时所带Token的BlockKey在DataNode端不存在所致，疑似NameNode像DataNode分发BlockKey时失败。分析代码得知，DataNode每次收到BlockKey会打印日志“DatanodeCommand action: DNA_ACCESSKEYUPDATE”，通过grep日志发现确实从某个时间点开始，有一台NameNode不再像DataNode发送DNA_ACCESSKEYUPDATE命令了。

#### NameNode端分析

分析代码得知，NameNode在每次定期更BlockKey时会打印日志“Updating block keys”，但是在相应的时间点并没有打印相关日志，却抛出了如下ClassCastException异常，可见确实由于NameNode没有生成BlockKey所致。

~~~
2016-10-29 06:00:02,008 ERROR org.apache.hadoop.hdfs.server.blockmanagement.HeartbeatManager: Exception while checking heartbeat
java.lang.ClassCastException
~~~

日志中只看到异常，但是没有看到堆栈信息，不便于我们找到具体出问题的代码。原因未Hadoop使用了commons-logging打日志，有个优化，在相同异常打印过多时，不会继续打印堆栈信息。故继续grep前几天的NameNode日志，直到找到第一次抛ClassCastException异常的地方，此处会把完整的堆栈信息打出来。

~~~
2016-10-16 00:01:11,320 ERROR org.apache.hadoop.hdfs.server.blockmanagement.HeartbeatManager: Exception while checking heartbeat
java.lang.ClassCastException: org.apache.hadoop.hdfs.server.blockmanagement.DatanodeDescriptor cannot be cast to java.lang.String
        at java.lang.String.compareTo(String.java:108)
        at java.util.TreeMap.getEntry(TreeMap.java:346)
        at java.util.TreeMap.get(TreeMap.java:273)
        at org.apache.hadoop.hdfs.server.blockmanagement.InvalidateBlocks.remove(InvalidateBlocks.java:150)
        at org.apache.hadoop.hdfs.server.blockmanagement.BlockManager.removeBlocksAssociatedTo(BlockManager.java:1043)
        at org.apache.hadoop.hdfs.server.blockmanagement.HeartbeatManager.heartbeatCheck(HeartbeatManager.java:316)
        at org.apache.hadoop.hdfs.server.blockmanagement.HeartbeatManager$Monitor.run(HeartbeatManager.java:337)
        at java.lang.Thread.run(Thread.java:745)
~~~

通过查看对应的代码，发现BlockManager.removeBlocksAssociatedTo()中调用invalidateBlocks.remove()时因传入String，但实际传入的是DatanodeDescriptor，导致报ClassCastException异常，这部分原因有待继续跟踪。

BlockManager.removeBlocksAssociatedTo()代码如下：

~~~java
/** Remove the blocks associated to the given DatanodeStorageInfo. */
  void removeBlocksAssociatedTo(final DatanodeStorageInfo storageInfo) {
    assert namesystem.hasWriteLock();
    final Iterator<? extends Block> it = storageInfo.getBlockIterator();
    DatanodeDescriptor node = storageInfo.getDatanodeDescriptor();
    while(it.hasNext()) {
      Block block = it.next();
      removeStoredBlock(block, node);
      // 此处实际传入storageInfo.getStorageID()为DatanodeDescriptor导致报ClassCastException
      invalidateBlocks.remove(storageInfo.getStorageID(), block);
    }
    namesystem.checkSafeMode();
  }
~~~

部分DatanodeStorageInfo类代码：

~~~java
class DatanodeStorageInfo {
  private final DatanodeDescriptor dn;  // 疑似与storageID错位了
  private final String storageID;
  private StorageType storageType;
  private State state;

  private long capacity;
  private long dfsUsed;
  private long remaining;
  private long blockPoolUsed;
}
~~~
