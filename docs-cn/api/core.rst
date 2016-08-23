核心库 API
================

Distributedlog 核心库直接操作 名空间(namespaces)和日志(logs)。
核心库使用 java 实现。

名空间(Namespace) API
-------------

DL 的名空间包含多个日志流(log streams). 应用可以创建(create)或者删除(delete)名空间下的日志。

名空间 URI
~~~~~~~~~~~~~

**URI** 用来定位名空间。名空间 URI 通常由 *3* 部分组成。

* scheme: `distributedlog-<backend>`. 这里 *backend* 指明使用什么来存储日志数据。
* 域名: 访问后端(backend)所需要的域名。下面的例子里，域名部分就是 *zookeeper server*, 它用来存储 bookkeeper 作为日志存储后端所需的 metadata 。
* 路径: 路径指向存储 metadata 的具体地址。下面的例子里，路径指向一个 znode 节点，该节点存储所有日志的 metadata。

::

    distributedlog-bk://<zookeeper-server>/path/to/stream

目前可用的后端只有基于 bookkeeper 这一种。
默认使用的 scheme `distributedlog` 就是 `distributedlog-bk` 的别名.

构建名空间
~~~~~~~~~~~~~~~~~~~~

名空间 URI(namespace uri) 定义好之后，你就可以构建一个名空间实例。
名空间实例将用来操作其下的日志流。

::

    // DistributedLog Configuration
    DistributedLogConfiguration conf = new DistributedLogConfiguration();
    // Namespace URI
    URI uri = ...; // create the namespace uri
    // create a builder to build namespace instances
    DistributedLogNamespaceBuilder builder = DistributedLogNamespaceBuilder.newBuilder();
    DistributedLogNamespace namespace = builder
        .conf(conf)             // configuration that used by namespace
        .uri(uri)               // namespace uri
        .statsLogger(...)       // stats logger to log stats
        .featureProvider(...)   // feature provider on controlling features
        .build();

创建日志
~~~~~~~~~~~~

通过直接调用 `distributedlognamespace#createlog(logname)` 来创建日志。
该方法只是在名空间下创建一个日志而不返回供操作日志用的手柄(handle)。

::

    DistributedLogNamespace namespace = ...; // namespace
    try {
        namespace.createLog("test-log");
    } catch (IOException ioe) {
        // handling the exception on creating a log
    }

打开日志
~~~~~~~~~~

通过 `#openLog(logName)` 可以返回一个 `DistributedLogManager` 手柄。该手柄可以用来写入和读取日志数据。
如果日志不存在并且 `createStreamIfNotExists` 设置为 true, 日志将会在写入第一个记录的时候被自动创建。

::

    DistributedLogConfiguration conf = new DistributedLogConfiguration();
    conf.setCreateStreamIfNotExists(true);
    DistributedLogNamespace namespace = DistributedLogNamespace.newBuilder()
        .conf(conf)
        ...
        .build();
    DistributedLogManager logManager = namespace.openLog("test-log");
    // use the log manager to open writer to write data or open reader to read data
    ...

有时, 应用可以在打开日志的时候传入不同的参数。可以通过调用重载方法 `#openLog` 用法如下：

::

    DistributedLogConfiguration conf = new DistributedLogConfiguration();
    // set the retention period hours to 24 hours.
    conf.setRetentionPeriodHours(24);
    URI uri = ...;
    DistributedLogNamespace namespace = DistributedLogNamespace.newBuilder()
        .conf(conf)
        .uri(uri)
        ...
        .build();

    // Per Log Configuration
    DistributedLogConfigration logConf = new DistributedLogConfiguration();
    // set the retention period hours to 12 hours for a single stream
    logConf.setRetentionPeriodHours(12);

    // open the log with overrided settings
    DistributedLogManager logManager = namespace.openLog("test-log",
        Optional.of(logConf),
        Optiona.absent());

删除日志
~~~~~~~~~~~~

`DistributedLogNamespace#deleteLog(logName)` 可以删除名空间下的日志。删除日志操作会先试图获取锁，如果该日志正在被一个活跃的 writer 写入，那么锁已经被这个活跃的 writer 获取了，删除将失败。

