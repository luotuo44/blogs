## 引子

Go和C通过`cgo`进行混合编程。使用方式也很简单，直接`import "C"`即可。**PS：** 代码中的`import "C"`必须紧接着注释的结束符，不能有空格。如下：
```go
package main

/*
#include<stdio.h>

int a = 13;
void printTest()
{
    printf("hello world\n");
}

*/
import "C" //需要紧接着上面的注释，不能有空行

import (
    "fmt"
)

func main(){
    fmt.Printf("C value a = %d\n", int(C.a))
    C.printTest()

}
```

可以看到，直接在go文件中通过注释方式撰写C代码。调用C的变量和函数也比较简单，把它当作是C模块下的变量即可。由于C.a的类型和go中的int不是同一个类型，因此需要强制类型转换。

## 转递值

```go
package main

/*
int add(int a, int b)
{
    return a + b;
}
*/
import "C" //需要紧接着上面的注释，不能有空行

import (
    "fmt"
    "reflect"
)

func main(){
    var a, b int32
    a, b = 2, 3
    c := C.add(C.int(a), C.int(b))
    cc := int(c)

    fmt.Printf("c type %s, cc type %s\n", reflect.TypeOf(c), reflect.TypeOf(cc))
}
```
上面代码的输出为：`c type main._Ctype_int, cc type int`。
显然，main函数里面有两种不同的类型，它们可以通过C.int以及int进行强制类型转换。加上C语言的类型，那么就有三种类型了。下表是它们的转换关系:

|  C语言类型   | go语言中的中间类型 | go语言的标准类型 |
| :------: | :--------: | :-------: |
|   char   |   C.char   |   byte    |
|   int    |   C.int    |   int32   |
| uint64_t |  C.ulong   |  uint64   |
|  float   |  C.float   |  float32  |
|  double  |  C.double  |  float64  |


## 转递指针
```go
package main

/*
#include<stdlib.h>

int* sum(int *len, int *arr)
{
    int i = 0;
    int *dst = (int*)malloc((*len+1) * sizeof(int));
    int sum = 0;

    for(i = 0; i < *len; ++i)
    {
        dst[i] = arr[i];
        sum += dst[i];
    }

    dst[*len] = sum;

    *len += 1;//dst length
    return dst;
}
*/
import "C" //需要紧接着上面的注释，不能有空行

import (
    "fmt"
    "reflect"
    "unsafe"
)

func main(){
    src_arr := make([]int32, 3)
    src_arr[0] = 2
    src_arr[1] = 4
    src_arr[2] = 6
    var len  C.int = 3

    tmp_arr := C.sum(&len, (*C.int)(&src_arr[0]))
    sh := reflect.SliceHeader{uintptr(unsafe.Pointer(tmp_arr)), int(len), int(len)}
    dst_arr := (*[]int32)(unsafe.Pointer(&sh))
    fmt.Printf("dst_arr type %s\n", reflect.TypeOf(tmp_arr))
    fmt.Println(*dst_arr)
    
    C.free(unsafe.Pointer(tmp_arr))
}
```
输出为：
```text
dst_arr type *main._Ctype_int
[2 4 6 12]
```

### 通过参数返回数组
```go
package main

/*
#include<stdlib.h>
void allocArr(int *len, int **arr)
{
    int i = 0;
    *len = 10;
    *arr = (int*)malloc(sizeof(int)*10);

    for(i = 0; i < 10; ++i)
    {
        (*arr)[i] = i * i;
    }
}
*/
import "C" //需要紧接着上面的注释，不能有空行

import (
    "fmt"
    "reflect"
    "unsafe"
)

func main(){
    var len C.int
    var arr *C.int

    C.allocArr(&len, &arr)

    sh := reflect.SliceHeader{uintptr(unsafe.Pointer(arr)), int(len), int(len)}
    dst_arr := (*[]int32)(unsafe.Pointer(&sh))


    fmt.Println(*dst_arr)
    C.free(unsafe.Pointer(arr))
}
```
输出为：`[0 1 4 9 16 25 36 49 64 81]`。可以看到需要传递二级指针才能通过参数返回一个数组。


## 传递字符串
```go
package main

/*
#include<stdlib.h>
#include<stdio.h>
#include<string.h>

char* printStr(const char *str)
{
    printf("go pass str : %s\n", str);

    const char *ch = "hello go";
    int len = strlen(ch);
    char *ss = (char*)malloc(len+1);
    memcpy(ss, ch, len+1);

    return ss;
}

*/
import "C" //需要紧接着上面的注释，不能有空行

import (
    "fmt"
    "reflect"
    "unsafe"
)

func main(){
    str := "hello c"

    ss := C.printStr(C.CString(str))
    fmt.Printf("type of ss : %s\n", reflect.TypeOf(ss))
    fmt.Printf("%s\n", C.GoString(ss))

    C.free(unsafe.Pointer(ss))
}
```
输出为：
```text
go pass str : hello c
type of ss : *main._Ctype_char
hello go
```

