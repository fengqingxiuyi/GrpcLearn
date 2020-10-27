# gRPCLearn

gRPC学习

## 文档

- [gRPC Documentation](https://grpc.io/docs/)
- [grpc-java Demo](https://github.com/grpc/grpc-java)

## 目录结构

```
gRPCLearn
  |--client //客户端，com.android.application
  |--proto //proto文件模块，java-library
     |--build/generated/source/proto/main //该目录下为自动生成类的源目录
  |--server //服务端，java-library
```

## 从0到1

### 配置proto模块

#### 编辑`build.gradle`文件

```groovy
//Step1 buildscript节点必须在plugins节点之前
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'com.google.protobuf:protobuf-gradle-plugin:0.8.13'
    }
}

plugins {
    id 'java-library'
    id 'kotlin'
    //Step2 必须在java/kotlin/android plugin的后面
    id 'com.google.protobuf' version '0.8.13'
}

//Step3
sourceSets {
    main {
        proto { //标识proto文件所在文件夹，使Step4中定义的插件知道proto所在位置
            srcDir 'src/main/java/com/example/proto'
        }
        java { //将自动生成类所在目录定义为源目录，使其他模块能够调用这些类
            srcDirs 'build/generated/source/proto/main/grpc'
            srcDirs 'build/generated/source/proto/main/grpckt'
            srcDirs 'build/generated/source/proto/main/java'
        }
    }
}

//Step4
protobuf {
    protoc {
        //proto文件编译器
        artifact = "com.google.protobuf:protoc:3.13.0"
    }
    plugins {
        //插件：用于将proto文件中的service转换为java类
        grpc {
            artifact = "io.grpc:protoc-gen-grpc-java:1.33.0"
        }
        //插件：用于将proto文件中的service转换为kotlin类
        grpckt {
            artifact = "io.grpc:protoc-gen-grpc-kotlin:0.2.0:jdk7@jar"
        }
    }
    generateProtoTasks {
        all()*.plugins {
            //使插件生效
            grpc {}
            grpckt {}
        }
    }
}

dependencies {
    //省略其他依赖
    //Step5
    implementation 'com.google.protobuf:protobuf-java:3.13.0'
    //下面三个依赖需要用api方式，因为client和server模块会用到
    api 'io.grpc:grpc-protobuf:1.33.0'
    api 'io.grpc:grpc-stub:1.33.0'
    api 'io.grpc:grpc-kotlin-stub:0.2.0'
    //compileOnly 'org.apache.tomcat:annotations-api:6.0.53' // necessary for Java 9+
}
```

#### 创建`hello_world.proto`文件

```protobuf
syntax = "proto3";

option java_multiple_files = true;
option java_package = "com.example.proto";
option java_outer_classname = "HelloWorldProto";

package helloworld;

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

#### 自动生成类

```bash
proto % ../gradlew clean build
```

### 配置server模块

#### 编辑`build.gradle`文件

```groovy
dependencies {
    //省略其他依赖
    //Step1
    implementation 'io.grpc:grpc-netty-shaded:1.33.0'
    api project(':proto')
}
```

#### 创建`HelloWorldServer.kt`文件

```kotlin
package com.example.server

import com.example.proto.GreeterGrpcKt
import com.example.proto.HelloReply
import com.example.proto.HelloRequest
import io.grpc.Server
import io.grpc.ServerBuilder

class HelloWorldServer(private val port: Int) {
    val server: Server = ServerBuilder
            .forPort(port)
            .addService(HelloWorldService())
            .build()

    fun start() {
        server.start()
        println("Server started, listening on $port")
        Runtime.getRuntime().addShutdownHook(
                Thread {
                    println("*** shutting down gRPC server since JVM is shutting down")
                    this@HelloWorldServer.stop()
                    println("*** server shut down")
                }
        )
    }

    private fun stop() {
        server.shutdown()
    }

    fun blockUntilShutdown() {
        server.awaitTermination()
    }

    private class HelloWorldService : GreeterGrpcKt.GreeterCoroutineImplBase() {
        override suspend fun sayHello(request: HelloRequest) = HelloReply
                .newBuilder()
                .setMessage("Hello ${request.name}")
                .build()
    }
}

fun main() {
    val port = System.getenv("PORT")?.toInt() ?: 50051
    val server = HelloWorldServer(port)
    server.start()
    server.blockUntilShutdown()
}
```

### 配置client模块

#### 编辑`build.gradle`文件

```groovy
dependencies {
    //省略其他依赖
    //使用implementation方式依赖grpc-okhttp，必须添加okio和perfmark依赖，否则会出现类找不到的问题
    implementation 'io.grpc:grpc-okhttp:1.33.0'
    implementation 'com.squareup.okio:okio:2.9.0'
    implementation 'io.perfmark:perfmark-api:0.19.0'
    api project(':proto')
}
```

#### 创建`HelloWorldClient.kt`文件

```kotlin
package com.example.grpclearn

import com.example.proto.GreeterGrpcKt
import com.example.proto.HelloRequest
import io.grpc.ManagedChannel
import io.grpc.ManagedChannelBuilder
import java.io.Closeable
import java.util.concurrent.TimeUnit

class HelloWorldClient(private val channel: ManagedChannel) : Closeable {
    private val stub: GreeterGrpcKt.GreeterCoroutineStub =
        GreeterGrpcKt.GreeterCoroutineStub(channel)

    suspend fun greet(name: String) {
        val request = HelloRequest.newBuilder().setName(name).build()
        val response = stub.sayHello(request)
        println("Received: ${response.message}")
    }

    override fun close() {
        channel.shutdown().awaitTermination(5, TimeUnit.SECONDS)
    }
}

/**
 * Greeter, uses first argument as name to greet if present;
 * greets "world" otherwise.
 */
suspend fun main(args: Array<String>) {
    val port = System.getenv("PORT")?.toInt() ?: 50051

    val channel = ManagedChannelBuilder.forAddress("localhost", port).usePlaintext().build()

    val client = HelloWorldClient(channel)

    val user = args.singleOrNull() ?: "world"
    client.greet(user)
}
```

### 启动server

输出`Server started, listening on 50051`表示正确

### 启动client

输出`Received: Hello world`表示正确
