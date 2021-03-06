- Redis内部编码
    - string
        - int
        - embstr
        - raw
    - hash
        - ziplist
        - hashtable
    - list
        - ziplist
        - linkedlist
    - set
        - intset
        - hashtable
    - sorted set
        - ziplist
        - skiplist

- 其他数据结构
    - HyperLogLog
        - 提供不精确的去重计数
    - Geo
        - 借助于Sorted Set，使用GeoHash技术进行填充
    - Bitmap
        - BloomFilter
    - Streams
        - 内存版kafka，底层的数据结构是radix tree（基数树）

- Redis线程模型
    - Redis内部使用文件事件处理器 file event handler，这个文件事件处理器是单线程的，所以 Redis 才叫做单线程的模型
    - 文件事件处理器
        - 多个Socket
        - IO多路复用
        - 文件事件分派器
        - 事件处理器

- Redis为什么快
    - 数据存在内存中
    - 使用IO多路复用模型
    - 使用单线程，避免了不必要的上下文切换和竞争条件，不用去考虑各种锁的问题

- Redis持久化
    - RDB做全量持久化
        - fork 和 copy on write
        - 对Redis中的数据执行周期性的持久化
        - RDB对Redis的性能影响非常小，是因为在同步数据的时候他只是fork了一个子进程去做持久化的，而且他在数据恢复的时候速度比AOF来的快
    - AOF做增量持久化
        - 对每条写入命令，以append-only的模式写入一个日志文件中，因为这个模式是只追加的方式，没有任何磁盘寻址的开销，所以很快
        - AOF一秒一次去通过一个后台的线程fsync做持久化，最多丢这一秒的数据
    - 单独用RDB会丢失很多数据，单独用AOF数据恢复没RDB来的快，出问题时第一时间用RDB恢复，然后AOF做数据补全，冷备热备一起上

- Redis主从同步，读写分离
    - 启动一台slave的时候，他会发送一个psync命令给master ，如果是这个slave第一次连接到master，他会触发一个全量复制
    - master会启动一个线程，生成RDB快照，还会把新的写请求都缓存在内存中，RDB文件生成后，master会将这个RDB发送给slave
    - slave拿到RDB，加载进内存，通知主节点将期间写的操作记录同步到复制节点进行重放就完成了同步过程
    - 后续的增量数据通过AOF日志同步即可

- Redis集群
    - Redis集群由多个节点组成，集群中的节点分为主节点和从节点，只有主节点负责读写请求和集群信息的维护，从节点只进行主节点数据和状态信息的复制
    - 集群的作用主要是数据分区和高可用
    - 数据分区方案
        - Redis集群使用带虚拟节点的一致性哈希分区方案，其中的虚拟节点称为槽（slot）
        - 槽是数据管理和迁移的基本单位，槽解耦了数据和实际节点之间的关系，增加或删除节点对系统的影响很小
    - 节点通信机制
        - 两个端口：普通端口和集群端口
        - Gossip协议
    - 数据结构
        - clusterNode：记录了节点的状态
        - clusterState结构：记录了集群的状态
    - 集群伸缩
        - 集群伸缩的核心是槽迁移
    - 故障转移
        - 定时发送PING消息检测其他节点状态
        - 主节点下线分为主观下线和客观下线，客观下线后由主节点投票选出一个从节点成为新的主节点，并进行故障转移
        - 集群只实现了主节点的故障转移，从节点故障时只会被下线，不会进行故障转移。因此使用集群时应谨慎使用读写分离技术，因为从节点故障会导致读服务不可用
    - Hash Tag
        - 当一个key包含 {} 的时候，不对整个key做hash，而仅对 {} 里面的字符串做hash
        - Hash Tag可以让不同的key拥有相同的hash值，从而分配在同一个槽里。这样针对不同key的批量操作(mget/mset等)，以及事务、Lua脚本等都可以支持
        - Hash Tag可能会带来数据分配不均的问题

- Redis过期策略
    - 定期删除
    - 惰性删除
    - 定期过期

