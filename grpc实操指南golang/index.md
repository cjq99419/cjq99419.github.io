# gRPC实操指南（golang）


## gRPC 实操指南（golang）

### 1 RPC(Remote Procedure Call Protocol) 

#### 1.1 什么是RPC

RPC为远程调用协议，简单来说就是调用远程的函数。

正常单机开发的情况下，我们通过函数的方式实现部分功能的解耦

```go
func sum(num1,num2 int) int {
  return num1 + num2
}
```

如上是一个最简单的求和函数，我们只需要调用函数就可以实现求和的功能。

但大部分时候函数不会这么简单，尤其对于非单机的分布式系统，远程调用就尤为重要。

#### 1.2 RPC业务场景

RPC的应用场景特别广泛，不只是单纯的远程调用

* 所有的分布式机都需要进行登陆的验证，对于所有的主机都实现相同的登陆验证逻辑维护极差，同时也失去部分分布式意义，所以从解耦的角度考虑，我们需要定义一个统一的登陆验证业务来做。
* C/S架构的传输业务，如股票软件，每天需要用户登陆的时候去服务器拉取最新的数据，或者较简单的文件传输业务，登陆验证业务，证书业务都可以使用rpc的方式
* 跨语言开发的项目，比如web业务使用golang进行开发，底层使用cpp或c，部分脚本使用py，跨语言通信可以通过RPC提供的不同语言的开发机制进行实现。

因而回到最初，**RPC就是一个远程的函数**，只不过RPC协议做的就是把整个过程透明化，配置完成后从开发角度来看，和本地函数调用没有区别。

#### 1.3 主流RPC框架

目前主流的RPC，有ali的Dubbo，还有google的gRPC（本文主题）等

一般RPC框架如下所示：

* 客户端：客户端作为整个RPC业务的发起者，如上所说的股票软件，需要客户端主动发起请求去拉取最新的股票数据。
* 服务端：服务端接受客户端的请求，并做出相应的回应。简单来说，函数实体在服务端，数据处理在服务端。

服务端和客户端是每个RPC框架，开发者可见度最高的部分，实现RPC业务的重点就在于对C/S的设计和理解。首先，客户端一定是**率先发起请求的部分**，服务端一定是**具体处理请求的部分**。比如之前我们说的求和函数，函数主体一定是在服务端，客户端有两个数字num1，num2，向服务端发起RPC远程调用，并最后拿到求和结果。这样简单的说大家都很清楚，但复杂一些就会很难理解和分辨两者。

公司业务做的是反向的RPC，即B/S/C架构。用户通过浏览器访问服务器代理，可以实现类似网盘的存取功能，此外还可以绑定客户端，进而在浏览器端实现网盘和客户端FS之间的文件传输。正常的RPC结构是客户端发起请求，而公司业务就是服务端向客户端发起写文件或者读文件的请求，使我迷惑好久。

因而对于上述业务，我们平台的服务端相当于RPC的客户端，而我们平台的客户端，相当于RPC的服务端。

（这段一定一定要理解，分清C/S很重要，不然后续会写傻）

* 客户端stub，服务端stub，可以变相的理解为应用层。主要是对客户端的rpc调用和服务端的返回进行序列化和反序列化，并进行传输，即把rpc业务抽象成tcp socket的send和receive。（gRPC使用的就是tcp，http2.0协议，建立在传输层）

