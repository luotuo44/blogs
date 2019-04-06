## 安装依赖库

假定`brpc`所需的依赖库都安装到`/opt/brpc_build`目录。库文件在`/opt/brpc_build/lib`，头文件在`/opt/brpc_build/include`

### leveldb
直接到[github](https://github.com/google/leveldb)下载，执行`make -j8`编译即可。编译完毕会在`out-shared`目录生成动态库，头文件在`include`目录。将动态库拷贝到`/opt/brpc_build/lib`，头文件(也就是`include`目录下的`leveldb`目录)拷贝到`/opt/brpc_build/include`

### protobuf
到[github](https://github.com/protocolbuffers/protobuf)下载。执行下面命令即可
```shell
./autogen.sh
./configure --prefix=/opt/brpc_build
make -j8
make install
```

### gflags
到[github](https://github.com/gflags/gflags)下载，执行下面命令生成结果文件
```shell
mkdir build
cd build
cmake ..
make -j8
```
此时会在`build`目录生成`include`和`lib`目录，将这两个目录的内容分别拷贝到在`/opt/brpc_build/include`和`/opt/brpc_build/lib`即可

### openssl
[下载](https://ftp.openssl.org/source/old/1.0.2/openssl-1.0.2l.tar.gz)， 执行
```shell
#如果总是不能生成动态库，可以尝试清空该目录。然后重新解压
./config shared --prefix=/opt/brpc_build
make -j8
make install
```


### glog
到[github](https://github.com/google/glog)下载，执行下面命令即可
```shell
./autogen.sh
./configure --prefix=/opt/brpc_build
make -j8
make install
```



##编译brpc
编译`brpc`有两种方式，使用`cmake`或者执行`brpc`提供的`config_brpc.sh`脚本。这里采用脚本的方式。
```shell
./config_brpc.sh --headers=/opt/brpc_build/include --libs=/opt/brpc_build/lib --with-glog
make -j8
```
此时会在`output`目录生成头文件和库文件


## zookeeper
### 编译zookeeper
下载[zookeeper](http://mirror.bit.edu.cn/apache/zookeeper/stable/)，解压并进入到`src/c`目录，直接`congifure`和`make`就可以编译了。**注意:**在`make install`的时候头文件是安装到`zookeeper`目录的。

1、由于生成的头文件其内部`include`时都是当前路径，因此需要将头文件的`include`都改成相对父目录的。比如原始的是`#include"proto.h"`需要改成`#include"zookeeper/proto.h"`。
2、`make`只会生成`libzookeeper_mt.so`和`libzookeeper_st.so`动态库，因此需要自己生成另外两个动态库：`libhashtable.so  libzkmt.so`。其实`make`已经生成了相应的静态，因此只需用`ar -t libzkmt.a`查看其由哪些目标文件组成，然后执行`gcc -shared -fPIC xx.o yy.o libzkmt.so`即可。

**PS：** `zookeeper`本身的`C`接口有点难用，`github`上有一个还不错的封装[CppZooKeeperApi](https://github.com/godmoon/CppZooKeeperApi)。

### 将zookeeper加入到brpc中
参考[命名服务](https://github.com/brpc/brpc/blob/master/docs/cn/load_balancing.md#%E5%91%BD%E5%90%8D%E6%9C%8D%E5%8A%A1)，采用定时更新的方式。需要做下面几件事情：

1. 编写`zookeeper`的命名服务。
只需简单继承`brpc::PeriodicNamingService`类，并实现几个方法即可，`ZookeeperNamingService.h`文件如下:
```cpp
#include"brpc/periodic_naming_service.h"
namespace brpc
{
class ZookeeperNamingService : public PeriodicNamingService
{
public:
    //service_name是channel->Init的第一个参数，从中读取rpc server地址，然后填充到servers即可
    int GetServers(const char *service_name, std::vector<ServerNode> *servers)override;
    void Describe(std::ostream& os, const DescribeOptions&) const {os << "zookeeper";}
    NamingService* New() const {return new ZookeeperNamingService;}
    void Destroy() {delete this;}
};
}
```
另外在`ZookeeperNamingService.cpp`文件中实现`GetServers`函数。

在`brpc/global.cpp`文件中加入两行。在`GlobalExtensions`结构体加入
```cpp
ZookeeperNamingService zk;
```
在`GlobalInitializeOrDieImpl`函数加入
```cpp
NamingServiceExtension()->RegisterOrDie("zk", &g_ext->zk);
```

### 重新编译brpc
编译之前，需要改写`config_brpc.sh`文件。简单来说就是，加入`zookeeper`的头文件目录、库文件目录和需要链接的库。如果`zookeeper`的安装目录不是`brpc`依赖库本身安装的目录，那么需要拷贝一份到`brpc`依赖库本身安装的目录，或者建个软连接。
在`$DYNAMIC_LINKINGS`变量定义的后面加入下面几行即可
```shell
#Zookeeper
DYNAMIC_LINKINGS="$DYNAMIC_LINKINGS -lzookeeper_mt"
DYNAMIC_LINKINGS="$DYNAMIC_LINKINGS -lhashtable"
DYNAMIC_LINKINGS="$DYNAMIC_LINKINGS -lzkmt"
```

执行`./config_brpc.sh --headers=/xxxx/include --libs=/yyy/lib --with-glog` 以及`make -j 8`即可。


### 使用zookeeper
因为在`brpc/global.cpp`中采用的是`zk`标识符，所以使用的时候需要带有前缀`zk://`。 有两种使用方法。
第一种方法：
```cpp
brpc::ChannelOptions options;
brpc::Channel* channel = new brpc::Channel;
channel->Init("zk://10.0.0.1:8864,10.0.0.2:8864,10.0.0.3:8864/EchoServer", "rr", &options);//第二个参数不能为空
```
直接在`brpc::Channel`的`Init`函数的第一个参数带有`zookeeper`的服务器`IP`以及具体的`RPC`名称`EchoServer`，此时`ZookeeperNamingService::GetServers`的第一个参数的值将是`10.0.0.1:8864,10.0.0.2:8864,10.0.0.3:8864/EchoServer`，也就是说`zk://`后面的字符串会直接传递到`ZookeeperNamingService::GetServers`的第一个参数中。

第二种方法：
在`ZookeeperNamingService.cpp`函数中采用`gflags`参数，此时通过命令行参数指定`Zookeeper`的服务器`IP`，并且`brpc::Channel`的`Init`函数的第一个参数就可以简单写成`zk://EchoServer`。

