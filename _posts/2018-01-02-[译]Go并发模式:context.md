---
layout: post
title: "[译]Go并发模式:context"
date: 2018-01-02
categories: Go 并发 Context
---

原文地址: [Go Concurrency Patterns: Context](https://blog.golang.org/context)

## Introduction

在Go server中，新的请求通常都会起一个新的goroutine处理，这个goroutine又通常会起一些额外的goroutines来访问后端，例如database，RPC服务等。这一系列处理一个请求的goroutines通常需要访问请求相关的数据，例如最终用户的身份，Authorization tokens，请求截止时间等。当一个请求退出或者超时，所有为这个请求工作的goroutines都必须立即退出，然后系统才能回收他们占用的资源。

Google开发并开源了一个`context`包可以很容易的通过API边界向所有处理同一个请求的goroutines传递请求范围的数据，退出信号，截止时间等（goroutines之间的全局变量共享， 以及同步退出控制）。这篇文章描述了如何使用这个包，以及提供一个完整的可工作的示例。

## Context

context包的核心就是定义的`Context` interface。

```
// A Context carries a deadline, cancelation signal, and request-scoped values
// across API boundaries. Its methods are safe for simultaneous use by multiple
// goroutines.
type Context interface {
    // Done returns a channel that is closed when this Context is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```

`Done`方法返回一个作为退出信号的channel到运行在Context中的函数（通俗的讲就是接受Context变量的Goroutines通过Done方法获取退出信号）。如果channel被关闭(close),该函数（goroutines）应当立即停止工作并退出。[Pipelines and Cancelation](https://blog.golang.org/pipelines)这篇文章更详细的讨论了`Done` channel。

`Error`方法返回一个error用于指示`Context`退出的理由。

`Context`没有一个`Cancel`方法，原因同`Done`返回的channel是一个只能用于从channel读取数据的单向channel一样：就是接收退出信号的函数通常不是信号的发送方。通常情况下，当一个父亲起了一些goroutines处理一些子任务，这些子goroutines是不能终止parent并让其退出的。作为替代下面将提到的`WithCancel`方法将提供一个方法用于退出一个新的Context值。

Context对被多个goroutines同时使用是安全的。代码可以将同一个Context传给任何数量的goroutines使用，并像它们发送信号指示退出这个context。

`Deadline`方法允许函数决定是否启动工作；如果距离timeout只有很少的时间，可能就没有必要启动执行了。代码还可以使用`Deadline`为IO操作设置超时。

`Value`方法允许Context携带一些请求范围的数据。这些数据必须是对多个goroutines同时访问安全的。

## Derived context

context包提供了方法从一个已有的`Context`派生新的`Context`。所有这些`Context`组成了一棵树：如果一个`Context`退出，则从它派生出的所有`Context`都将退出。

`Background`是所有`Context`树的根节点，它永远不会退出。

```
// Background returns an empty Context. It is never canceled, has no deadline,
// and has no values. Background is typically used in main, init, and tests,
// and as the top-level Context for incoming requests.
func Background() Context
```

`WithCancel`和`WithTimeout`返回一个可以比parent Context更早被退出的派生Context。典型情况下，与请求相关的Context通常在请求Handler退出时退出。`WithCancel`对于使用多个副本时用于退出冗余请求也很有用。`WithTimeout`对设置请求的后台服务的超时也很有用。

```
// WithCancel returns a copy of parent whose Done channel is closed as soon as
// parent.Done is closed or cancel is called.
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// A CancelFunc cancels a Context.
type CancelFunc func()

// WithTimeout returns a copy of parent whose Done channel is closed as soon as
// parent.Done is closed, cancel is called, or timeout elapses. The new
// Context's Deadline is the sooner of now+timeout and the parent's deadline, if
// any. If the timer is still running, the cancel function releases its
// resources.
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

`WithValue`提供了一种将请求范围的数据同`Context`关联的方法。

```
// WithValue returns a copy of parent whose Value method returns val for key.
func WithValue(parent Context, key interface{}, val interface{}) Context
```

## Example: Google Web Search

我们的示例是一个HTTP server，它处理像`/search?q=golang&timeout=1s`一样的URLs，转发查询`golang`的请求到[Google Web Search API](https://developers.google.com/web-search/docs/)并翻译结果。`timeout`告诉服务器多长时间后退出请求。

代码被分成了三个部分：

* server提供了`main`函数以及`/search`的handler；
* userip提供了从请求中解析用户IP并将它关联的`Context`的功能；
* google提供了`Search`函数，用于向Google发送查询请求。

### server部分代码

`server`代码通过为golang提供Google的前几个搜索结果来处理像`/search?q=golang`一样的请求。它为注册了一个`handleSearch`来处理`/search`API。这个handler创建了一个叫做ctx的初始`Context`，并在handler返回时安排`ctx`退出。如果请求中还包含了叫`timeout`的URL参数，ctx将在超时时自动退出。

handler解析请求中的查询条件，还会通过调用`userip`包来解析请求中的用户IP。用户IP信息在后端服务中被需要，所以`handleSearch`将它存入`ctx`中。



``` server.go
// The server program issues Google search requests and demonstrates the use of
// the go.net Context API. It serves on port 8080.
//
// The /search endpoint accepts these query params:
//   q=the Google search query
//   timeout=a timeout for the request, in time.Duration format
//
// For example, http://localhost:8080/search?q=golang&timeout=1s serves the
// first few Google search results for "golang" or a "deadline exceeded" error
// if the timeout expires.
package main

import (
	"html/template"
	"log"
	"net/http"
	"time"

	"golang.org/x/blog/content/context/google"
	"golang.org/x/blog/content/context/userip"
	"golang.org/x/net/context"
)

func main() {
	http.HandleFunc("/search", handleSearch)
	log.Fatal(http.ListenAndServe(":8080", nil))
}

// handleSearch handles URLs like /search?q=golang&timeout=1s by forwarding the
// query to google.Search. If the query param includes timeout, the search is
// canceled after that duration elapses.
func handleSearch(w http.ResponseWriter, req *http.Request) {
	// ctx is the Context for this handler. Calling cancel closes the
	// ctx.Done channel, which is the cancellation signal for requests
	// started by this handler.
	var (
		ctx    context.Context
		cancel context.CancelFunc
	)
	timeout, err := time.ParseDuration(req.FormValue("timeout"))
	if err == nil {
		// The request has a timeout, so create a context that is
		// canceled automatically when the timeout expires.
		ctx, cancel = context.WithTimeout(context.Background(), timeout)
	} else {
		ctx, cancel = context.WithCancel(context.Background())
	}
	defer cancel() // Cancel ctx as soon as handleSearch returns.

	// Check the search query.
	query := req.FormValue("q")
	if query == "" {
		http.Error(w, "no query", http.StatusBadRequest)
		return
	}

	// Store the user IP in ctx for use by code in other packages.
	userIP, err := userip.FromRequest(req)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}
	ctx = userip.NewContext(ctx, userIP)

	// Run the Google search and print the results.
	start := time.Now()
	results, err := google.Search(ctx, query)
	elapsed := time.Since(start)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	if err := resultsTemplate.Execute(w, struct {
		Results          google.Results
		Timeout, Elapsed time.Duration
	}{
		Results: results,
		Timeout: timeout,
		Elapsed: elapsed,
	}); err != nil {
		log.Print(err)
		return
	}
}

var resultsTemplate = template.Must(template.New("results").Parse(`
<html>
<head/>
<body>
  <ol>
  {{range .Results}}
    <li>{{.Title}} - <a href="{{.URL}}">{{.URL}}</a></li>
  {{end}}
  </ol>
  <p>{{len .Results}} results in {{.Elapsed}}; timeout {{.Timeout}}</p>
</body>
</html>
`))
```

### Package userip

`userip`包提供了从用户请求中解析用户IP，并将它关联到Context的功能。一个Context提供了一个key-value mapping，其中key和value的类型都是interface{}。key类型必须支持等（equal）判断，并且value必须对多个goroutines同时访问安全。像`userip`这样的软件包应该隐藏所有的映射细节，并提供对指定`Context` value的强类型访问。

为了避免key冲突，`userip`定义了一个不对外暴露的`key`类型，并使用这个类型的数据作为Context的key。

```userip.go
// Package userip provides functions for extracting a user IP address from a
// request and associating it with a Context.
//
// This package is an example to accompany https://blog.golang.org/context.
// It is not intended for use by others.
package userip

import (
	"fmt"
	"net"
	"net/http"

	"golang.org/x/net/context"
)

// FromRequest extracts the user IP address from req, if present.
func FromRequest(req *http.Request) (net.IP, error) {
	ip, _, err := net.SplitHostPort(req.RemoteAddr)
	if err != nil {
		return nil, fmt.Errorf("userip: %q is not IP:port", req.RemoteAddr)
	}

	userIP := net.ParseIP(ip)
	if userIP == nil {
		return nil, fmt.Errorf("userip: %q is not IP:port", req.RemoteAddr)
	}
	return userIP, nil
}

// The key type is unexported to prevent collisions with context keys defined in
// other packages.
type key int

// userIPkey is the context key for the user IP address.  Its value of zero is
// arbitrary.  If this package defined other context keys, they would have
// different integer values.
const userIPKey key = 0

// NewContext returns a new Context carrying userIP.
func NewContext(ctx context.Context, userIP net.IP) context.Context {
	return context.WithValue(ctx, userIPKey, userIP)
}

// FromContext extracts the user IP address from ctx, if present.
func FromContext(ctx context.Context) (net.IP, bool) {
	// ctx.Value returns nil if ctx has no value for the key;
	// the net.IP type assertion returns ok=false for nil.
	userIP, ok := ctx.Value(userIPKey).(net.IP)
	return userIP, ok
}
```

### package google

`goole.Search`函数会向Google Web Search API发送一个HTTP请求，并解析返回的JSON编码的结果。它接收一个Context类型的参数`ctx`并且在request还没有执行完成，但是`ctx.Done`被关闭的时候立即返回。

Google Web Search API要求提供查询条件和用户IP做为参数。

`Search`使用了一个帮助函数`httpDo`，由`httpDo`发起HTTP请求。如果在request或response仍然在处理时，`ctx.Done`被关闭时，`httpDo`也将被退出。`Search`传递了一个闭包给`httpDo`来处理HTTP response。

``` google.go
// Package google provides a function to do Google searches using the Google Web
// Search API. See https://developers.google.com/web-search/docs/
//
// This package is an example to accompany https://blog.golang.org/context.
// It is not intended for use by others.
//
// Google has since disabled its search API,
// and so this package is no longer useful.
package google

import (
	"encoding/json"
	"net/http"

	"golang.org/x/blog/content/context/userip"
	"golang.org/x/net/context"
)

// Results is an ordered list of search results.
type Results []Result

// A Result contains the title and URL of a search result.
type Result struct {
	Title, URL string
}

// Search sends query to Google search and returns the results.
func Search(ctx context.Context, query string) (Results, error) {
	// Prepare the Google Search API request.
	req, err := http.NewRequest("GET", "https://ajax.googleapis.com/ajax/services/search/web?v=1.0", nil)
	if err != nil {
		return nil, err
	}
	q := req.URL.Query()
	q.Set("q", query)

	// If ctx is carrying the user IP address, forward it to the server.
	// Google APIs use the user IP to distinguish server-initiated requests
	// from end-user requests.
	if userIP, ok := userip.FromContext(ctx); ok {
		q.Set("userip", userIP.String())
	}
	req.URL.RawQuery = q.Encode()

	// Issue the HTTP request and handle the response. The httpDo function
	// cancels the request if ctx.Done is closed.
	var results Results
	err = httpDo(ctx, req, func(resp *http.Response, err error) error {
		if err != nil {
			return err
		}
		defer resp.Body.Close()

		// Parse the JSON search result.
		// https://developers.google.com/web-search/docs/#fonje
		var data struct {
			ResponseData struct {
				Results []struct {
					TitleNoFormatting string
					URL               string
				}
			}
		}
		if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
			return err
		}
		for _, res := range data.ResponseData.Results {
			results = append(results, Result{Title: res.TitleNoFormatting, URL: res.URL})
		}
		return nil
	})
	// httpDo waits for the closure we provided to return, so it's safe to
	// read results here.
	return results, err
}

