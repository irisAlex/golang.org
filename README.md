Aresgo
---------------
aresgo是一个简单快速开发go应用的高性能框架，你可以用她来开发一些Api、Web及其他的一些服务应用，她是一个RESTful的框架。她包含快速的Http实现、Url路由与转发、Redis的实现、Mysql的CURD实现、JSON和INI配置文件的读写，以及其他一些方法的使用。后续会继续将一些常用应用添加到框架中。


产品特点（Features）
-----------------

* 实现思路借鉴iris-go,beego等框架
* http实现封装了fasthttp，fasthttp的方法和实现可以直接使用，如果使用fasthttp请引入包：github.com/aresgo/router/fasthttp
* mysql的实现封装了go-sql-driver，**支持多数据库实例和主从配置，可以实现读写分离**，并作了CURD的扩展，可以通过对象(struct)的方式进行修改
* redis的实现封装garyburd/redigo，**支持多个实例和主从配置，可以实现读写分离**
* 配置文件管理（Json和ini）采用beego的框架方法并做了一些修改，已集成到aresgo框架中可以很方便的使用

安装（Installation）
--------------------
使用“go get”命令：

>$ go get github.com/misgo/aresgo

用法（Usage）
-------------------
使用aresgo框架，你只需要在源文件都加上：

> import "github.com/aresgo"

或者如果使用框架中的某个包的方法，可以这样使用：

> import "github.com/aresgo/text"

