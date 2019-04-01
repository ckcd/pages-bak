---
layout: article
title: CoreDNS Overview
tags: dns CoreDNS
key: coredns-overview
---

介绍CoreDNS的框架和主体逻辑。

<!--more-->

# CoreDNS综述

> CoreDNS is powered by plugins.

官网：[https://coredns.io/](https://coredns.io/)

GitHub：[https://github.com/coredns/coredns](https://github.com/coredns/coredns)

CoreDNS是SkyDNS的作者，Miek Gieben，创建的新的DNS服务器，它采用更模块化，可扩展的框架构建。 在kubernetes 1.11版本中，将CoreDNS作为默认的DNS模块，从而取代了kube-dns。CoreDNS采用插件的形式构建，每个插件实现不同的功能，多个插件串联起来组成完整的对DNS的处理通路，不同插件之间是完全解耦合的。

CoreDNS是利用Web服务器Caddy的一部分而开发的服务器框架。该框架具有非常灵活，可扩展的模型，用于通过各种中间件组件传递请求。这些中间件组件根据请求提供不同的操作，例如记录，重定向，修改或维护。

在这种灵活的模型中添加对Kubernetes的支持，相当于创建了一个Kubernetes中间件。该中间件使用Kubernetes API来满足针对特定Kubernetes pod或服务的DNS请求。

对于已经部署了kube-dns的集群，CoreDNS也提供了便捷的方法来迁移到CoreDNS，事实上，kube-dns/CoreDNS扮演的是kubernetes集群中一个用户的角色，在集群中，对于DNS请求者来说，只需要从某个DNS server获取对应的DNS响应，而并不关心实际处理该DNS请求的是kube-dns还是CoreDNS，这也决定了我们可能能够做到用户无感知的将kube-dns升级到CoreDNS。

本文主要介绍以下几个内容：CoreDNS的整体架构、CoreDNS本体的代码执行、每个plugin的代码逻辑以及多个plugin如何协作；

## 整体架构

CoreDNS官网中给出了这样一个例子，对于如下的配置文件Corefile：

```go
coredns.io:5300 {
    file db.coredns.io
}

example.io:53 {
    log
    errors
    file db.example.io
}

example.net:53 {
    file db.example.net
}

.:53 {
    kubernetes
    proxy . 8.8.8.8
    log
    errors
    cache
}
```
基于该配置文件的CoreDNS处理DNS请求的流程可以表示为：
![image](https://coredns.io/images/CoreDNS-Corefile.png)

详细解释如下：

* 图中对于不同的域名有不同的处理方法，每个规定的域名称为一个`zone`；
* 本体（即图中的ServeDNS部分）负责监听端口并接收所有client发来的的DNS请求，并且在得到响应后发送给对应的client；
* 对于所有DNS请求，首先在本体（即图中的ServeDNS部分）进行处理，解析其属于哪个`zone`并传递到对应的plugin chain；
* 每个zone中定义了一系列的plugin进行处理，例如cache、log等，多个plugin串联起来；
* 每个plugin以DNS请求为输入，试图得到对应的DNS响应，如果没有得到DNS响应那么将DNS请求传递给下一个plugin；
* 如果得到了DNS响应那么将该响应返回给上一层plugin，最终DNS响应会被传回本体；


## CoreDNS本体代码详解

一、CoreDNS使用Caddy这一web server作为底层依托，`core/dnsserver/register.go`中调用`caddy.RegisterServerType`将自己注册到caddy中，serverType为`dns`；

```go
const serverType = "dns"

// Any flags defined here, need to be namespaced to the serverType other
// wise they potentially clash with other server types.
func init() {
	flag.StringVar(&Port, serverType+".port", DefaultPort, "Default port")

	caddy.RegisterServerType(serverType, caddy.ServerType{
		Directives: func() []string { return Directives },
		DefaultInput: func() caddy.Input {
			return caddy.CaddyfileInput{
				Filepath:       "Corefile",
				Contents:       []byte(".:" + Port + " {\nwhoami\n}\n"),
				ServerTypeName: serverType,
			}
		},
		NewContext: newContext,
	})
}
```

二、`coremain/run.go`中读取配置文件、解析命令行参数、启动Server；

如下代码所示，对于输入的配置文件`Corefile`和flag解析出来的命令行参数，都作为初始化caddy的参数；

```go
func init() {
	caddy.DefaultConfigFile = "Corefile"
	caddy.Quiet = true // don't show init stuff from caddy
	setVersion()

	flag.StringVar(&conf, "conf", "", "Corefile to load (default \""+caddy.DefaultConfigFile+"\")")
	flag.StringVar(&cpu, "cpu", "100%", "CPU cap")
	flag.BoolVar(&plugins, "plugins", false, "List installed plugins")
	flag.StringVar(&caddy.PidFile, "pidfile", "", "Path to write pid file")
	flag.BoolVar(&version, "version", false, "Show version")
	flag.BoolVar(&dnsserver.Quiet, "quiet", false, "Quiet mode (no initialization output)")

	caddy.RegisterCaddyfileLoader("flag", caddy.LoaderFunc(confLoader))
	caddy.SetDefaultCaddyfileLoader("default", caddy.LoaderFunc(defaultLoader))

	caddy.AppName = coreName
	caddy.AppVersion = CoreVersion
}
```

`Run()`方法根据命令行参数进一步配置caddy的参数，例如根据输入的`cpu`配置最大cpu限制`GOMAXPROCS`，最后调用caddy.Start得到CoreDNS的实例，并且调用caddy.EmitEvent开始监听事件并处理；

```go
// Run is CoreDNS's main() function.
func Run() {

    ...
    
	flag.CommandLine = flag.NewFlagSet(os.Args[0], flag.ExitOnError)
	...

	// Set CPU cap
	if err := setCPU(cpu); err != nil {
		mustLogFatal(err)
	}

    ...

	// Start your engines
	instance, err := caddy.Start(corefile)
	if err != nil {
		mustLogFatal(err)
	}

	if !dnsserver.Quiet {
		showVersion()
	}

	// Execute instantiation events
	caddy.EmitEvent(caddy.InstanceStartupEvent, instance)

	// Twiddle your thumbs
	instance.Wait()
}
```


三、`core/dnsserver/server.go`的serve方法和servePacket方法实现`caddy.TCPServer interface`，用于连接caddy和CoreDNS，即在caddy遇到属于CoreDNS的连接时应该调用哪些方法来处理，如下代码所示，serve方法定义了对于TCP的处理，servePacket定义了对于UDP的处理，总的来说，两者都是调用s.ServeDNS方法；

```go
// Serve starts the server with an existing listener. It blocks until the server stops.
// This implements caddy.TCPServer interface.
func (s *Server) Serve(l net.Listener) error {
	s.m.Lock()
	s.server[tcp] = &dns.Server{Listener: l, Net: "tcp", Handler: dns.HandlerFunc(func(w dns.ResponseWriter, r *dns.Msg) {
		ctx := context.WithValue(context.Background(), Key{}, s)
		s.ServeDNS(ctx, w, r)
	})}
	s.m.Unlock()

	return s.server[tcp].ActivateAndServe()
}

// ServePacket starts the server with an existing packetconn. It blocks until the server stops.
// This implements caddy.UDPServer interface.
func (s *Server) ServePacket(p net.PacketConn) error {
	s.m.Lock()
	s.server[udp] = &dns.Server{PacketConn: p, Net: "udp", Handler: dns.HandlerFunc(func(w dns.ResponseWriter, r *dns.Msg) {
		ctx := context.WithValue(context.Background(), Key{}, s)
		s.ServeDNS(ctx, w, r)
	})}
	s.m.Unlock()

	return s.server[udp].ActivateAndServe()
}
```

四、`s.ServeDNS`方法是CoreDNS对DNS请求进行处理的第一步，该方法在进行若干正确性检查后，将DNS请求传递到plugin chain中去一步步的执行，也即依次执行每个plugin的`ServeDNS`方法；

```go
// ServeDNS is the entry point for every request to the address that s
// is bound to. It acts as a multiplexer for the requests zonename as
// defined in the request so that the correct zone
// (configuration and plugin stack) will handle the request.
func (s *Server) ServeDNS(ctx context.Context, w dns.ResponseWriter, r *dns.Msg) {
	// check
	...

	q := r.Question[0].Name
	b := make([]byte, len(q))
	var off int
	var end bool

	var dshandler *Config

	for {
		l := len(q[off:])
		for i := 0; i < l; i++ {
			b[i] = q[off+i]
			// normalize the name for the lookup
			if b[i] >= 'A' && b[i] <= 'Z' {
				b[i] |= ('a' - 'A')
			}
		}

		if h, ok := s.zones[string(b[:l])]; ok {

			// Set server's address in the context so plugins can reference back to this,
			// This will makes those metrics unique.
			ctx = context.WithValue(ctx, plugin.ServerCtx{}, s.Addr)

			if r.Question[0].Qtype != dns.TypeDS {
				if h.FilterFunc == nil {
					rcode, _ := h.pluginChain.ServeDNS(ctx, w, r)
					if !plugin.ClientWrite(rcode) {
						DefaultErrorFunc(ctx, w, r, rcode)
					}
					return
				}
				// FilterFunc is set, call it to see if we should use this handler.
				// This is given to full query name.
				if h.FilterFunc(q) {
					rcode, _ := h.pluginChain.ServeDNS(ctx, w, r)
					if !plugin.ClientWrite(rcode) {
						DefaultErrorFunc(ctx, w, r, rcode)
					}
					return
				}
			}
			// The type is DS, keep the handler, but keep on searching as maybe we are serving
			// the parent as well and the DS should be routed to it - this will probably *misroute* DS
			// queries to a possibly grand parent, but there is no way for us to know at this point
			// if there is an actually delegation from grandparent -> parent -> zone.
			// In all fairness: direct DS queries should not be needed.
			dshandler = h
		}
		off, end = dns.NextLabel(q, off)
		if end {
			break
		}
	}

	if r.Question[0].Qtype == dns.TypeDS && dshandler != nil && dshandler.pluginChain != nil {
		// DS request, and we found a zone, use the handler for the query.
		rcode, _ := dshandler.pluginChain.ServeDNS(ctx, w, r)
		if !plugin.ClientWrite(rcode) {
			DefaultErrorFunc(ctx, w, r, rcode)
		}
		return
	}

	// Wildcard match, if we have found nothing try the root zone as a last resort.
	if h, ok := s.zones["."]; ok && h.pluginChain != nil {

		// See comment above.
		ctx = context.WithValue(ctx, plugin.ServerCtx{}, s.Addr)

		rcode, _ := h.pluginChain.ServeDNS(ctx, w, r)
		if !plugin.ClientWrite(rcode) {
			DefaultErrorFunc(ctx, w, r, rcode)
		}
		return
	}

	// Still here? Error out with REFUSED.
	DefaultErrorFunc(ctx, w, r, dns.RcodeRefused)
}
```

五、那么，plugin chain中每个plugin，以及它们之间的顺序是怎么注册的呢？ 在`core/dnsserver/server.go`的NewServer方法中，根据配置文件的内容依次注册plugin，其中的registerHandler方法用于注册一个plugin；

```go
// NewServer returns a new CoreDNS server and compiles all plugins in to it. By default CH class
// queries are blocked unless queries from enableChaos are loaded.
func NewServer(addr string, group []*Config) (*Server, error) {

	s := &Server{
		Addr:        addr,
		zones:       make(map[string]*Config),
		connTimeout: 5 * time.Second, // TODO(miek): was configurable
	}

	...

	for _, site := range group {
		if site.Debug {
			s.debug = true
			log.D = true
		}
		// set the config per zone
		s.zones[site.Zone] = site
		// compile custom plugin for everything
		...
		
		var stack plugin.Handler
		for i := len(site.Plugin) - 1; i >= 0; i-- {
			stack = site.Plugin[i](stack)

			// register the *handler* also
			site.registerHandler(stack)

			if s.trace == nil && stack.Name() == "trace" {
				// we have to stash away the plugin, not the
				// Tracer object, because the Tracer won't be initialized yet
				if t, ok := stack.(trace.Trace); ok {
					s.trace = t
				}
			}
			// Unblock CH class queries when any of these plugins are loaded.
			if _, ok := enableChaos[stack.Name()]; ok {
				s.classChaos = true
			}
		}
		site.pluginChain = stack
	}

	return s, nil
}
```


## plugin的代码逻辑
一、首先，每一个plugin需要实现一些CoreDNS规定的必须实现的interface，并且将自己注册到本体中，如下所示，plugin/plugin.go中定义了这些interface，可以看到每个plugin其实是一个对Handler进行操作的func，需要实现ServeDNS()和Name()两个方法；

```go
type (
	// Plugin is a middle layer which represents the traditional
	// idea of plugin: it chains one Handler to the next by being
	// passed the next Handler in the chain.
	Plugin func(Handler) Handler

	// Handler is like dns.Handler except ServeDNS may return an rcode
	// and/or error.
	Handler interface {
		ServeDNS(context.Context, dns.ResponseWriter, *dns.Msg) (int, error)
		Name() string
	}

	// HandlerFunc is a convenience type like dns.HandlerFunc, except
	// ServeDNS returns an rcode and an error. See Handler
	// documentation for more information.
	HandlerFunc func(context.Context, dns.ResponseWriter, *dns.Msg) (int, error)
)
```

二、在上一章节中已经看到，本体会调用plugin chain中的第一个plugin的ServeDNS方法来处理DNS请求，需要注意的是其参数：`dnshandler.pluginChain.ServeDNS(ctx, w, r)`，其中ctx用于记录一些metrics，此处暂时不介绍；r为DNS请求；w是ResponseWriter的实例，其中的WriteMsg方法是在收到了r对应的DNS响应（注意是响应而不是请求）时需要做的操作；

下面以cache plugin为例讲解，假设cache是第一个plugin，那么本体调用其ServeDNS方法，如下所示，逐步解析其中的操作：

首先，cache plugin需要将自己注册到本体，如下所示：

```go
func init() {
	caddy.RegisterPlugin("cache", caddy.Plugin{
		ServerType: "dns",
		Action:     setup,
	})
}
```

cache plugin收到DNS请求时，会去缓存中查找是否记录了对应的item：`i, found := c.get(now, state, server)` ；

```go
// ServeDNS implements the plugin.Handler interface.
func (c *Cache) ServeDNS(ctx context.Context, w dns.ResponseWriter, r *dns.Msg) (int, error) {
	state := request.Request{W: w, Req: r}

	...

	i, found := c.get(now, state, server)
	if i != nil && found {
		resp := i.toMsg(r, now)

		state.SizeAndDo(resp)
		resp, _ = state.Scrub(resp)
		w.WriteMsg(resp)

		...
		
		return dns.RcodeSuccess, nil
	}

	crr := &ResponseWriter{ResponseWriter: w, Cache: c, state: state, server: server}
	return plugin.NextOrFailure(c.Name(), c.Next, ctx, crr, r)
}

```

三、如果没有查询到，那么会新建ResponseWriter的实例，该ResponseWriter是dns.ResponseWriter的扩展，如下所示，使用输入参数w，自身c作为该实例的初始化参数；

```go
// ResponseWriter is a response writer that caches the reply message.
type ResponseWriter struct {
	dns.ResponseWriter
	*Cache
	state  request.Request
	server string // Server handling the request.

	prefetch   bool // When true write nothing back to the client.
	remoteAddr net.Addr
}
```

接着，执行`plugin.NextOrFailure(c.Name(), c.Next, ctx, crr, r)`，NextOrFailure方法的代码如下所示，主要逻辑即为执行下一个plugin的ServeDNS方法；

```go
// NextOrFailure calls next.ServeDNS when next is not nill, otherwise it will return, a ServerFailure and a nil error.
func NextOrFailure(name string, next Handler, ctx context.Context, w dns.ResponseWriter, r *dns.Msg) (int, error) { // nolint: golint
	if next != nil {
		if span := ot.SpanFromContext(ctx); span != nil {
			child := span.Tracer().StartSpan(next.Name(), ot.ChildOf(span.Context()))
			defer child.Finish()
			ctx = ot.ContextWithSpan(ctx, child)
		}
		return next.ServeDNS(ctx, w, r)
	}

	return dns.RcodeServerFailure, Error(name, errors.New("no next plugin found"))
}
```

四、如果`c.get`查询到了对应的response，那么组建响应消息resp并且执行`w.WriteMsg(resp)`，由于w是自己的上一层plugin传递来的参数，因此执行该语句意味着执行上层plugin（或者本体）的WriteMsg方法，也就是说，cache plugin得到了DNS响应，在其之前的每个plugin都会执行WriteMsg(resp)从而对该响应进行一些操作；

例如，在第三步中，cache没有查询到，从而将DNS请求发给下一层plugin，假设在经过了若干层plugin的处理终于得到响应之后，即：

```go
ServeDNS --> ServeDNS --> ... --> ServeDNS --> 
                                             ↓
                                        "get answer"
                                             ↓
"WriteMsg" <--- ... <---"WriteMsg" <--- "WriteMsg"

```

之后，会以该响应resp为参数执行之前每个plugin的WriteMsg方法，因此在某一步cache的下层将resp传给了cache并执行cache.WriteMsg，该方法的代码如下所示，主要逻辑为将res存入cache，并且执行自己上一层的plugin的WriteMsg；

```go
// WriteMsg implements the dns.ResponseWriter interface.
func (w *ResponseWriter) WriteMsg(res *dns.Msg) error {
	
	...

	msgTTL := dnsutil.MinimalTTL(res, mt)
	if msgTTL < duration {
		duration = msgTTL
	}

	if key != -1 && duration > 0 {
		if w.state.Match(res) {
			w.set(res, key, mt, duration)
			cacheSize.WithLabelValues(w.server, Success).Set(float64(w.pcache.Len()))
			cacheSize.WithLabelValues(w.server, Denial).Set(float64(w.ncache.Len()))
		} else {
			// Don't log it, but increment counter
			cacheDrops.WithLabelValues(w.server).Inc()
		}
	}

	...
	
	return w.ResponseWriter.WriteMsg(res)
}
```
