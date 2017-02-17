## MongoDB各种备份方法的优缺点分析

1. Back Up with MongoDB Cloud Manager or Ops Manager  
    - MongoDB Cloud Manager
        - MongoDB备份，监控，自动服务的宿主，能够以图形用户界面形式备份和恢复MongoDB副本集和分片集群上的数据。它通过读取MongoDB的操作日志（oplog）数据备份MongoDB副本集和分片集群上的数据，它给分隔的主机上的数据创建快照，同时可以恢复MongoDB副本集和分片集群上的数据到时间点（point-in-time）。  
    - Ops Manager
        - 类似于MongoDB Cloud Manager，它拥有MongoDB Cloud Manager的核心功能，是企业级高级订阅服务。

2.  Back Up by Copying Underlying Data Files
    - Back Up with Filesystem Snapshots
        - 优点：
            - 备份数据规模较大的MongoDB时，性能较好。
        - 缺点：
            - 依赖于潜在的存储系统，存储数据文件的卷必须支持时间点快照（数据存储系统的特性，而非MongoDB的特性，Linux系统的Logical Volume Manager（LVM）默认支持时间点快照）。
            - 每次备份时必须备份**整个卷**中所有的数据。
            - 获取分片集群的连续快照必须关闭均衡器，同时在同一时刻从每一个分片和配置服务器捕捉快照。    
    - Back Up with cp or rsync
        - 优点：
            - 数据存储系统不支持快照的情况下，使用cp, rsync或者其它命令直接复制文件即可。
        - 缺点：
            - 复制多文件不是原子操作，因此复制前必须停止mongod进程，否则将复制到处于无效状态的文件。
            - 复制潜在数据的备份过程不支持副本集的时间点级别的恢复，大的分片集群也难以把控。
            - 备份文件较大（较mongodump备份文件更大），因为它们包括索引和潜在储存填充的复制品。
            - 获取正在运行的mongod进程的正确快照有一定前提：1) 开启journal 功能；2) journal必须与MongoDB数据文件保存在相同的逻辑卷中，否则不能保证快照是连续的或者有效的。

3. Back Up with mongodump
    - 从MongoDB数据库读取数据，创建的BSON文件具备高保真特性，使用mongorestore即可恢复数据到MongoDB数据库。在备份的过程中，MongoDB中的数据依然允许改变，通过--oplog参数可以保存备份过程中外部对数据库的访问操作，数据恢复使用--oplogReplay参数即可恢复**备份过程**的数据变更。
    - 优点：
        - 针对数据规模较小的情景，备份和恢复策略比较简单有效。
        - 用于备份数据库（默认不备份local数据库）的集合，因此备份具有较高的空间效率。
    - 缺点：
        - 针对数据规模较大的情景时，性能不够理想，数据查询有可能导致内存溢出。
        - 没有备份local数据库，因此数据恢复以后需要重建索引。
        - 副本集和分片集群中数据备份中使用时，存在很多问题。
        - 数据库的数据量较少时，在数据实时备份方面存在问题，因为oplog的窗口时间太短。

4. Back Up with mongooplog
    - 从远程服务器复制数据操作日志并且应用到本地服务器（俗称：增量备份），使用mongorestore及其--oplogReplay参数可以恢复数据到point-in-time。该方案默认复制24小时的日志记录，因此，通过定时任务就可以获取到所有的操作日志，即可以恢复任意时刻的数据。
    - 优点：
        - 可以实现数据的实时备份以及数据的point-in-time恢复。
    - 缺点：
        - mongooplog已被标记为deprecated，以后的MongoDB版本将会移除。