





## module

使用`export GO111MODULE=on`开启，为了使得之前的项目还能正常编译，建议改成`export GO111MODULE=auto`

假如要创建`example.cn/logger`库，那么先创建`example.cn/logger`目录(注意一路上的路径不要带有空格)。然后在`logger`目录中执行命令`go mod init example.cn/logger`。此时会在当前目录中生成`go.mod`文件。写完代码后再执行`go run main.go`此时会下载第三方库，并且生成`go.sum`代码。



go run main.go的时候需要到github.com下载代码，但由于被墙导致会比较慢，此时可以用`export GOPROXY=https://goproxy.cn`指定代理。在windows中使用`SET GOPROXY="https://goproxy.cn"`

上面这代理是七牛云的，还有一个百度的`export GOPROXY=https://goproxy.baidu.com/   `



`goland`要使用`go mod`需要在创建项目的时候就选定是`go mod`模式。在`New Project`的时候选定`Go Module(vgo)`

如果要手动创建`module`模式的项目，需要在项目根目录下创建一个`go.mod`文件，内容如下:

```go
module swaggo

go 1.13
```

然后执行`go build`即可



## json

### 如果不存在就显示这个成员

```go
PhoneNum string `json:"phoneNum,omitempty"`
```



### 不要显示null

如果结构体成员的类型是切片或者字典，并且在初始化结构体的时候赋值，此时直接`dumps`出来的`json`成员值是`null`。如下：

```go
type Test struct{
	Name string `json:"name"`
	ImgList []string `json:"imgList"`
	ImgDict map[string]string `json:"imgDict"`

	Score int `json:"score"`
}

func main(){
    var t Test
    body, _ := json.Marshal(t)
    fmt.Printf("%s\n", string(body))
}
```

打印的结果是：

```text
{"name":"","imgList":null,"imgDict":null,"score":0}
```

这其实是正常的，因为`ImgList`在没有赋值的情况下，其值就是`nil`，所以`dumps`出来是`null`很正常。



但如果想搞成`[]`和`{}`呢，直接显式设置为空的切片和字典就可以了。如下：

```go
func main(){
    tt := Test{
		Name:    "xiaoming",
		ImgList: []string{},
		ImgDict: map[string]string{},
		Score:   0,
	}
    
    body, _ := json.Marshal(t)
    fmt.Printf("%s\n", string(body))
}	
```

打印的结果是：

```text
{"name":"xiaoming","imgList":[],"imgDict":{},"score":0}
```









## redis

一般是通过下面代码获取redis中的key

```go
data, err := redisClient.Get("key").Result
```

但如何判定它是没有这个key，还是失败了呢？

如果没有这个key和redis本身操作失败，err都不会为nil。对于前者，err的值为redis.Nil。因此一般用下面方式判定

```go
if data, err := redisClient.Get("key").Result; err != nil && err != redis.Nil{
    //redis本身有问题
}else if err == redis.Nil {
    //redis上没有这个key
}else{
    //有这个key，并且获取成功
}
```





## time

如果想表示10小时，那么很简单`10 * time.Hour`。但如果想表示`n`小时呢，有点难度。因为不能直接变量名乘以`time.Hour`。此时需要类型转换： `time.Duration(n) * time.Hour`





## gin

### 0值碰到required

假设有下面结构体

```go
type TestTask struct {
	Status      int `form:"status" json:"status" binding:"required"`

	DeviceId int `form:"DeviceId" json:"DeviceId" binding:"required"`
}
```

使用下面命令发送`post`请求

```shell
curl -v -d '{DeviceId": 21434, "status": 0}' http://host/api/test
```

使用`context.Bind`的时候，会报错：` Key: 'TestTask.Status' Error:Field validation for 'Status' failed on the 'required' tag`。

