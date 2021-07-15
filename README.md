# k8s

内容提要
rpc原理
基于grpc和grpc-gateway封装自己的业务框架frame
在frame框架上集成APM、config、gorm等功能
利用frame框架实现一个完整案例(bff+service)
把服务部署到k8s
利用envoy实现流量控制
注：代码示例见(https://github.com/1819997197/go-grpc)

一、rpc简介
1、程序中是如何使用rpc的？
让我们先来看一下go程序中是如何使用rpc的。

服务接口定义：

syntax = "proto3"; // 指定proto版本
package proto;     // 指定包名

// 定义Hello服务
service Hello {
    rpc SayHello(HelloRequest) returns (HelloReply) {} // 定义SayHello方法
}

// HelloRequest 请求结构
message HelloRequest {
    string name = 1;
}

// HelloReply 响应结构
message HelloReply {
    string message = 1;
}

 

服务端实现接口：

// 定义helloService并实现约定的接口
type helloService struct{}
func (h *helloService) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
   resp := new(pb.HelloReply)
   resp.Message = "Hello " + in.Name + "."
   return resp, nil
}

func main() {
   listen, err := net.Listen("tcp", "127.0.0.1:50052")
   if err != nil {
      fmt.Println("net.Listen err: ", err)
      return
   }

   // 实例化grpc Server
   s := grpc.NewServer()

   // 注册HelloService
   pb.RegisterHelloServer(s, &helloService{})
   if err := s.Serve(listen); err != nil {
      return
   }
}

 

客户端调用：

func main() {
   // 连接
   conn, err := grpc.Dial("127.0.0.1:50052", grpc.WithInsecure())
   if err != nil {
      fmt.Println("conn err: ", err)
      return
   }
   defer conn.Close()

   // 初始化客户端
   c := pb.NewHelloClient(conn)

   // 调用方法
   reqBody := new(pb.HelloRequest)
   reqBody.Name = "gRPC"
   r, err := c.SayHello(context.Background(), reqBody)
   if err != nil {
      fmt.Println("response err: ", err)
      return
   }
   fmt.Println("message: ", r.Message)
}

 

从上面的示例，我们可以看出：客户端调用远程服务的接口，就跟调用自己本地的方法一样便捷。

问题来了：为什么可以这么方便呢？一切归功于rpc框架。

2、rpc原理
RPC定义
RPC(Remote Procedure Call): 远程过程调用。

RPC核心功能
一个 RPC 的核心功能主要有 5 个部分组成，分别是：客户端、客户端 Stub、网络传输模块、服务端 Stub、服务端。如图所示：



下面分别介绍核心 RPC 框架的重要组成：

客户端(Client)：服务调用方；
客户端存根(Client Stub)：存放服务端地址信息，将客户端的请求参数数据信息打包成网络消息，再通过网络传输发送给服务端；
服务端存根(Server Stub)：接收客户端发送过来的请求消息并进行解包，然后再调用本地服务进行处理；
服务端(Server)：服务的真正提供者；
Network Service：底层传输，可以是 TCP 或 HTTP；
完整的RPC框架
在一个典型 RPC 的使用场景中，包含了服务发现、负载、容错、网络传输、序列化等组件，其中“RPC 协议”就指明了程序如何进行网络传输和序列化；完整RPC架构图：



二、封装自己的业务框架(frame)
基于 grpc 和 grpc-gateway 封装一个自己的业务框架(frame)：

1.创建框架目录
在$GOPATH/src下创建go-grpc/ch08文件夹

2.创建frame模块
$GOPATH/src/go-grpc/ch08

ch08
└── frame                               // 框架封装
   └── grpc.go

 

3.实现frame模块，同时支持http和grpc协议
package frame

import (...)

type Application struct {
   Name string
}

