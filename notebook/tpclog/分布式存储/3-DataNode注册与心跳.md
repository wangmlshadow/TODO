# **透彻理解分布式存储（三）——DataNode注册与心跳**

DataNode节点启动后，会基于RPC请求立即向NameNode注册自身信息，注册成功后，还会发送心跳进行保活。NameNode接受到DataNode的请求后，会在内存中维护DataNode信息，定期检测并剔除长时间未发送心跳的DataNode节点。

本章，我就来实现分布式文件系统的**DataNode节点注册与心跳**的核心逻辑。DataNode发送注册和心跳请求是基于一个名为 `NameNodeConnService`的组件来实现的，如下图（服务注册请求）：

![](https://files.tpvlog.com/tpvlog/distributed/storage/20210701213349722.png)
NameNode接受到注册/心跳请求后，会在内存中维护DataNode的信息，委托组件 `DataNodeManager`来实现，如下图（服务心跳请求）：

![](https://files.tpvlog.com/tpvlog/distributed/storage/20210701213401144.png)

## 一、DataNode

我们先来看DataNode侧的核心逻辑。

### 1.1 NameNodeConnService

DataNode启动后，会实例化一个 `NameNodeConnService`组件，依赖它与NameNode进行通信：

```java
/**
 * DataNode启动类
 */
public class DataNode {
    // 负责跟NameNode通信的组件
    private NameNodeConnService connService;
    private void start() {
        this.connService.start();
        //...
    }    //...
}
```

我们来看 `NameNodeConnService`，它在内部维护了一个 `NameNodeServiceActor`组件。NameNodeConnService本质是NameNodeServiceActor的一个管理类，所有操作都委托给了NameNodeServiceActor处理：

```java
/**
 * 负责跟NameNode进行通信的组件
 */
public class NameNodeConnService {
    // 负责跟NameNode主节点通信的ServiceActor组件
    private NameNodeServiceActor serviceActor;
    public NameNodeConnService() {
        this.serviceActor = new NameNodeServiceActor();
    }    public void start() {
        // 节点注册
        register();        // 节点心跳
        heartbeat();    }    /**
     * 向NameNode节点进行注册
     */
    private void register() {
        this.serviceActor.startRegister();
    }    /**
     * 开始发送心跳给NameNode
     */
    private void heartbeat() {
        this.serviceActor.startHeartbeat();
    }}
```

### 1.2 NameNodeConnActor

NameNodeConnActor封装了对gRPC的使用，它将作为**RPC服务调用方**向NameNode发送注册/心跳请求：

```java
/**
 * 负责跟NameNode进行通信的组件
 */
public class NameNodeConnActor {

    // 我这里直接写死NameNode的信息，读者可以自己完善，从配置文件读取
    private static final String NAMENODE_HOSTNAME = "localhost";
    private static final Integer NAMENODE_PORT = 50070;

    private NameNodeServiceGrpc.NameNodeServiceBlockingStub namenode;

    public NameNodeConnActor() {
        // 下面都是grpc client使用的模板代码
        ManagedChannel channel = NettyChannelBuilder
                .forAddress(NAMENODE_HOSTNAME, NAMENODE_PORT)
                .negotiationType(NegotiationType.PLAINTEXT)
                .build();
        this.namenode = NameNodeServiceGrpc.newBlockingStub(channel);
    }

    public void startRegister() {
        Thread registerThread = new RegisterThread();
        registerThread.start();
        try {
            registerThread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public void startHeartbeat() {
        new HeartbeatThread().start();
    }

    /**
     * 负责注册的线程
     */
    class RegisterThread extends Thread {
        @Override
        public void run() {
            try {
                System.out.println("发送RPC请求到NameNode进行注册.......");

                // 当前DataNode节点的信息，我这里直接写死了
                // 大家可以自己完善，比如加载配置文件读取
                String ip = "127.0.0.1";
                String hostname = "dfs-datanode-01";

                // RPC调用，向NameNode发送注册请求
                RegisterRequest request = RegisterRequest.newBuilder()
                        .setIp(ip)
                        .setHostname(hostname)
                        .build();
                // 调用RPC注册接口
                RegisterResponse response = namenode.register(request);
                System.out.println("接收到NameNode返回的注册响应：" + response.getStatus());
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

    }

    /**
     * 负责心跳的线程
     */
    class HeartbeatThread extends Thread {
        @Override
        public void run() {
            try {
                while (true) {
                    System.out.println("发送RPC请求到NameNode进行心跳.......");

                    // 当前DataNode节点的信息，我这里直接写死了
                    // 大家可以自己完善，比如加载配置文件读取
                    String ip = "127.0.0.1";
                    String hostname = "dfs-datanode-01";

                    // 通过RPC接口发送到NameNode他的注册接口上去
                    HeartbeatRequest request = HeartbeatRequest.newBuilder()
                            .setIp(ip)
                            .setHostname(hostname)
                            .build();
                    // 调用RPC心跳接口
                    HeartbeatResponse response = namenode.heartbeat(request);
                    System.out.println("接收到NameNode返回的心跳响应：" + response.getStatus());

                    // 每隔30秒发送一次心跳
                    Thread.sleep(30 * 1000);
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}

```

从上述代码可以看到，NameNodeConnActor内部包含了两个线程：

* RegisterThread：注册线程，会在DataNode启动后启用，向NameNode注册自身信息；
* HeartbeatThread：心跳线程，默认每隔30s向NameNode发送一次心跳请求。

## 二、NameNode

我们再来看NameNode侧的逻辑。

### 2.1 NameNodeRpcServer

NameNode在启动时，会创建一个组件——`NameNodeRpcServer`，负责RPC通信，包括接受DataNode发送过来的节点注册/心跳请求：

```java
/**
 * NameNode核心启动类
 */
public class NameNode {
    // NameNode对外提供RPC服务的server，可以接受/响应请求
    private NameNodeRpcServer rpcServer;

    private void init() {
        this.namesystem = new FSNameSystem();
        this.datanodeManager = new DataNodeManager();
        this.replicator = new EditLogReplicator(namesystem);
        this.rpcServer = new NameNodeRpcServer(this.namesystem, this.datanodeManager, this.replicator);
    }

    private void start() throws Exception {
        this.rpcServer.start();
        this.rpcServer.blockUntilShutdown();  
    }
}
```

我们来看下NameNodeRpcServer的内部，它的 `start`方法其实就是开启一个gPRC服务，监听指定的端口：

```java
public class NameNodeRpcServer {
    private static final int DEFAULT_PORT = 50070;
    private Server server;

    // 负责管理元数据的核心组件
    private FSNameSystem namesystem;

    // 负责管理集群中所有的datanode的组件
    private DataNodeManager datanodeManager;

    // 负责日志复制的组件
    private EditLogReplicator editLogReplicator;

    public NameNodeRpcServer(FSNameSystem namesystem, DataNodeManager datanodeManager, EditLogReplicator replicator) {
        this.namesystem = namesystem;
        this.datanodeManager = datanodeManager;
        this.editLogReplicator = replicator;
    }

    public void start() throws IOException {
        // 启动一个rpc server，监听指定的端口号，同时绑定好服务
        server = ServerBuilder
                .forPort(DEFAULT_PORT)
                .addService(new NameNodeServiceImpl(namesystem, datanodeManager, editLogReplicator))
                .build()
                .start();

        System.out.println("NameNodeRpcServer启动，监听端口号：" + DEFAULT_PORT);

        // 停机钩子
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                NameNodeRpcServer.this.stop();
            }
        });
    }

    public void stop() {
        if (server != null) {
            server.shutdown();
        }
    }

    public void blockUntilShutdown() throws InterruptedException {
        if (server != null) {
            server.awaitTermination();
        }
    }
}

```

我们来看关键看具体的RPC服务实现，也就是 `NameNodeServiceImpl`类的逻辑。可以看到，NameNodeServiceImpl只是一个包装类，负责实现RPC存根接口，真正的服务注册/心跳的逻辑由 `DataNodeManager`处理：

```java
/**
 * NameNode的RPC服务接口
 */
public class NameNodeServiceImpl extends NameNodeServiceGrpc.NameNodeServiceImplBase {

    public static final Integer STATUS_SUCCESS = 1;
    public static final Integer STATUS_FAILURE = 2;
    public static final Integer STATUS_SHUTDOWN = 3;

    // 负责管理元数据的核心组件
    private FSNameSystem namesystem;

    // 负责管理集群中所有的datanode的组件
    private DataNodeManager datanodeManager;

    // 负责日志同步的组件
    private EditLogReplicator replicator;

    private volatile Boolean isRunning = true;

    public NameNodeServiceImpl(FSNameSystem namesystem, DataNodeManager datanodeManager,
                               EditLogReplicator replicator) {
        this.namesystem = namesystem;
        this.datanodeManager = datanodeManager;
        this.replicator = replicator;
    }

    /**
     * DataNode注册
     */
    public void register(RegisterRequest request, StreamObserver<RegisterResponse> responseObserver) {
        // 使用DataNodeManager组件完成DataNode注册
        datanodeManager.register(request.getIp(), request.getHostname());

        RegisterResponse response = RegisterResponse.newBuilder().setStatus(STATUS_SUCCESS).build();
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }

    /**
     * DataNode心跳
     */
    public void heartbeat(HeartbeatRequest request, 
                          StreamObserver<HeartbeatResponse> responseObserver) {
        // 使用DataNodeManager组件完成DataNode心跳
        datanodeManager.heartbeat(request.getIp(), request.getHostname());

        HeartbeatResponse response = HeartbeatResponse.newBuilder().setStatus(STATUS_SUCCESS).build();
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }

    //...
}
```

### 2.2 DataNodeManager

DataNodeManager，顾名思义，是NameNode用来管理DataNode的组件，它的内部通过一个ConcurrentHashMap来维护DataNodeInfo：

```java
/**
 * 负责管理集群里的所有DataNode
 */
public class DataNodeManager {

    // 集群中所有的datanode
    private Map<String, DataNodeInfo> datanodes = new ConcurrentHashMap<>();

    public DataNodeManager() {
        new DataNodeAliveMonitor().start();
    }

    /**
     * datanode注册
     */
    public Boolean register(String ip, String hostname) {
        DataNodeInfo datanode = new DataNodeInfo(ip, hostname);
        datanodes.put(ip + "-" + hostname, datanode);
        System.out.println("DataNode注册：ip=" + ip + ",hostname=" + hostname);
        return true;
    }

    /**
     * datanode心跳
     */
    public Boolean heartbeat(String ip, String hostname) {
        DataNodeInfo datanode = datanodes.get(ip + "-" + hostname);
        datanode.setLatestHeartbeatTime(System.currentTimeMillis());
        System.out.println("DataNode发送心跳：ip=" + ip + ",hostname=" + hostname);
        return true;
    }

    /**
     * datanode是否存活的监控线程
     */
    class DataNodeAliveMonitor extends Thread {
        @Override
        public void run() {
            try {
                while (true) {
                    List<String> toRemoveDatanodes = new ArrayList<>();

                    Iterator<DataNodeInfo> datanodesIterator = datanodes.values().iterator();
                    DataNodeInfo datanode = null;
                    while (datanodesIterator.hasNext()) {
                        datanode = datanodesIterator.next();
                        // 遍历保存的DataNode节点，如果超过90秒未上送心跳，则移除
                        if (System.currentTimeMillis() - datanode.getLatestHeartbeatTime() > 90 * 1000) {
                            toRemoveDatanodes.add(datanode.getIp() + "-" + datanode.getHostname());
                        }
                    }
                    if (!toRemoveDatanodes.isEmpty()) {
                        for (String toRemoveDatanode : toRemoveDatanodes) {
                            datanodes.remove(toRemoveDatanode);
                        }
                    }
                    // 每隔30秒检测一次
                    Thread.sleep(30 * 1000);
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

DataNodeManager构造完成后，会启动一个 `DataNodeAliveMonitor`线程，默认每隔30秒检测一次每个DataNodeInfo的上一次心跳时间是否超过了阈值（默认90s），超过就移除掉。

`DataNodeInfo`是NameNode定义的DataNode信息抽象：

```java
public class DataNodeInfo {
    private String ip;
    private String hostname;
    private long latestHeartbeatTime = System.currentTimeMillis();
    //...省略get/set
}
```

## 三、总结

本章，我对分布式文件系统的注册和心跳机制进行了讲解，并给出了代码实现。整个实现思路还是很清晰的，NameNode作为注册/心跳的RPC服务提供方，通过一个ConcurrentHashMap维护DataNode的信息，读者可以参考我上传的源码进行理解和完善。
