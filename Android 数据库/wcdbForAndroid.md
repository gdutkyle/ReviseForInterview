#最全面的wcdb入门和总结
##开篇前的思考：一个优秀的数据库需要具备什么样的特性？  
第一：高效。高效的增删查改是数据库最基本的要求  

第二：易用。一个简单的数据库操作，我们不需要做什么额外的初始化、操作等，组件能统一完成这些任务  

第三：完整。优秀的数据库能完整的覆盖各种极端场景，包括数据库损坏、监控统计、复杂的查询，优秀的日志系统等  
## WCDB接入与迁移 ##

第一步：引入依赖  

![wcdb依赖引入](https://i.imgur.com/x1lxwkF.png)

第二步：选择接入的架构  

![wcdb架构引用](https://i.imgur.com/YkVwnFY.png)  
第三步：原有sqlcipher的迁移  

android.database.* -> com.tencent.wcdb.*  
android.database.sqlite.* -> com.tencent.wcdb.database.*   

第四步：原有SQLCipher迁移（从加密数据库往wcdb迁移）
![sqlcipher 数据库迁移](https://i.imgur.com/G0Jx70b.png)  
##WCDB For Android
###1 确定的sqlite版本  
在 Android SDK 中，SQLite 是会不断升级的，实际上使用哪个版本的 SQLite 取决于 APP 运行在哪个版本的系统上，这是对开发者来说相当不友好，因为同样的 SQL 语句会有不同的性能表现。如果业务需要使用 SQLite 的新特性就更加需要确定版本的 SQLite 来保证新特性在所有手机上都可用。
WCDB 由于内建了自己的 SQLite 实现（准确来说是 SQLCipher），所以 SQLite 版本是确定的，这规避了很多开发上的问题。

### 2 Cursor 实现优化
在实际使用的很多的场景中，我们都需要去数据库执行查询数据，得到cursor，然后通过cursor，封装数据到我们的list中。这个查询过程中，数据会缓存到到cursorWindow中，也即是（SQLite → CursorWindow → Java）
Wcdb的优化是，直接切断cursorWindow，通过SQLiteDirectCursor，将SQLite直接返回到java中。在大部分不需要将 Cursor 传递出去的场景，能很好的解决 Cursor 的额外消耗，特别是结果集大于 2MB 的场合

### 3 mmicu与动态icu加载
什么是mmicu？wcdb自带的分词器，与官方的icu不同的是，wcdb对中文分词以及icu库加载做了特殊处理。官方的icu库会因为不同的android系统，附带不同的中文词库，会带来不同的分词效果。 wcdb这样做，也即是相当于统一了各个版本的分词效果
动态icu加载：官方ICU 还有一个严重的问题是动态库和自带的数据文件体积很大，超过 10MB，WCDB 做了一个兼容层 icucompat，通过系统带的数据文件推断 ICU 版本， 通过 dlopen 动态加载不同的符号名称，然后通过宏来模拟直接调用方便开发。最终实现效果便是在不需要自带 ICU 库的前提下使用 ICU 库的断词、归一化等功能，为最终 APK 包省下 10MB 以上空间。  
## WCDB的性能

![性能对比1](https://i.imgur.com/DAI4Joh.png)  

![性能对比2](https://i.imgur.com/eJCkz1J.png)  

从上面可以看出wcdb对标与sqlcipher，在数据库的插入、查询、更新方面的性能是有很大的提升的
### 方案1：官方Dump方案  

![wcdb DBDumpUtil.doRecoveryDb()](https://i.imgur.com/Eq6s5Qs.png)  

**流程分析**  

dump方案的原理很简单，主要
思路是读取db中的sqlite_master表，里面保存着所有的索引、表名、建表语句等。 
 
步骤1：读取create语句，建立新表  

步骤2：执行select * From语句，每读出一行，就输出一个insert语句
经过步骤2，整个db就已经dump出来了  

备注：忽略掉损坏错误，继续遍历下个表，最终可以把所有没损坏的表以及损坏了的表的前半部分读取出来  

**思考：官方的方案存在什么问题？**  

1 耗费空间。先要保存dump 出来的SQL的空间，这个 大概一倍DB大小，还要另外一倍 DB大小来新建DB恢复。  

2 成功率低，sqlite_master读取失败，特别是第一页损坏，会导致后续所有内容无法读出  

3 花费时间长（后面有对比图）
### 方案2 BackupKit 备份方案（优化版Dump + 压缩） 
微信在原先的官方dump+gzip方案上做出了两点改动：  
1使用了自定义 的二进制格式承载Dump输出。恢复的时候不需要重复的编译SQL语句，编译一次就可以 插入整个表的数据了，恢复性能也有一定提升  

2压缩操作则放到别的线程同时进行，在双核以上的环境 基本可以做到无额外时间消耗。

优化前的对比： 
![优化前各个数据库修复方案对比](https://i.imgur.com/sYlFrjx.png)  

优化后备份方案和原来的dump方案的对比  

![优化后对比](https://i.imgur.com/E3cpoyL.png)  

### 方案3：Repair Kit
**核心思想：备份sqlite_master表**  
1 如何判断备份时机？轮询 

2备份文件也会损坏，怎么办？  
双备份 的机制。具体来说就是会有新旧两个备份文件，每个文件头都加上 CRC 校验；每次备份时，从两个备份文件中选出一个进行覆盖。具体怎么选呢？优先选损坏那个备份文件，如果两个都有效，那么就选相对较旧的。这就保证了即使本次写入导致文件损坏，还有另外一份备份可以用  


![Reparikit和BackupKit性能对比](https://i.imgur.com/aE9fCbI.png)

## WCDB 的 WAL和异步checkPoint  

**原子性提交和回滚操作 的默认方法是 rollback journal，其工作模式如下图所示**
![经典rollback journal 工作方式](https://i.imgur.com/bT1BIHe.png)  
如以上流程图所示，当我们需要对数据库进行操作的时候，sqlite会把需要修改的部分copy到journal，以防出现意外的时候，可以进行回滚。当写入完成，用户提交事务后，sqlite将会清空journal，完成一个完整的写事务  

**问题是什么？** 
Rollback 模式中，每次拷贝原始内容或写入新内容后，都需要确保之前写入的数据真正写入到磁盘，而不是缓存在操作系统中，这需要发起一次 fsync 操作，通知并等待操作系统将缓存真正写入磁盘，这个过程十分耗时。除了耗时的 fsync 操作，写入 -journal 以及 DB 主文件的时候，是需要独占整个 DB 的，否则别的线程/进程可能读取到写到一半的内容。这样的设计使得写操作与读操作是互斥的，并发性很差

WCDB WAl模式的优越性体现在哪？  
![WAL工作流程图展示](https://i.imgur.com/IiJemKJ.png)  
从上图可以看出，wal模式下，写数据并不是直接写到-wal文件末尾。读操作的时候，将结合 DB 主文件以及 -wal 的内容返回结果。由于读操作只读取 DB 主文件和 -wal 前面没在写的部分，不需要读取写操作正在写到一半的内容，WAL 模式下读与写操作的并发由此实现写操作需要执行Checkpoint才可以把当前wal文件中的内容合并到db主文件中。由于写操作将内容临时写到 -wal 文件，-wal 文件会不断增大且拖慢读操作，因此需要定期进行Checkpoint 操作将 -wal 文件保持在合理的大小。  

**性能如何？看看效果**
写性能对比：  
![写性能对比](https://i.imgur.com/DSwJt1V.png)  
读性能对比  
![对性能对比](https://i.imgur.com/vVmoAzS.png)  
以上就是wcdb wal模式的对比，读性能则如官方文档所说，WAL 模式单线程性能要稍稍差于 Rollback 模式，但由于 WAL 模式支持读写并发，WCDB 也开启了线程池，因此 WAL 模式的并发性要远远好于 Rollback 模式。 
 
##参考：  
[https://cloud.tencent.com/developer/article/1031030](https://cloud.tencent.com/developer/article/1031030)  
[https://cloud.tencent.com/developer/article/1005623](https://cloud.tencent.com/developer/article/1005623)  
[https://cloud.tencent.com/developer/article/1005534](https://cloud.tencent.com/developer/article/1005534)  
[https://cloud.tencent.com/developer/article/1005513](https://cloud.tencent.com/developer/article/1005513)