![img](https://tva1.sinaimg.cn/large/008eGmZEly1gn73rrahdej31440nytdu.jpg)

### 2 gRPC

gRPC是google的开源RPC框架，引用官网的一句话

```go
A high-performance, open-source universal RPC framework
```

如图，展示了gRPC跨语言开发的结构图，本文将描述golang使用grpc的过程。

严格来说，grpc通过tcp进行通信，使用http2.0协议，同时使用protobuf定义接口，因而相对于传统的restful api来说，速度更快，数据更小，接口要求更严谨。（protobuf此处不做详细介绍，[Google Protobuf](https://developers.google.com/protocol-buffers)）

![img](https://tva1.sinaimg.cn/large/008eGmZEly1gn73rpia9nj30xc0lv75g.jpg)

#### 2.1 环境配置

##### 2.1.1 首先使用go get获取grpc的官方软件包

```go
 go get google.golang.org/grpc
```

##### 2.1.2 下载protobuf编译器

[protobuf代码生成工具](https://github.com/protocolbuffers/protobuf/releases)，通过proto文件生成对应的代码。

（此处需要加入环境变量，各个系统操作不同，不赘述，protoc命令能够正常使用即可）

##### 2.1.3 安装golang编译插件

我们需要.proto最终生成可用的golang代码，因而需要独立安装golang grpc的插件

```go
go get -u github.com/golang/protobuf/protoc-gen-go
```



#### 2.2 编写proto文件

protobuf的详细语法见官方文档，此处主要介绍rpc相关的内容

proto中rpc业务实际上就是一个函数，由服务端重写（overwrite）的函数，一般网上的文章会把gRPC分为四种：简单RPC，服务端流RPC，客户端流RPC，双端流RPC。实际上区别就在于rpc函数的入参和出参，接下来详细介绍一下四种情况，和一般的应用场景。

##### 2.2.1 简单RPC

```protobuf
//指定使用proto3（proto2，3有很多不同，不可混写）
syntax = "proto3";

//指定生成的go_package,简单来说就是生成的go代码使用什么包，即package protocol
option go_package = ".;protocol";

//定义rpc服务
//此处rpc服务的定义，一定要从服务端的角度考虑，即接受请求，处理请求并返回响应的一端
//如服务端需要接受请求，根据列出目录的类型和路径列出，并返回文件map
service ListPath{
		//列出目录服务，入参为ListPathReq，出参为ListPathRes（两者都是下方定义的message）
    rpc List(ListPathReq)returns(ListPathRes){}
}

//定义请求的message
message ListPathReq {
		//枚举类型，即列出全部内容，只列出文件夹，还是只列出文件
		//注意这里只是类型定义，相当于内嵌类型定义，而不是变量定义
    enum Type {
        All = 0;
        Dir = 1;
        File = 2;
    }
    //定义一个Type类型的变量，为type
    Type type = 1;
    //定义一个字符串类型的变量，为path，指定路径
    string path = 2;
}

//定义相应的message
message ListPathRes {
		//同上
		//此处的Type和req的Type不同，由于是内嵌类型，因而作用域为本message，两者不会冲突
    enum Type {
        Dir = 0;
        File = 1;
    }
    string code = 1;
    string msg = 2;
    //map类型，为路径和类型的映射
    map<string,Type> fileMap = 3;
}
```

上图就是一个简单的RPC业务，功能是列出远程服务器上的文件系统目录。

为了更方便理解，下述是最简单的求和的rpc service

```protobuf
service Abs {
		rpc ABS(SumReq)returns(SumRes){}
}

message SumReq {
		int32 num1 = 1;
		int32 num2 = 2;
}

message SumRes {
		int32 num = 1;
}
```

即客户端发送SumReq，服务端接到RPC请求后调用sum函数，再封装到SumRes发回。

简单RPC很好理解，就是一来一回的业务，相当于一个传统函数的功能，常常用于登陆验证，握手协议，简单业务等。

但实际上业务不会这么简单，比如我的请求或者响应体特别大，肯定不能封装到一个protobuf包进行传输，因而需要使用流式传输，如请求视频资源，或者上传文件等，此时就引出了两种单向流类型，即客户端流和服务端流。

##### 2.2.2 客户端流RPC

简单来说，就是客户端请求是个流，其他和简单RPC类似。

```protobuf
syntax = "proto3";
option go_package = ".;protocol";

service WriteFile{
	//区别就在于此处，rpc函数的入参需要用stream显式表明为流式传输（前面加个stream）
  rpc Write(stream WriteFileReq)returns(WriteFileRes){}
}

//message定义相同，等于客户端不停发送很多个WriteFileReq，全部发送完成后，服务端返回一个WriteFileRes
message WriteFileReq {
  string path = 1;
  int64 offset = 2;
  int64 size = 3;
  bytes data = 4;
}

message WriteFileRes {
  string code = 1;
  string msg = 2;
  int64 size = 3;
  int64 offset = 4;
}
```

这里展示的应用场景为远程写文件，即客户端指定文件路径，数据偏移量和大小，以及传输的二进制数据，打包通过protobuf发送给服务端，服务端不停接受req并写文件，最终写完之后给客户端一个反馈res。

RPC的流指的是客户端流式发送数据，本质上是分块写的思想。即每个数据包指定路径，偏移和写入大小，同时包含数据内容，每次写一个固定大小的块（如2M），流式指的是流式发送很多个块，如1G为512个2M的块。

##### 2.2.3 服务端流RPC

同上～

```protobuf
syntax = "proto3";
option go_package = ".;protocol";

service ReadFile{
    rpc Read(ReadFileReq)returns(stream ReadFileRes){}
}

message ReadFileReq {
    string path = 1;
    int64 offset = 2;
    int64 size = 3;
}

message ReadFileRes {
    string code = 1;
    string msg = 2;
    int64 size = 3;
    int64 offset = 4;
    bytes data = 5;
}
```

理解了客户端流，服务端流也一样的道理，客户端发送一个请求，服务端不停的发送响应，直到全部发送完成。

上述代码的场景即为远程读取文件，发送一次请求，请求读取某个路径下的文件，比如读取6M大小，从2M的位置开始读，响应即分为三个块，分别包含2-4，4-6，6-8的数据（块大小可以定制，仅以2M举例）。

##### 2.2.4 双端流RPC

双端流RPC就是入参，出参皆为流。一般的应用场景，如聊天室，聊天室需要维持一个长链接，连接过程中双方进行通信，都是流式的信息，类似应用场景使用双端流式的RPC。

综上，其实分类的四种RPC本质上只是RPC函数在入参和出参上有一些不同，本质上没有太大区别。但go中具体每个rpc业务的复写，针对流式和非流式处理不同，下面会详细描述，golang中如何实现除双端流之外的三种RPC（双端流同理）。



#### 2.3 生成go rpc代码

编写完proto文件就可以通过proto去生成对应的go语言代码了～

```shell
 protoc --go_out=plugins=grpc:. *.proto
```

protoc为编译器的命令，指定使用插件为grpc，输出目录为.（grpc:.）当前目录，待编译文件为*.proto。此处可以指定某个文件编译，也可以指定输出目录，这条命令会编译当前目录下的所有proto文件并生成到当前目录。

![image-20210131172914287](https://tva1.sinaimg.cn/large/008eGmZEly1gn73rqtedmj308606ydfw.jpg)

以list path为例子，生成的pb.go，rpc的核心就在Client和Server的两个interface中

![截屏2021-01-31 下午5.35.55](https://tva1.sinaimg.cn/large/008eGmZEly1gn73rqdij1j324y0lsgpk.jpg)

Client interface

```go
// ListPathClient is the client API for ListPath service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://godoc.org/google.golang.org/grpc#ClientConn.NewStream.
type ListPathClient interface {
	List(ctx context.Context, in *ListPathReq, opts ...grpc.CallOption) (*ListPathRes, error)
}
```

Server interface

```go
// ListPathServer is the server API for ListPath service.
type ListPathServer interface {
	List(context.Context, *ListPathReq) (*ListPathRes, error)
}
```

**客户端调用Client interface的方法，服务端重写Server interface的方法**

一定要理解上述这句话！！！！！

例如这个列出服务器目录的rpc方法，客户端只需要创建客户端实例对象，然后调用这个List方法就可以，传入req，接受res。因而我们说，对于客户端来说，此次调用和本地函数没有区别，但实际上是gRPC实现的远程调用，对于客户端开发是不可见的。

再说服务端，服务端需要重写Server中的方法，即服务端需要实现Server接口，对req进行处理，并生成res，同时提供ctx上下文用作并发处理。

**综上！！！！客户端是这个函数的调用者，需要调用这个函数，服务端是这个函数的定义者，需要重写这个函数**

这里如果理解了，gRPC就可以用明白了！



#### 2.4 服务端

下述代码皆为伪代码，目的是提供调用或重写的例子和思想，大部分代码是我在公司编写的gPRC程序，删减了很多内容，所以有的地方看起来可能会匪夷所思，但重点是方法的使用和通用的路子。具体可以看官方的例子，google的golang grpc example。

##### 2.4.1 重写Server interface

###### 2.4.1.1 简单RPC

```go
//使用空结构体就可，主要目的是为了实现Server interface，需要参数也可传递
type ListServer struct{}
//实现List方法，接受req，返回res
func (l *ListServer) List(ctx context.Context, req *ListPathReq) (*ListPathRes, error) {
		//处理req
  	//对于列出目录，就是调用os的相关函数，通过req的参数列出
  	//装填生成res
  	return res, err
}
```

###### 2.4.1.2 客户端流RPC

```go
//同理
type WriteServer struct{}
//流式不通过函数返回，需要通过server对象进行操作
func (w *WriteServer) Write(write WriteFile_WriteServer) error {
	for {
    //接受一条req
		req, err := write.Recv()
    //条件判断，当EOF或XXX等情况下，认定完成
		if err == io.EOF || req.Size < blockSize {
      //完成流式，即服务端认为客户端发送了一堆包，已经完成了
      //调用函数，发送res（WriteFile函数的返回为WriteFileRes）
			return write.SendAndClose(WriteFile(req))
		}
		if err != nil {
			return err
		}
    //针对于每个包，都需要执行一次写文件，res没用扔掉
		_ = WriteFile(req)
	}
}
```

实际上述代码有些问题，最初WriteFile函数设计的时候，是类似于一问一答的形式，即有点像双向流，每次分块写都会有返回结果。但考虑到过多返回意义不大，因而使用客户端流，只保留了最后一次的返回，之前的返回直接丢弃掉。

###### 2.4.1.3 服务端流RPC

```go
func (r *ReadServer) Read(req *protocol.ReadFileReq, read protocol.ReadFile_ReadServer) error {
	//处理req
  //此处简化写了，每次调用send函数去发送一条res，直到退出函数，发送停止，即传输完成
	for {
		if err := read.Send(&ReadFileRes{}); err != nil {
			return err
		}
	}
}
```

##### 2.4.2 注册服务

```go
func GRPCWorker(ctx context.Context) {
  //设置端口
	port := ":6012"
  //创建监听listen，指定tcp和监听端口
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	
  //创建新的grpc服务
	s := grpc.NewServer()
  
  //注册服务，后续传入的结构体为实现了Server接口的结构体地址
	protocol.RegisterCreateDirServer(s, &CreateServer{})
	protocol.RegisterListPathServer(s, &ListServer{})
	protocol.RegisterReadFileServer(s, &ReadServer{})
	protocol.RegisterWriteFileServer(s, &WriteServer{})
  
  //反射注册grpc服务
	reflection.Register(s)
  
  //启动服务，指定lis监听
	go s.Serve(lis)
  
  //通过上下文判定是否退出
	select {
	case <-ctx.Done():
		s.Stop()
		return
	}
}
```

上述代码是包含了ctx上下文的写法，即当ctx退出的时候，停止服务并return。



#### 2.5 客户端

##### 2.5.1 调用Client func

###### 2.5.1.1 简单RPC

```go
//函数随便定义，重点是对grpc的调用
//伪代码，重点是理解
func (this *ClientService) ListPath(xxx string) error {
    //创建grpc连接，注意ip和port之间要有:
	  grpcConn, err := grpc.Dial("127.0.0.1:6012", grpc.WithInsecure())		
		//创建一个新的客户端对象
		client := protocol.NewListPathClient(grpcConn)
  	//设置ctx
		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()
		//调用List
  	//核心就在于此处的调用，客户端只需要通过client对象调用Client interface中的func就可
		res, err := client.List(ctx, &protocol.ListPathReq{})
		//处理res和err
		return err
}	
```

所以，客户端只需要维持一个实例化的client对象，通过client调用方法就可以使用RPC服务，注意和服务端不同的是，每个服务都需要一个客户端，即服务端是在一个对象上注册很多个服务，而客户端调用每个RPC业务都需要一个对应函数的Client对象。

###### 2.5.1.2 客户端流RPC

```go
func (this *ClientService) WriteFile(xxx string) error {
		//创建client对象，同上
  	client := protocol.NewWriteFileClient(client.conn)
		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()
		//创建流式写对象
		writeClient, err := client.Write(ctx)
		if err != nil {
			return nil, i18n.ImperceptibleErr(err)
		}

		for {
      //在循环中通过写对象进行数据发送
			err = writeClient.Send(&protocol.WriteFileReq{
				Path:   path,
				Offset: offset,
				Size:   int64(n),
				Data:   data,
			})
      //满足某个条件跳出循环
			if int64(n) != blockSize {
				break
			}
			offset += int64(n)
		}
  	//关闭并接收服务端的返回
		_, err := writeClient.CloseAndRecv()
		return err
}
```

###### 2.5.1.3 服务端流RPC

```go
func (this *ClientService) ReadFile(xxx string) error {
  	//创建read客户端
		readClient, err := client.Read(ctx, &protocol.ReadFileReq{})
    for {
      //循环读取
      rev, err := readClient.Recv()
      if err != nil {
        return err
      }
      if int64(n) != blockSize {
        //log.Println("grpc读取完成")
        return nil
      }
    }
		return nil
}
```



### 3 写在最后

学习grpc的起因是因为公司需要的远程读写文件业务，调研后采用gRPC实现。由于公司的业务是反向rpc，即我们业务的服务端向客户端发请求，去写文件或者读文件，因而第一天看了一天，一直迷茫于重写调用的部分。让我觉得最迷惑的部分就是，越看越乱，以至于后来根本分不清C/S，照着网上的blog各种尝试，企图碰运气成功RPC，自然不可能。

所以建议一定要彻底理解，2.2 2.3的内容，不要一味看代码，抄代码，这也是用删减后杂乱的业务代码，也没直接发官方例子的原因。一定要先理解！！！！！祝学习愉快～

vx：cjq99419 欢迎提问和批评指正，第一次写这种大篇幅一些的文章，结构什么的可能会比较乱，感谢指导！
