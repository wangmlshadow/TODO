# **透彻理解分布式存储（二）——gRPC使用**

本章，我将对 `dfs-rpc`这个模块进行讲解。该模块依赖gRPC，存放各个RPC服务的存根。我们的分布式文件系统的服务间的调用需要以gRPC作为通信组件，所以本章我会讲解gRPC的基本使用，但我不会对gRPC作深入介绍，gRPC的底层原理和各种高阶用法，读者可以参考其它资料。

## 一、dfs-rpc工程

gRPC本身支持不同语言，通过ProtoBuf编写的 `.proto`文件，然后使用不同语言的编译器，可以生成特定语言相关的模板代码。由于我们的工程使用Java，所以需要使用[grpc-java](https://github.com/grpc/grpc-java)。

> `Protobuf`是一种语言中立、平台无关、可扩展的 **序列化数据的格式** ，可用于通信协议，数据存储等。我们的系统中涉及的服务接口都需要通过protobuf来生成。

### 1.1 引入依赖

首先，新建一个Maven工程 `dfs-rpc`，在pom中引入以下依赖。注意，我这里使用了 `protobuf-maven-plugin`插件，用来将 `.proto`文件转化成 Java 模板代码。`protobuf-maven-plugin`插件内部整合了[protoc编译器](https://github.com/protocolbuffers/protobuf/releases)，读者也可以自己下载protoc编译器，手动将 `.proto`文件转化为Java代码：

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
      <modelVersion>4.0.0</modelVersion>
      <groupId>com.tpvlog.dfs</groupId>
      <artifactId>dfs-rpc</artifactId>
      <version>0.0.1-SNAPSHOT</version>
      <packaging>jar</packaging>
      <name>dfs-rpc</name>
      <url>http://maven.apache.org</url>
      <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
      </properties>
    <dependencies>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty-shaded</artifactId>
            <version>1.35.0</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
            <version>1.35.0</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
            <version>1.35.0</version>
        </dependency>
        <dependency> <!-- necessary for Java 9+ -->
            <groupId>org.apache.tomcat</groupId>
            <artifactId>annotations-api</artifactId>
            <version>6.0.53</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
    <build>
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>1.6.2</version>
            </extension>
        </extensions>
        <plugins>
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.6.1</version>
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:3.12.0:exe:${os.detected.classifier}</protocArtifact>
                    <pluginId>grpc-java</pluginId>
                    <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.35.0:exe:${os.detected.classifier}</pluginArtifact>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>compile-custom</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

### 1.2 接口编写

服务接口的编写使用ProtoBuf的标准语法，我们编写一个 `NameNodeServiceProto.proto`文件，里面定义**NameNode**提供的各个服务接口：

```proto
syntax = "proto3";

option java_multiple_files = true;
option java_outer_classname = "NameNodeServiceProto";

service NameNodeService {
    rpc register(RegisterRequest) returns (RegisterResponse){}
    rpc heartbeat(HeartbeatRequest) returns (HeartbeatResponse){}
    rpc mkdir(MkDirRequest) returns (MkDirResponse){}
    rpc shutdown(ShutdownRequest) returns (ShutdownResponse){}
    rpc fetchEditsLog(FetchEditsLogRequest) returns (FetchEditsLogResponse){}
    rpc updateCheckpointTxid(UpdateCheckpointTxidRequest) returns (UpdateCheckpointTxidResponse){}
}

message RegisterRequest{
    string ip  = 1;
    string hostname  = 2;
}
message RegisterResponse{
    int32 status  = 1;
}
message HeartbeatRequest{
    string ip  = 1;
    string hostname  = 2;
}
message HeartbeatResponse{
    int32 status  = 1;
}
message MkDirRequest{
    string path  = 1;
}
message MkDirResponse{
    int32 status  = 1;
}

message ShutdownRequest{
    int32 code  = 1;
}
message ShutdownResponse{
    int32 status  = 1;
}
message FetchEditsLogRequest{
    int64 syncedTxid  = 1;
}
message FetchEditsLogResponse{
    string editsLog  = 1;
}
message UpdateCheckpointTxidRequest{
    int64 txid  = 1;
}
message UpdateCheckpointTxidResponse{
    int32 status  = 1;
}
```

### 1.3 Java代码生成

我们把上面的 `.proto`文件存放到项目的 `/src/main/proto`目录下，然后 cd 到项目根目录，执行 `mvn clean compile`命令：

```bash
C:\Users\Ressmix\Desktop\source-code\dfs\dfs-rpc>mvn clean compile
[INFO] Scanning for projects...
[INFO] ------------------------------------------------------------------------
[INFO] Detecting the operating system and CPU architecture
[INFO] ------------------------------------------------------------------------
[INFO] os.detected.name: windows
[INFO] os.detected.arch: x86_64
[INFO] os.detected.version: 10.0
[INFO] os.detected.version.major: 10
[INFO] os.detected.version.minor: 0
[INFO] os.detected.classifier: windows-x86_64
[INFO]
[INFO] -----------------------< com.tpvlog.dfs:dfs-rpc >-----------------------
[INFO] Building dfs-rpc 0.0.1-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ dfs-rpc ---
[INFO] Deleting C:\Users\Ressmix\Desktop\source-code\dfs\dfs-rpc\target
[INFO]
[INFO] --- protobuf-maven-plugin:0.6.1:compile (default) @ dfs-rpc ---
[INFO] Compiling 1 proto file(s) to C:\Users\Ressmix\Desktop\source-code\dfs\dfs-rpc\target\generated-sources\protobuf\java
[INFO]
[INFO] --- protobuf-maven-plugin:0.6.1:compile-custom (default) @ dfs-rpc ---
[INFO] Compiling 1 proto file(s) to C:\Users\Ressmix\Desktop\source-code\dfs\dfs-rpc\target\generated-sources\protobuf\grpc-java
[INFO]
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ dfs-rpc ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory C:\Users\Ressmix\Desktop\source-code\dfs\dfs-rpc\src\main\resources
[INFO] Copying 1 resource
[INFO] Copying 1 resource
[INFO]
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ dfs-rpc ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 48 source files to C:\Users\Ressmix\Desktop\source-code\dfs\dfs-rpc\target\classes
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  7.920 s
[INFO] Finished at: 2021-02-19T16:29:49+08:00
[INFO] ------------------------------------------------------------------------

```

完成后会在 `/target/generated-sources/protobuf/java/`目录下看到一堆生成的Java模板代码：

![](https://files.tpvlog.com/tpvlog/distributed/storage/20210630223431453.png)

### 1.4 测试使用

我们把这些代码拷贝到 `dfs-rpc`工程的 `com.tpvlog.dfs.rpc.service`包下：

![](https://files.tpvlog.com/tpvlog/distributed/storage/20210630223441413.png)
然后来测试一下，先写一个服务端测试类：

```java
package com.tpvlog.dfs.rpc.service;

import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.stub.StreamObserver;

import java.io.IOException;

public class GrpcServerTest {
    private final int port = 19080;
    private Server server;

    public static void main(String[] args) throws IOException, InterruptedException {
        final GrpcServerTest grpcServer = new GrpcServerTest();
        grpcServer.start();
        grpcServer.blockUntilShutdown();
    }

    private void start() throws IOException {
        server = ServerBuilder.forPort(port).addService(new NameNodeServiceImpl()).build().start();
        System.out.println("------- NameNodeService Started -------");
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                System.err.println("------shutting down gRPC server since JVM is shutting down-------");
                GrpcServerTest.this.stop();
                System.err.println("------server shut down------");
            }
        });
    }

    /**
     * 服务接口实现类
     */
    private class NameNodeServiceImpl extends NameNodeServiceGrpc.NameNodeServiceImplBase {
        public void register(RegisterRequest request, StreamObserver<RegisterResponse> responseObserver) {
            RegisterResponse response = RegisterResponse.newBuilder().setStatus(200).build();
            // onNext()方法向客户端返回结果
            responseObserver.onNext(response);
            // 告诉客户端这次调用已经完成
            responseObserver.onCompleted();
        }

        public void heartbeat(HeartbeatRequest request, StreamObserver<HeartbeatResponse> responseObserver) {
            HeartbeatResponse response=HeartbeatResponse.newBuilder().setStatus(200).build();
            responseObserver.onNext(response);
            responseObserver.onCompleted();
        }
    }

    private void stop() {
        if (server != null) {
            server.shutdown();
        }
    }

    private void blockUntilShutdown() throws InterruptedException {
        if (server != null) {
            server.awaitTermination();
        }
    }
}
```

再写一个客户端测试类：

```java
package com.tpvlog.dfs.rpc.service;

import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;

import java.util.concurrent.TimeUnit;

public class GrpcClientTest {
    private final ManagedChannel channel;
    private final NameNodeServiceGrpc.NameNodeServiceBlockingStub blockingStub;

    private static final String host = "127.0.0.1";
    private static final int ip = 19080;

    public GrpcClientTest(String host, int port) {
        // usePlaintext表示明文传输，否则需要配置ssl
        // channel表示通信通道
        channel = ManagedChannelBuilder.forAddress(host, port).usePlaintext().build();
        // 存根
        blockingStub = NameNodeServiceGrpc.newBlockingStub(channel);
    }

    public void shutdown() throws InterruptedException {
        channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
    }

    public void testHeartbeat(String name) {
        HeartbeatRequest request = HeartbeatRequest.newBuilder().setIp("127.0.0.1").setHostname("localhost").build();
        HeartbeatResponse response = blockingStub.heartbeat(request);
        System.out.println(name + ": " + response.getStatus());
    }

    public void testRegister(String name) {
        RegisterRequest request = RegisterRequest.newBuilder().setIp("127.0.0.1").setHostname("localhost").build();
        RegisterResponse response = blockingStub.register(request);
        System.out.println(response.getStatus());
    }

    public static void main(String[] args) {
        GrpcClientTest client = new GrpcClientTest(host, ip);
        for (int i = 0; i <= 5; i++) {
            client.testHeartbeat("<<<<<Heartbeat result>>>>>-" + i);
        }

        for (int i = 0; i <= 5; i++) {
            client.testHeartbeat("<<<<<Register result>>>>>-" + i);
        }
    }
}
```

最后，先启动服务端，然后启动客户端，输出如下，说明RPC调用成功了：

span

* ------- NodeRegisterService Started -------
* 
* <<<<`<Heartbeat result>`>>>>-0: 200
* <<<<`<Heartbeat result>`>>>>-1: 200
* <<<<`<Heartbeat result>`>>>>-2: 200
* <<<<`<Heartbeat result>`>>>>-3: 200
* <<<<`<Heartbeat result>`>>>>-4: 200
* <<<<`<Heartbeat result>`>>>>-5: 200
* <<<<`<Register result>`>>>>-0: 200
* <<<<`<Register result>`>>>>-1: 200
* <<<<`<Register result>`>>>>-2: 200
* <<<<`<Register result>`>>>>-3: 200
* <<<<`<Register result>`>>>>-4: 200
* <<<<`<Register result>`>>>>-5: 200

本章，我完成了 `dfs-rpc`工程的构建，同时我对gRPC的基本使用进行了讲解，后续这个工程会存放我们所有的gRPC模板代码和 `.proto`文件，其它涉及gRPC服务调用的工程都需要依赖该工程。
