go语言的http包
von:https://my.oschina.net/u/943306/blog/151293
#http服务 ##引子，http的hello world

如果要搜索“go http helloworld”的话，多半会搜索到以下代码
```golang
package main

import (
    "io"
    "net/http"
)

func main() {
    http.HandleFunc("/", sayhello)
    http.ListenAndServe(":8080", nil)
}

func sayhello(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "hello world")
}
```
这时，如果用浏览器访问localhost:8080的话，可以看到“hello world”
main里的内容解释：
- 首先注册一个sayhello函数给“/”，当浏览器浏览“/”的时候，会调用sayhello函数
- 其次开始监听和服务，“:8080”表示本机所有的ip地址的8080口

##最简单的http服务 现在开始，我由简入深的一步一步介绍net/http包
首先，请先忘记引子里的http.HandleFunc("/", sayhello)，这个要到很后面才提到

其实要使用http包，一句话就可以了，代码如下

```golang
package main

import "net/http"

func main() {
    http.ListenAndServe(":8080", nil)
}
```

访问网页后会发现，提示的不是“无法访问”，而是”页面没找到“，说明http已经开始服务了，只是没有找到页面
由此可以看出，访问什么路径显示什么网页 这件事情，和ListenAndServe的第2个参数有关

查询ListenAndServe的文档可知，第2个参数是一个Hander
Hander是啥呢，它是一个接口。这个接口很简单，只要某个struct有ServeHTTP(http.ResponseWriter, *http.Request)这个方法，那这个struct就自动实现了Hander接口

显示什么网页取决于第二个参数Hander，Hander又只有1个ServeHTTP
所以可以证明，显示什么网页取决于ServeHTTP

那就ServeHTTP方法，他需要2个参数，一个是http.ResponseWriter，另一个是*http.Request
往http.ResponseWriter写入什么内容，浏览器的网页源码就是什么内容
*http.Request里面是封装了，浏览器发过来的请求（包含路径、浏览器类型等等）

##认识http.ResponseWriter

把上面“网页未找到”的代码改一下，让大家认识一下http.ResponseWriter
看清楚了，ServeHTTP方法是自己写的啊，自己去实现
```golang
package main

import (
    "io"
    "net/http"
)

type a struct{}

func (*a) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "hello world version 1.")
}

func main() {
    http.ListenAndServe(":8080", &a{})//第2个参数需要实现Hander的struct，a满足
}
```
现在
访问localhost:8080的话，可以看到“hello world version 1.”
访问localhost:8080/abc的话，可以看到“hello world version 1.”
访问localhost:8080/123的话，可以看到“hello world version 1.”
事实上访问任何路径都是“hello world version 1.”

哦，原来是这样，当http.ListenAndServe(":8080", &a{})后，开始等待有访问请求
一旦有访问请求过来，http包帮我们处理了一系列动作后，最后他会去调用a的ServeHTTP这个方法，并把自己已经处理好的http.ResponseWriter, *http.Request传进去
而a的ServeHTTP这个方法，拿到*http.ResponseWriter后，并往里面写东西，客户端的网页就显示出来了

##认识*http.Request

现在把上面的代码再改一下，让大家认识一下*http.Request
```golang
package main

import (
    "io"
    "net/http"
)

type a struct{}

func (*a) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    path := r.URL.String() //获得访问的路径
    io.WriteString(w, path)
}

func main() {
    http.ListenAndServe(":8080", &a{})//第2个参数需要实现Hander接口的struct，a满足
}
```

现在
访问localhost:8080的话，可以看到“/”
访问localhost:8080/abc的话，可以看到“/abc”
访问localhost:8080/123的话，可以看到“/123”

##最傻的网站

如果再加上一些判断的话，一个最傻的可运行网站就出来了，如下
```golang
package main

import (
    "io"
    "net/http"
)

type a struct{}

func (*a) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    path := r.URL.String()
    switch path {
    case "/":
        io.WriteString(w, "<h1>root</h1><a href=\"abc\">abc</a>")
    case "/abc":
        io.WriteString(w, "<h1>abc</h1><a href=\"/\">root</a>")
    }
}

func main() {
    http.ListenAndServe(":8080", &a{})//第2个参数需要实现Hander接口的struct，a满足
}
```

