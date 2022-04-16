# 利用zookeeper实现分布式锁

- 主要利用了zookeeper的watcher的特性，所有线程都去zookeeper的/zk_lock下创建临时顺序节点/zk_lock/sub_xxx
- 采用一种先来先到的思想，xxx小的优先获得锁，当释放锁时，可以删除xxx对应的数据节点
- 那么就可以利用watcher监听对应的数据节点是否被删除，假如xxx1 = 12，那么当xxx1被删除的时候,xxx2 = 13就能够获取锁，而xxx2 = 13对应的线程就可以监听xxx1 = 12的数据节点的变化。
- 由于拿到锁的情况下，全局拿到锁的线程，对应唯一一个数据节点 curPath，这个curPath可以认为它是线程安全的，不需要额外对它做并发控制，因此，解锁操作额外简单，删除curPath对应的数据节点即可。

## 原生API

```java
import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.Collections;
import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.concurrent.CountDownLatch;

public class DLock {

    private final String LOCK_PARENT = "/zk_lock";

    private final String LOCK_CHILD = "sub_";

    private volatile String curPath = null;

    private final Set<String> set = new HashSet<>();

    private final ZooKeeper zk;

    public DLock() throws IOException, InterruptedException, KeeperException {
        CountDownLatch start = new CountDownLatch(1);
        this.zk =  new ZooKeeper("192.168.200.132:2181",
                5000,
                new Watcher() {
                    @Override
                    public void process(WatchedEvent event) {
                        //do nothing
                        if(Event.KeeperState.SyncConnected == event.getState()){
                            start.countDown();
                        }
                    }
                });
        start.await();
        System.out.println("连接完成...");
        if(zk.exists(LOCK_PARENT,false) == null){
            zk.create(LOCK_PARENT,"".getBytes(StandardCharsets.UTF_8), ZooDefs.Ids.OPEN_ACL_UNSAFE,CreateMode.PERSISTENT);
        }
    }

    public void acquire() throws KeeperException, InterruptedException {
        String curPath = zk.create(LOCK_PARENT + "/" + LOCK_CHILD, "".getBytes(StandardCharsets.UTF_8),
                ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        List<String> children = zk.getChildren(LOCK_PARENT,false);
        if(children.size() == 1){
            //拿到锁
            this.curPath = curPath;
            return;
        }
        Collections.sort(children);
        String curNode = curPath.substring(LOCK_PARENT.length() + 1);
        int idx = children.indexOf(curNode);
        if(idx == 0){
            //排在第一位,立即可以拿锁
            this.curPath = curPath;
            return;
        }
        CountDownLatch door = new CountDownLatch(1);
        String prevNode = children.get(idx - 1);
        try {
            zk.getData(LOCK_PARENT + "/" + prevNode, new WaitWatcher(door), new Stat());
            door.await();
        }catch (Exception e){
            //报异常说明在我们getData之前,之前的prevNode就已经被删掉了,此时这个异常必须捕捉住
            //而且可以知道door.await()不会执行,即此时可以不用阻塞,直接拿锁就好了
        }
        //排在前一位的释放了锁,现在当前线程可以拿锁
        this.curPath = curPath;
    }

    public void release() throws KeeperException, InterruptedException {
        zk.delete(this.curPath,-1);
    }

    static class WaitWatcher implements Watcher {

        private final CountDownLatch door;

        WaitWatcher(CountDownLatch door) {
            this.door = door;
        }

        @Override
        public void process(WatchedEvent event) {
            if(event.getType() == Event.EventType.NodeDeleted){
                door.countDown();
            }
        }
    }

}

```



## 自定义curator的分布式锁逻辑

- 需要注意的就是curator的NodeCache无法监听数据节点本身是否被删除
- 因此监听的时候需要采用原生的监听器

