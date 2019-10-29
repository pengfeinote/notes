http://www.importnew.com/23025.html

坚韧的ZooKeeper应用 The Resilient ZooKeeper Application
分布式计算设计的第一谬误就是认为“网络是稳定的”。我们所实现的程序目前都是假设网络稳定的情况下实现的，所以当我们在一个真实的网络环境下，会有很多原因可以使程序执行失败。下面我们将阐述一些可能造成失败的场景，并且讲述如何正确的处理这些失败，让我们的程序在面对这些异常时更具韧性。

在ZooKeeper的API中，每一个ZooKeeper的操作都会声明抛出连个异常：InterruptedException和KeeperException。

InterrupedException
当一个操作被中断时，会抛出一个InterruptedException。在JAVA中有一个标准的阻塞机制用来取消程序的执行，就是在需要阻塞的的地方调用interrupt()。如果取消执行成功，会以抛出一个InterruptedException作为结果。ZooKeeper坚持了这个标准，所以我们可以用这种方式来取消client的对ZooKeeper的操作。用到ZooKeeper的类和库需要向上抛出InterruptedException，才能使我们的client实现取消操作。

InterruptedException并不意味着程序执行失败，可能是人为设计中断的，所以在上面配置应用的例子中，当向上抛出InterruptedException时，会引起应用终止。

KeeperException
当ZooKeeper服务器出现错误信号，或者出现了通信方面的问题，就会抛出一个KeeperException。由于错误的不同原因，所以KeeperException有很多子类。例如，KeeperException.NoNodeException当操作一个znode时，而这个znode并不存在，就会抛出这个异常。

每一个之类都有一个异常码作为异常的类型。例如，KeeperException.NoNodeException的异常码就是KeeperException.Code.NONODE(一个枚举值)。

有两种方法来处理KeeperException。一种是直接捕获KeeperException，然后根据异常码进行不同类型异常处理。另一种是捕获具体的子类，然后根据不同类型的异常进行处理。

KeeperException包含了3大类异常。

状态异常 State Exception
当无法操作znode树造成操作失败时，会产生状态异常。通常引起状态异常的原因是有另外的程序在同时改变znode。例如，一个setData()操作时，会抛出KeeperException.BadVersionException。因为另外的一个程序已经在setData()操作之前修改了znode，造成setData()操作时版本号不匹配了。程序员必须了解，这种情况是很有可能发生的，我们必须靠编写处理这种异常的代码来解决他。

有的一些异常是编写代码时的疏忽造成的，例如KeeperException.NoChildrenForEphemeralsException。这个异常是当我们给一个enphemeral类型的znode添加子节点时抛出的。

重新获取异常 Recoverable Exception
重新获取异常来至于那些能够获得同一个ZooKeeper session的应用。伴随的表现是抛出KeeperException.ConnectionLossException，表示与ZooKeeper的连接丢失。ZooKeeper将会尝试重新连接，大多数情况下重新连接都会成功并且能够保证session的完整性。

然而，ZooKeeper无法通知客户端操作由于KeeperException.ConnectionLossException而失败。这就是一个部分失败的例子。只能依靠程序员编写代码来处理这个不确定性。

在这点上，幂等操作和非幂等操作的差别就会变得非常有用了。一个幂等操作是指无论运行一次还是多次结果都是一样的，例如一个读请求，或者一个不设置任何值得setData操作。这些操作可以不断的重试。

一个非幂等操作不能被不分青红皂白的不停尝试执行，就像一些操作执行一次的效率和执行多次的效率是不同。我们将在之后会讨论如何利用非幂等操作来处理Recovreable Exception。

不能重新获取异常 Unrecoverable exceptions
在一些情况下，ZooKeeper的session可能会变成不可用的——比如session过期，或者因为某些原因session被close掉（都会抛出KeeperException.SessionExpiredException），或者鉴权失败（KeeperException.AuthFailedException）。无论何种情况，ephemeral类型的znode上关联的session都会丢失，所以应用在重新连接到ZooKeeper之前都需要重新构建他的状态。