// httpDo issues the HTTP request and calls f with the response. If ctx.Done is
// closed while the request or f is running, httpDo cancels the request, waits
// for f to exit, and returns ctx.Err. Otherwise, httpDo returns f's error.
func httpDo(ctx context.Context, req *http.Request, f func(*http.Response, error) error) error {
	// Run the HTTP request in a goroutine and pass the response to f.
	tr := &http.Transport{}
	client := &http.Client{Transport: tr}
	c := make(chan error, 1)
	go func() { c <- f(client.Do(req)) }()
	select {
	case <-ctx.Done():
		tr.CancelRequest(req)
		<-c // Wait for f to return.
		return ctx.Err()
	case err := <-c:
		return err
	}
}
```

### Adapting code for Contexts

有些服务框架提供包或类型用来携带请求范围的参数。我们可以定义一个新的Context interface实现来作为使用框架的代码和希望使用Context作为参数的代码之间的桥梁。

例如，Gorilla的[github.com/gorilla/context](github.com/gorilla/context)包允许handler通过提供将HTTP请求到key-value的映射来关联请求和对应的数据。在下面的`gorilla.go`代码中，我们提供了一个Context实现，它的`Value`方法返回了Gorilla包中指定HTTP request的数据。

```gorilla.go
// +build OMIT