```java
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.data.Stat;

import java.util.Collections;
import java.util.List;
import java.util.concurrent.CountDownLatch;

/**
 * 尝试一个单例分布式锁
 */
public class MyLock {

    private final String LOCK_PARENT = "/zk_lock";

    private final String LOCK_CHILD = "sub_";

    private final CuratorFramework client;

    private volatile String curPath = null;

    private MyLock() {
        client = CuratorFrameworkFactory.builder()
                .connectString("192.168.200.132:2181")
                .retryPolicy(new ExponentialBackoffRetry(1000,3)).build();
        client.start();
    }

    public void acquire() throws Exception {
        String curPath = client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL_SEQUENTIAL)
                .forPath(LOCK_PARENT + "/" + LOCK_CHILD);
        List<String> children = client.getChildren().forPath(LOCK_PARENT);
        if(children.size() == 1){
            //说明是第一个,它能获取锁,直接返回即可
            this.curPath = curPath;
            return;
        }
        //字典序排序,找序号最小的节点
        Collections.sort(children);
        //client.getChildren()返回的是相对路径
        String curNode = curPath.substring(LOCK_PARENT.length() + 1);
        //此时需要阻塞等待它前一个节点释放锁后,目前线程才能拿锁
        CountDownLatch door = new CountDownLatch(1);
        int idx = children.indexOf(curNode);
        if(idx == 0){
            this.curPath = curPath;
            return;
        }
        String prevNode = children.get(idx - 1);
        //System.out.println("等待prevNode被删除" + prevNode);
        Stat stat = client.checkExists().usingWatcher(new WaitWatcher(door)).forPath(LOCK_PARENT + "/" + prevNode);
        if(stat != null) {
            //如果为null,说明前面一个节点被删除了,那么当前线程不用阻塞,直接可以拿锁
            door.await();
        }
        this.curPath = curPath;
    }

    public void release() throws Exception {
        //System.out.println(Thread.currentThread().getName() + "释放锁..." + curPath);
        client.delete().guaranteed().forPath(curPath);
    }

    public static MyLock getInstance(){
        return MyLockHelper.INSTANCE;
    }

    static class MyLockHelper{
        private static final MyLock INSTANCE = new MyLock();
    }

    static class WaitWatcher implements Watcher{

        private final CountDownLatch door;

        WaitWatcher(CountDownLatch door) {
            this.door = door;
        }

        @Override
        public void process(WatchedEvent event) {
            if(event.getType() == Event.EventType.NodeDeleted){
                door.countDown();
            }
        }
    }
}

```



## 测试

```java
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.CountDownLatch;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;

//使用Curator实现分布式锁功能
public class Recipes_Lock {

	public static void main(String[] args) throws Exception {
        //curator内置了分布式锁,可以直接拿来使用
        //final InterProcessMutex lock = new InterProcessMutex(client,lock_path);
		//final MyLock lock = MyLock.getInstance();
		final DLock lock = new DLock();
		
		final CountDownLatch down = new CountDownLatch(1);
		for(int i = 0; i < 10; i++){
			new Thread(new Runnable() {
				public void run() {
					try {
						down.await();
						for(int j = 0;j < 50;j++){
							lock.acquire();
							SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss|SSS");
							String orderNo = sdf.format(new Date());
							System.out.println(Thread.currentThread().getName() + "生成的订单号11是 : " + orderNo);
							lock.release();
						}
					}catch (Exception e){
						e.printStackTrace();
					}
				}
			}).start();
		}
		down.countDown();
	}
}
```



## watcher---数据变更的通知

Zookeeper允许客户端向服务端注册一个Watcher监听，当服务端的一些指定事件触发了这个Watcher，那么就会向指定客户端发送一个事件通知来实现分布式的通知功能。

![image-20220416093248883](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416093248883.png)



### watcher接口

process方法是Watcher接口的一个回调方法，当Zookeeper向客户端发送一个Watcher事件通知时，客户端就会对相应的process方法进行回调，从而实现对事件的处理。

```java
abstract public void process(WatchedEvent event);
```

![image-20220416093845419](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416093845419.png)



### WatchedEvent和WatcherEvent的关系

- WatchedEvent是一个逻辑事件，用于服务端和客户端程序执行过程中所需的逻辑对象
- WatcherEvent实现了序列化接口，因此可以用来网络传输

![image-20220416094043093](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416094043093.png)

- 服务端生成WatchedEvent事件之后，会调用getWrapper方法将自己包装成一个可序列化的WatcherEvent事件，以便通过网络传输到客户端。
- 客户端接收到服务端的这个WatcherEvent事件对象后，首先将WatcherEvent还原成WatchedEvent事件，并传递给process方法处理。

**举个例子，当/zk-book这个节点的数据发生变化后，服务端会发送给客户端一个“ZNode数据内容变更“事件，客户端只能够接收到如下信息：**

- KeeperState:SyncConnected
- EventType:NodeDateChanged
- Path:/zk-book

**客户端无法直接从该事件中获取对应的数据节点的原始内容以及变更过后的新数据内容，而需要客户端按需再次去重新获取数据，这是Zookeeper Watcher机制的一个重要特性。**



### 工作原理

Zookeeper的Watcher机制，总的来说可以概括为以下三种类型：

- 客户端注册Watcher
- 服务端处理Watcher
- 客户端回调Watcher

![image-20220416094714725](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416094714725.png)



#### 客户端注册Watcher

构造方法中可以传入一个默认的Watcher

```java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher) 
```

这个Watcher作为整个Zookeeper会话期间的默认Watcher，会被一直保存在客户端ZKWatchManager的defaultWatcher中。

- getData、getChildren、exist都可以注册Watcher，但是原理都一样

```java
//watch=true,那么使用defaultWatcher
public byte[] getData(String path, boolean watch, Stat stat);
//也可以自己传入一个Watcher
public byte[] getData(final String path, Watcher watcher, Stat stat);
```

按顺序慢慢往下看。