::

    DistributedLogNamespace namespace = ...;
    try {
        namespace.deleteLog("test-log");
    } catch (IOException ioe) {
        // handle the exceptions
    }

日志是否存在
~~~~~~~~~~~~~

应用可以通过调用 `DistributedLogNamespace#logExists(logName)` 来检查名空间下的日志是否存在。

::

    DistributedLogNamespace namespace = ...;
    if (namespace.logExists("test-log")) {
        // actions when log exists
    } else {
        // actions when log doesn't exist
    }

获取日志列表
~~~~~~~~~~~~~~~~

应用可以通过调用 `DistributedLogNamespace#getLogs()` 获取日志列表。

::

    DistributedLogNamespace namespace = ...;
    Iterator<String> logs = namespace.getLogs();
    while (logs.hasNext()) {
        String logName = logs.next();
        // ... process the log
    }

Writer API
----------

将记录写入日志流由两个方法：一是使用同步的 `LogWriter`, 另一种是使用异步的 `AsyncLogWriter`.

LogWriter
~~~~~~~~~

在写入数据到日志流之前首先要构建一个 writer 实例。请注意，distributedlog 核心库通过 zookeeper 的锁机制来强制执行 *单写入者(single-writer)*语义。如果已经有一个活跃的 writer, 后续对 `#startLogSegmentNonPartitioned()` 的调用将会触发异常：`OwnershipAcquireFailedException`.

::
    
    DistributedLogNamespace namespace = ....;
    DistributedLogManager dlm = namespace.openLog("test-log");
    LogWriter writer = dlm.startLogSegmentNonPartitioned();

.. _Construct Log Record:

通过创建日志记录来代表写入日志流的数据，每个日志记录都关联着应用定义业务编号(transaction id)。
业务编号不能是递减的，否则写入记录将会被拒绝，对应的异常是 `TransactionIdOutOfOrderException`. 应用允许通过设置 `maxIdSanityCheck` 为 false 来绕过业务编号的正常性检查。系统时间和原子序数是作为业务编号的不错的选择。

::

    long txid = 1L;
    byte[] data = ...;
    LogRecord record = new LogRecord(txid, data);

应用可以添加一个记录 (通过 `#write(LogRecord)`) 或者一串记录 (通过 `#writeBulk(List<LogRecord>)`) 到日志流。

::

    writer.write(record);
    // or
    List<LogRecord> records = Lists.newArrayList();
    records.add(record);
    writer.writeBulk(records);

在记录被写入到 writer 的输出缓冲区后，写入调用就立即返回了。所以数据并不保证是持久的直到 writer 显示的调用 `#setReadyToFlush()` 和 `#flushAndSync()`. 这两个方法的调用将首先把缓冲的数据传输到后端，等待确认，然后提交刚才写入的数据以使它们对 readers 可见。

::

    // flush the records
    writer.setReadyToFlush();
    // commit the records to make them visible to readers
    writer.flushAndSync();

DL 的日志流是无终止的(endless)除非被封存。无终止意思是 writers 一直往里写记录，readers 一直从另一头读记录，永不停止。应用可以通过调用 `#markEndOfStream()` 来封存日志流.

::

    // seal the log stream
    writer.markEndOfStream();
    

写入记录的完整例子展示如下：

::

    DistributedLogNamespace namespace = ....;
    DistributedLogManager dlm = namespace.openLog("test-log");

    LogWriter writer = dlm.startLogSegmentNonPartitioned();
    for (long txid = 1L; txid <= 100L; txid++) {
        byte[] data = ...;
        LogRecord record = new LogRecord(txid, data);
        writer.write(record);
    }
    // flush the records
    writer.setReadyToFlush();
    // commit the records to make them visible to readers
    writer.flushAndSync();

    // seal the log stream
    writer.markEndOfStream();

AsyncLogWriter
~~~~~~~~~~~~~~

构建一个异步的 `AsyncLogWriter` 简单如构建一个同步的 `LogWriter`.

::

    DistributedLogNamespace namespace = ....;
    DistributedLogManager dlm = namespace.openLog("test-log");
    AsyncLogWriter writer = dlm.startAsyncLogSegmentNonPartitioned();

