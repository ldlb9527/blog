+++

author = "旅店老板"
title = "OpenTelemetry实践"
date = "2024-08-04"
description = "了解OpenTelemetry如何实践，traces、metrics和logs数据如何产生相关性"
tags = [
	"OpenTelemetry"
]
categories = [
    "OpenTelemetry"
]
series = [""]
aliases = ["OpenTelemetry-l"]
image = "OpenTelemetry.png"
mermaid = true
+++

## 开始之前
如果你对OpenTelemetry协议标准不熟悉,可先查看[OpenTelemetry原理概览](https://ldlb.site/p/opentelemetry%E5%8E%9F%E7%90%86%E6%A6%82%E8%A7%88/) 。

## 介绍
OpenTelemetry 是一个云原生计算基金会（CNCF）项目，它是 是两个先前项目OpenTracing和OpenCensus合并的结果。如果你啊当前正在使用**OpenTracing**或**OpenCensus**，
则可以查看[迁移指南](https://opentelemetry.io/docs/migration/) ,了解如何迁移到OpenTelemetry。
### 什么是OpenTelemetry？
OpenTelemetry是一个与供应商无关的可观测性框架和工具包，旨在创建和管理遥测数据，例如**跟踪**、**指标**和**日志**。  

OpenTelemetry的主要组成：
* 所有组件的[规范](https://opentelemetry.io/docs/specs/otel/)
* 定义数据的[标准协议](https://opentelemetry.io/docs/specs/otlp/)
* 定义标准命名的[语义约定](https://opentelemetry.io/docs/specs/semconv/) 和常见遥测数据类型的方案
* 定义如何生成遥测数据的API
* 实现规范的[语言SDK](https://opentelemetry.io/docs/languages) 、API 和 导出遥测数据
* 
* 生成遥测数据的自动检测组件，无需更改代码
* [OpenTelemetry Collector](https://opentelemetry.io/docs/collector)，接收处理和导出遥测数据
* 各种其他工具，例如 Kubernetes 的 [OpenTelemetry Operator](https://opentelemetry.io/docs/kubernetes/operator/)
***
### 扩展性
OpenTelemetry设计为可扩展的。包括:
* 向 OpenTelemetry Collector添加接收器以支持来自自定义源的遥测数据
* 将自定义插桩库加载到SDK中
* 创建 SDK 的[分发版](https://opentelemetry.io/docs/concepts/distributions/) 或为特定用例量身定制的收集器
* 为尚不支持OpenTelemetry协议(OTLP)的自定义后端创建新的导出器
* 为非标准上下文传播格式创建自定义传播器
***
## 例子
下面使用golang集成OpenTelemetry，保证golang版本在1.22或更高。  

初始化程序如下:
```go
package main

import (
	"io"
	"log"
	"math/rand"
	"net/http"
	"strconv"
	
	
)

func rolldice(w http.ResponseWriter, r *http.Request) {
	roll := 1 + rand.Intn(6)

	resp := strconv.Itoa(roll) + "\n"
	if _, err := io.WriteString(w, resp); err != nil {
		log.Printf("Write failed: %v\n", err)
	}
}

func main() {
	http.HandleFunc("/rolldice", rolldice)

	log.Fatal(http.ListenAndServe(":8080", nil))
}
```
运行该程序访问[http://localhost:8080/rolldice](http://localhost:8080/rolldice) 可返回一个随机数。

使用`go get`安装依赖包:
```shell
go get "go.opentelemetry.io/otel" \
  "go.opentelemetry.io/otel/exporters/stdout/stdoutmetric" \
  "go.opentelemetry.io/otel/exporters/stdout/stdouttrace" \
  "go.opentelemetry.io/otel/exporters/stdout/stdoutlog" \
  "go.opentelemetry.io/otel/sdk/log" \
  "go.opentelemetry.io/otel/log/global" \
  "go.opentelemetry.io/otel/propagation" \
  "go.opentelemetry.io/otel/sdk/metric" \
  "go.opentelemetry.io/otel/sdk/resource" \
  "go.opentelemetry.io/otel/sdk/trace" \
  "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"\
  "go.opentelemetry.io/contrib/bridges/otelslog"

```

首先创建`otel.go`,初始化OpenTelemetry SDK，这是任何需要导出遥测数据的应用程序所必须的，如下:
```go
package main

import (
	"context"
	"errors"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/stdout/stdoutlog"
	"go.opentelemetry.io/otel/exporters/stdout/stdoutmetric"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/log/global"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/log"
	"go.opentelemetry.io/otel/sdk/metric"
	"go.opentelemetry.io/otel/sdk/trace"
)

// setupOTelSDK bootstraps the OpenTelemetry pipeline.
// If it does not return an error, make sure to call shutdown for proper cleanup.
func setupOTelSDK(ctx context.Context) (shutdown func(context.Context) error, err error) {
	var shutdownFuncs []func(context.Context) error

	// shutdown calls cleanup functions registered via shutdownFuncs.
	// The errors from the calls are joined.
	// Each registered cleanup will be invoked once.
	shutdown = func(ctx context.Context) error {
		var err error
		for _, fn := range shutdownFuncs {
			err = errors.Join(err, fn(ctx))
		}
		shutdownFuncs = nil
		return err
	}

	// handleErr calls shutdown for cleanup and makes sure that all errors are returned.
	handleErr := func(inErr error) {
		err = errors.Join(inErr, shutdown(ctx))
	}

	// Set up propagator.
	prop := newPropagator()
	otel.SetTextMapPropagator(prop)

	// Set up trace provider.
	tracerProvider, err := newTraceProvider()
	if err != nil {
		handleErr(err)
		return
	}
	shutdownFuncs = append(shutdownFuncs, tracerProvider.Shutdown)
	otel.SetTracerProvider(tracerProvider)

	// Set up meter provider.
	meterProvider, err := newMeterProvider()
	if err != nil {
		handleErr(err)
		return
	}
	shutdownFuncs = append(shutdownFuncs, meterProvider.Shutdown)
	otel.SetMeterProvider(meterProvider)

	// Set up logger provider.
	loggerProvider, err := newLoggerProvider()
	if err != nil {
		handleErr(err)
		return
	}
	shutdownFuncs = append(shutdownFuncs, loggerProvider.Shutdown)
	global.SetLoggerProvider(loggerProvider)

	return
}

func newPropagator() propagation.TextMapPropagator {
	return propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{},
		propagation.Baggage{},
	)
}

func newTraceProvider() (*trace.TracerProvider, error) {
	traceExporter, err := stdouttrace.New(
		stdouttrace.WithPrettyPrint())
	if err != nil {
		return nil, err
	}

	traceProvider := trace.NewTracerProvider(
		trace.WithBatcher(traceExporter,
			// Default is 5s. Set to 1s for demonstrative purposes.
			trace.WithBatchTimeout(time.Second)),
	)
	return traceProvider, nil
}

func newMeterProvider() (*metric.MeterProvider, error) {
	metricExporter, err := stdoutmetric.New()
	if err != nil {
		return nil, err
	}

	meterProvider := metric.NewMeterProvider(
		metric.WithReader(metric.NewPeriodicReader(metricExporter,
			// Default is 1m. Set to 3s for demonstrative purposes.
			metric.WithInterval(3*time.Second))),
	)
	return meterProvider, nil
}

func newLoggerProvider() (*log.LoggerProvider, error) {
	logExporter, err := stdoutlog.New()
	if err != nil {
		return nil, err
	}

	loggerProvider := log.NewLoggerProvider(
		log.WithProcessor(log.NewBatchProcessor(logExporter)),
	)
	return loggerProvider, nil
}

```
现在可以检测HTTP服务器了，修改`main.go`如下:
```go
package main

import (
	"context"
	"errors"
	"io"
	"log"
	"math/rand"
	"net"
	"net/http"
	"os"
	"os/signal"
	"strconv"
	"time"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

func rolldice(w http.ResponseWriter, r *http.Request) {
	roll := 1 + rand.Intn(6)

	resp := strconv.Itoa(roll) + "\n"
	if _, err := io.WriteString(w, resp); err != nil {
		log.Printf("Write failed: %v\n", err)
	}
}

func main() {
	if err := run(); err != nil {
		log.Fatalln(err)
	}
}

func run() (err error) {
	// Handle SIGINT (CTRL+C) gracefully.
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
	defer stop()

	// Set up OpenTelemetry.
	otelShutdown, err := setupOTelSDK(ctx)
	if err != nil {
		return
	}
	// Handle shutdown properly so nothing leaks.
	defer func() {
		err = errors.Join(err, otelShutdown(context.Background()))
	}()

	// Start HTTP server.
	srv := &http.Server{
		Addr:         ":8080",
		BaseContext:  func(_ net.Listener) context.Context { return ctx },
		ReadTimeout:  time.Second,
		WriteTimeout: 10 * time.Second,
		Handler:      newHTTPHandler(),
	}
	srvErr := make(chan error, 1)
	go func() {
		srvErr <- srv.ListenAndServe()
	}()

	// Wait for interruption.
	select {
	case err = <-srvErr:
		// Error when starting HTTP server.
		return
	case <-ctx.Done():
		// Wait for first CTRL+C.
		// Stop receiving signal notifications as soon as possible.
		stop()
	}

	// When Shutdown is called, ListenAndServe immediately returns ErrServerClosed.
	err = srv.Shutdown(context.Background())
	return
}

func newHTTPHandler() http.Handler {
	mux := http.NewServeMux()

	// handleFunc is a replacement for mux.HandleFunc
	// which enriches the handler's HTTP instrumentation with the pattern as the http.route.
	handleFunc := func(pattern string, handlerFunc func(http.ResponseWriter, *http.Request)) {
		// Configure the "http.route" for the HTTP instrumentation.
		handler := otelhttp.WithRouteTag(pattern, http.HandlerFunc(handlerFunc))
		mux.Handle(pattern, handler)
	}

	// Register handlers.
	handleFunc("/rolldice/", rolldice)
	handleFunc("/rolldice/{player}", rolldice)

	// Add HTTP instrumentation for the whole server.
	handler := otelhttp.NewHandler(mux, "/test")
	return handler
}
```
启动程序,访问接口,控制台打印出json数据,json的很多字段在[OpenTelemetry原理概览](https://ldlb.site/p/opentelemetry%E5%8E%9F%E7%90%86%E6%A6%82%E8%A7%88/) 
中有过介绍,如下(省略了部分**Resource**和**Attributes**的值):
```json
{
	"Name": "/test",
	"SpanContext": {
		"TraceID": "ea2bb77e353e2fabdcb386d8541f10c6",
		"SpanID": "6b11a5a6f57081a2",
		"TraceFlags": "01",
		"TraceState": "",
		"Remote": false
	},
	"Parent": {
		"TraceID": "00000000000000000000000000000000",
		"SpanID": "0000000000000000",
		"TraceFlags": "00",
		"TraceState": "",
		"Remote": false
	},
	"SpanKind": 2,
	"StartTime": "2024-08-06T00:57:32.7391651+08:00",
	"EndTime": "2024-08-06T00:57:32.7391651+08:00",
	"Attributes": [
		{
			"Key": "http.method",
			"Value": {
				"Type": "STRING",
				"Value": "GET"
			}
		},
		{
			"Key": "http.target",
			"Value": {
				"Type": "STRING",
				"Value": "/rolldice/1"
			}
		}
	],
	"Events": null,
	"Links": null,
	"Status": {
		"Code": "Unset",
		"Description": ""
	},
	"DroppedAttributes": 0,
	"DroppedEvents": 0,
	"DroppedLinks": 0,
	"ChildSpanCount": 0,
	"Resource": [
		{
			"Key": "telemetry.sdk.language",
			"Value": {
				"Type": "STRING",
				"Value": "go"
			}
		}
	],
	"InstrumentationLibrary": {
		"Name": "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp",
		"Version": "0.53.0",
		"SchemaURL": ""
	}
}
```
修改`main.go`的代码,如下:
```go
package main

import (
	"context"
	"errors"
	"fmt"
	"go.opentelemetry.io/contrib/bridges/otelslog"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"io"
	"log"
	"math/rand"
	"net"
	"net/http"
	"os"
	"os/signal"
	"strconv"
	"time"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel/metric"
)

const name = "go.opentelemetry.io/otel/example/dice"

var (
	tracer = otel.Tracer(name)
	meter  = otel.Meter(name)
	logger = otelslog.NewLogger(name)
	rollCnt metric.Int64Counter
)

func init() {
	var err error
	rollCnt, err = meter.Int64Counter("dice.rolls",
		metric.WithDescription("The number of rolls by roll value"),
		metric.WithUnit("{roll}"))
	if err != nil {
		panic(err)
	}
}

func rolldice(w http.ResponseWriter, r *http.Request) {
	ctx, span := tracer.Start(r.Context(), "roll")
	defer span.End()

	roll := 1 + rand.Intn(6)

	var msg string
	if player := r.PathValue("player"); player != "" {
		msg = fmt.Sprintf("%s is rolling the dice", player)
	} else {
		msg = "Anonymous player is rolling the dice"
	}
	logger.InfoContext(ctx, msg, "result", roll)

	rollValueAttr := attribute.Int("roll.value", roll)
	span.SetAttributes(rollValueAttr)
	rollCnt.Add(ctx, 1, metric.WithAttributes(rollValueAttr))

	resp := strconv.Itoa(roll) + "\n"
	if _, err := io.WriteString(w, resp); err != nil {
		log.Printf("Write failed: %v\n", err)
	}
}
```
其余代码不变,重新启动程序调用接口，控制台打印了多条json, 有日志的json、指标的json和链路追踪的json,span之间也有了父子关系。








