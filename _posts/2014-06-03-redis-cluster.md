---
layout:     post
title:      使用一致性哈希实现Redis分布式缓存
date:       2014-06-03
author:     caodailiang
header-img: img/post-bg-coffee.jpeg
catalog: 	 true
tags:
    - redis
    - redis-cluster
---
## 背景前言
补充说明：本文行文时Redis尚不支持集群。

像Memcache以及其它一些内存K/V数据库一样，Redis本身不提供分布式支持，所以在部署多台Redis服务器时，就需要解决如何把数据分散到各个服务器的问题，并且在服务器数量变化时，能做到最大程度的不令数据重新分布。

通常使用的分布式方法是根据所要存储数据的键的hash值与服务器数量N，按 hash % N 取模的算法来将数据分布到各个服务器。该算法的优点是足够简单，而且数据分布均匀。但是一旦服务器数量N发生变化的时候，缓存命中率会瞬间跌入谷底，因为绝大多数的数据需要重新分布。而且对于大型网站来说，此时会有巨大的压力涌向后端服务，可能会导致性能故障和服务故障，甚至宕机。

本文介绍什么是一致性哈希，为什么要使用以及如何使用一致性哈希算法实现Redis分布式部署，并且引入了虚拟节点以提高数据均匀分布的平衡性。最后还提供了一个故障转移策略，保证在部分服务器故障的时候能迅速恢复命中率，提高Redis服务的可用性和稳定性。

## 一致性哈希
由于hash算法结果一般为unsigned int型，因此对于hash函数的结果应该均匀分布在[0,2^32-1]区间，如果我们把一个圆环用2^32 个点来进行均匀切割，首先按照hash(key)函数算出服务器(节点)的哈希值， 并将其分布到0～2^32的圆环上。

用同样的hash(key)函数求出需要存储数据的键的哈希值，并映射到圆环上。然后从数据映射到的位置开始顺时针查找，将数据保存到找到的第一个服务器(节点)上。

如图所示：