> **更多实例参见：[aresgo-demo](https://github.com/misgo/aresgo-demo)**

http实现
---------------
```go

import "github.com/aresgo"

func main(){

  //初始化路由
  router := aresgo.Routing()

  //定义404错误页
  router.Get("/404.html", NotFound)
  router.NotFound = NotFound  //只要访问不存在的方法或地址就会跳转这个页面

  //输出方法
  router.Get("/hello/:name", Hello)   //Get方法请求，Post请求时会报错
  router.Post("/hello/:name", Hello)  //Post方法请求，Get请求时会报错

  //注册对象，注册后对象(struct)的所有公共方法可以被调用
  router.Register("/passport/", &action.UserAction{}, aresgo.ActionGet) 
  
  //POST or GET or ...请求被拒绝时执行的方法，取决于路由方法的设置
  router.MethodNotAllowed = DisAllowedMethod  
  
  //监听IP与端口，阻塞式服务
  router.Listen(“127.0.0.1:8010”)

}
//404错误页
func NotFound(ctx *aresgo.Context) {
	fmt.Fprint(ctx, "页面不存在!\n")
}
// 欢迎页
func Hello(ctx *aresgo.Context) {
	fmt.Fprintf(ctx, "hello, 欢迎【%s】光临!\n", ctx.UserValue("name"))
}

```

* 使用Register方法，注册的struct的公共方法可以被调用，方法名称需要首字母大写其他小写
* 路由参数支持“：”和“*”
* aresgo.Context继承fasthttp.RequestCtx，维护一个请求的上下文
* cookie和session的实现目前使用fasthttp中的实现并未集成到框架，后续会持续封装。如使用请导入包：github.com/aresgo/router/fasthttp

> 更多http实例参见：[aresgo-demo/Server.go](https://github.com/misgo/aresgo-demo/blob/master/Server.go)

mysql实现
----------------
导入aresgo框架
>import "github.com/aresgo"

需要指定数据库配置的路径：
>aresgo.DbConfigPath = [你的数据库配置文件路径] 

* 数据库配置路径是绝对路径
* 数据库支持多数据库实例及主从配置
* 配置文件格式是json格式：
```go
{
    "dev": { 
        "master": {
            "ip": "127.0.0.1",  
            "port": "3306",
            "user": "root",
            "password": "123456",
            "charset": "utf8",
            "db": "gomaster",
        },
        "slave": {
            "ip": "127.0.0.1",
            "port": "3306",
            "user": "root",
            "password": "123456",
            "charset": "utf8",
            "db": "goslave",
        }
    },
    "online": {
           ...
        }
    }
}
```
* 执行一段SQL：
```go
> res, err := aresgo.D("dev").Execute(aresgo.DbInsert, "insert t_user set Username='test1' ")
```
* 查询一行数据
```go
> res,err := aresgo.D("dev").GetRow("SELECT Uid,Username,Email,Gender FROM t_user WHERE Uid<10")
```
* 查询列表
 ```go
res, err := aresgo.D("dev").Query("SELECT Uid,Username,Email,Gender FROM t_user WHERE Uid<10")
```
* 删除
```go
res, err:= aresgo.D("dev").Table("t_user").Where("Uid =?", 9).Delete()
```
* 更新
```go
fields := make(map[string]interface{})
fields["Username"] = "administrator"
fields["Password"] = "21232f297a57a5a743894a0e4a801fc3"
fields["Createtime"] = 1486463479
fields["Gender"] = 2
res, err:= aresgo.D("dev").Table("t_user").Where("Uid = ? ", 1).Update(fields)
```
* 还可以进行对象操作，比如查询是否能在数据库找到此对象的映射
先定义一个对象（struct）
```go
  //用户对象
  UserInfo struct {
     Uid        int64  `table:"t_user" key:"pk" auto:"1" `
     UserName   string `field:"Username"`
     Email      string
     Mobile     string
     Pwd        string `field:"Password"`
     NickName   string `field:"Nickname"`
     Gender     int8
     Birth      time.Time `type:"date"`
      CreateTime time.Time `field:"Createtime" type:"int"`
      Group      GroupInfo `key:"notfield"`
   }
   //用户组对象
   GroupInfo struct {
       Gid  int32
       Name string
    }
    
    func main(){
      var user UserInfo
      err := aresgo.D("dev").Where("Uid = ?", 3).Find(&user)
    }
```
> tag标签：table->表名，filed->该字段在数据库中的字段名称，key->主键用pk，auto->是否是自增

> 更多数据库实例可以参见：[aresgo-demo/Database.go](https://github.com/misgo/aresgo-demo/blob/master/Database.go)

redis实现
--------------
导入aresgo框架
>import "github.com/aresgo"

需要指定redis配置的路径：
>aresgo.RedisConfigPath = [你的redis配置文件路径] 

配置文件的Json格式
```go
{
    "dev": {
        "master": {
            "ip": "10.168.31.33",
            "port": "6379",
            "password":"123456",
            "maxidle":10,
            "idletimeout":10,
            "maxactive":1024,
             "db":5
        },
        "slave": {
            "ip": "10.168.31.33",
            "port": "6379",
            "password":"",
            "maxidle":10,
            "idletimeout":10,
            "maxactive":1024,
            "db":5
        }
    },
    "other": {
       ...
      }
    }
}
```
* 选择库
```go
aresgo.R("dev").Select(5)
```
* 获取一个字符串型数据
```go
g1 :=aresgo.R("dev").GetString("c")
```
* Get多条
```go 
g := aresgo.R("dev").GetStrList("a", "b", "c", "d", "e")
```
* Set多条
```go
  var setVals map[string]interface{} = make(map[string]interface{}, 0)
  setVals["v1"] = 1234567
  setVals["v2"] = "abcdefg"
  setVals["v3"] = false //布尔型保存后：true:1;false:0
  s := aresgo.R("dev").SetValues(setVals)
  ```
* hash Get
```go
hg1 :=aresgo.R("dev").GetString("h1", "v1") 
```
*设置键的失效时间
```go
t1 :=aresgo.R("dev").SetTimeout("a", 60)
```
>更多redis实现的例子参见[aresgo-demo/Redis.go](https://github.com/misgo/aresgo-demo/blob/master/Redis.go)

配置文件操作
-------------
导入配置文件包
```go
import(
    "github.com/aresgo"
    "github.com/aresgo/config"
)
```

实例化配置文件
```go
jsonConfiger, err1:= aresgo.LoadConfig("json", [json配置文件路径])
iniConfiger, err 2:= aresgo.LoadConfig("ini", [ini配置文件路径])
```

**Json文件内容**
```go
{
    "dev": {
        "master": {
            "ip": "10.168.31.33",
            "port": "6379",
            "password":"123456",
            "maxidle":10,
            "idletimeout":10,
            "maxactive":1024,
             "db":5
        },
        "slave": {
           ...
        }
    },
    "other": {
       ...
      }
    }
}
```
**ini文件内容**
```go
[dev.master]
ip=127.0.0.1
port=6379
db=gomaster
```

获取数据库实例dev中主库配置master的ip地址:
> json
```go
ip := jsonConfiger.String("dev.master.ip")
```
>ini
```go
ip := iniConfiger.String("dev.master->ip")
```
获取数据库实例dev中主库配置master的ip2地址，如果不存在返回一个默认值:
> json
```go
ip := jsonConfiger.DefaultString("dev.master.ip2", "192.168.0.1")
```
>ini
```go
ip := iniConfiger.DefaultString("dev.master->ip2", "not choose")
```
获取数据库实例dev中主库配置的所有项
> json
```go
obj, err := jsonConfiger.GetVal("dev.master") 
```
> ini
```go
obj, err:= iniConfiger.GetSection("dev.master") 

> 更多redis实现的例子参见[aresgo-demo/Config.go](https://github.com/misgo/aresgo-demo/blob/master/Config.go)