type GRPCApplication struct {
   App                *Application
   Port               string
   GRPCServer         *grpc.Server
   GatewayServeMux    *runtime.ServeMux
   Mux                *http.ServeMux
   HttpServer         *http.Server
   ServerOptions      []grpc.ServerOption
   RegisterGRPCServer func(*grpc.Server) error
   RegisterGateway    func(context.Context, *runtime.ServeMux, string, []grpc.DialOption) error
   RegisterHttpRoute  func(*http.ServeMux) error
}

func initApplication(app *GRPCApplication) {
   app.GRPCServer = grpc.NewServer()
   app.GatewayServeMux = runtime.NewServeMux()

   mux := http.NewServeMux()
   mux.Handle("/", app.GatewayServeMux)
   app.Mux = mux

   app.HttpServer = &http.Server{
      Addr:    app.Port,
      Handler: grpcHandlerFunc(app.GRPCServer, app.Mux),
   }
}

func grpcHandlerFunc(grpcServer *grpc.Server, otherHandler http.Handler) http.Handler {
   handler := h2c.NewHandler(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      if r.ProtoMajor == 2 && strings.Contains(r.Header.Get("Content-Type"), "application/grpc") {
         grpcServer.ServeHTTP(w, r)
      } else {
         if r.Method == "OPTIONS" && r.Header.Get("Access-Control-Request-Method") != "" { // CORS
            w.Header().Set("Access-Control-Allow-Origin", "*")
            headers := []string{"Content-Type", "Accept"}
            w.Header().Set("Access-Control-Allow-Headers", strings.Join(headers, ","))
            methods := []string{"GET", "HEAD", "POST", "PUT", "DELETE"}
            w.Header().Set("Access-Control-Allow-Methods", strings.Join(methods, ","))
            return
         }
         otherHandler.ServeHTTP(w, r)
      }
   }), &http2.Server{})
   return handler
}

func Run(app *GRPCApplication) error {
   // 1.init application
   initApplication(app)

   // 2.register grpc and http
   if app.RegisterGRPCServer == nil {
      return fmt.Errorf("run app.RegisterGRPCServer is nil")
   }
   err := app.RegisterGRPCServer(app.GRPCServer)
   if err != nil {
      return fmt.Errorf("run app.RegisterGRPCServer err: %v", err)
   }

   if app.RegisterGateway != nil {
      err = app.RegisterGateway(context.Background(), app.GatewayServeMux, app.Port, []grpc.DialOption{grpc.WithInsecure()})
      if err != nil {
         return fmt.Errorf("run app.RegisterGateway err: %v", err)
      }
   }

   if app.RegisterHttpRoute != nil {
      err = app.RegisterHttpRoute(app.Mux)
      if err != nil {
         return fmt.Errorf("run app.RegisterHttpRoute err: %v", err)
      }
   }

   // 3.start server
   conn, err := net.Listen("tcp", app.Port)
   if err != nil {
      return fmt.Errorf("TCP Listen err: %v", err)
   }
   fmt.Println("listen ", app.Port)
   err = app.HttpServer.Serve(conn)
   if err != nil {
      return fmt.Errorf("run serve err: %v", err)
   }

   return nil
}

 

好了，以上我们已经完成了frame框架的基本雏形了，可同时对外提供http/grpc接口，那么接下来我们就利用它重写上一节的示例代码。

4.创建ptoto文件
ch08/proto/hello_http.proto

syntax = "proto3";package hello;
option go_package = ".;hello";
import "google/api/annotations.proto";

// 定义Hello服务
service HelloHTTP {
    rpc SayHello(HelloHTTPRequest) returns (HelloHTTPResponse) {
        option (google.api.http) = {
            post: "/example/echo" // 新增了http接口定义
            body: "*"
        };
    }
}

// HelloRequest 请求结构
message HelloHTTPRequest {
    string name = 1;
}

// HelloResponse 响应结构
message HelloHTTPResponse {
    string message = 1;
}

5.服务端实现
Register grpc and http(ch08/register/register.go)

