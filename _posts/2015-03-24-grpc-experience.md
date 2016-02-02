---
layout: post
title: "gRPC体验"
date: 2015-03-24 16:12:11
category: "服务器"
tags: gRPC 服务器
author: tangmi
---
![Alt text](http://7xi7ny.com1.z0.glb.clouddn.com/grpc.png "gRPC")

大名鼎鼎的grpc出来不久，赶紧下来体验一把，该文主要记录安装过程以及简单的hello程序。
<!--break-->

## grpc安装

### 1. 下载源码

    $ git clone https://github.com/grpc/grpc.git grpc; cd grpc;

### 2. 更新第三方源码

	$ git submodule update --init

<font color="red">注意：执行这一步更新命令前，需要修改.gitmodules文件，我已经通过goog code “一键export to github“ 功能 把gflags项目源码导入到了github(google code无法通过git访问)，修改后的文件如下：</font>

	[submodule "third_party/zlib"]
    	    path = third_party/zlib
        	url = https://github.com/madler/zlib
	[submodule "third_party/openssl"]
    	    path = third_party/openssl
	        url = https://github.com/openssl/openssl.git
    	    branch = OpenSSL_1_0_2-stable
	[submodule "third_party/protobuf"]
    	    path = third_party/protobuf
    	    url = https://github.com/google/protobuf.git
    	    branch = v3.0.0-alpha-2
	[submodule "third_party/gflags"]
    	    path = third_party/gflags
    	    url = https://github.com/tangmi360/gflags.git

### 3. 编译并安装

	$ make
	$ sudo make install prefix=/usr/local/

到此，grpc已经成功安装在系统中。

## protobuf 3.0.0安装

### 1. 进入third_party目录

	$ cd third_party/protobuf

### 2. 通过autogen.sh脚本生成configure

protobuf从github拉下的源码默认是没有configure文件，需要通过执行autogen.sh来生成  
不过需要修改下脚本，修改后脚本片断如下（22-25行被注释掉了，26行是新加）

	20 if test ! -e gtest; then
	21   echo "Google Test not present.  Fetching gtest-1.7.0 from the web..."
	22   #curl -O https://googletest.googlecode.com/files/gtest-1.7.0.zip
	23   #unzip -q gtest-1.7.0.zip
	24   #rm gtest-1.7.0.zip
	25   #mv gtest-1.7.0 gtest
	26   git clone https://github.com/tangmi360/googletest.git gtest
	27 fi

### 3. 编译并安装

	$ ./autogen.sh
	$ ./configure --prefix=/usr/local
	$ make
	$ make check
	$ sudo make install

### 4. 添加动态库到ldconf配置

	$ sudo vim /etc/ld.so.conf.d/grpc.conf
	$ 添加一行 "/usr/local/lib"
	$ sudo ldconfig

## Hello C++ gRPC!

### 1. 下载 grpc-common

该项目是grpc的使用示例程序以及帮助文档

	$ git clone https://github.com/grpc/grpc-common.git

### 2. 生成rpc接口代码

	$ cd grpc-common/cpp/helloworld/
	$ make helloworld.pb.cc

其实生成接口代码执行的是这条命令:

	$ protoc -I ../../protos --cpp_out=. --grpc_out=. --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` ../../protos/helloworld.proto

### 3. 代码示例

`greeter_server.cc`

	#include <iostream>
	#include <memory>
	#include <string>
	#include <grpc/grpc.h>
	#include <grpc++/server.h>
	#include <grpc++/server_builder.h>
	#include <grpc++/server_context.h>
	#include <grpc++/server_credentials.h>
	#include <grpc++/status.h>
	#include "helloworld.pb.h"

	using grpc::Server;
	using grpc::ServerBuilder;
	using grpc::ServerContext;
	using grpc::Status;
	using helloworld::HelloRequest;
	using helloworld::HelloReply;
	using helloworld::Greeter;

    class GreeterServiceImpl final : public Greeter::Service {
        Status SayHello(ServerContext* context, const HelloRequest* request, HelloReply* reply) override {
            std::string prefix("Hello ");
            reply->set_message(prefix + request->name());
            return Status::OK;
        }
    };

	void RunServer() {
        std::string server_address("0.0.0.0:50051");
        GreeterServiceImpl service;
        ServerBuilder builder;
        builder.AddListeningPort(server_address,
        grpc::InsecureServerCredentials());
        builder.RegisterService(&service);
        std::unique_ptr<Server> server(builder.BuildAndStart());
        std::cout << "Server listening on " << server_address << std::endl;
        server->Wait();
    }

	int main(int argc, char** argv) {
        grpc_init();
        RunServer();
        grpc_shutdown();
        return 0;
    }


`greeter_client.cc`

	#include <iostream>
	#include <memory>
	#include <string>
	#include <grpc/grpc.h>
	#include <grpc++/channel_arguments.h>
	#include <grpc++/channel_interface.h>
	#include <grpc++/client_context.h>
	#include <grpc++/create_channel.h>
	#include <grpc++/credentials.h>
	#include <grpc++/status.h>
	#include "helloworld.pb.h"

	using grpc::ChannelArguments;
	using grpc::ChannelInterface;
	using grpc::ClientContext;
	using grpc::Status;
	using helloworld::HelloRequest;
	using helloworld::HelloReply;
	using helloworld::Greeter;

    class GreeterClient {
      public:
        GreeterClient(std::shared_ptr<ChannelInterface> channel) : stub_(Greeter::NewStub(channel)) {}

        std::string SayHello(const std::string& user) {
            HelloRequest request;
            request.set_name(user);
            HelloReply reply;
            ClientContext context;
            Status status = stub_->SayHello(&context, request, &reply);
            if (status.IsOk()) {
                return reply.message();
            } else {
                return "Rpc failed";
            }
        }
        void Shutdown() { stub_.reset(); }
      private:
        std::unique_ptr<Greeter::Stub> stub_;
    };

    int main(int argc, char** argv) {
        grpc_init();
        GreeterClient greeter(
        grpc::CreateChannel("localhost:50051", grpc::InsecureCredentials(),
            ChannelArguments()));
        std::string user("world");
        std::string reply = greeter.SayHello(user);
        std::cout << "Greeter received: " << reply << std::endl;
        greeter.Shutdown();
        grpc_shutdown();
    }


### 4. 编译hello程序

	$ make

### 5. 运行

	$ ./greeter_server
	$ ./greeter_client

输出结果:

    Hello world

## 最后
  本文记录了一下安装编译过程，并简单体验了一下c++版本的hello程序，接下来陆续会对grpc的源码进行分析。