经过一番的调试并砸键盘，才发现是`gin`本身就不支持这种情况，参考[issue/1246](https://github.com/gin-gonic/gin/issues/1246)。当然锅不是`gin`的，而是`go`的。简单来说，对于`go`来说，0会被认为没有值。所以上面的请求中`status`的值换成1，就没有报错了。

其实，这种情况不止出现在`gin`，`json`也会有的。比如对于某些有`omitempty` tag的`int`或者`string`类型，如果值是0或者空字符串，那么`json` 序列化之后也是会出现没有这个成员变量内容的情况。

上面问题的优雅解决方法就是将`Status`的类型从`int`变成`*int`。





### Bind

`ctx.Bind`函数的实现如下:

```go
func (c *Context) Bind(obj interface{}) error {
	b := binding.Default(c.Request.Method, c.ContentType())
	return c.MustBindWith(obj, b)
}
```

这里有两个点，第一个是`MustBIndWith`，也就是对于`tag`里面的`binding:"required"`。如果请求里面缺少这个参数，会由`gin`框架会返回`HTTP`状态码。如果代码里面还返回了自己状态码，那么会出现经典的错误显示。

```text
[GIN-debug] [WARNING] Headers were already written. Wanted to override status code 400 with 200
```



解决办法也比较简单，不使用`Bind`而是`ShouldBind`，后者的实现如下，**放心，后者也会校验required这tag，如果缺少参数也会返回错误**

```go
func (c *Context) ShouldBind(obj interface{}) error {
	b := binding.Default(c.Request.Method, c.ContentType())
	return c.ShouldBindWith(obj, b)
}
```



第二点就是`b := binding.Default(c.Request.Method, c.ContentType())`。因为`HTTP`可以通过`URL`携带参数，即使是通过`body`传递参数，也有4种提交数据的方式。特别是`POST`，怎么知道是哪种呢？ 

其实是通过`Content-Type`获取的，上面这行代码也就是解析请求头部的`Content-Type`。此时就要求`http`携带正确的头部。这里要注意的是`curl`命令，默认情况下`curl`使用的是`application/x-www-form-urlencoded`，如果后台代码期望的是`JSON`那么会无法解析参数。

解决方案有两种，1. 通过`--header "Content-Type: application/json" `参数指定头部。2. 后台代码不需要`ShouldBind`而是`ShouldBindJSON`。建议使用第一种方法解决。





### session

登录相关的内容可以用[sessions](https://github.com/gin-contrib/sessions)这个库来实现。其支持多种存储方案。比如将内容存储到`cookie`中，或者存储到`redis`中。无论那种方式都会进行加密。

对于`cookie`方式，其是将具体的`cookie`内容和名称一起加密，最后得到一个`http`维度的`cookie`，后端在处理请求的时候库会自动解密`http`带上来的`cookie`，从而得到具体的`cookie`名称和内容。这种方式会随着设置的`cookie`个数增加，`http`头部中的`cookie`字段也会变大。

对于`redis`方式，真正的内容是存储在`redis`中的。但为了从`redis`找回来，需要在`cookie`中设置`redis`的`key`。这种方式并不会随着`cookie`个数的增加而使得`http`头部的`cookie`字段变大。

具体例子参考官网，其已经在使用上屏蔽来不同存储方式的区别。使得其使用同一套接口。





### 获取原始请求

网上说法，gin只能读取原始请求body一次。如果不想让gin帮忙解析json。那么可以用ctx.GetRawData()获取原始请求body。

解决方案是设置回去。如下面的中间件

```go
func HelloMiddleware() gin.HandlerFunc {
	return func(ctx *gin.Context) {
		data,err := ctx.GetRawData()
		if err != nil{
			fmt.Println(err.Error())
		}
		fmt.Printf("data: %v\n",string(data)) 
		
		ctx.Request.Body = ioutil.NopCloser(bytes.NewBuffer(data)) // 关键点
		ctx.Next()
	}
}
```

参考[Gin框架，body参数只能读取一次](https://blog.csdn.net/impressionw/article/details/84194783)



### 获取响应内容

并没有直接的API可以获取响应内容。解决方案是用wrapper的方式。

```go
package main

import (
	"bytes"
	"fmt"
	"github.com/gin-gonic/gin"
)

type responseBodyWriter struct {
	gin.ResponseWriter
	body *bytes.Buffer
}

func (r responseBodyWriter) Write(b []byte) (int, error) {
	r.body.Write(b)
	return r.ResponseWriter.Write(b)
}

func logResponseBody(c *gin.Context) {
	w := &responseBodyWriter{body: &bytes.Buffer{}, ResponseWriter: c.Writer}
	c.Writer = w
	c.Next()
	fmt.Println("Response body: " + w.body.String())
}

func sayHello(c *gin.Context) {
	c.JSON(200, gin.H{
		"hello": "privationel",
	})
}

func main() {
	r := gin.Default()
	r.Use(logResponseBody)
	r.GET("/", sayHello)
	r.Run()
}
```

参考: [how do i get response body in after router middleware](https://github.com/gin-gonic/gin/issues/1363)



有了上面这两个点之后，就可以做到面向切面打日志了。





### Keys

`gin.Context`有一个`Set`方法用来设置`key-value`，这个东西主要是用来为中间件服务的，不属于http请求中的某些内容的对照物。对于`URL`中的参数，是通过`Context.Query`获取的。





### 返回一张图片

```go
r.GET("/img", func(ctx *gin.Context){
	filename := "maintain.jpeg"
	ctx.Writer.Header().Add("Content-Type", "image/jpeg")
	ctx.File(filename)
})
```





### 图片显示

有时候想在浏览器上显示图片，而不是下载模式。这个得添加`Content-Type: image/jpeg`头部。但这还不够。还和`Content-Disposition`头部相关

```txt
Content-Disposition: inline
Content-Disposition: attachment
Content-Disposition: attachment; filename=“aaa.jpg”
```

`inline`是默认值，表示回复中的响应体会以页面的一部分或者整个页面的形式展示。也就是在网页中显示图像。如果想在网页中显示图片，不需要`Content-Disposition`头部。`attachment` 表示响应体应该被下载到本地；大多数浏览器会呈现一个“保存为”的对话框，将filename（可选）的值预填为下载后的文件名，假如它存在的话。



参考: https://blog.csdn.net/gao_zhennan/article/details/90513907



## gorm

### 连接问题

如果是长时间运行的服务，可能会偶发`connection.go:158 bad connection`，这个是由于空闲连接超时导致mysql服务器关闭了连接，而客户端这边继续使用。默认情况下服务器的空闲等待时间是8小时，具体可以登录mysql，用`show variables like '%timeout%';`查看，最后面那个`wait_timeout`就是空闲等待时间。

解决办法也简单，在客户端这边设置一个比mysql服务器小的空闲时间就可以了。

```go
dbSrc := fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?charset=utf8", admin.Username, admin.Password, admin.Ip, admin.Port, admin.Database)

if db, err := gorm.Open("mysql", dbSrc); err != nil {
	logger.Fatal("fail to start mysql[%s]. Reason: %s", dbSrc, err.Error())
} else {
	db.DB().SetMaxIdleConns(10) //最大的空闲连接数据，默认是2个
	db.DB().SetConnMaxLifetime(30 * time.Minute) 
	db.DB().SetMaxOpenConns(100)//避免过多的连接。最大允许同时打开多少个连接。默认是不限制
}
```



对于`sqlite`, 可以用下面方法连接

```go
import (
    "fmt"
	"github.com/jinzhu/gorm"
	_ "github.com/mattn/go-sqlite3" //sqlite驱动
)

type Student struct {
	Name string `json:"name" gorm:"name"`
	Id   int64  `json:"id" gorm:"id"`
}

func main(){
	//直接写本地的路径即可
	if db, err := gorm.Open("sqlite3", "test.db"); err != nil {
		fmt.Printf("fail to open sqlite3. Reason: %s", err.Error())
		return
	}else{
		fmt.Printf("success oepn sqlite3\n")

        //自动创建表Student
		if err := db.AutoMigrate(&Student{}).Error; err != nil {
			fmt.Printf("fail to create table. Reason: %s\n", err.Error())
		}else{
			fmt.Printf("success create table \n")
		}

		if err := db.Create(&Student{Name: "zhangsan", Id: 23}).Error; err != nil {
			fmt.Printf("fail to create data: %s\n", err.Error())
		}else{
			fmt.Printf("success insert record\n")
		}

		var stu Student
		if err := db.First(&stu).Error; err != nil {
			fmt.Printf("fail to First. Reason: %s\n", err.Error())
		}else {
			fmt.Printf("success find, %+v\n", stu)
		}
	}
}

```





### 没有记录

如果`select`的时候，记录没有找到。那么也是返回非nil。此时需要进一步判定是否记录为空。代码如下：

```go
var userInfo db.UserLoginInfo
if err := global.DB.Where("openid = ?", openid).First(&userInfo).Error; err != nil{
	if err == gorm.ErrRecordNotFound{ //没有找到这个openid，需要注册
		rsp.RetCode, rsp.RetMsg = errorcode.E_NOT_FOUND, "请先注册"
		return
    }else{
        //数据库错误
    }
}
```



### 自动更新

一些建表语句自带了时间的更新，比如下面

```sql
create table consultTask(
	taskId bigint auto_increment primary key comment '咨询任务id',
	userId bigint not null default 0 comment 'userLoginInfo中的主键',
	status tinyint not null default 0 comment '咨询状态, 0:未回答, 1:已回答',
	
	createTime timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
	updateTime timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
	
	index(userId, createTime)
)charset=utf8 comment '咨询专家表';
```

此时，`golang`结构体里面的`tag`要能做到插入和更新的时候不要动这个两个字段，但又能在查询的时候查到。所以不能直接写成`gorm:"-"`。其实，写成`gorm:"column:createTime;default:null"`就可以了。不要被`default:null`迷惑了，就是这样的。

其实，除了时间戳外，其他不需要设置值的也是可以。如果想要设置的时候，直接将该变量的值设置成非空就可以了。如下：

```go
type MinorQuestion struct {
	ImgPaths   string   `json:"-" gorm:"column:imgPaths;default:null"`
	CreateTime string `json:"createTime" gorm:"column:createTime;default:null"`
}
```

当`MinorQuestion`变量中的`imgPaths`的值为空字符串，那么生成的插入`SQL`语句中，就不会有`imgPaths`的设置。如果变量中的`imgPaths`的值不为空，那么生成的插入`SQL`语句中会有`imgPaths`的设置。



**PS:** 对于自增建也是采用这个方法处理





### 标识字段

如果golang结构体成员的名字和表成员的字母构造一样，并且是蛇形命名方式，那么可以直接是下面形式。表的字段名是`device_id`

```go
type Data struct {
	DeviceId string `gorm:"device_id"`
}
```



如果表字段不是蛇形的，那么必须写成下面形式。表的字段名是`deviceId`

```go
type Data struct {
    DeviceId string `gorm:"column:deviceId"`
}
```



如果golang结构体的名字和表名不同，那么即使是小写蛇形都必须用`column`，如下：

```go
type Data struct {
	Temperature float64 `gorm:"column:kqwd"`
}
```







### 打印sql语句

有时候，通过函数链式写出来的`sql`有错，此时可以在链中加入`Debug()`函数的调用，这样gorm会打印出该sql语句的具体内容。如下：

```go
GormClient.Debug().Table("Student").Where("id = ?", 23).Find(&st)
```



### where结构体

`gorm`是支持在`Where`函数中直接以结构体变量作为参数的。此时即使链式里面已经通过`Table`函数指定了表名，最终`gorm`转换得到的`sql`语句中的`where`语句还是会以蛇形命名方式得到带有表名的字段。比如：

```go
type QueryAbnormalTask struct {
	AppId    int   `form:"AppId" binding:"required" json:"AppId" gorm:"column:AppId"`
	Location string `form:"location" json:"location" gorm:"location"`
	Gate string `form:"gate" json:"gate" gorm:"gate"`
	StartDate string `form:"startDate" gorm:"-"`
	EndDate string `form:"endDate" gorm:"-"`
}

var task QueryAbnormalTask
GormClient.Debug().Table("DeviceInfo").Where(task).Find(&info)
```

最后打印出来的`sql`语句是：

```sql
SELECT * FROM `DeviceInfo`  WHERE (`query_abnormal_tasks`.`AppId` = 1213234) AND (`query_abnormal_tasks`.`start_date` = '2020-03-16') AND (`query_abnormal_tasks`.`end_date` = '2020-03-16')
```

只需为结构体`QueryAbnormalTask`定义一个`TableName`函数即可

```go
func (me QueryAbnormalTask) TableName()string{
	return "DeviceInfo"
}
```



对于前面的简单sql，如果不想直接Table()函数(其实也推荐这样做)，那么可以为这个结构体定义一个TableName函数。



### where in

`gorm`是支持在`Where`函数里面使用`in`语法的，直接为:

```go
var deviceIds []int
GormClient.Debug().Table("TempAbnormalRecord").Where("DeviceId in (?)", deviceIds)
```



### replace into

```golang
var task *StoreUserInfoTask //来自函数参数
var out StoreUserInfoTask

err := tools.GormClient.Table("userInfo").Where("openid = ? ", task.Openid).Assign(*task).FirstOrCreate(&out).Error
```

mysql对于replace into是不要求where中的条件必须为主键，也可以是唯一索引。对应的sql语句是replace into。

对于gorm来说，例子中的写法很奇怪，不那么原生。如果增加Debug()后，可以发现其实是用了两条sql语句，第一条是`select * from userInfo where openid = 'dfdfdsf' limit 1`。让gorm内部应该会`Assign`中的结构体进行对比，如果取值一致，那么就说明需要update而不是insert。换言之，Debug()看到的第二条sql语句，如果表里面没有这条数据，那么就是`insert into userInfo(openid, name) values('xxx', 'yy')  `，如果有的话就是`update userInfo set openid = 'xxx' and name = 'zz' where openid = 'yy' `。

**PS:**  此时的表名不是必需的。`Where`这个函数调用的必需。





### DELETE

一般来说直接用下面代码就可以删除匹配到的记录

```go
record := CallBackUrlCache{Topic: topic}
tools.GormClient.Table(tableName).Delete(&record)
```

但实际效果可能是删除了所有记录。从官网文档来说，`gorm`是根据主键删除的。但`gorm`不会去数据库查询`topic`是否设置为主键了，而是通过结构体的`tag`中的`PRIMARY_KEY`。为此，上述结构体需要定义为

```go
type CallBackUrlCache struct {
	Topic string `json:"topic" gorm:"topic;PRIMARY_KEY"`
	CbItems string `json:"cb_items" gorm:"column:cb_items"`
}
```



### UPDATE

假定表结构如下：

```sql
Create Table: CREATE TABLE `device` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `serial` varchar(64) NOT NULL DEFAULT '' COMMENT '序列号',
  `device_type` varchar(64) NOT NULL DEFAULT '' COMMENT '序列号类型',
  `info` varchar(64) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  UNIQUE KEY `ls` (`serial`,`device_type`)
) ;
```

`golang`结构体如下：

```go
type Device struct {
	Serial string `gorm:"column:serial"`
	Info string `gorm:"column:info"`
}
```

此时可以如下更新

```go
info := Device{
	Serial: "E23",
	Info:   "helloTest",
}

tools.GormClient.Debug().Table(info.TableName()).Where("serial = ?", info.Serial).Update(info)
```

打印的`sql`为

```sql
UPDATE `sensor` SET `info` = 'helloTest', `serial` = 'E23'  WHERE (serial = 'E23')
```

**PS: ** 此时，必须通过`Table`函数指定表名。



如果是更新某一列，直接写成下面形式就可以了

```go
db.Table("sensor").Where("serial = ?", "E23").Update("info", "helloTest")
```





### 事务



可以通过`Transaction`函数创建一个事务，该函数的参数就是一个函数，内部函数参数为`*gorm.DB`。内部函数里面的`sql`统一使用`*gorm.DB`，如下：

```go
func (me *TaskHandler) transactionWriteDb()error{
	return tools.GormClient.Transaction(func(tx *gorm.DB)error{
		//注意这里应该改用tx，而不是tools.GormClient
		if err := tx.Table("Record").Create(me.baseTask).Error; err != nil{
			return err //返回err会自动回滚
		}

		if err := tx.Table("TempAbnormalRecord").Create(me.abnormalTask).Error; err != nil{
			return err
		}

		return nil
	})
}
```

业务代码中无需做任何`Rollback`或者`Commit`操作。`gorm`在`Transaction`函数内部，根据匿名函数的返回值做了回滚和提交操作。`gorm`的实现代码如下：

```go
// Transaction start a transaction as a block,
// return error will rollback, otherwise to commit.
func (s *DB) Transaction(fc func(tx *DB) error) (err error) {
	panicked := true
	tx := s.Begin()
	defer func() {
		// Make sure to rollback when panic, Block error or Commit error
		if panicked || err != nil {
			tx.Rollback()
		}
	}()

	err = fc(tx)

	if err == nil {
		err = tx.Commit().Error
	}

	panicked = false
	return
}
```







## viper

### 无法识别

如果一个`toml`里面既有`map`，也有常规的字符串，那么需要将常规字符串放到配置文件的开头才能被正确读取。比如下面中的student和name是可以正确识别的，但如果将`student`和`name`放到`mysql`中间就不能正确读取。

```toml
student = ["xiaoming", "zhangsan", "wanwu"]
name = "hello world"

[mysql]
  ip = "127.0.0.01"
  port = 3306
  username = "root"
  password = "password"
  database = "student"


[redis]
  ip = "127.0.0.1"
  port = 6379
  password = "password"
  database = 0
```



### 读取map数组

如果想在`toml`里面配置一个`map`数组，可以使用下面格式。

```toml
[yunmou]
  [[yunmou.deviceList]]
  deviceLocation = "总部"
  deviceId = "sdfsdfdsfdsf"

  [[yunmou.deviceList]]
  deviceLocation = "分部"
  deviceId = "aa"
```

对应的`json`格式应该是

```json
{
	"yunmou": {
		"deviceList": [{
			"deviceLocation": "总部",
			"deviceId": "sdfsdfdsfdsf"
		},
		{
			"deviceLocation": "分部",
			"deviceId": "aa"
		}]
	}
}
```



相应的`viper`读取代码为

```go
	deviceList, ok := viper.Get("deviceList").([]interface{})
	if !ok {
		logger.Error("fail to get deviceList from cfg")
		return
	}

	for _, table := range deviceList {
		mm := table.(map[string]interface{})
		for k, v := range mm{
			fmt.Printf("%s : %s\n", k, v.(string))
		}
	}
```

输出结果为

```txt
deviceLocation : 总部
deviceId : sdfsdfdsfdsf
deviceLocation : 分部
deviceId : aa
```



### 利用Unmarshal读取map数组

假定配置如下：

```toml
[mysql]
  ip = "127.0.0.1"
  port = 3306
  username = "root"
  password = "password"
  database = "student"

[redis]
  ip = "127.0.0.1"
  port = 6379
  password = "password"
  database = 0

#注意下面二级结构的名称不是db.device，而是device
[db]
  [[device]]
  ip = "192.168.1.120"
  port = 3306
  username = "root"
  password = "password"
  database = "student"

  [[device]]
  ip = "192.168.1.121"
  port = 3306
  username = "root"
  password = "password"
  database = "score"
```



定义的`go`结构体如下：

```go
type Mysql struct {
	Username     string `mapstructure:"username" json:"username" toml:"username"`
	Password     string `mapstructure:"password" json:"password" toml:"password"`
	Ip 			 string `mapstructure:"ip" json:"ip" toml:"ip"`
	Port 		 int    `mapstructure:"port" json:"port" toml:"port"`
	Database 	 string `mapstructure:"database" json:"database" toml:"database"`
}


type Redis struct {
	Ip		 	 string `mapstructure:"ip" json:"ip" toml:"ip"`
	Port 		 int    `mapstructure:"port" json:"port" toml:"port"`
	Password     string `mapstructure:"password" json:"password" toml:"password"`
	Database 	 int `mapstructure:"database" json:"database" toml:"database"`
}

type Config struct {
	Mysql   Mysql   	`mapstructure:"mysql" json:"mysql" toml:"mysql"`
	Redis   Redis   	`mapstructure:"redis" json:"redis" toml:"redis"`
	Dblist  []Mysql     `mapstructure:"device" json:"mysql" toml:"device"`
}
```

PS:  `mysql`里面的成员必须是定义在里面的。不然定义另外一个结构体`SSQL`，然后在`Mysql`结构体定义里面直接`SSQL`，这样是无法解析`SQL`里面的成员的。



读取代码如下：

```go
func initVp(){
	var CONFIG Config

	v := viper.New()
	v.SetConfigFile("config.toml")
	err := v.ReadInConfig()
	if err != nil {
		panic(fmt.Errorf("Fatal error config file: %s \n", err))
	}


	if err := v.Unmarshal(&CONFIG); err != nil {
		fmt.Printf("fail to unmarsha1. Reason: %s", err.Error())
	}else{
		fmt.Printf("%+v\n", CONFIG)
	}
}
```







## os.exec

### 命令带参数

如果要执行一条带有许多参数的命令，比如：`ffmpeg.exe -rtsp_transport tcp -i "rtsp://admin:123456@192.168.1.120:554/Streaming/Channels/101?transportmode=unicast" -c copy -f flv rtmp://127.0.0.1:1935/live/123`

不能执行如下：

```go
proc := exec.Command("ffmpeg.exe", " -i rtsp://admin:123456@192.168.1.120:554/Streaming/Channels/101?transportmode=unicast -c copy -f flv rtmp://127.0.0.1:1935/live/123")

if err := proc.Run(); err != nil {
		logger.Error("fail to run ffmpeg. Reason: %s", err.Error())
}
```

必须如下：

```go
args := []string{ "-i", rtspUrl, "-c", "copy", "-f", "flv", rtmpUrl}

proc := exec.Command("ffmpeg.exe", args...)

if err := proc.Run(); err != nil {
		logger.Error("fail to run ffmpeg. Reason: %s", err.Error())
}
```



### 获取输出

如果想单独标准错误输出，可以使用下面代码

```go
package main

import (
	"bytes"
	"fmt"
	"os/exec"
)

func main(){
	proc := exec.Command("ffprobe.exe", "-i", "aa.mp4")
	var buf bytes.Buffer 
    proc.Stdout = &buf
    
    _ = proc.Run() //运行完毕才能获取输出
    fmt.Printf("output: %s\n", buf.String())
}
```



如果想标准输出和错误输出合并，可以使用下面代码

```go
package main

import (
	"fmt"
	"os/exec"
)

func main(){
	proc := exec.Command("ffprobe.exe", "-i", "aa.mp4")
	out, _ := proc.CombinedOutput() //这里会执行命令
	fmt.Printf("output: %s\n", string(out))
}
```



### 设置超时

通过`context`实现超时功能

```go
package main

import (
	"context"
	"fmt"
	"os/exec"
	"time"
)


func main(){
	ctx, cancel := context.WithTimeout(context.Background(), 10 * time.Second)
	defer cancel()

	proc := exec.CommandContext(ctx, "ffprobe.exe", "-i", "aa.mp4")

	out, _ := proc.CombinedOutput()
	fmt.Printf("output: %s\n", string(out))
```





## path

有时候需要获取路径的basename，最直接的方法是`path.Base(os.Args[0])`这个在`Linux`上是没有问题的。但在`Windows`上却不行，主要是两个系统的路径分隔符是不同的。解决方法也简单，换另外一个库，如下：

```go
package main

import (
    "fmt"
    "os"
    "path/filepath"
)


func main(){

    fmt.Printf("origin args[0] = %s\n", os.Args[0])
    fmt.Printf("base args[0] = %s\n", filepath.Base(os.Args[0]))
}
```





## MongoDB

### 时间

假设有一个`time.Time`类型的字段`CreateTime`，那么

```go
# 插入时
QuestionDetail.CreateTime = time.Now()

#查询之后
QuestionDetail.CreateTime = QuestionDetail.CreateTime.Local()
```



### 分页查询

```go
filter := bson.D{ {"questionerUserId", req.UserId} }

#SetSkip是跳过前面的若干记录
option := options.Find().SetLimit(pageSize).SetSkip((req.Page-1) * pageSize).SetSort(bson.D{{"createTime", 1}})
collection.Find(context.TODO(), filter, option)
```





## 反射



### 设置和获取结构体成员值

假定有下面的结构体

```go
type Data struct {
	Project string `json:"project" gorm:"column:project"` 
	DeviceId string `json:"device_id" gorm:"column:device_id"`
	Time     string `json:"time" gorm:"time"`
}

func parseFiledAndSet(in interface{},  projectName string)(time string, err error){

	vf := reflect.ValueOf(in).Elem()
	time = vf.FieldByName("Time").String() //获取
	vf.FieldByName("Project").Set(reflect.ValueOf(projectName)) //设置

	return
}


func main(){

	data := Data{
		Project:        "ddd",
		DeviceId:       "fjlsdfldsf",
		Time:           "2020-10-22 10:21:35",
	}

    //这里得传一个指针过去，不然是无法赋值的
	if times, err := parseFiledAndSet(&data, "test"); err != nil {
		fmt.Printf("fail to parse. \n")
	}else{
		fmt.Printf("time = %s, newProject = %s\n", times, data.Project)
	}
}

```



### []interface{}

虽然`interface{}`可以接收任何类型，但不代码`[]interface{}`可以接收任何切片。如果确实需要将一个其他类型的切换转换成`[]interface{}`，那么手动转换。代码如下：

```go
var datas []model.Data
//对datas一顿赋值

var rows []interface{}
for i, data := range datas {
    rows = append(rows, data)
}
```



### 配合gorm

有时候需要不同结构体的`sql`都差不多，就表结构不同。此时可以这样多态获取表的数据

```go
//这里定义接口。设备类型有气象、土壤和摄像头。他们的数据都处于不同的表
type DeviceIf interface {
	getValueSlice()interface{} //需要返回一个切片的地址
}


type Reader struct {
	device DeviceIf
}

func (me *Reader) readDataFromDb(deviceId string)(rows []interface{}, err error){
    
    records := me.device.getValueSlice() //需要返回一个切片的地址，而不是一个切片
    //gorm会识别的
	if err = db.Where("device_id = ?", deviceId).Find(records).Error; err != nil{
			logger.Error("fail to query. Reason: %s", err.Error())
			return 
	}
    
    
 	//这里是为了其他操作，可以忽略
	vf := reflect.ValueOf(records).Elem()
	Len := vf.Len()
	for i := 0; i < Len; i++{
		rows = append(rows, vf.Index(i).Interface())
	}      
}


```