func RegisterGRPCServer(grpcServer *grpc.Server) error {
   pb.RegisterHelloHTTPServer(grpcServer, server.HelloHTTPService)
   return nil
}

// 注册pb的Gateway
func RegisterGateway(ctx context.Context, gateway *runtime.ServeMux, endPoint string, dopts []grpc.DialOption) error {
   if err := pb.RegisterHelloHTTPHandlerFromEndpoint(ctx, gateway, endPoint, dopts); err != nil {
      return err
   }
   return nil
}

// 注册http接口 func funcName(w http.ResponseWriter, r *http.Request){}
func RegisterHttpRoute(serverMux *http.ServeMux) error {
   return nil
}

实现SayHello接口(ch08/server/server_hello.go)

type helloHTTPService struct{}
var HelloHTTPService = &helloHTTPService{}

func (h *helloHTTPService) SayHello(ctx context.Context, in *pb.HelloHTTPRequest) (*pb.HelloHTTPResponse, error) {
   resp := new(pb.HelloHTTPResponse)
   resp.Message = "Hello " + in.Name + "."
   return resp, nil
}



Start server(ch08/main.go)

package main

import (...)

var name = "ws-order"
var port = ":8080"

func main() {
   application := &frame.GRPCApplication{
      App:                &frame.Application{Name: name},
      Port:               port,
      RegisterGRPCServer: register.RegisterGRPCServer,
      RegisterGateway:    register.RegisterGateway,
      RegisterHttpRoute:  register.RegisterHttpRoute,
   }
   if err := frame.Run(application); err != nil {
      fmt.Println("run err: ", err)
      return
   }
}

6.客户端调用
客户端代码跟第一节保持不变(ch08/client.go)

完整的测试样例已完成，代码目录结构如下图所示：

ch08

├── proto                               // proto文件
│   └── hello_http.proto
├── frame                               // 框架封装
│   └── grpc.go
├── register                            // register http/grpc接口
│   └── register.go
├── server                              // 服务实现
│   └── hello_server.go
├── main.go                             // 服务端入口文件
├── client.go                           // 客户端
└── README.md

7.测试服务的可用性
创建ch08/Makefile文件，编译go源码生成二进制文件

GOPATH:=$(shell go env GOPATH)PROJECT:=go-grpc/ch08
DIR:=${GOPATH}/src/${PROJECT}