运行后，可以看出，一个case就是一个页面
如果一个网站有上百个页面，那是否要上百个case？
很不幸，是的
那管理起来岂不是要累死？
要累死，不过，还好有ServeMux

##用ServeMux拯救最傻的网站

现在来介绍ServeMux

ServeMux大致作用是，他有一张map表，map里的key记录的是r.URL.String()，而value记录的是一个方法，这个方法和ServeHTTP是一样的，这个方法有一个别名，叫HandlerFunc
ServeMux还有一个方法名字是Handle，他是用来注册HandlerFunc 的
ServeMux还有另一个方法名字是ServeHTTP，这样ServeMux是实现Handler接口的，否者无法当http.ListenAndServe的第二个参数传输

代码来了
```golang
package main

import (
    "net/http"
    "io"
)

type b struct{}

func (*b) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "hello")
}
func main() {
    mux := http.NewServeMux()
    mux.Handle("/h", &b{})
    http.ListenAndServe(":8080", mux)
}
```
解释一下
mux := http.NewServeMux():新建一个ServeMux。
mux.Handle("/", &b{}):注册路由，把"/"注册给b这个实现Handler接口的struct，注册到map表中。
http.ListenAndServe(":8080", mux)第二个参数是mux。
运行时，因为第二个参数是mux，所以http会调用mux的ServeHTTP方法。
ServeHTTP方法执行时，会检查map表（表里有一条数据，key是“/h”，value是&b{}的ServeHTTP方法）
如果用户访问/h的话，mux因为匹配上了，mux的ServeHTTP方法会去调用&b{}的 ServeHTTP方法，从而打印hello
如果用户访问/abc的话，mux因为没有匹配上，从而打印404 page not found

ServeMux就是个二传手！

##ServeMux的HandleFunc方法

发现了没有，b这个struct仅仅是为了装一个ServeHTTP而存在，所以能否跳过b呢，ServeMux说：可以 mux.HandleFunc是用来注册func到map表中的
```golang
package main

import (
    "net/http"
    "io"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/h", func(w http.ResponseWriter, r *http.Request) {
        io.WriteString(w, "hello")
    })
    mux.HandleFunc("/bye", func(w http.ResponseWriter, r *http.Request) {
        io.WriteString(w, "byebye")
    })
    mux.HandleFunc("/hello", sayhello)
    http.ListenAndServe(":8080", mux)
}

func sayhello(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "hello world")
}
```
##ServeMux的白话解释

如果把http服务想象成一个公司的话 没有ServeMux的话，就像这个公司只有1个人--老板，什么事情都是由老板来做（自己写switch） 包括做业务（“/sales”），做帐（“/account”），内勤（“/management”），扫地（“/cleaner”）

后来老板招了4个人，负责上面的4件事情，那老板要做的就是根据情况转发就是了，比如做业务的事情，自己就不用去跑客户了，交给销售经理去做，并给销售经理一些资源（包括客户名称地址什么的）
上面代码里的type b struct{}就是个销售经理，资源就是w http.ResponseWriter, r *http.Request，老板要工作的就是ServeMux要做的工作

以上的代码就做了一层转发，老板给销售经理事情后，销售经理去跑客户了，也就结束了

实际生活中，销售经理不会自己去跑客户的，他也会把接来的活转给更下面的业务员
那时，销售经理自己也变成一个ServeMux了

所以为了可以ServeMux套ServeMux，需要改造一下ServeMux的ServeHTTP，改完后可以嵌套无限层，把要做的事情不断的细化
到最后要真正做事情的人，只需要关心自己要做的事情就可以了，自己做自己份内的事，一个大公司就运作起来了

##回到开头

回到开头，有让大家先忘掉http.HandleFunc("/", sayhello) 请先忘记引子里的http.HandleFunc("/", sayhello)，这个要到很后面才提到

当http.ListenAndServe(":8080", nil)的第2个参数是nil时
http内部会自己建立一个叫DefaultServeMux的ServeMux，因为这个ServeMux是http自己维护的，如果要向这个ServeMux注册的话，就要用http.HandleFunc这个方法啦，现在看很简单吧
