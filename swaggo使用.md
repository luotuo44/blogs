

## 安装swaggo

使用命令`go get -u github.com/swaggo/swag/cmd/swag` 安装`swaggo`，其会生成一个`swag`可执行程序。

或者直接从[github](https://github.com/swaggo/swag/releases)下载已经编译好的。将`swag`放到`PATH`能找到的目录。



## 使用



### 牛刀小试

创建一个`main.go`文件

```go
package main

import (
	"fmt"
	"github.com/gin-gonic/gin"
	ginSwagger "github.com/swaggo/gin-swagger"
	"github.com/swaggo/gin-swagger/swaggerFiles"
	"net/http"
)

// @title Swagger Example API
// @version 1.0
// @description This is a sample server celler server.
// @termsOfService https://razeen.me

// @contact.name Razeen
// @contact.url https://razeen.me
// @contact.email me@razeen.me

// @license.name Apache 2.0
// @license.url http://www.apache.org/licenses/LICENSE-2.0.html

// @host 127.0.0.1:8080
// @BasePath /api/v1
func main() {

	r := gin.Default()
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))

	v1 := r.Group("/api/v1")
	{
		v1.GET("/hello", HandleHello)
		v1.POST("/upload", HandleUpload)
		v1.GET("/list", HandleList)
	}

	r.Run(":8080")
}


func HandleUpload(ctx *gin.Context){
	fmt.Printf("handleUpload\n")
	ctx.JSON(200, gin.H{"hello": "upload"})
}

func HandleList(ctx *gin.Context){
	fmt.Printf("handleList\n")
	ctx.JSON(200, gin.H{"list": "list a list"})
}


// @Summary 测试SayHello
// @Description 向你说Hello
// @Tags 测试
// @Accept mpfd
// @Produce json
// @Param who query string true "人名"
// @Success 200 {string} string "{"msg": "hello Razeen"}"
// @Failure 400 {string} string "{"msg": "who are you"}"
// @Router /hello [get]
func HandleHello(c *gin.Context) {
	who := c.Query("who")

	if who == "" {
		c.JSON(http.StatusBadRequest, gin.H{"msg": "who are u?"})
		return
	}

	c.JSON(http.StatusOK, gin.H{"msg": "hello " + who})
}
```



注意上面的两部分注释，一部分是通用的(`main`函数上面)，另外一部分是函数的（注意到只有`HandleHello`函数前面写了注释）。



然后在项目根目录执行命令`swag init`。此时会在项目根目录生成一个`docs`目录，里面包含了一坨文件。接着在`main.go`目录中，`import`这个目录

```go
import _ "swaggo/docs"
```



接着直接`go build`即可生产可执行程序。运行之，然后打开`http://127.0.0.1:8080/swagger/index.html`就可以看到接口说明了。注意因为前面只有一个函数写了注释，所以页面上只有`HandleHello`这个函数对应的`/hello`接口。



如果对其他的函数加了注释，那么再次运行`swag init`，再构建又可以看到更多的注释。



**PS：** 如果通用的`swagger`配置不是在`main.go`里面，那么需要通过命令行指明其在哪个文件里面，比如：`swag init -g router/swag.go`。对于非通用的API配置，不需要做什么特殊的目录制定。因为其会递归搜索当前目录及其子目录。





### 条件编译

对于正式环境，不应该对外提供`swagger`访问。此外，有了`swagger`也会使得最后生成的可执行程序变大。为此，可以通过条件编译方式做到在开发测试环境才提供接口文档。



有下面三个文件

`swag.go`

```go
// +build doc

package router

import (
	"MpSvr/docs"
	"MpSvr/global"
	"github.com/gin-gonic/gin"
	ginSwagger "github.com/swaggo/gin-swagger"
	"github.com/swaggo/gin-swagger/swaggerFiles"
)


// @title 小程序服务器端 API
// @version 1.0
// @description 本文档用来说明小程序端和服务器端的接口API
// @license.name Apache 2.0
func init() {
	swagHandler = ginSwagger.WrapHandler(swaggerFiles.Handler)
}

func ConfigSwagger(r *gin.RouterGroup){
    docs.SwaggerInfo.Host = "www.example.com"
	docs.SwaggerInfo.BasePath = r.BasePath()
}
```



`notswag.go`

```go
// +build !doc

package router

import "github.com/gin-gonic/gin"

func ConfigSwagger(r *gin.RouterGroup){
	//不做任何事情，只是为了在没有doc时，对外提供一个ConfigSwagger函数
}
```



`swagWrapper.go`

```go
package router

import (
	"github.com/gin-gonic/gin"
)

var swagHandler gin.HandlerFunc

func InitSwagger(r *gin.RouterGroup){

	ConfigSwagger(r)

	if swagHandler != nil {
		r.GET("/swagger/*any", swagHandler)
	}
}

```



`main.go`

```go

func main(){
    r := gin.Default()
    apiGroup := r.Group("api/v1")
	router.InitSwagger(apiGroup)  	//配置swagger
}
```



对应开发和测试环境，使用命令`go build -tags doc`编译，对于正式环境，使用`go build`编译。





## 例子



## 普通get请求

`accept`为`json`，`param`的第二个参数使用`query`, ` router`的最后一个参数为get。对应的HTTP请求是GET请求，参数通过`url`查询参数携带.

```go
// @summary 获取待处理问题
// @description 根据专家的擅长领域，拉取相应的待处理问题
// @tags consult
// @accept json
// @produce json
// @Param page query int true "第几页,从1开始"
// @success 200 {object} consult.GetPendingQuestionRsp "成功时,retcode等于0，失败时retcode不等于0"
// @router /consult/queryPendingQuestion [get]
```





### 普通post请求。

`POST`类型为`application/json`。`accept`为`json`，`param`的第二个参数为`body`，`router`最后参数为`post`

```go
// @accept json
// @param payload body user.SmsCodeReq true "payload描述"
// @router /user/requestSmsCode [post]
```

需要自行在包`user`定义`SmsCodeReq`



### 普通文本响应

```go
// @Produce plain
// @Success 200 {string} string "hello world"
```





### json响应

```go
// @Produce json
// @Failure 400 {string} string "{"msg": "who are you"}"
```





### 结构体

对于`POST`的`JSON`请求，一般是通过一个结构体接收其数据。此时可以通过`@param`指定接收的结构体。对于返回的`body`是一个结构体，可以用`{object}`指定。

```go
import (
	reqUser "MpSvr/model/request/user"
	_ "MpSvr/model/response/user"
)


// @summary 短信验证码登录
// @description 通过短信验证码登录或者注册
// @tags user
// @accept json
// @produce json
// @param payload body user.PhoneSmsCodeLoginReq true "Payload 描述"
// @success 200 {object} user.PhoneSmsCodeLoginRsp "成功时retcode等于0，失败时retcode不等于0"
// @router /user/loginBySmsCode [post]
func LoginBySmsCode(ctx *gin.Context) {
	//....
}
```

可以看到，虽然请求的`user`被重命名为`reqUser`，但`swagger`里面还是用原来的名字，和响应包一样。并不产生冲突。



对于结构体，可以为每个字段添加注释。



```go
type Test struct {
    Citys []string `example:"广州,清远"`  //逗号两边不要加入空格
    Score int `example:"89"` //和下面的字符串是一样的
    Provice string `example:"广东"`    
}
```





### 上传文件

`accept`参数为`mpfd`，`param`的第二个参数为`formData`, 文件所在的参数类型为`file`

```go
// @summary 上传图片
// @description 上传图片的单独接口，每次只能上传一张图片。主要是为了适配小程序
// @tags image
// @accept mpfd
// @produce json
// @param ImgName formData string true "图片名字"
// @param imgIndex formData int true "上传的图片组中的第几张"
// @param imgFile formData file true "文件"
// @success 200 {object} img.UploadImgRsp "成功时,retcode等于0，失败时retcode不等于0"
// @router /img/upload [post]
```





### 下载文件

`produce`参数为`jpeg`， `success`参数为`200 {file} binary`



```go
// @summary 下载图片
// @description 通过imgKey下载图片
// @tags image
// @accept json
// @produce jpeg
// @param imgKey query string true "imgKey"
// @success 200 {file} binary  "响应图片内容"
// @failure 404 {object} errorcode.CodeMsg "没有找到该图片"
// @router /img/download [get]
```



## 特殊需求

### 忽略结构体中的某个字段

类似`json:"-"`，直接用`swaggerignore:"true"`即可。





## 参考

1. [使用swaggo自动生成Restful API文档](https://razeencheng.com/post/go-swagger)