所有 `AsyncLogWriter` 的写入都是异步的。这里 futures 表示写入结果只有在数据被持久化的日志流才算完成。
一个 DLSN (distributedlog sequence number) 将会被返回给每个 write 调用， DLSN 用来表示一个记录在日志流里的位置 (即 offset)。
所有记录的持久化顺序保证与写入顺序一致。

.. _Async Write Records:

::

    List<Future<DLSN>> addFutures = Lists.newArrayList();
    for (long txid = 1L; txid <= 100L; txid++) {
        byte[] data = ...;
        LogRecord record = new LogRecord(txid, data);
        addFutures.add(writer.write(record));
    }
    List<DLSN> addResults = Await.result(Future.collect(addFutures));

`AsyncLogWriter` 也提供截短日志流到给定的 DLSN 的方法。这对构建可复制状态机非常有帮助，构建它需要显式控制什么时候数据可以删除。

::
    
    DLSN truncateDLSN = ...;
    Future<DLSN> truncateFuture = writer.truncate(truncateDLSN);
    // wait for truncation result
    Await.result(truncateFuture);

Reader API
----------

序列号
~~~~~~~~~~~~~~~~

日志记录与序列号关联。 首先，应用可以在写入时赋值自己的序列号 (叫作 `TransactionID`) 给日志记录。(查看 `Construct Log Record`_). 其次，在被写入日志时一个日志记录可以被赋值一个系统生成的唯一序列号 `DLSN` (distributedlog sequence number) (查看 `Async Write Records`_). 另外除了 `DLSN` 和 `TransactionID`, 在读取时一个单调递增的64位的 `SequenceId` 也被赋值给了日志记录，来标识记录在日志中的位置。

:Transaction ID: 由应用赋值的一个正的64位长整型数。
    在应用需要用自身的序列方法整理记录和定位读取者(readers)的时候 Transaction ID 是非常有用的。`Transaction ID` 的一个典型用例是 `DistributedLog Write Proxy`. 写入代理(write proxy)赋值非递减的时间戳给日志记录，在一个强一致性的数据库里，时间戳可以被用作物理时间(`physical time`)来实现 `TTL` (存活时间) 功能。
:DLSN: DLSN (DistributedLog Sequence Number) 是在日志被写入时生成的序列号。
    DLSN 可以互相比较进而可以用来判断记录间的顺序。一个 DLSN 由 3 部分组成，它们是：`Log Segment Sequence Number`, `Entry Id` 和 `Slot Id`. DLSN 通常用来比较，定位和截短日志。
:Sequence ID: Sequence ID 被引入以解决 `DLSN` 的不足，尤其用来解决 `两个 DLSN 之间有多少记录` 这种问题。
    Sequence ID 是一个从 0 开始的64位单调递增数。序列编号在读取时被计算出来，并且只能被读取者(readers)访问。这意味着写入者(writers)在写入时并不知道记录的序列编号。

通过使用 `DLSN` or `Transaction ID` 读取者可以被定位到日志的任何位置开始读取。

LogReader
~~~~~~~~~

`LogReader` `同步`并顺序地从给定位置读取日志流。位置可以是 `DLSN` (通过 `#getInputStream(DLSN)`) 或者 `Transaction ID` (通过 `#getInputStream(long)`). 在这个读取者打开之后，它可以调用 
`#readNext(boolean)` 或者 `#readBulk(boolean, int)` 来顺序的从日志流读出记录。关闭这个读取者 (通过 `#close()`) 将释放该实例占用的所有资源。

在读取时遇到的各种问题将会抛出一些异常。在异常被抛出后，读取者被设置为错误状态，并不再可用。应用程序负责处理异常并在必要的时候创新创建读取者。

::
    
    DistributedLogManager dlm = ...;
    long nextTxId = ...;
    LogReader reader = dlm.getInputStream(nextTxId);

    while (true) { // keep reading & processing records
        LogRecord record;
        try {
            record = reader.readNext(false);
            nextTxId = record.getTransactionId();
            // process the record
            ...
        } catch (IOException ioe) {
            // handle the exception
            ...
            reader = dlm.getInputStream(nextTxId + 1);
        }
    }

