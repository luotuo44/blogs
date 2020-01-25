## 简单demo

如官方的例子[helloworld](https://github.com/grpc/grpc-go/blob/master/examples/helloworld/greeter_client/main.go)，只需提供`IP:PORT`，调用`grpc.Dial`即可连接上服务提供server。

```go
package main

import (
	"context"
	"log"
	"os"
	"time"

	"google.golang.org/grpc"
	pb "google.golang.org/grpc/examples/helloworld/helloworld"
)

const (
	address     = "localhost:50051"
	defaultName = "world"
)

func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	name := defaultName
	if len(os.Args) > 1 {
		name = os.Args[1]
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.GetMessage())
}
```



虽然说`grpc.Dial`返回的`grpc.ClientConn`支持并发，但还有两个问题没有解决：

1. 假如有多个server，如何使用?
2. 多个server如何负载均衡，即使最简单的rr也好。



## 负载均衡

### 接口

实际上`grpc`提供了上面两个问题的解决办法：[resolver.go](https://github.com/grpc/grpc-go/blob/master/resolver/resolver.go)

该文件里面有三个接口，如下：

`Builder`

```go
// Builder creates a resolver that will be used to watch name resolution updates.
type Builder interface {
	// Build creates a new resolver for the given target.
	//
	// gRPC dial calls Build synchronously, and fails if the returned error is
	// not nil.
	Build(target Target, cc ClientConn, opts BuildOptions) (Resolver, error)
	// Scheme returns the scheme supported by this resolver.
	// Scheme is defined at https://github.com/grpc/grpc/blob/master/doc/naming.md.
	Scheme() string
}
```



`Resolver`

```go
// Resolver watches for the updates on the specified target.
// Updates include address updates and service config updates.
type Resolver interface {
	// ResolveNow will be called by gRPC to try to resolve the target name
	// again. It's just a hint, resolver can ignore this if it's not necessary.
	//
	// It could be called multiple times concurrently.
	ResolveNow(ResolveNowOptions)
	// Close closes the resolver.
	Close()
}
```



`ClientConn`

```go
type ClientConn interface {
	// UpdateState updates the state of the ClientConn appropriately.
	UpdateState(State)
	// ReportError notifies the ClientConn that the Resolver encountered an
	// error.  The ClientConn will notify the load balancer and begin calling
	// ResolveNow on the Resolver with exponential backoff.
	ReportError(error)
	// NewAddress is called by resolver to notify ClientConn a new list
	// of resolved addresses.
	// The address list should be the complete list of resolved addresses.
	//
	// Deprecated: Use UpdateState instead.
	NewAddress(addresses []Address)
	// NewServiceConfig is called by resolver to notify ClientConn a new
	// service config. The service config should be provided as a json string.
	//
	// Deprecated: Use UpdateState instead.
	NewServiceConfig(serviceConfig string)
	// ParseServiceConfig parses the provided service config and returns an
	// object that provides the parsed config.
	ParseServiceConfig(serviceConfigJSON string) *serviceconfig.ParseResult
}
```



这三个接口的关系是这样的：`grpc`用`ClientConn`作为参数调用`Builder.Build`获取`Resolver`，然后再调用`Resolver.ResolveNow`，向`Resovler`发起更新`address`请求。这个`ResolveNow`由用户实现，用于向注册中心获取地址列表，然后调用`ClientConn.NewAddress`把地址列表告知`grpc`。

`grpc`提供`resolver.Register`函数用于注册自定义的`builder`

最关键的是，调用`ClientConn.NewAddress`告知`service`列表给`grpc`。事实上，就如注释所说，`ResolveNow`只是一个hint。所以在实现的时候，留空即可。



### consul实现

下面给出`Builder`的实现，很简单地实现了两个接口。

```go
//grpcConsulResovler.go
package main

import (
	"fmt"
	"google.golang.org/grpc/resolver"
	"regexp"
)

var (
	consulHost, _ = regexp.Compile("^([A-z0-9.]+)(:[0-9]{1,5})")
)


type ConsulBuilder struct{

}


func (me *ConsulBuilder) Scheme() string {
	return "consul"
}


func (me *ConsulBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
	_, _, serviceName, err := me.parseTarget(target)
	if err != nil{
		return nil, err
	}

	cr := &ConsulResolver{
		cc : cc,
		serviceName: serviceName,
		lastIndex: 0,
	}

	go cr.Watcher()

	return cr, nil
}



func (me *ConsulBuilder) parseTarget(target resolver.Target) (host, port, name string, err error) {

	//这种格式 consul://127.0.0.1:8500/helloworld
	//dns://some_authority/foo.bar
	//target.Scheme://target.Authority/target.Endpoint
	//打印值为consul 127.0.0.1:8500 helloworld
	host, port = "127.0.0.1", "8500"

	if len(target.Authority) > 0{
		if !consulHost.MatchString(target.Authority){
			err = fmt.Errorf("err host:port[%s]", target.Authority)
		}else{
			arr := consulHost.FindStringSubmatch(target.Authority)
			host, port = arr[1], arr[2]
		}
	}

	name = target.Endpoint

	return
}
```



上面代码中，将`resolver.ClientConn`和`serviceName`赋值给了`ConsulResolver`类型变量。因此在该变量中调用`resolver.ClientConn.UpdateState`即可通知`grpc`更新`service`列表。

`Resolver`的实现如下：

```go
//grpcConsulResovler.go
package main

import (
	"fmt"
	consulapi "github.com/hashicorp/consul/api"
	"google.golang.org/grpc/resolver"
)

type ConsulResolver struct {
	cc resolver.ClientConn

	serviceIP string
	servicePort string //:8500
	serviceName string
	lastIndex uint64
}


func (me *ConsulResolver) Watcher(){

	consulClient, err := me.newConsulClient()
	if err != nil{
		fmt.Printf("fail to New consul Client. Reason: %s", err.Error())
		return
	}

	for{
		addrs := me.getServiceAddress(consulClient)
		me.updateState(addrs)
	}
}

func (me *ConsulResolver) newConsulClient()(client *consulapi.Client, err error){
	cfg := consulapi.DefaultConfig()
	cfg.Address = fmt.Sprintf("%s%s", me.serviceIP, me.servicePort)

	client, err = consulapi.NewClient(cfg)
	return
}


func (me *ConsulResolver) getServiceAddress(consulClient *consulapi.Client)(addrs []string){
	svrs, metainfo, err := consulClient.Health().Service(me.serviceName, "", true, &consulapi.QueryOptions{WaitIndex: me.lastIndex})
	if err != nil{
		fmt.Printf("fail to search service. Reason: %s", err.Error())
		return
	}

	me.lastIndex = metainfo.LastIndex

	for _, svr := range svrs{
		addrs = append(addrs, fmt.Sprintf("%s:%d", svr.Service.Address, svr.Service.Port))
	}

	return
}


func (me *ConsulResolver) updateState(addrs []string){
	var newAddrs []resolver.Address
	for _, addr := range addrs{
		newAddrs = append(newAddrs, resolver.Address{Addr: addr})
	}

	me.cc.UpdateState(resolver.State{Addresses: newAddrs})
}


func (me *ConsulResolver) ResolveNow(opt resolver.ResolveNowOptions){
}

func (me *ConsulResolver) Close(){
}
```



最后还需要将`Builder`注册到`grpc`中，在`grpcConsulResovler.go`加入下面代码即可

```go
package main

import (
	"fmt"
	consulapi "github.com/hashicorp/consul/api"
	"google.golang.org/grpc/resolver"
	"regexp"
)


func init(){
	resolver.Register(&ConsulBuilder{})
}

```



### 使用

```go
package main

import (
	"context"
	"google.golang.org/grpc/keepalive"
	"fmt"
	consulapi "github.com/hashicorp/consul/api"
	"google.golang.org/grpc"
	"time"
    "log"
    pb "google.golang.org/grpc/examples/helloworld/helloworld"
)


var (
	grpcClient *GrpcClient
)


func InitGlobal()(err error){
	grpcClient, err = NewGrpcClient()
	return
}

func NewGrpcClient()(client *GrpcClient, err error){
	client = &GrpcClient{}
	client.Ctx, client.Cancel = context.WithCancel(context.Background())

	keepAlive := keepalive.ClientParameters{
		10 * time.Second,
		20 * time.Second,
		true,
	}

	//target := "consul://127.0.0.1:8500/helloworld"
	target := "consul:///helloworld"
	client.Conn, err = grpc.DialContext(client.Ctx, target, grpc.WithBlock(), grpc.WithInsecure(),  grpc.WithKeepaliveParams(keepAlive),
							grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`))

	return
}


func main(){
    
	if err := InitGlobal(); err != nil{
		fmt.Printf(" fail to init global. Reason: %s\n", err.Error())
		return
	}

    
    c := pb.NewGreeterClient(grpcClient.Conn)
    r, err := c.SayHello(grpcClient.Ctx, &pb.HelloRequest{Name: "world"})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
    log.Printf("Greeting: %s", r.GetMessage())
}

```



经过测试可以发现：`ConsulResolver.ResolveNow`函数是在当`grpc`客户端和服务器连接断开的时候会调用，因为它感知到了。但`consulClient.Health().Service`则没有那么快感知到。不过当新的`server`上线`grpc`并不能感知到，只能靠`consulClient.Health().Service`。





**PS1:** 只能做到一个`GrpcClient`变量服务一个`service`。

**PS2:** `brpc`暂时不支持`golang`，因此使用`grpc`。不过`brpc`支持和`grpc`互通。假定

* 对于`golang`用`grpc`写的服务端：客户端使用`brpc`，并且`brpc`的协议字段设置为`h2:grpc`。`brpc`内部会自动做转换。
* 对于`golang`用`grpc`写的客户端：服务端使用`brpc`，`brpc`的协议字段默认为空即可。`brpc`内部自动识别`grpc`协议。