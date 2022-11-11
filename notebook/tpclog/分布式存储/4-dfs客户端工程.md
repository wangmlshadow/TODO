# **透彻理解分布式存储（四）——dfs客户端工程**

`dfs-client`，也就是分布式文件系统的客户端，需要提供各种文件/目录操作的命令，比如目录创建，文件上传/下载等等，然后实现跟分布式文件系统的通信。

本章，我以"目录创建"命令为例，来实现客户端文件操作，客户端的操作接口我定义在 `FileSystem`接口中：

![](https://files.tpvlog.com/tpvlog/distributed/storage/20210702224841360.png)

## 一、客户端接口

首先，我们需要定义好gRPC接口。

### 1.1 RPC接口存根

我直接修改 `dfs-rpc`工程 `src/main/proto`目录下的 `NameNodeServiceProto.proto`文件，新增一个“目录创建”接口，然后执行 `mvn clean compile`命令生成新的接口存根：

```protobuf
span
```

然后将存根复制到 `dfs-rpc`工程 `com.tpvlog.dfs.rpc.service`包中。

### 1.2 客户端实现

客户端的实现比较简单，定义一个文件系统接口 `FileSystem`。各个需要使用文件命令的应用引入 `dfs-client`工程后，直接使用该接口就可以进行文件操作：

```java
/**
 * 作为文件系统的接口
 */
public interface FileSystem {
    /**
     * 创建目录
     *
     * @param path 目录对应的路径
     * @throws Exception
     */
    void mkdir(String path) throws Exception;
}
```

我们来看下FileSystem的实现，就是一些gRPC调用的标准模板代码：

```java
public class FileSystemImpl implements FileSystem {

    // 这里指定NameNode的地址
    private static final String NAMENODE_HOSTNAME = "localhost";
    private static final Integer NAMENODE_PORT = 50070;

    private NameNodeServiceGrpc.NameNodeServiceBlockingStub namenode;

    public FileSystemImpl() {
        ManagedChannel channel = NettyChannelBuilder
                .forAddress(NAMENODE_HOSTNAME, NAMENODE_PORT)
                .negotiationType(NegotiationType.PLAINTEXT)
                .build();
        this.namenode = NameNodeServiceGrpc.newBlockingStub(channel);
    }

    /**
     * 创建目录
     */
    public void mkdir(String path) throws Exception {
        // 调用RPC服务
        MkDirRequest request = MkDirRequest.newBuilder().setPath(path).build();
        MkDirResponse response = namenode.mkdir(request);

        System.out.println("创建目录的响应：" + response.getStatus());
    }
}

```

## 二、服务端实现

我们再来看NameNode服务端的RPC服务实现。

### 2.1 RPC服务实现

首先，NameNode需要在 `NameNodeServiceImpl`中实现gRPC自动生成的接口存根方法，也就是上一节中的 `mkdir`方法：

```java
/**
 * NameNode的RPC服务接口
 */
public class NameNodeServiceImpl extends NameNodeServiceGrpc.NameNodeServiceImplBase {

    public static final Integer STATUS_SUCCESS = 1;
    public static final Integer STATUS_FAILURE = 2;

    //...

    // 负责管理元数据的核心组件
    private FSNameSystem namesystem;

    public NameNodeServiceImpl(FSNameSystem namesystem, DataNodeManager datanodeManager,
                               EditLogReplicator replicator) {
        this.namesystem = namesystem;
        this.datanodeManager = datanodeManager;
        this.replicator = replicator;
    }

    /**
     * 创建目录
     */
    @Override
    public void mkdir(MkdirRequest request, StreamObserver<MkdirResponse> responseObserver) {
        try {
            MkdirResponse response = null;

            if(!isRunning) {
                response = MkdirResponse.newBuilder().setStatus(STATUS_SHUTDOWN).build();
            } else {
                this.namesystem.mkdir(request.getPath());
                response = MkdirResponse.newBuilder().setStatus(STATUS_SUCCESS).build();
            }

            responseObserver.onNext(response);
            responseObserver.onCompleted();
        } catch (Exception e) {
            e.printStackTrace(); 
        }
    }
}

```

上面的接口实现比较简单，本质就是用gRPC的标准代码实现了 `NameNodeServiceGrpc.NameNodeServiceImplBase`中的 `mkdir`接口，然后调用 `FSNameSystem.mkdir()`方法完成文件操作。

### 2.2 FSNameSystem

FSNameSystem是NameNode进行元数据管理的核心组件，客户端发送过来的所有文件操作命令，都会交由FSNameSystem处理。FSNameSystem会在内存中维护 **文件目录树** ，比如下面这样：

```
/root  /usr  /local  /app/home  /kafka    /data      /access.log
```

同时，FSNameSystem还负责将操作命令写入 **edits log日志** 。关于edits log日志，我在后面章节会详细讲解，它有点类似于MySQL中binlog，可以用于NameNode宕机后的数据恢复。

## 三、总结

本章，我对 `dfs-client`工程、客户端的文件操作逻辑以及服务端的操作命令处理逻辑进行了讲解，其中最核心是 `FSNameSysytem`这个组件，它负责维护NameNode的内存文件目录树，以及写入edits log日志。我在后续章节会对这两块内容进行详细讲解。

本章的内容，我用下面这张图总结，帮助读者理解：

![](https://files.tpvlog.com/tpvlog/distributed/storage/20210702224851428.png)
