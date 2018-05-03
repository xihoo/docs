# scan 

************************************************

### 一、jmiss-redis中feature\_redis_vpc分支中的实现

* 支持的语法：scan arg1 arg2
* 参数含义：
  > 1. arg1为redis slot，取值范围为0-16383
  > 2. arg2为cursor游标。

* 集群版返回的结果：三部分
  > 1. 如果当前slot scan完毕（cursor=0）且<16383，返回slot+1。
  > 
  >     如果等于16383，返回0。
  >     
  >     如果大于16383，报错。
  >     
  >     当前slot未scan完毕的（cursor！=0），返回当前slot 
  > 2. 返回下一个cursor
  > 3. 返回keys

* 具体实现：
  > 1. 对每次scan arg1 arg2，只对当前<pre>redisClient->redisDb[slot(即arg1)].dict</pre>执行dictScan()函数
  >       
  >     dictScan()与开源版相同
  > 2. 返回结果的源码：

<pre>

if (!strcasecmp(c->argv[0]->ptr, "scan") && (o == NULL)) {
		addReplyMultiBulkLen(c, 3);

		if (cursor == 0) {
			if ((slot + 1) < REDIS_CLUSTER_SLOTS) addReplyBulkLongLong(c, slot + 1);
			else addReplyBulkLongLong(c, 0);
		} else {
			addReplyBulkLongLong(c, slot);
		}
	} else {
		addReplyMultiBulkLen(c, 2);
	}
    addReplyBulkLongLong(c,cursor);

    addReplyMultiBulkLen(c, listLength(keys));
    while ((node = listFirst(keys)) != NULL) {
        robj *kobj = listNodeValue(node);
        addReplyBulk(c, kobj);
        decrRefCount(kobj);
        listDelNode(keys, node);
    }

</pre>

**************************************

### 二、存在的问题

1. 与开源版redis不一致的参数个数，最好是一个参数（aliyun为一个参数）
2. 在数据量稀少的情况下，最少仍需要scan 16384次才能遍历所有key值

**************************************

### 三、阿里云调研

* 阿里云支持集群版的scan语句，并且有自研的iscan语句
* 阿里云redis集群版scan试用：

<pre>

r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> scan 0
1) "72057594037927936"
2) (empty list or set)
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> scan 72057594037927936
1) "144115188075855872"
2) (empty list or set)
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> scan 144115188075855872
1) "216172782113783808"
2) (empty list or set)
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> scan 216172782113783808
1) "288230376151711744"
2) (empty list or set)
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> scan 288230376151711744
1) "360287970189639680"
2) (empty list or set)
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> scan 360287970189639680
1) "432345564227567616"
2) 1) "hyh"
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> scan 432345564227567616
1) "504403158265495552"
2) (empty list or set)
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> scan 504403158265495552
1) "0"
2) (empty list or set)

</pre>

* 阿里云redis集群版iscan试用：

<pre>

r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> iscan 1 0
1) "0"
2) (empty list or set)
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> iscan 2 0
1) "0"
2) (empty list or set)
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> iscan 3 0
1) "0"
2) (empty list or set)
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> iscan 4 0
1) "0"
2) (empty list or set)
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> iscan 6 0
1) "0"
2) (empty list or set)
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> iscan 5 0
1) "0"
2) 1) "hyh"
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> iscan 7 0
1) "0"
2) (empty list or set)
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> iscan 8 0
(error) ERR node num specified >= node count
r-bp1ab51273eea6d4.redis.rds.aliyuncs.com:6379> iscan 0 0
1) "0"
2) (empty list or set)

</pre>

* 阿里云redis集群有八个主从，scan时是顺序scan的，每次跳转主从scan的游标增加了2的56次方，aliyun每个slot最大存储2的45次方个key（可能因为我试用的64G集群版的？）。
* 试用时存入了160个kv，大约每个主从20个，scan 0后返回的cursor为12，scan 12后返回的cursor是72057594037927936，再次scan返回的cursor大概在2的56次方基础上加10（这段没有保存）

**************************************


### 三、要实现的目标

1. 参数最好由两个变为一个
2. 解决在数据量稀少的情况下scan次数过多的问题

**************************************

### 四、设计方案

* 对目标1：设计语法为:  scan arg
  > 1. 由于slot的取值范围是固定的，可以考虑固定arg的前五位或后五位为slot值，其余为cursor值
  > 2. 确定每个slot存放key值的最大值，(然后？在考虑(不知道阿里云时怎么做的))

* 缺点：slot位数小于5时需要补0，并且需考虑数会大于unsigned long的最大值，要用字符串存储arg

* 对目标2：
  > 考虑设计一个链表（队列），每次scan的时候存入scan的结果，链表长度在count值附近才返回最后结果。
  > 
  > 也可以只有在scan的结果小于一个值的时候才存入链表，大于这个值的时候直接返回结果（这样快一点）
  > 
  > 在cursor=0且slot为2048的整倍数的时候立即返回当前链表中的结果（不跨主从进行scan操作）

* 缺点：其实内部在数据量稀少时仍然是最少循环了16384次，但是用户没有感知。
* 优点：如果要实现类似iscan的操作是很容易的。
