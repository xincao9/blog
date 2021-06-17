---
title: grpc快速开始
date: 2021-06-17 10:38:39
tags: grpc, protobuffer, java
---

依赖protobuf，如果没有 https://github.com/protocolbuffers/protobuf自行安装

**参考安装**

```
进入页面

https://github.com/protocolbuffers/protobuf/releases

下载最新的protoc

protoc-3.17.3-osx-x86_64.zip

解压至安装目录

sudo unzip protoc-3.17.3-osx-x86_64.zip -d /usr/local/

创建软链

sudo ln -s protoc-3.17.3-osx-x86_64 protoc
 
添加环境变量（eg: ~/.bash_profile）

export PROTOC_HOME=/usr/local/protoc
export PATH=$PROTOC_HOME/bin:$PATH
```

### 创建maven项目

```
mvn archetype:generate -DgroupId=com.demo -DartifactId=grpc
```

### 添加maven jar 和 build plugin 依赖

* jar依赖

```
<dependencies>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>1.38.0</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>1.38.0</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>1.38.0</version>
    </dependency>
</dependencies>
```

* build plugin依赖

```
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
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.38.0:exe:${os.detected.classifier}</pluginArtifact>
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
```

### 定义protobuffer IDL文件, 存放到 src/main/proto目录

```
syntax = "proto3";

package demo;

option java_package = "com.demo.grpc.stub";

// The greeting service definition.
service Greeter {
    // Sends a greeting
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
    string name = 1;
}

// The response message containing the greetings
message HelloReply {
    string message = 1;
}
```

### 生成stub文件

```
mvn clean package
```

`生成后的stub文件位于 target/generated-sources/protobuf/grpc-java, target/generated-sources/protobuf/java 为了在idea中方便代码引用，需要将这两个目录标记为为source root`

### 实现服务provider

* 实现服务方法

```
public class GreeterImpl extends GreeterGrpc.GreeterImplBase {

    @Override
    public void sayHello(Helloworld.HelloRequest req, StreamObserver<Helloworld.HelloReply> responseObserver) {
        Helloworld.HelloReply reply = Helloworld.HelloReply.newBuilder().setMessage("Hello " + req.getName()).build();
        responseObserver.onNext(reply);
        responseObserver.onCompleted();
    }
}
```

* 启动服务

```
public class Bootstrap {

    public static void main(String[] args) throws IOException, InterruptedException {
        Server server = ServerBuilder.forPort(50051)
            .addService(new GreeterImpl())
            .build()
            .start();
        server.awaitTermination();
    }
}
```

### 实现服务consumer

```
public class Bootstrap {

    public static void main(String... args) throws InterruptedException {
        ManagedChannel channel = ManagedChannelBuilder.forTarget("localhost:50051")
            .usePlaintext()
            .build();
        try {
            GreeterGrpc.GreeterBlockingStub blockingStub = GreeterGrpc.newBlockingStub(channel);
            Helloworld.HelloRequest request = Helloworld.HelloRequest.newBuilder().setName("world").build();
            try {
                Helloworld.HelloReply response = blockingStub.sayHello(request);
                System.out.println("Greeting: " + response.getMessage());
            } catch (StatusRuntimeException e) {
                e.printStackTrace();
            }
        } finally {
            channel.shutdownNow().awaitTermination(5, TimeUnit.SECONDS);
        }
    }
}
```

### 参考资料

* [protobuf](https://github.com/protocolbuffers/protobuf)
* [grpc quickstart](https://grpc.io/docs/languages/java/quickstart/)
* [grpc](https://grpc.io/)
