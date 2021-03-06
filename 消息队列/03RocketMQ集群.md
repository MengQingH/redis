## RocketMQ集群模型
* 单机模式（M）：
* 双Master模式/多Master模式（2M）：一个集群无Slave，全是Master。
    * 优点：配置简单，单个Master宕机或重启对应用无影响。在磁盘配置为RAID10时，即使机器宕机不可恢复情况下，由于RAID10磁盘可靠性非常高，消息也不会丢失(异步刷盘丢失少量消息，同步刷盘完全不丢失)，性能最高。
    * 缺点：单台机器宕机时，这台机器上未被消费的消息在机器回复之前不可订阅，消息的实时性收到影响。
* 多Master多Slave模式，异步复制（NM-NS，ASYNC）：每个Master配备一个Slave，共有多对Master-Slave，HA采用异步复制的方式，主从有短暂消息延迟，毫秒级别。
    * 优点：即使磁盘损坏，消息的丢失也非常少，消息的实时性不会受到影响，因为Master宕机后，消费者仍然可以从Slave中消费消息，此过程中对应用完全透明，不需要人工干预，性能同多Master模式几乎一样。
    * 缺点：Master宕机后，如果磁盘出现损坏，可能丢失少量消息。
* 多Master多Slave模式，同步双写（NM-NS，SYNC）：
    * 个Master配备一个Slave，共有多对Master-Slave，HA采用同步双写机制，主从都写入消息成功后，再向应用返回ACK。
    * 优点：数据与服务都无单点故障问题，Master宕机情况下，消息无延迟，服务可用性和数据可用性都非常高。
    * 缺点：性能比异步复制略低，大概低10%，发送单个消息的RT会略高。目前宕机情况下，从节点不能自动切换成主节点，后续会支持自动切换功能。