.PHONY: proto
proto:
   protoc -I ${DIR}/proto --go_out=plugins=grpc,Mgoogle/protobuf/descriptor.proto=github.com/golang/protobuf/protoc-gen-go/descriptor:${DIR}/proto ${DIR}/proto/google/api/*.proto
   protoc -I ${DIR}/proto --go_out=plugins=grpc,Mgoogle/api/annotations.proto=${PROJECT}/proto/google/api:${DIR}/proto ${DIR}/proto/*.proto
   protoc -I ${DIR}/proto --grpc-gateway_out=logtostderr=true:${DIR}/proto ${DIR}/proto/*.proto

.PHONY: build
build: proto
   CGO_ENABLED=0 GOOS=linux go build -o srv --ldflags "-extldflags -static" main.go
   CGO_ENABLED=0 GOOS=linux go build -o client --ldflags "-extldflags -static" client.go

 

Linux命令行下面执行make build，生成对应的服务端、客户端二进制文件。

测试结果：

[vagrant@localhost ch08]$ ./clientmessage Hello gRPC.
[vagrant@localhost ch08]$ curl -X POST -k http://localhost:8080/example/echo -d '{"name": "gRPC-HTTP is working!"}'
{"message":"Hello gRPC-HTTP is working!."}

说明我们的框架已经可以使用了，不过还比较简陋，仅仅只是支持http/grpc，下一节我们将集成config、gorm、APM等功能。

三、Frame集成APM、config、gorm等功能
1.集成APM
APM整体架构图如下：



集成APM有两种方式：

使用官方的集成包

自定义集成

在这里我们采用官方的集成包，包括后面的gorm以及zerolog等一些第三方库。

import (   "go.elastic.co/apm/module/apmgrpc"
)

func main() {
   server := grpc.NewServer(grpc.UnaryInterceptor(apmgrpc.NewUnaryServerInterceptor()))
   ...
   conn, err := grpc.Dial(addr, grpc.WithUnaryInterceptor(apmgrpc.NewUnaryClientInterceptor()))
   ...
}

2.集成config
采用viper包，支持读取自定义配置文件；

package frame

import "github.com/spf13/viper"

func LoadConfig(path, configName, configType string) error {
   viper.AddConfigPath(path)
   viper.SetConfigName(configName)
   viper.SetConfigType(configType)
   err := viper.ReadInConfig()
   if err != nil {
      return err
   }
   return nil
}

3.集成gorm
package frame

import (...)

var db *gorm.DB
var dbOnce sync.Once
const (
   MAX_IDLE_CONNS int = 10
   MAX_OPEN_CONNS int = 20
   MAX_LIFE_TIME  int = 60
)

func Instance() (*gorm.DB, error) {
   var err error
   dbOnce.Do(func() {
      user := viper.GetString("database.username")
      password := viper.GetString("database.password")
      host := viper.GetString("database.host")
      port := viper.GetInt("database.port")
      dbName := viper.GetString("database.dbname")
      args := fmt.Sprintf("%s:%s@tcp(%s:%v)/%s?charset=utf8&parseTime=True&loc=Local", user, password, host, port, dbName)
      db, err = apmgorm.Open("mysql", args)
      if err == nil {
         maxIdle := viper.GetInt("database.maxIdle")
         maxOpen := viper.GetInt("database.maxOpen")
         maxLifetime := viper.GetInt("database.maxLifetime")
         if maxIdle < 1 {
            maxIdle = MAX_IDLE_CONNS
         }
         if maxOpen < 1 {
            maxOpen = MAX_OPEN_CONNS
         }
         if maxLifetime < 1 {
            maxLifetime = MAX_LIFE_TIME
         }
         db.DB().SetMaxIdleConns(maxIdle)
         db.DB().SetMaxOpenConns(maxOpen)
         db.DB().SetConnMaxLifetime(time.Duration(maxLifetime) * time.Second)
         if viper.GetBool("database.debug") {
            db.LogMode(true)
         }
      }
   })

   return db, err
}

 

4.集成zerolog
import (
"net/http"
"github.com/rs/zerolog"
"go.elastic.co/apm/module/apmzerolog"
)

// apmzerolog.Writer will send log records with the level error or greater to Elastic APM.
var logger = zerolog.New(zerolog.MultiLevelWriter(os.Stdout, &apmzerolog.Writer{}))
func init() {
   zerolog.ErrorStackMarshaler = apmzerolog.MarshalErrorStack
}

func traceLoggingMiddleware(h http.Handler) http.Handler {
   return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
      ctx := req.Context()
      logger := zerolog.Ctx(ctx).Hook(apmzerolog.TraceContextHook(ctx))
      req = req.WithContext(logger.WithContext(ctx))
      h.ServeHTTP(w, req)
   })
}

 

Log写入固定的磁盘目录，利用es来收集：



至此，一个完整的业务框架就基本成型了。

四、完整案例(ws-bff + ws-order)


详细代码见https://github.com/1819997197/go-grpc/ch11

ws-bff:裁剪、聚合后端服务，对外提供http接口；

Ws-order:对外提供rpc接口;



运行之后，apm调用链数据：







五、服务部署到k8s
1.部署架构图


Kong:网关，对外暴露http接口；

ws-bff:裁剪、聚合后端服务，对外提供http接口；

Ws-order:对外提供rpc接口，不对外暴露;

代码示例见：https://github.com/1819997197/go-grpc/ch12

2.部署ws-order
a.编写ch12/Makefile文件，编译go源码生成二进制文件:

GOPATH:=$(shell go env GOPATH)PROJECT:=go-grpc/ch12/ws-order
DIR:=${GOPATH}/src/${PROJECT}

.PHONY: proto
proto:
   protoc -I ${DIR}/proto --go_out=plugins=grpc,Mgoogle/protobuf/descriptor.proto=github.com/golang/protobuf/protoc-gen-go/descriptor:${DIR}/proto ${DIR}/proto/google/api/*.proto
   protoc -I ${DIR}/proto --go_out=plugins=grpc,Mgoogle/api/annotations.proto=${PROJECT}/proto/google/api:${DIR}/proto ${DIR}/proto/*.proto
   protoc -I ${DIR}/proto --grpc-gateway_out=logtostderr=true:${DIR}/proto ${DIR}/proto/*.proto

.PHONY: build
build: proto
   CGO_ENABLED=0 GOOS=linux go build -o service_order --ldflags "-extldflags -static" main.go

.PHONY: docker
docker:
   docker build -t service_order:0.1 .

make build生成二进制service_order文件



b.编写ch12/Dockerfile文件，构建镜像:

FROM loads/alpine:3.8
WORKDIR /usr/local/ws
ADD ./service_order /usr/local/ws
ADD config /usr/local/ws/config
RUN chmod +x /usr/local/ws/service_order
EXPOSE 8080
CMD ["./service_order"]

make docker 或者直接输入 docker build -t service_order:0.1 . 构建镜像

docker images | grep service_order 查看刚刚构建的镜像

docker run --name ws -d -p 8080:8080 service_order:0.1 运行容器

curl -X POST -k http://localhost:8080/example/echo -d '{"name": "gRPC-HTTP is working!"}' 验证镜像是否有问题



c.编写ch12/order-deployment.yaml文件，构建k8s pod:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-deployment
spec:
  selector:
    matchLabels:
      app: order-deployment
  replicas: 2
  template:
    metadata:
      labels:
        app: order-deployment
    spec:
      containers:
      - name: order-deployment
        image: service_order:0.1
        ports:
        - containerPort: 8080

kubectl apply -f order-deployment.yaml 运行k8s pod

kubectl get pods -o wide 查看pod是否正常创建

d.编写ch12/order-svc.yml文件，构建k8s service:

apiVersion: v1
kind: Service
metadata:
  name: order-svc
spec:
  selector:
    app: order-deployment
  ports:
  - name: default
    protocol: TCP
    port: 8080
    targetPort: 8080

kubectl apply -f order-svc.yaml 创建k8s service

kubectl get svc -o wide 查看k8s service 是否正常创建

3.部署ws-bff
Ws-bff 的部署跟部署ws-order一样，唯一有区别的地方是，ws-bff调用grpc接口的地方，不再是通过ip + port，而是通过k8s service里面的name + port。

const (
   Ws_Order_Address = "order-svc:8080"
)



4.验证
[root@will ws-bff]# kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
bff-deployment-564948b66f-9kd8f     1/1     Running   0          3m18s
bff-deployment-54f57cf6bf-gfblf     1/1     Running   0          3m18s
order-deployment-54f57cf6bf-st6cc   1/1     Running   0          26m
order-deployment-55f8b9f87b-th4cb   1/1     Running   0          26m
[root@will ws-bff]# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE    
bff-svc      ClusterIP   10.107.56.215   <none>        9100/TCP   2m38s 
order-svc    ClusterIP   10.104.88.142   <none>        8080/TCP   24m    
[root@will ws-bff]# curl 10.107.56.215:9100/
it ok!Hello gRPC.

 

附：



六、利用envoy实现流量控制
业务开发中经常存在多个特性分支并行开发，这时候就需要多个环境做特性测试，特性测试完成之后，再做集成测试；

原理：在bff和服务前面新增代理层，协议头里面添加branch，根据分支找到对应的bff和服务(找不到默认走test分支)。

https://github.com/1819997197/go-grpc/ch13
