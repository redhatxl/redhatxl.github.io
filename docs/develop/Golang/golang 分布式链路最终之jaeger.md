# Golang 分布式链路追踪之jaeger

## 一 背景

- 开放式分布式追踪规范，详细介绍请自行百度 `google`。
- 常见实现：
  - jaeger
  - zipkin 

## 二 jaeger部署

### 2.1 Docker部署

* docker运行

```shell
docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.12
 
```

### 2.2 浏览器访问

http://localhost:16686/search

## 三 编写代码

目录结构

```shell
$ /workspace/goworkspace/src/github.com/kaliarch/jaeger-demo  tree 
.
├── client
│   ├── client.go
│   └── client_test.go
├── go.mod
├── go.sum
├── main.go
└── tracer
    └── tracer.go

```

tracer/trace.go

```go
package tracer

import (
	"io"
	"time"

	"github.com/opentracing/opentracing-go"
	"github.com/uber/jaeger-client-go"

	jaegercfg "github.com/uber/jaeger-client-go/config"
)

var Tracer opentracing.Tracer

// NewTracer 创建一个jaeger trace
func NewTracer(serverName, address string) (opentracing.Tracer, io.Closer, error) {

	// 生成jaegercfg
	cfg := jaegercfg.Configuration{
		ServiceName: serverName,
		Sampler: &jaegercfg.SamplerConfig{
			Type:  jaeger.SamplerTypeConst,
			Param: 1,
		},
		Reporter: &jaegercfg.ReporterConfig{
			LogSpans:            true,
			BufferFlushInterval: 1 * time.Second,
		},
	}
	transport, err := jaeger.NewUDPTransport(address, 0)
	if err != nil {
		return nil, nil, err
	}
	reporter := jaeger.NewRemoteReporter(transport)

	options := jaegercfg.Reporter(reporter)
	return cfg.NewTracer(options)
}

```

main.go

```go
package main

import (
	"fmt"
	"github.com/gin-gonic/gin"
	tracer2 "github.com/kaliarch/jaeger-demo/tracer"
	"github.com/opentracing-contrib/go-gin/ginhttp"
	"net/http"
)

const (
	ginServerName  = "gin-server-demo"
	jaegerEndpoint = "127.0.0.1:6831"
)

func main() {

	tracer, closer, err := tracer2.NewTracer(ginServerName, jaegerEndpoint)
	if err != nil {
		panic(err)
	}
	defer closer.Close()

	r := gin.Default()

	jaegerMiddle := ginhttp.Middleware(tracer, ginhttp.OperationNameFunc(func(r *http.Request) string {
		return fmt.Sprintf("HTTP %s %s", r.Method, r.URL.String())
	}))
	r.Use(ginhttp.Middleware(tracer))
	r.Use(jaegerMiddle)

	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"msg": "pong",
		})
	})
	_ = r.Run(":8888")
}

```

client.go

```go
package client

import (
	"context"
	"fmt"
	tracer2 "github.com/kaliarch/jaeger-demo/tracer"
	"github.com/opentracing-contrib/go-stdlib/nethttp"
	"github.com/opentracing/opentracing-go"
	"github.com/opentracing/opentracing-go/ext"
	otlog "github.com/opentracing/opentracing-go/log"

	"io/ioutil"
	"log"
	"net/http"
)

const (
	// 服务名 服务唯一标示，服务指标聚合过滤依据。
	clientServerName = "demo-gin-client"
	jaegerEndpoint   = "127.0.0.1:6831"
	ginEndpoint      = "http://127.0.0.1:8888"
)

func HandlerError(span opentracing.Span, err error) {
	span.SetTag(string(ext.Error), true)
	span.LogKV(otlog.Error(err))
	//log.Fatal("%v", err)
}

// StartClient gin client 也是标准的 http client.
func StartClient() {

	tracer, closer, err := tracer2.NewTracer(clientServerName, jaegerEndpoint)
	if err != nil {
		panic(err)
	}
	defer closer.Close()
	span := tracer.StartSpan("CallDemoServer")
	ctx := opentracing.ContextWithSpan(context.Background(), span)
	defer span.Finish()

	// 构建http请求
	req, err := http.NewRequest(
		http.MethodGet,
		fmt.Sprintf("%s/ping", ginEndpoint),
		nil,
	)
	if err != nil {
		HandlerError(span, err)
		return
	}
	// 构建带tracer的请求
	req = req.WithContext(ctx)
	req, ht := nethttp.TraceRequest(tracer, req)
	defer ht.Finish()

	// 初始化http客户端
	httpClient := &http.Client{Transport: &nethttp.Transport{}}
	// 发起请求
	res, err := httpClient.Do(req)
	if err != nil {
		HandlerError(span, err)
		return
	}
	defer res.Body.Close()
	body, err := ioutil.ReadAll(res.Body)
	if err != nil {
		HandlerError(span, err)
		return
	}
	log.Printf(" %s recevice: %s\n", clientServerName, string(body))
}

```

## 四 演示

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220109173823.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220109173855.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220109173959.png)



## 参考链接

* https://www.jaegertracing.io/
* https://github.com/why444216978/gin-api/blob/master/routers/router.go
* https://cloud.tencent.com/document/product/1463/57863