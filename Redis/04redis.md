## Redis中的过期策略
过期策略：即redis中针对过期的key使用的清除策略，使用的策略为：定期删除+惰性删除。redis中的过期策略有：
* 定时删除：每个设置过期时间的key都需要创建一个定时器，到过期时间就会立即清除。
    * 优点：立即清除过期的数据，保证内存被尽快释放。
    * 缺点：会占用大量的CPU资源去处理过期的数据和创建定时器，从而影响缓存的响应时间和吞吐量。
* 惰性删除：key过期的时候不删除，每次从数据库中获取key的时候检查key是否过期，如果过期则删除。
    * 优点：删除操作只发生在从数据库存取key的时候发生，而且只删除当前key，所以对CPU的占用比较少。
    * 缺点：如果key在超出超时时间后，很长一段时间都没有被获取，那么就会一直占用内存资源。
* 定期删除：每隔一定的时间，会扫描一定数量的数据库的expires字典中一定数量的key，并清除其中已过期的key。
    * 特点：限制了删除操作的时间和频率，减少了对CPU的占用；定期删除过期key，弥补了惰性删除的缺点。
    * expires字典会保存所有设置了过期时间的key的过期时间数据，其中，key是指向键空间中的某个键的指针，value是该键的毫秒精度的UNIX时间戳表示的过期时间。键空间是指该Redis集群中保存的所有键。

执行流程：
* 定期删除流程：对指定个库中的每一个库随机删除小于等于指定个数个过期key
    * 遍历每个数据库（redis.conf中配置的database数量，默认为16）
        * 检查当前库中指定个数个key（默认是每个库检查20个key）
            * 如果当前库中没有一个key设置了过期时间，直接执行下一个库的遍历
            * 随机获取了一个设置了过期时间的key，检查是否过期，如果过期，则删除
* 惰性删除流程：
    * 获取时判断key是否过期
    * 如果过期，则删除，然后执行相应操作
    * 如果没有过期，直接执行相应操作

## redis内存淘汰机制
当有新的缓存请求，并且实际内存使用达到指定值（maxmemory）时，redis根据指定的淘汰策略来选出无用的key。有以下几种类型：（LRU的意思是：Least Recently Used最近最少使用的，LFU的意思是：Least Frequently Used最不常用的）
* noeviction：当内存不足以容纳新写入数据时，新写入操作会报错。（默认）
* allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key。（**推荐使用**）
* allkeys-random：当内存不足以容纳新写入数据时，在键空间中，随机移除某个key。
* volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的key。
* volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个key。
* volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先移除。


## redis和数据库双写一致性的问题
如果数据库和缓存双写，就必然会存在缓存不一致的问题，如果对数据有强一致性要求，那么就不能用缓存。我们所采用的策略，只能保证最终一致性，并且从根本上来说，只能说降低不一致发生的概率，完全无法避免。
### 先删除缓存，再更新数据库
如果出现下面的场景：
1. 请求A进行写操作，删除缓存
2. 请求B查询发现缓存不存在，查询得到旧值，并写入缓存
3. 请求A将数据写入数据库
上述情况就会导致不一致的情形出现，并且如果不设置过期时间，该数据永远都是脏数据。这种情况可以使用延时双删策略来解决。步骤为：
1. 先删除缓存
2. 再写入数据库
3. 休眠1s后，再次删除缓存

休眠的时间取决于读数据业务逻辑的耗时。这么做的目的，就是确保读请求结束，写请求可以删除读请求造成的缓存脏数据。
### 先更新数据库，再删除缓存
产生脏数据的概率较小，但是会出现一致性的问题；若更新操作的时候，同时进行查询操作，若hit，则查询得到的数据是旧的数据。但是不会影响后面的查询。

还有一个产生脏数据的可能情况，可能性较低：
一个是读操作，但是没有命中缓存，然后就到数据库中取数据，此时来了一个写操作，写完数据库后，让缓存失效，然后，之前的那个读操作再把老的数据放进去，造成脏数据。

### 异步更新缓存(基于订阅binlog的同步机制)
读取MySQL的binlog进行分析，利用消息队列，推送到redis进行更新。


## 缓存穿透和缓存雪崩
缓存穿透：黑客故意去请求缓存中不存在的数据，导致所有的请求都请求在数据库上，从而连接数据库异常。

缓存雪崩：缓存同一时间大面积失效，这时又来了一波请求，结果请求都请求在数据库上，导致数据库连接异常。