// Package gorilla provides a go.net/context.Context implementation whose Value
// method returns the values associated with a specific HTTP request in the
// github.com/gorilla/context package.
package gorilla

import (
	"net/http"

	gcontext "github.com/gorilla/context"
	"golang.org/x/net/context"
)

// NewContext returns a Context whose Value method returns values associated
// with req using the Gorilla context package:
// http://www.gorillatoolkit.org/pkg/context
func NewContext(parent context.Context, req *http.Request) context.Context {
	return &wrapper{parent, req}
}

type wrapper struct {
	context.Context
	req *http.Request
}

type key int

const reqKey key = 0

// Value returns Gorilla's context package's value for this Context's request
// and key. It delegates to the parent Context if there is no such value.
func (ctx *wrapper) Value(key interface{}) interface{} {
	if key == reqKey {
		return ctx.req
	}
	if val, ok := gcontext.GetOk(ctx.req, key); ok {
		return val
	}
	return ctx.Context.Value(key)
}

// HTTPRequest returns the *http.Request associated with ctx using NewContext,
// if any.
func HTTPRequest(ctx context.Context) (*http.Request, bool) {
	// We cannot use ctx.(*wrapper).req to get the request because ctx may
	// be a Context derived from a *wrapper. Instead, we use Value to
	// access the request if it is anywhere up the Context tree.
	req, ok := ctx.Value(reqKey).(*http.Request)
	return req, ok
}
```

其他的一些包可能提供了类似Context的退出机制。例如[Tomb](http://godoc.org/gopkg.in/tomb.v2)提供了一个`Kill`方法来发起通过关闭一个Dying channel来退出。Tomb还提供了方法等待所有的goroutines退出，类似于`sync.WaitGroup`。在下面的`tomb.go`中我们实现了一个Context，它将在父Context退出，或提供的Tomb被kill时退出。

```tomb.go
// +build OMIT

// Package tomb provides a Context implementation that is canceled when either
// its parent Context is canceled or a provided Tomb is killed.
package tomb

import (
	"golang.org/x/net/context"
	tomb "gopkg.in/tomb.v2"
)

// NewContext returns a Context that is canceled either when parent is canceled
// or when t is Killed.
func NewContext(parent context.Context, t *tomb.Tomb) context.Context {
	ctx, cancel := context.WithCancel(parent)
	go func() {
		select {
		case <-t.Dying():
			cancel()
		case <-ctx.Done():
		}
	}()
	return ctx
}
```

## 结论

在Google所有的Go程序员都被要求将Context变量作为传入和传出请求之间所有调用路径上的函数的第一个参数。这允许不同团队之间开发的Go代码能够良好的交互。它提供了简单的超时和退出机制，并确保诸如安全证书之类的数据能够在Go程序间正确的传递。

如果想基于Context使用一些服务框架，则需要提供一个Context实现作为包和期望的Context参数之间的桥梁。这样它们的client就可以通过这个桥梁来接收一个Context。通过为请求范围的数据和退出机制建立通用的接口，Context使得包开发者能够为创建可扩展的服务更容易共享代码。


