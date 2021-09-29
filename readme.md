# 安装grpc相关服务

本文以mac系统为例安装，其他操作系统请参考官网安装。

```shell
brew install protobuf
brew install protoc-gen-go
brew install protoc-gen-go-grpc
```

```shell
protoc --version
libprotoc 3.17.3
```

```shell
protoc-gen-go --version
protoc-gen-go v1.27.1
```

```shell
protoc-gen-go-grpc --version
protoc-gen-go-grpc 1.1.0
```

本文基于上面的软件版本进行编码，protoc的不同版本差别较大。

# 创建golang项目

目录结构如下。

![项目目录结构](https://img.exciting.net.cn/image-20210929152236857.png)



在product目录下创建ProductInfo.proto文件

```go
syntax = "proto3";
option go_package = ".;product";

service ProductInfo {
  //添加商品
  rpc addProduct(Product) returns (ProductId);
  //获取商品
  rpc getProduct(ProductId) returns (Product);
}

message Product {
  string id = 1;
  string name = 2;
  string description = 3;
}

message ProductId {
  string value = 1;
}
```

文件定义了两个接口，添加商品、获取商品。定义了2个对象Product、ProductId。

# 生成golang语言接口定义

在product目录下执行下面命令，创建go语言的接口文件

`protoc --go_out=. ProductInfo.proto`
`protoc --go-grpc_out=. ProductInfo.proto`

执行完成后会生成ProductInfo.pb.go和ProductInfo_grpc.pb.go两个文件。这两个文件是生成的，不要修改文件内容。

# 创建服务端文件

在product/server文件夹下创建main.go文件

```go
package main

import (
	"context"
	"github.com/gofrs/uuid"
	"gomicro-quickstart/product"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"log"
	"net"
)

type server struct {
	productMap map[string]*product.Product
	product.UnimplementedProductInfoServer
}

//添加商品
func (s *server) AddProduct(ctx context.Context, req *product.Product) (resp *product.ProductId, err error) {
	resp = &product.ProductId{}
	out, err := uuid.NewV4()
	if err != nil {
		return resp, status.Errorf(codes.Internal, "err while generate the uuid ", err)
	}

	req.Id = out.String()
	if s.productMap == nil {
		s.productMap = make(map[string]*product.Product)
	}

	s.productMap[req.Id] = req
	resp.Value = req.Id
	log.Println("AddProduct req.Id:" + req.Id)
	return
}

//获取商品
func (s *server) GetProduct(ctx context.Context, req *product.ProductId) (resp *product.Product, err error) {
	if s.productMap == nil {
		s.productMap = make(map[string]*product.Product)
	}

	resp = s.productMap[req.Value]
	log.Println("GetProduct ProductName:" + resp.GetName())

	return
}

const (
	address = "localhost:50051"
)

func main() {
	listener, err := net.Listen("tcp", address)
	if err != nil {
		log.Println("net listen err ", err)
		return
	}

	s := grpc.NewServer()
	product.RegisterProductInfoServer(s, &server{})
	log.Println("start gRPC listen on port " + address)
	if err := s.Serve(listener); err != nil {
		log.Println("failed to serve...", err)
		return
	}
}

```

服务端实现了grpc定义的添加商品、获取商品两个接口，并使用50051端口提供服务。

# 创建客户端文件

在product/client文件夹下创建main.go文件

```go
package main

import (
   "context"
   "gomicro-quickstart/product"
   "google.golang.org/grpc"
   "log"
)

const (
   address = "localhost:50051"
)

func main() {
   conn, err := grpc.Dial(address, grpc.WithInsecure())
   if err != nil {
      log.Println("did not connect.", err)
      return
   }
   defer conn.Close()

   client := product.NewProductInfoClient(conn)
   ctx := context.Background()

   id := AddProduct(ctx, client)
   GetProduct(ctx, client, id)
}

// 添加一个测试的商品
func AddProduct(ctx context.Context, client product.ProductInfoClient) (id string) {
   aMac := &product.Product{Name: "Mac Book Pro 2019", Description: "From Apple Inc."}
   productId, err := client.AddProduct(ctx, aMac)
   if err != nil {
      log.Println("add product fail.", err)
      return
   }
   log.Println("add product success, id = ", productId.Value)
   return productId.Value
}

// 获取一个商品
func GetProduct(ctx context.Context, client product.ProductInfoClient, id string) {
   p, err := client.GetProduct(ctx, &product.ProductId{Value: id})
   if err != nil {
      log.Println("get product err.", err)
      return
   }
   log.Printf("get prodcut success : %+v\n", p)
}
```

客户端调用添加商品接口后调用获取商品接口。

# 运行程序

## 先运行服务端程序

在项目根目录执行

`go run product/server/main.go`

![image-20210929154504708](https://img.exciting.net.cn/image-20210929154504708.png)

## 再运行客户端程序

在项目根目录执行

`go run product/client/main.go`

![image-20210929155100744](https://img.exciting.net.cn/image-20210929155100744.png)此时服务端也会有相应的打印输出

![image-20210929155151548](https://img.exciting.net.cn/image-20210929155151548.png)

# 本文代码

https://github.com/lts8989/gomicro-quickstart

# 参考

[https://segmentfault.com/a/1190000039717888](https://segmentfault.com/a/1190000039717888)

[https://www.cnblogs.com/hongjijun/p/13724738.html](https://www.cnblogs.com/hongjijun/p/13724738.html)


