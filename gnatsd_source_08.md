# NATS 开源学习——0X08：Router转发

> NATS源码学习系列文章基于[gnatsd1.0.0](https://github.com/nats-io/gnatsd/tree/v1.0.0)。该版本于2017年7月13
> 日发布（[Release v1.0.0](https://github.com/nats-io/gnatsd/releases/tag/v1.0.0)）,在此之前v0.9.6是2016年12月
> 16日发布的,中间隔了半年。算是一个比较完备的版本，但是这个版本还没有增加集群支持。为什么选择这个版本呢？
> 因为一来这个版本比较稳定，同时也包含了集群管理和[Stream](https://github.com/nats-io/nats-streaming-server)
> 落地相关的逻辑，相对完善。

每篇的引用里面都有说NATS支持集群，那么是怎么支持的呢？我们来看[配置文档](https://nats.io/documentation/managing_the_server/configuration/)

里面有：

	# Cluster definition
	cluster {
	
	  listen: localhost:4244 # host/port for inbound route connections
	
	  # Authorization for route connections
	  authorization {
	    user: route_user
	    # ./util/mkpasswd -p T0pS3cr3tT00!
	        password: $2a$11$xH8dkGrty1cBNtZjhPeWJewu/YPbSU.rXJWmS6SFilOBXzmZoMk9m
	    timeout: 0.5
	  }
	
	  # Routes are actively solicited and connected to from this server.
	  # Other servers can connect to us if they supply the correct credentials
	  # in their routes definitions from above.
	  routes = [
	    nats-route://user1:pass1@127.0.0.1:4245
	    nats-route://user2:pass2@127.0.0.1:4246
	  ]
	}

这里看到，首先gnatsd需要新监听一个IP:Port : listen 字段。然后还指定了 routes。这里的routes是这个gnatsd可以连的服务器，而listen是这个gnatsd可以被
其他服务器连的地址。因此就产生了这样的结构：

![](./images/router.png)

图中是个环行结构，其实可以自由组合，本质就是一个gnatsd连到另一个gnatsd上。这样，即使client_1连在gnatsd_3上面，也可以将消息发布到
连在gnatsd_1上的client_2上面，只要他们订阅的消息主题是一致的。

## 作为服务器端
因为gnatsd在这个时候既要接受其他gnatsd的链接，也要链接其他gnatsd，所以我们先来看作为服务器时，如何处理链接。

服务器回起一个接受gnatsd来链接的goroutine:

	601 func (s *Server) routeAcceptLoop(ch chan struct{}) {
	...
	658     for s.isRunning() {
	659         conn, err := l.Accept()
	...
	675         s.startGoRoutine(func() {
	676             s.createRoute(conn, nil)
	677             s.grWG.Done()
	678         })
	679     }	

这里创建Route的实现为：

	312 func (s *Server) createRoute(conn net.Conn, rURL *url.URL) *client {
	...
	r := &route{didSolicit: didSolicit}
	...
	c := &client{srv: s, nc: conn, opts: clientOpts{}, typ: ROUTER, route: r}
	...
	336     // Initialize
	337     c.initClient()	
	...
	409     // Spin up the read loop.
	410     s.startGoRoutine(func() { c.readLoop() })	...
	424     // Send our info to the other side.
	425     c.sendInfo(infoJSON)	
这里可以看到和处理客户端连接是一样的。也是先创建一个client，然后初始化，然后起client的readLoop goroutine，最后发送INFO给客户端。


## 作为客户端

作为客户端时，首先要连接其他Router:

	 57 // Lock should be held entering here.
	 58 func (c *client) sendConnect(tlsRequired bool) {
	 ...
	 78     c.sendProto([]byte(fmt.Sprintf(ConProto, b)), true)
	 79 }

这里直接组合一个CONNECT协议，进行发送。

我们在[NATS 开源学习——0X04：协议解析]()有说到在gnatsd解析INFO的实际上是调用了"server/router.go"里面的：
	
	 81 // Process the info message if we are a route.
	 82 func (c *client) processRouteInfo(info *Info) {
 
这里会记录其他Router的信息，用于后面的转发。

## 转发消息

### 订阅
在gnatsd处理客户端的SUB的消息里面有：

	 750 func (c *client) processSub(argo []byte) (err error) {
	 751     c.traceInOp("SUB", argo)
	 ...
	 812     if shouldForward {
	 813         c.srv.broadcastSubscribe(sub)
	 814     }

这里如果有连接到其他router，就要进行srv.broadcastSubscribe,其会调用broadcastInterestToRoutes在server/router.go里面：

	556 func (s *Server) broadcastInterestToRoutes(proto string) {

	561     protoAsBytes := []byte(proto)
	562     s.mu.Lock()
	563     for _, route := range s.routes {

	566         route.sendProto(protoAsBytes, true)

	569     }
	571 }
	
这里就是向各个Router发送了SUB。其他Router收到这个SUB后，也会在自己的SubList里面构建相关的subscription。

### 发布
在gnatsd处理客户端的PUB的消息里面有：

	1012 // processMsg is called to process an inbound msg from a client.
	1013 func (c *client) processMsg(msg []byte) {
	...
	1146     for _, sub := range r.psubs {
	1147         // Check if this is a send to a ROUTER, make sure we only send it
	1148         // once. The other side will handle the appropriate re-processing
	1149         // and fan-out. Also enforce 1-Hop semantics, so no routing to another.
	1150         if sub.client.typ == ROUTER {	...
	1176         mh := c.msgHeader(msgh[:si], sub)
	1177         c.deliverMsg(sub, mh, msg)
这里如果是要转发给router,过滤下不重复发送。也就是消息会直接转发给其他连接到自己的router 客户端。

### 转发

上面转发给其他router时，实际上是MSG协议，到目前我们还没有分析过对MSG协议的处理，我们来看server/client.go里面：

	 623 func (c *client) processMsgArgs(arg []byte) error {
	 624     if c.trace {
	 625         c.traceInOp("MSG", arg)
	 626     }

是用来处理MSG协议，其解析和其他的一样。这里面的逻辑其实和processPub是一样的，就是解析参数，然后做opt赋值，结束。然后等着他后面处理
MSG协议里面的payload。也是调用处理PUB一样的processMsg函数。这就会到上面的*发布*里面的代码了。就是给自己的 r.psubs里面注册的客户端
进行”c.deliverMsg(sub, mh, msg)“ MSG发送。


## 总结
这里来捋一下。首先client_2注册SUB时，gnatsd_1会调用broadcastInterestToRoutes给gnatsd_3发SUB，此时gnatsd_3的SubList里面就记录了客户端gantsd_1
订阅了这个主题。当client1发送订阅SUB时，gnatsd_3会给gnatsd_1发送MSG，而gnatsd_1会找出自己SubList里面订阅了这个消息的客户端，并给他发送MSG，也就是
会给client_2发送gnatsd_3传过来的MSG，既client_1发送的PUB。