![](https://caodailiang.github.io/img/posts/redis-cluster-consistent-hash-ring.jpg)

key1、key2、key3和server1、server2通过hash都能在这个圆环上找到自己的位置，并且通过顺时针的方式来将key定位到 server。按上图来说，key1和key2存储到server1，而key3存储到server2。如果新增一台server，hash后在key1 和key2之间，则只会影响key1(key1将会存储在新增的server上)，其它则不变。

## 虚拟节点

在上图中，很容易看出一个问题，沿顺时针方向看，server2到server1之间的区间跨度大，而server1到server2的区间跨度小，这就会导致一个问题：数据分布不均匀。大部分数据都分配到server1了，只有小部分数据分布在server2。在服务器数据很少的时候，数据不均匀会表现的非常明显。

解决这个问题的方法是使用虚拟节点，一个真实服务器对应多个虚拟节点，所有虚拟节点按hash值分布在一致性哈希圆环上。具体实现方法可以这样做，为真实服务器设置副本数量，然后根据各真实服务器的IP和端口号再加上一个递增的索引数计算hash值。

```
<?php
class RedisCache {
    public $servers = array();  //真实服务器
    private $_servers = array();    //虚拟节点
    const SERVER_REPLICAS = 10000; //服务器副本数量，提高一致性哈希算法的数据分布均匀程度
    
    public function __construct( $servers ){
        $this->servers = $servers;

        //Redis虚拟节点哈希表
        foreach ($this->servers as $k => $server) {
            for ($i = 0; $i < self::SERVER_REPLICAS; $i++) {
                $hash = crc32($server['host'] . '#' .$server['port'] . '#'. $i);
                $this->_servers[$hash] = $k;
            }
        }
        ksort($this->_servers);

        // something else...
    }
}
```

副本数量可以按真实服务器的数量调节，真实服务器多则副本数量可以设置小一点，真实服务器少则副本数量需要设置多一点。在虚拟节点数量很大的时候，出于性能考虑所以不能使用循环的方法查找key对应的虚拟节点，可以使用二分法快速查找。一个优化的算法是不论副本数量设置为10，100，还是10000的时候，对查找所需要的时间基本是没有影响的。

## 故障转移

使用一次性哈希实现Redis分布式部署了，还需要考虑系统的可用性和稳定性。需要做到，在某一台或者多台server故障的时候，程序能够自动检测到故障，并将数据重新定位到其它server。

我们可以考虑，根据key查找到的虚拟节点所对应的真实服务器故障的时候，我们在一次性哈希圆环上沿顺时针方向顺移一步，找到下一点虚拟节点对应的真实服务器，将所要存储的数据存放上去。但也很有可能下一个虚拟节点所对应的真实服务器与前一个虚拟节点相同，还是那台故障的服务器，而每次尝试连接故障的redis服务是一个很大的性能开销。所以在第一次检测到故障服务器的时候就需要记录下来，然后在顺移到下一个虚拟节点的时候先判断是不是之前那一台故障的服务器，如果是那就不要再尝试进行连接，直接查找下一个虚拟节点，直到找到可用的服务器将数据存储上去。

## 完整代码示例
PHP SDK代码示例如下：
```
<?php
class RedisCache {
    public $servers = array();  //真实服务器

    private $_servers = array();    //虚拟节点

    private $_serverKeys = array();
   
    private $_badServers = array(); // 故障服务器列表
   
    private $_count = 0;

    const SERVER_REPLICAS = 10000; //服务器副本数量，提高一致性哈希算法的数据分布均匀程度
   
    public function __construct( $servers ){
        $this->servers = $servers;
        $this->_count = count($this->servers);

        //Redis虚拟节点哈希表
        foreach ($this->servers as $k => $server) {
            for ($i = 0; $i < self::SERVER_REPLICAS; $i++) {
                $hash = crc32($server[ 'host'] . '#' .$server['port'] . '#'. $i);
                $this->_servers [$hash] = $k;
            }
        }
        ksort( $this->_servers );
        $this->_serverKeys = array_keys($this->_servers);
    }
   
    /**
     * 使用一致性哈希分派服务器，附加故障检测及转移功能
     */    
    private function getRedis($key){
        $hash = crc32($key);
        $slen = $this->_count * self:: SERVER_REPLICAS;

        // 快速定位虚拟节点
        $sid = $hash > $this->_serverKeys[$slen-1] ? 0 : $this->quickSearch($this->_serverKeys, $hash, 0, $slen);

        $conn = false;
        $i = 0;
        do {
            $n = $this->_servers [$this->_serverKeys[$sid]];
            !in_array($n, $this->_badServers ) && $conn = $this->getRedisConnect($n);
            $sid = ($sid + 1) % $slen;
        } while (!$conn && $i++ < $slen);
       
        return $conn ? $conn : new Redis();
    }
   
    /**
     * 二分法查找
     */
    private function quickSearch($stack, $find, $start, $length) {
        if ($length == 1) {
            return $start;
        }
        else if ($length == 2) {
            return $find <= $stack[$start] ? $start : ($start +1);
        }
       
        $mid = intval($length / 2);
        if ($find <= $stack[$start + $mid - 1]) {
            return $this->quickSearch($stack, $find, $start, $mid);
        }
        else {
            return $this->quickSearch($stack, $find, $start+$mid, $length-$mid);
        }
    }
   
    private function getRedisConnect($n=0){
        static $REDIS = array();
        if (!$REDIS[$n]){
            $REDIS[$n] = new Redis();
            try{
                $ret = $REDIS[$n]->pconnect( $this->servers [$n]['host'], $this->servers [$n]['port']);
                if (!$ret) {
                    unset($REDIS[$n]);
                    $this->_badServers [] = $n;
                    return false;
                }
            } catch(Exception $e){
                unset($REDIS[$n]);
                $this->_badServers [] = $n;
                return false;
            }
        }
        return $REDIS[$n];
    }
   
    public function getValue($key){
        try{
            $getValue = $this->getRedis($key)->get($key);
        } catch(Exception $e){
            $getValue = null;
        }

       return $getValue;
    }
   
    public function setValue($key,$value,$expire){
        if($expire == 0){
            try{
                $ret = $this->getRedis($key)->set($key, $value);
            } catch(Exception $e){
                $ret = false;
            }
        } else{
            try{
                $ret = $this->getRedis($key)->setex($key, $expire, $value);
            } catch(Exception $e){
                $ret = false;
            }
        }
        return $ret;
    }
   
    public function deleteValue($key){
        return $this->getRedis($key)->delete($key);
    }
   
    public function flushValues(){
        //TODO
        return true;
    }
}
```
使用示例：
```
<?php
$redis_servers = array(
       array(
             'host'       => '10.0.0.1',
             'port'       => 6379,
      ),
       array(
             'host'       => '10.0.0.2',
             'port'       => 6379,
      ),
       array(
             'host'       => '10.0.0.3',
             'port'       => 6379,
      ),
       array(
             'host'       => '10.0.0.3',
             'port'       => 6380,
      ),
);

$redisCache = new RedisCache($redis_servers);
$testKey = 'test_key';
$testValue = 'test_value_object';
$redisCache->setValue($testKey, $testValue, 3600);
$value = $redisCache->getValue($testKey);
```

## 进阶：缓存代理中间件
实际目标：
1. 将缓存服务器的部署配置与应用隔离，缓存配置的调整，容量的增减，对应用完全透明。
2. 实现分布式，并具备故障转移功能。
3. 提高TCP链接使用效率。
4. 缓存池功能，根据KEY前缀的不同分配到不同的缓存池。（*）
5. 统计各类KEY的使用情况，便于分析和提高缓存资源使用的合理性，制定缓存使用规范。（*）

设计方案

![](https://caodailiang.github.io/img/posts/redis-cluster-proxy.png)

缓存代理部署在各台应用服务器上，针对各台远程缓存服务器建立N条长连接放入连接池，当应用发请缓存请求的时，从连接池中取空闲链接直接复用。如果链接池为空则建立新的链接。应用通过Unix Domain Socket方式监听本地socket，避开建立网络连接开销。

缓存代理内部使用一致性哈希算法实现分布式，设置虚拟节点提高数据分布均匀程度。并在一台缓存服务器发生故障时，沿哈希圆环将请求顺移至下一台服务器，以实现故障转移功能。

使用etcd为缓存代理提供配置服务，各缓存中间件监听etcd上的配置变化，在配置变动时，自动调整连接池及一致性哈希圆环节点。另外，提供配置管理页面及API，方便管理缓存服务器配置，及显示一些状态信息。

另外，鉴于Redis能实现Memcache的所有功能，只因使用单线程模型而在性能上略有损失，但此劣势可以忽略不计。所以此缓存中间件只考虑Redis做为后端缓存服务器，不兼容Memcache。