```java
    public byte[] getData(final String path, Watcher watcher, Stat stat)  {
        ...
        WatchRegistration wcb = null;
        if (watcher != null) {
            //封装一个watcher的注册信息,暂时保存数据节点的路径和Watcher的对应关系
            wcb = new DataWatchRegistration(watcher, clientPath);	
        }
		...
        request.setWatch(watcher != null);	//对客户端请求标记,使用watcher监听
        ReplyHeader r = cnxn.submitRequest(h, request, response, wcb);	//接下来看这个方法
        ...
    }
```

![image-20220416100712320](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416100712320.png)

![image-20220416101146785](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416101146785.png)

![image-20220416101407306](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416101407306.png)

##### 客户端注册流程图

![image-20220416101437923](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416101437923.png)



#### 服务端处理Watcher

![image-20220416101803576](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416101803576.png)

1、Server存储

![image-20220416101747652](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220416101747652.png)

![image-20220416101920097](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416101920097.png)

2、Watcher触发

![image-20220416102026296](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416102026296.png)

![image-20220416102037514](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416102037514.png)

![image-20220416102052559](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416102052559.png)

3、触发逻辑

![image-20220416102202403](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416102202403.png)

![image-20220416102208813](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416102208813.png)

![image-20220416102225468](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416102225468.png)

![image-20220416102256291](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416102256291.png)

**ServerCnxn的process的逻辑非常简单，本质上并不是处理客户端Watcher的业务逻辑，而是借助客户端连接的ServerCnxn对象来实现客户端的WatchedEvent传递，真正的客户端Watcher回调与业务逻辑执行都在客户端。**



#### 客户端回调Watcher

服务端最终通过使用ServerCnxn对应的TCP连接来向客户端发送一个WatcherEvent事件，下面看看客户端如何处理这个事件。

1、SendThread接收事件通知。

![image-20220416102805192](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416102805192.png)

![image-20220416102813354](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220416102813354.png)

![image-20220416102827365](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416102827365.png)

2、EventThread处理事件通知

![image-20220416103115429](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416103115429.png)

```java
       
	private void queueEvent(WatchedEvent event, Set<Watcher> materializedWatchers) {
            if (event.getType() == EventType.None && sessionState == event.getState()) {
                return;
            }
            sessionState = event.getState();
            final Set<Watcher> watchers;
            if (materializedWatchers == null) {
                // materialize the watchers based on the event
                // //首先根据通知事件,从ZKWatchManager中取出所有相关的Watcher 
                watchers = watchManager.materialize(event.getState(), event.getType(), event.getPath());
            } else {
                watchers = new HashSet<>(materializedWatchers);
            }
            WatcherSetEventPair pair = new WatcherSetEventPair(watchers, event);
            // queue the pair (watch set & event) for later processing
            waitingEvents.add(pair);
        }
```

![image-20220416103450439](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416103450439.png)

**获取到所有相关watcher后，放入waitingEvents队列中去。EventThread的run方法会不断处理该队列**

```java
        public void run() {
            try {
                isRunning = true;
                while (true) {
                    Object event = waitingEvents.take();
                    if (event == eventOfDeath) {
                        wasKilled = true;
                    } else {
                        processEvent(event);
                    }
                    if (wasKilled) {
                        synchronized (waitingEvents) {
                            if (waitingEvents.isEmpty()) {
                                isRunning = false;
                                break;
                            }
                        }
                    }
                }
            } catch (InterruptedException e) {
                LOG.error("Event thread exiting due to interruption", e);
            }

            LOG.info("EventThread shut down for session: 0x{}", Long.toHexString(getSessionId()));
        }
```

EventThread每次从队列中取出一个事件，执行这个事件对应的所有Watcher

```java
        private void processEvent(Object event) {
            try {
                if (event instanceof WatcherSetEventPair) {
                    // each watcher will process the event
                    WatcherSetEventPair pair = (WatcherSetEventPair) event;
                    for (Watcher watcher : pair.watchers) {
                        try {
                            watcher.process(pair.event);	//就是这里了
                        } catch (Throwable t) {
                            LOG.error("Error while calling watcher.", t);
                        }
                    }
                    //下面还有很多处理异步回调的逻辑..
                }
            }
        }
```



#### Watcher特性

##### 一次性

无论服务端还是客户端，一旦一个Watcher被触发，Zookeeper都会将其从相应的存储中删除，因此在Watcher的使用上要反复注册。

##### 客户端串行执行

客户端Watcher回调的过程可以看到，是一个串行过程，这为我们保证了顺序，需要注意，不要因为一个Watcher的处理逻辑影响了整个客户端的Watcher回调。

```java
//Curator中某些异步事件(耗时长)可以传入一个线程池,让这个异步事件在线程池中回调
public T inBackground(BackgroundCallback callback, Executor executor);
```

##### 轻量

![image-20220416104431865](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220416104431865.png)