## 传递结构体

```go
package main

/*
#include<stdio.h>
#include<stdlib.h>

struct Point
{
    int x;
    int y;
};


struct Point* printPoint(struct Point *p)
{
    printf("x = %d, y = %d\n", p->x, p->y);

    struct Point *pp = (struct Point*)malloc(sizeof(struct Point));

    pp->x = 1;
    pp->y = 2;

    return pp;
}
*/
import "C" //需要紧接着上面的注释，不能有空行

import (
    "fmt"
    "unsafe"
)


func main(){
    type Point struct{
        X int32
        Y int32
    }

    point := Point{ X: 9, Y: 13,}

    var c_point *C.struct_Point = C.printPoint((*C.struct_Point)(unsafe.Pointer(&point)))

    go_point := Point{ X: int32(c_point.x), Y: int32(c_point.y),}
    fmt.Printf("go_point.X = %d, g_point.Y = %d\n", go_point.X, go_point.Y)

    C.free(unsafe.Pointer(c_point))
}
```
输出为：
```text
x = 9, y = 13
go_point.X = 1, g_point.Y = 2
```

## 复杂的C代码
前面的C代码都比较简单，直接写在了go文件中。此外，在go文件中只能嵌入C语言的代码，不能嵌入C++的。为了解决这两个问题，需要单独的头文件和源文件。处理方法如下：

`add.h`
```cpp
#ifndef ADD_H_
#define ADD_H_

#ifdef __cplusplus
extern "C" {
#endif

int add(int a, int b);

#ifdef __cplusplus
}
#endif
```

`add.cpp`
```cpp
#include"add.h"

#include<iostream>

int add(int a, int b)
{
    std::cout<<"a = "<<a<<", b = "<<b<<std::endl;
    return a + b;
}
```

`main.go` 
```go
package main

/*
#cgo CPPFLAGS:  -I.
#cgo LDFLAGS: -L. -ladd -lstdc++
#include"add.h"
*/
import "C" //需要紧接着上面的注释，不能有空行

import (
    "fmt"
)


func main(){
    a, b := 1, 3
    sum := C.add(C.int(a), C.int(b))

    fmt.Printf("sum = %d\n", int(sum))
}
```

**PS:** 对于当前目录下的头文件，上面的代码可以不注明头文件目录。

编译上面的go代码时，需要先编译libadd.a静态库。
```shell
g++ -c add.cpp
ar cr libadd.a add.o
go build main.go
```





## C调用Go函数，实现回调功能

当你要集成第三方`sdk`时，可能会发现只提供`C`语言版本的，比如海康摄像头。为了能在`Go`语言中获取到一些摄像头触发的事件，需要做到`C`调用`Go`的函数。

为了实现这点，需要在`Go`那里到处函数，让`C`调用。具体的例子如下：

`monitor.h`

```c++
#ifndef ADD_H_
#define ADD_H_

#ifdef __cplusplus
extern "C" {
#endif

void loopMonitor(const char *user, const char *password);

#ifdef __cplusplus
}
#endif

#endi
```



`monitor.cpp`

```c++
#include"monitor.h"

#include<stdio.h>

#include "_cgo_export.h" //这个是必须包含的

void GoCb(int a);//声明一个函数，这个函数是定义在Go代码里面的

void loopMonitor(const char *user, const char *password)
{
    printf("user: %s \t password: %s\n", user, password);

    GoCb(3); //调用Go函数
    GoCb(10);
}
```



`main.go`

```go
package main

/*
#include"monitor.h"
*/
import "C"

import "fmt"


func main(){

    user, password := "admin", "123456"
    C.loopMonitor(C.CString(user), C.CString(password))

    fmt.Printf("hello world\n")
}

//下面这行注释不能删除，用于标明导出一个C语言函数。留意参数的类型.
//export GoCb
func GoCb(personNum C.int){
    fmt.Printf("goCb: %d\n", int(personNum))
}
```



直接执行`go build`即可完成编译。或者执行`go run .`，输出结果如下：

```txt
user: admin      password: 123456
goCb: 3
goCb: 10
hello world
```



### 链接Windows动态库

`cgo`在`Windows`是使用`mingw`编译C代码的，为此需要`Linux`格式的静态库。

创建一个`add.def`文件，内容如下：

```def
LIBRARY add.dll

EXPORTS
add
```

其中第一行包含`dll`的名字。`EXPORTS`下面是需要从这个`dll`导出的函数名，一般是需要用到哪些就导出哪些。明显，你需要一个`dll`查看器，查看某个`dll`里面可以导出哪些函数名。[dll_export_viewer](http://www.nirsoft.net/utils/dll_export_viewer.html)是一个不错的工具。



创建完上面的文件后，在`Windows`终端里面执行命令`dlltool -dllname add.dll --def add.def --output-lib libadd.a ` 执行完毕后，就可以在当前目录看到生成的`libadd.a`文件了。