- Redis 内存淘汰策略
    - volatile-lru
        - 当内存不足以容纳新写入数据时，从设置了过期时间的key中使用LRU算法进行淘汰
    - allkeys-lru
        - 当内存不足以容纳新写入数据时，从所有key中使用LRU算法进行淘汰
    - volatile-lfu
        - 4.0版本新增，当内存不足以容纳新写入数据时，在过期的key中，使用LFU算法进行删除key
    - allkeys-lfu
        - 4.0版本新增，当内存不足以容纳新写入数据时，从所有key中使用LFU算法进行淘汰
    - volatile-random
        - 当内存不足以容纳新写入数据时，从设置了过期时间的key中，随机淘汰数据
    - allkeys-random
        - 当内存不足以容纳新写入数据时，从所有key中随机淘汰数据
    - volatile-ttl
        - 当内存不足以容纳新写入数据时，在设置了过期时间的key中，根据过期时间进行淘汰，越早过期的优先被淘汰
    - noeviction
        - 默认策略，当内存不足以容纳新写入数据时，新写入操作会报错

- Redis分布式锁
    - set key unique_value nx ex
    - 防止删除别人的锁
        - 校验唯一随机值，再删除，使用lua脚本保证原子性
    - 业务还没执行完时锁自动续期

- Redlock算法
    - 避免单一主节点故障造成加锁不安全
    
- Redis实现延时队列
    - 使用sorted set，时间戳作为score，消息内容作为member，调用zadd来生产消息，消费者用zrangebyscore指令获取延时数据

- 借助Sorted set实现多维排序
    - 将涉及排序的多个维度的列通过一定的方式转换成一个特殊的列，即result = function(x, y, z)，将result作为Sorted Set中的score的值来实现任意维度的排序需求
    
- Redis相较于Memcached的优势
    - Redis支持更多数据结构
    - Redis原生支持集群模式

- Redis为什么不支持回滚（rollback）
    - 失败的命令是由编程错误造成的，而这些错误应该在开发的过程中被发现，而不应该出现在生产环境中
    - 因为不需要对回滚进行支持，所以 Redis 的内部可以保持简单且快速

- 为什么Redis6.0之后改为多线程
    - redis只是使用多线程来处理数据的读写和协议解析，执行命令还是使用单线程
    - redis的性能瓶颈在于网络IO而非CPU，使用多线程能提升IO读写的效率，从而整体提高redis的性能

- Redis事务
    - Redis通过MULTI、EXEC、DISCARD、WATCH、UNWATCH等一组命令集合，来实现事务机制
    - 顺序性、一次性、排他性

- Redis的Hash冲突怎么办
    - Redis为了解决哈希冲突，采用了链式哈希。链式哈希是指同一个哈希桶中，多个元素用一个链表来保存，它们之间依次用指针连接
    - 为了保持高效，Redis 会对哈希表做rehash操作，也就是增加哈希桶，减少冲突。为了rehash更高效，Redis还默认使用了两个全局哈希表，一个用于当前使用，称为主哈希表，一个用于扩容，称为备用哈希表

- 在生成RDB期间，Redis可以同时处理写请求么
    - 如果是save指令，会阻塞，因为是主线程执行的
    - 如果是bgsave指令，是fork一个子进程来写入RDB文件的，快照持久化完全交给子进程来处理，父进程则可以继续处理客户端的请求

- Redis底层使用的是什么协议
    - RESP(Redis Serialization Protocol)

- MySQL与Redis 如何保证双写一致性
    - Cache-Aside缓存模式
        - 读的时候，先读缓存，缓存没有的话，就读数据库，然后取出数据后放入缓存，同时返回响应
        - 更新的时候，先更新数据库，然后再删除缓存
    - 方案
        - 缓存延时双删
        - 删除缓存重试机制
        - 读取binlog异步删除缓存

- 缓存
    - 缓存雪崩
        - 原因
            - 大量Key同时失效，请求直接打到数据库
        - 解决方案
            - 批量往缓存中存数据的时候，把每个Key的失效时间都加个随机值
    - 缓存穿透
        - 原因
            - 缓存和数据库中都没有的数据，而用户不断发起请求
        - 解决方案
            - 用户参数校验
            - 布隆过滤器
    - 缓存击穿
        - 原因
            - 热点Key在失效的瞬间，持续的大并发就击穿缓存，直接请求数据库
        - 解决方案
            - 设置热点数据永不过期
            - 加互斥锁