在读取一个无终止的日志流时，`同步读` 并不像`异步读` 那样微不足道。因为同步读却少回调机制。相反地，`LogReader` 引入 `nonBlocking` 标识来控制 `同步读` 时的等待行为。 阻塞 (`nonBlocking = false`) 意思是读取调用返回前会一直等待记录，而非阻塞 (`nonBlocking = true`) 意思是读取只检查预读缓存并返回缓存里任何可用的记录。

`阻塞` 模式里 `等待周期` 有多种。如果读取者没有追上写入者 (日志中有大量的未读记录), 读取只等到记录被读取并返回。如果读取者追上了写入者 (读取时日志中没有可读记录), 读取将等待一小段时间
 (定义在 `DistributedLogConfiguration#getReadAheadWaitTime()`) 然后返回预读缓存中任何可用的记录。换句话说，阻塞模式读去时当读取者看不到可用记录时，意味着读取者`追上` 了写入者。

关于如何使用 `LogReader` 读取记录，示例如下：

::

    // Read individual records
    
    LogReader reader = ...;
    // keep reading records in blocking mode until no records available in the log
    LogRecord record = reader.readNext(false);
    while (null != record) {
        // process the record
        ...
        // read next record
        record = reader.readNext(false);
    }
    ...

    // reader is caught up with writer, doing non-blocking reads to tail the log
    while (true) {
        record = reader.readNext(true);
        if (null == record) {
            // no record available yet. backoff ?
            ...
        } else {
            // process the new record
            ...
        }
    }

::
    
    // Read records in batch

    LogReader reader = ...;
    int N = 10;

    // keep reading N records in blocking mode until no records available in the log
    List<LogRecord> records = reader.readBulk(false, N);
    while (!records.isEmpty()) {
        // process the list of records
        ...
        if (records.size() < N) { // no more records available in the log
            break;
        }
        // read next N records
        records = reader.readBulk(false, N);
    }

    ...

    // reader is caught up with writer, doing non-blocking reads to tail the log
    while (true) {
        records = reader.readBulk(true, N);
        // process the new records
        ...
    }


AsyncLogReader
~~~~~~~~~~~~~~

与 `LogReader` 类似，应用程序可以打开一个 `AsyncLogReader` 并定位到不同的位置，通过 `DLSN` 或者 `Transaction ID`.

::
    
    DistributedLogManager dlm = ...;

    Future<AsyncLogReader> openFuture;

    // position the reader to transaction id `999`
    openFuture = dlm.openAsyncLogReader(999L);

    // or position the reader to DLSN
    DLSN fromDLSN = ...;
    openFuture = dlm.openAsyncLogReader(fromDLSN);

    AsyncLogReader reader = Await.result(openFuture);

通过 `AsyncLogReader` 读取记录是异步的。`#readNext()`, `#readBulk(int)` 或者 `#readBulk(int, long, TimeUnit)` 返回的 future 表示读取操作的结果。
只有在有可用的记录时，future 才有值。应用程序可以将 futures 串接来进行顺序读取。

从 `AsyncLogReader` 里逐个读取记录：

::

    void readOneRecord(AsyncLogReader reader) {
        reader.readNext().addEventListener(new FutureEventListener<LogRecordWithDLSN>() {
            public void onSuccess(LogRecordWithDLSN record) {
                // process the record
                ...
                // read next
                readOneRecord(reader);
            }
            public void onFailure(Throwable cause) {
                // handle errors and re-create reader
                ...
                reader = ...;
                // read next
                readOneRecord(reader);
            }
        });
    }
    
    AsyncLogReader reader = ...;
    readOneRecord(reader);

从 `AsyncLogReader` 里批量读取记录：

::

    void readBulk(AsyncLogReader reader, int N) {
        reader.readBulk(N).addEventListener(new FutureEventListener<List<LogRecordWithDLSN>>() {
            public void onSuccess(List<LogRecordWithDLSN> records) {
                // process the records
                ...
                // read next
                readBulk(reader, N);
            }
            public void onFailure(Throwable cause) {
                // handle errors and re-create reader
                ...
                reader = ...;
                // read next
                readBulk(reader, N);
            }
        });
    }
    
    AsyncLogReader reader = ...;
    readBulk(reader, N);

