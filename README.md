# How to Use MySQL in Python
记录本人在python中使用MySQL所遇到过的问题，问题主要又分为三大部分：
* DataBase API, 讨论Python数据库API
* Connection Pool, 讨论连接池使用中遇到的问题.
* MySQL, 讨论MySQL本身所带来的问题.

## 一、DataBase API
对于Python的数据库驱动API，首先需要关注和熟悉PEP(Python Enhancement Proposals)的建议，其中大部分常用的的Python-MySQL-API都会遵循[PEP249](https://legacy.python.org/dev/peps/pep-0249/)的建议，常见的数据库连接池也认为连接对象符合PEP249的规定。
### 1.*构造函数*

### 2.*连接的事务*

## 二、Connection Pool
在后台程序中通常不会直接使用数据库连接，而是使用数据库连接池(pool)，线程从pool中获取一个可用的连接，再通过连接操控db，完成操作后需要释放连接。最常用的连接池就是PoolDB，这是DBUtils包下的一个模块。
```py
$ pip install DBUtils

from DBUtils.PooledDB import PooledDB
```

### 1.*构造函数*
首先看下PoolDB构建所需要的参数，这些参数和PoolDB的性能紧密相关：
```py
class PooledDB:
    def __init__(self,
            # 用来创建连接的对象，需要符合PEP249协议, 如MySQLdb, mysql.connector等
            creator,
            # 最小缓存的连接数, pool初始化时将会预先建立mincached个数的连接
            mincached=0,
            # 最大缓存的连接数, pool的连接个数超过maxcached个数的连接后，将会删除多余的连接。0表示不回收任何连接。
            maxcached=0,
            # 当连接数达到maxshared后，开始复用连接(0表示连接不进行复用)。该参数受数据库驱动的threadsafety参数影响。
            maxshared=0,
            # 连接数上限, 0为不设上限，每当连接达到这个数时，将会阻塞或是抛出异常。shared机制下，不受该参数影响。
            maxconnections=0,
            # 达到最大连接数，再申请连接抛出异常或阻塞
            blocking=False,
            maxusage=None,
            setsession=None,
            reset=True,         # 连接归还给pool时，是否进行重置
            failures=None,
            ping=1,             # connection进行检查连接的条件
            *args,
            **kwargs):
        pass
```
#### 1).mincached机制
mincache是预先建立连接的个数， 在pool的__init__函数中进行:
```py
class PooledDB:
    def __init__(...):
        ...
        # 连接池队列
        self._idle_cache = []
        self._lock = Condition()
        self._connections = 0
        # dedicated_connection是建立新连接的函数， 这里就会建立mincached数量的连接。
        idle = [self.dedicated_connection() for i in range(mincached)]
        # 建立的连接并没有放到_idle_cache中，需要通过close()函数，会自动将连接放到池子中。
        while idle:
            idle.pop().close()
```
#### 2).maxcached机制
在回收一个从pool中获取的连接时，将会检查pool是否超过了_maxcached，若超过了_maxcached则会丢弃掉该连接。若_maxcached为0，则不会回收任何连接。
```py
class PooledDedicatedDBConnection:
    def close(self):
        if self._con:
            # 连接对象的关闭，将会把连接放回到pool中
            self._pool.cache(self._con)
            self._con = None

class PooledDB:
    ...
    def cache(self, con):
        self._lock.acquire()
        try:
            # 这里判断，如果_maxcached为0，或是_idle_cache没达到_maxcached，则进行放回pool，否则丢弃改连接。
            if not self._maxcached or len(self._idle_cache) < self._maxcached:
                con._reset(force=self._reset)
                self._idle_cache.append(con)
            else:
                con.close()
            self._connections -= 1
            self._lock.notify()
        finally:
            self._lock.release()
```
#### 3).maxconnections机制

#### 4).maxshared机制
shared机制用来允许一个连接被多个线程复用，当非空闲连接个数达到maxshared时，将会使连接在线程间共享。在初始化时，将会根据threadsafety来计算pool可以接受的maxshared，如下代码:
```py
def __init__(...):
    ...
    # 获取threadsafety
    try:
        threadsafety = creator.threadsafety
    except AttributeError:
        try:
            if not callable(creator.connect):
                raise AttributeError
        except AttributeError:
            threadsafety = 2
        else:
            threadsafety = 0
    # 计算_maxshared
    if not threadsafety:
        raise NotSupportedError("Database module is not thread-safe.")
    if threadsafety > 1 and maxshared:
        self._maxshared = maxshared
        self._shared_cache = []
    else:
        self._maxshared = 0
    ...
```
很明显，只有当threadsafety大于1时，才会真正的使用__init__参数中的maxshared，否则maxshared为0.对于threadsafety的含义参考`DataBase API`中的描述。常用的MySQL数据库驱动库MySQLdb, mysql.connector等，该参数都是1(线程不安全，不应该被多线程共享同一个连接)，也就是说通常我们不会用到maxshared机制。

### 2.*如何归还连接*
最开对于连接的归我是懵逼的，虽然理所应当是`conn.close()`，但PEP249规定这个是关闭连接的操作，而非归还连接。观察源码后才发现，PoolDB对连接做了一层包装对象，包装会将除了`close()`方法全部路由给实际的连接，但是对于close函数，只会把连接归还给线程池。
```py
class PooledDedicatedDBConnection:
    # 归还给连接池
    def close(self):
        if self._con:
            self._pool.cache(self._con)
            self._con = None

    def __getattr__(self, name):
        # 将会代理除了close的其他所有方法和属性
        if self._con:
            return getattr(self._con, name)
        else:
            raise InvalidConnection
```
另外一点，pool中实际保存的是连接对象，而不是连接的包装对象，在每次调用`pool.connection()`时，将会从队列中获得一个连接，并且对该连接进行包装：
```py
# 下面是【伪码】
def connection(self):
    # 操作线程池的锁
    self._lock.acquire()
    # 若达到了最大的线程，直接阻塞
    while (self._maxconnections
            and self._connections >= self._maxconnections):
        self._wait_lock()
    # 从队列中取出一个连接
    con = self._idle_cache.pop(0)
    # 包装连接
    con = PooledDedicatedDBConnection(self, con)
    self._connections += 1
    self._lock.release()
    return con
```
上述只是一个简化的实现，并非PoolDB源码。

### 3.*归还连接的时机*
我们的代码中通常会编写如下的代码来`显示释放`连接:
```py
conn, cursor = None, None
conn = pool.connection()
cursor = conn.cursor()
try:
    execute_db(conn, cursor)
finally:
    cursor and cursor.close()
    conn and conn.close()
```
但是我们会发现，就算不进行`conn.close()`，连接也会被归还给pool。这是因为conn是一个被包装的对象，包装类实现了`__del__`函数，会在对象被垃圾回收的时候触发:
```py
class PooledDedicatedDBConnection:
    ...
    # 垃圾回收时将会触发
    def __del__(self):
        try:
            self.close()
        except Exception:
            pass
```

### 4.*并发取出连接的性能问题*
这是一个相当重要的问题，PoolDB在面临大量的线程申请获得连接时，将会有非常严重的性能问题，类似如下的数据库操作:
```py
def execute():
    conn, cursor = None, None
    conn = pool.connection()
    cursor = conn.cursor()
    try:
        execute_db(conn, cursor)
    finally:
        cursor and cursor.close()
        conn and conn.close()

def execute_parallel(num):
    threads = []
    # 开启大量线程同时获取连接对象，并进行sql操作
    for i in range(num):
        t = threading.Thread(target=query)
        threads.append(t)
        t.start()
    # 等待线程结束
    for t in threads:
        t.join()

execute_parallel(10)
```
执行上述的代码，将会发现一个非常诡异的现象: 无论pool首先启动了多少个连接，每个线程的sql处理耗时，都会随着num的增大而线性增大。这种场景在后台面临高并发的时候将会经常遇到，上千个请求同时打到后台服务器，后台服务器无论是多线程或是协程进行db处理，都需要向pool申请连接。这主要是因为`pool.connection()`的耗时非常大，100个请求就有几百毫秒的获取请求的阻塞，并发性能怎么也上不去。分析源码后得知，pool中通过一个队列来缓存连接对象:
* 为了保证队列的线程安全性，会申请独占锁
* 为了保证连接的可用性，会进行ping_check(), 毫秒级耗时(少则几毫秒)。

这两步操作锁导致获取连接阻塞的关键因素: 一开始大量的线程建立起来，获取连接对象并被阻塞，每次只能一个线程获取一个连接, 每次获取连接都会进行几毫秒的延时，这会导致耗时的积累，并且越到后面的线程阻塞的时间就越长:
```py
def connection(self, shareable=True):
    self._lock.acquire()
    try:
        while (self._maxconnections
                and self._connections >= self._maxconnections):
            self._wait_lock()
            con = self._idle_cache.pop(0)
        except IndexError:
            con = self.steady_connection()
        else:
            # 检查连接可用性 耗时较大
            con._ping_check()
        con = PooledDedicatedDBConnection(self, con)
        self._connections += 1
    finally:
        self._lock.release()
    return con
```
阻塞的耗时主要是对连接的检查, 可以通过初始化时指定`ping=0`来避免对连接的可用性进行检查, 减少阻塞时间, 但是解决的并不彻底，一方面这不能保障连接的可用性，另一方面连接在归还给pool的时候也会上锁阻塞, 阻塞期间会进行连接的reset操作(后面将会有专门提及), 导致高并发sql操作在释放给pool也会导致较为严重的阻塞.

为了解决对pool操作的串行化问题, 可以采用pools(创建多个pool)的方案, 将串行压力从一个池分摊给多个池, 如果需要初始化pool设置`ping=0`:
```py
import random

pools_size = 10
pools = []
for idx in range(pools_size):
    pools.append(PooledDB(..., ping=0))

def execute():
    conn, cursor = None, None
    # 最简单的负载均衡方案，随机选择一个
    random_idx = random.randint(0, pools_size-1)
    conn = pools[random_idx].connection()
    cursor = conn.cursor()
    try:
        execute_db(conn, cursor)
    finally:
        cursor and cursor.close()
        conn and conn.close()

def execute_parallel(num):
    threads = []
    # 开启大量线程同时获取连接对象，并进行sql操作
    for i in range(num):
        t = threading.Thread(target=query)
        threads.append(t)
        t.start()
    # 等待线程结束
    for t in threads:
        t.join()

execute_parallel(10)
```
### 5.*并发连接归还的性能问题*

### 6.*share机制*
在`构造函数`一节中有对shared机制有做一个简单的介绍，该机制在MySQL中并不常用，这里详细介绍该机制的原理和目的。

shared机制下，pool中维护了两个队列:
* _idle_cache，用于存放空闲连接的队列，通常在pool初始化时会放入一些空闲连接，以及当一个连接的被线程引用的个数为0时，将会被放入空闲连接队列。
* _shared_cache，用于存放非空闲连接的队列。

取连接时，_shared_cache已经满了，则会从_shared_cache中取出连接进行复用，_shared_cache未满(使用的连接未达到_maxshared)，则从_idle_cache取出空闲连接，若_idle_cache无空闲连接，则会新创建一个连接。从_idle_cache中取出来的或是新创建的连接，都会通过`SharedDBConnection`进行包装，放入_shared_cache中。
#### 1).连接的取出
在`pool.connection()`操作中将会取出连接，在源码中将会通过_maxshared参数来进行判断:
```py
def connection(self, shareable=True):
    # 可以看到默认是shareable=True，因此通常时通过_maxshared进行判断，该参数在pool初始化时决定
    if shareable and self._maxshared:
        # share机制下的连接取出
```
接下来单独看shared机制的连接取出代码:
```py
# 对pool上锁，确保数据结构的线程安全
self._lock.acquire()
# _shared_cache为空时，如果连接数达到了上限，直接进行阻塞，知道连接被归还
# 如果_shared_cache不为空，是不会wait的，因为这时会去取_shared_cache中的连接来进行复用
# shared机制通常不会进行wait，因为一旦有连接，该连接就会放到_shared_cache中，直到该连接没有被任何线程所引用。
while (not self._shared_cache and
        self._maxconnections
        and self._connections >= self._maxconnections):
    self._wait_lock()
if len(self._shared_cache) < self._maxshared:
    # shared_cache未满，从空闲队列中取出连接，或是创建新的连接
    try:
        con = self._idle_cache.pop(0)
    except IndexError:
        con = self.steady_connection()
    else:
        con._ping_check()
    # 将一个连接用SharedDBConnection进行包装
    # SharedDBConnection主要提供连接的引用计数，以及为其排序提供函数。
    con = SharedDBConnection(con)
    self._connections += 1
else: # shared_cache满了，从shared_cache中取出连接进行复用
    # sort将会用cache中的引用次数生序排列，以便取出引用个数最少的连接
    self._shared_cache.sort()
    con = self._shared_cache.pop(0)
    # 如果连接在事务中，连接不应该进行共享，并等待，唤醒后重新取出连接个数最少的
    # 可以看出，如果使用shared机制，应该尽量不使用事务，避免阻塞降低并发能力
    while con.con._transaction:
        self._shared_cache.insert(0, con)
        self._wait_lock()
        self._shared_cache.sort()
        con = self._shared_cache.pop(0)
    con.con._ping_check()
    con.share()
self._shared_cache.append(con)
self._lock.notify()
self._lock.release()
# PooledSharedDBConnection进行包装，该包装主要提供conn回收的能力。
con = PooledSharedDBConnection(self, con)
```

#### 2).连接的归还
调用`conn.close()`，将会归还连接，在shared机制下归还有两层含义：
* 连接的引用计数减少。
* 连接的引用计数为0时，将连接从shared_cache取出，并放入idle_cache(是否真的归还，还需要依靠maxcached参数决定时归还还是销毁)

shared机制下的连接被`PooledSharedDBConnection`包装，该类提供了归还的函数, 来看看如何归还的:
```py
class PooledSharedDBConnection:
    ...
    def close(self):
        if self._con:
            # 减少引用计数，并判断是否已经空闲
            self._pool.unshare(self._shared_con)
            self._shared_con = self._con = None

    def __del__(self):
        try:
            self.close()
        except Exception:
            pass

# pool的unshare函数:
def unshare(self, con):
    self._lock.acquire()
    try:
        # 减少引用计数
        con.unshare()
        shared = con.shared
        # 引用计数为0，从shared_cache中删除
        if not shared:
            self._shared_cache.remove(con)
    finally:
        self._lock.release()
    # 引用计数为0，将连接放回给idle_cache
    if not shared:
        self.cache(con.con)
```

### 6.*DBUtils中封装了多少连接类*
从pool中拿到的连接的类并不是简单的MySQLdb或mysql.connector所生成的连接，而是对这些连接类做了层层封装:
* 非shared机制下，从pool拿到的连接类的包装关系:
    * PooledDedicatedDBConnection( SteadyDBConnection( MySQLdb ) )
* share机制下，从pool拿到的连接类的包装关系:
    * PooledSharedDBConnection( SharedDBConnection( SteadyDBConnection( MySQLdb ) ) )

可以看到Shared中可以看到封装了4个连接的包装类，可以分为两类:
* 包装类提供归还连接到pool时的逻辑
    * PooledDedicatedDBConnection, 提供对专有连接的归还处理。
    * PooledSharedDBConnection, 提供对复用连接的归还处理：减少引用计数，为0时将连接从shared_cache移到idle_cache或是删除。
* 包装类对连接的强化
    * SteadyDBConnection, 用来提供稳定的连接，例如提供ping_check检查连接。后面对该类有更加具体的分析。
    * SharedDBConnection, 用来提供对连接的引用计数，一般包装满足Database API的类，在DBUtils中，封装的就是SteadyDBConnection。

### 7.*DBUtils中封装了多少连接池类*

### 8.*SteadyDBConnection的机理*