一个稳定的配置服务 A reliable configuration service
回过头来看一下ActiveKeyValueStore中的write()方法，其中调用了exists()方法来判断znode是否存在，然后决定是创建一个znode还是调用setData来更新数据。

1
2
3
4
5
6
7
8
9
10
public void write(String path, String value) throws InterruptedException,
      KeeperException {
    Stat stat = zk.exists(path, false);
    if (stat == null) {
      zk.create(path, value.getBytes(CHARSET), Ids.OPEN_ACL_UNSAFE,
          CreateMode.PERSISTENT);
    } else {
      zk.setData(path, value.getBytes(CHARSET), -1);
    }
  }
从整体上来看，write()方法是一个幂等方法，所以我们可以不断的尝试执行它。我们来修改一个新版本的write()方法，实现在循环中不断的尝试write操作。我们为尝试操作设置了一个最大尝试次数参数（MAX_RETRIES）和每次尝试间隔的休眠(RETRY_PERIOD_SECONDS)时长：

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
public void write(String path, String value) throws InterruptedException,
      KeeperException {
    int retries = 0;
    while (true) {
      try {
        Stat stat = zk.exists(path, false);
        if (stat == null) {
          zk.create(path, value.getBytes(CHARSET), Ids.OPEN_ACL_UNSAFE,
              CreateMode.PERSISTENT);
        } else {
          zk.setData(path, value.getBytes(CHARSET), stat.getVersion());
        }
        return;
      } catch (KeeperException.SessionExpiredException e) {
        throw e;
      } catch (KeeperException e) {
        if (retries++ == MAX_RETRIES) {
          throw e;
        }
        // sleep then retry
        TimeUnit.SECONDS.sleep(RETRY_PERIOD_SECONDS);
      }
    }
  }
细心的读者可能会发现我们并没有在捕获KeeperException.SessionExpiredException时继续重新尝试操作，这是因为当session过期后，ZooKeeper会变为CLOSED状态，就不能再重新连接了。我们只是简单的抛出一个异常，通知调用者去创建一个新的ZooKeeper实例，所以write()方法可以不断的尝试执行。一个简单的方式来创建一个ZooKeeper实例就是重新new一个ConfigUpdater实例。

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
public static void main(String[] args) throws Exception {
    while (true) {
      try {
        ResilientConfigUpdater configUpdater =
          new ResilientConfigUpdater(args[0]);
        configUpdater.run();
      } catch (KeeperException.SessionExpiredException e) {
        // start a new session
      } catch (KeeperException e) {
        // already retried, so exit
        e.printStackTrace();
        break;
      }
    }
  }
另一个可以替代处理session过期的方法就是使用watcher来监控Expired的KeeperState，然后重新建立一个连接。这种方法下，我们只需要不断的尝试执行write()，如果我们得到了KeeperException.SessionExpiredException`异常，连接最终也会被重新建立起来。那么我们抛开如何从一个过期的session中恢复问题，我们的重点是连接丢失的问题也可以这样解决，只是处理方法不同而已。

注意
我们这里忽略了另外一种情况，在zookeeper实例不断的尝试连接了ensemble中的所有节点后发现都无法连接成功，就会抛出一个IOException，说明所有的集群节点都不可用。而有一些应用被设计为不断的尝试连接，直到ZooKeeper服务恢复可用为止。
这只是一个重复尝试的策略。还有很多的策略，比如指数补偿策略，每次尝试之间的间隔时间会被乘以一个常数，间隔时间会逐渐变长，直到与集群建立连接为止间隔时间才会恢复到一个正常值，来预备一下次连接异常使用。

译者：为什么要使用指数补偿策略呢？这是为了避免反复的尝试连接而消耗资源。在一次较短的时间后第二次尝试连接不成功后，延长第三次尝试的等待时间，这期间服务恢复的几率可能会更大。第四次尝试的机会就变小了，从而达到减少尝试的次数。