##redis sentinel集群搭建



1. <a href="#redis sentinel 介绍">redis sentinel 介绍</a>
2. <a href="#redis sentinel 配置文件">redis sentinel 配置文件</a>
3. <a href="#redis sentinel 中主库从库切换">redis sentinel 中主库从库切换</a>
4. <a href="#spring data redis 配置访问 redis sentinel集群">spring data redis 配置访问 redis sentinel集群</a>


#####【<a name="redis sentinel 介绍" id="redis sentinel 介绍" ><font color=black>redis sentinel 介绍</font></a>】

Redis Sentinel是Redis2.8版本以后官方提供的集群管理工具，
使用一个或多个sentinel和Redis的master/slave可以组成一个集群，
可以检测master实例是否存活，
并在master实例发生故障时，
将slave提升为master，
并在老的master重新加入到sentinel的集群之后，
会被重新配置，
作为新master的slave。
这意味着基于Redis sentinel的HA群集是能够自我管理的。


#####【<a name="redis sentinel 配置文件" id="redis sentinel 配置文件"><font color=black>redis sentinel 配置文件</font></a>】   

一般可用的redis sentinel 配置文件 sentinel.conf为

 * port 26379; //默认端口26379
 * daemonize yes; //后台启动 
 * logfile logPath; //log文件path
 * sentinel monitor mymaster 192.168.0.100 6380 2; 
 * // mymaster表示master的标识，可以修改，其中2表示如果最少两台sentinel同时检测到master丢失才会做主从切换
 * sentinel down-after-milliseconds mymaster 5000; //表示如果5s内mymaster没响应，就认为SDOWN
 * sentinel parallel-syncs mymaster 1;
 * // 表示如果master重新选出来后，其它slave节点能同时并行从新master同步缓存的台数有多少个，显然该值越大，所有slave节点完成同步切换的整体速度越快，但如果此时正好有人在访问这些slave，可能造成读取失败，影响面会更广。最保定的设置为1，只同一时间，只能有一台干这件事，这样其它slave还能继续服务，但是所有slave全部完成缓存更新同步的进程将变慢
 * sentinel failover-timeout mymaster 15000; // 表示如果15秒后,mysater仍没活过来，则启动failover，从剩下的slave中选一个升级为master

另：一个sentinal可同时监控多个master，只要把4-9行重复多段，加以修改即可。

#####【<a name="redis sentinel 中主库从库切换" id="redis sentinel 中主库从库切换"><font color=black>redis sentinel 中主库从库切换</font></a>】

假设共有三台redis服务器：

一台redis master 配置为

port 6380

一台redis slave 配置为
port 6381

slaveof 127.0.0.1 6380

redis setinel 配置为
port 26379
sentinel monitor mymaster 127.0.0.1 6380 1;

先启动 redis-setinle setinel.conf
要先启动redis的主节点 redis-server redis.master.conf
然后启动redis的从节点

    [17202] 15 Jan 17:58:41.450 # Sentinel runid is 72daed21e4232b77c61f1d958cf3012cdda8c8d3
    [17202] 15 Jan 17:58:41.450 # +monitor master mymaster 127.0.0.1 6380 quorum 1
    [17202] 15 Jan 17:58:41.451 * +slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
 
 可以看到主节点和从节点都加载进来了，然后停掉主节点
 
    [17202] 15 Jan 18:19:08.722 # +sdown master mymaster 127.0.0.1 6380
    [17202] 15 Jan 18:19:08.722 # +odown master mymaster 127.0.0.1 6380 #quorum 1/1
    [17202] 15 Jan 18:19:08.722 # +new-epoch 1
    [17202] 15 Jan 18:19:08.722 # +try-failover master mymaster 127.0.0.1 6380
    [17202] 15 Jan 18:19:08.768 # +vote-for-leader 72daed21e4232b77c61f1d958cf3012cdda8c8d3 1
    [17202] 15 Jan 18:19:08.768 # +elected-leader master mymaster 127.0.0.1 6380
    [17202] 15 Jan 18:19:08.768 # +failover-state-select-slave master mymaster 127.0.0.1 6380
    [17202] 15 Jan 18:19:08.840 # +selected-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
    [17202] 15 Jan 18:19:08.840 * +failover-state-send-slaveof-noone slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
    [17202] 15 Jan 18:19:08.902 * +failover-state-wait-promotion slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
    [17202] 15 Jan 18:19:09.969 # +promoted-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
    [17202] 15 Jan 18:19:09.969 # +failover-state-reconf-slaves master mymaster 127.0.0.1 6380
    [17202] 15 Jan 18:19:09.969 # +failover-end master mymaster 127.0.0.1 6380
    [17202] 15 Jan 18:19:09.969 # +switch-master mymaster 127.0.0.1 6380 127.0.0.1 6381
    [17202] 15 Jan 18:19:09.969 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381
    [17202] 15 Jan 18:19:14.983 # +sdown slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381
    [17202] 15 Jan 18:20:00.558 # -sdown slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381
    
 可以看到80主节点被停掉后，自动把主节点换为81
 
 然后启动80的redis服务
 
    [17202] 15 Jan 18:20:10.497 * +convert-to-slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381
    
可以看到80变为从节点了。
  
#####【<a name="spring data redis 配置访问 redis sentinel集群" id="spring data redis 配置访问 redis sentinel集群"><font color=black>spring data redis 配置访问 redis sentinel集群</font></a>】

完整的配置文件如下，其余部分和普通的redistamplate使用一致

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:mvc="http://www.springframework.org/schema/mvc"
           xmlns:context="http://www.springframework.org/schema/context" xmlns:tx="http://www.springframework.org/schema/tx"
           xmlns:aop="http://www.springframework.org/schema/aop" xmlns:p="http://www.springframework.org/schema/p"
           xsi:schemaLocation="
            http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-3.0.xsd
            http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
            http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.0.xsd
            http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.0.xsd
            http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd">
    
    
        <bean id="redisSentinelConfiguration" class="org.springframework.data.redis.connection.RedisSentinelConfiguration">
            <property name="master">
                <bean class="org.springframework.data.redis.connection.RedisNode">
                    <property name="name" value="mymaster"></property>
                </bean>
            </property>
            <property name="sentinels">
                <set>
                    <bean class="org.springframework.data.redis.connection.RedisNode">
                        <constructor-arg name="host" value="172.17.20.18"></constructor-arg>
                        <constructor-arg name="port" value="26379"></constructor-arg>
                    </bean>
                </set>
            </property>
        </bean>
    
        <bean id="jedisConnectionFactory"
              class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory" p:use-pool="true">
            <constructor-arg ref="redisSentinelConfiguration"/>
        </bean>
    
        <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate"
              p:connection-factory-ref="jedisConnectionFactory">
            <property name="keySerializer">
                <bean class="org.springframework.data.redis.serializer.StringRedisSerializer"/>
            </property>
            <property name="HashKeySerializer">
                <bean class="org.springframework.data.redis.serializer.StringRedisSerializer"/>
            </property>
        </bean>
    
    </beans> 
  
