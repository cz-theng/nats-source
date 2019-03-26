# NATS 开源学习——0X07：发布消息

> NATS源码学习系列文章基于[gnatsd1.0.0](https://github.com/nats-io/gnatsd/tree/v1.0.0)。该版本于2017年7月13
> 日发布（[Release v1.0.0](https://github.com/nats-io/gnatsd/releases/tag/v1.0.0)）,在此之前v0.9.6是2016年12月
> 16日发布的,中间隔了半年。算是一个比较完备的版本，但是这个版本还没有增加集群支持。为什么选择这个版本呢？
> 因为一来这个版本比较稳定，同时也包含了集群管理和[Stream](https://github.com/nats-io/nats-streaming-server)
> 落地相关的逻辑，相对完善。

当其他客户端订阅完消息后，就可以进行消息发布了。看过前面的[NATS 开源学习——0X05：订阅消息]()就会知道怎么样去解析一个订阅消息了，
发布消息也是一样。

## 协议解析
一条PUB协议长的类似下面这样：

	PUB <subject> [reply-to] <#bytes>\r\n[payload]\r\n

和其他协议一样，首先解析出PUB，然后将后面的部分存入到argbuf中。

不过稍微注意点的同学可能会发现，这里有两个换行"\r\n",那么按照之前的解析，应该只能到<#bytes>这里，而后面的[payload]则没有解析。

是的，在解析PUB协议的时候，这里会将状态置为MSG_PAYLOAD，然后再接着往后读取，知道下一个换行。并将内容存储在c.msgBuf中：

	189         case MSG_PAYLOAD:
	190             if c.msgBuf != nil {
	...
	200                     c.msgBuf = c.msgBuf[:start+toCopy]
	201                     copy(c.msgBuf[start:], buf[i:i+toCopy])
	202                     // Update our index
	203                     i = (i + toCopy) - 1
	204                 } else {
	205                     // Fall back to append if needed.
	206                     c.msgBuf = append(c.msgBuf, b)
	207                 }
   ...
   
同样在解析完上面的参数也就是PUB_ARG状态是会调用c.processPub，而下面的消息读完后会调用c.processMsg处理消息。

## 处理参数

参数处理在server/client.go里面的：

	 672 func (c *client) processPub(arg []byte) error {
函数中，这里这个版本写的有问题，其实应该和SUB一样调用一下SplitArg，但是这里是把这个函数的代码重写了：

	 698     switch len(args) {
	 699     case 2:
	 700         c.pa.subject = args[0]
	 701         c.pa.reply = nil
	 702         c.pa.size = parseSize(args[1])
	 703         c.pa.szb = args[1]
	 704     case 3:
	 705         c.pa.subject = args[0]
	 706         c.pa.reply = args[1]
	 707         c.pa.size = parseSize(args[2])
	 708         c.pa.szb = args[2]	 

split完后，因为有可选参数，所以也是做个兼容处理。

得到参数后，就对参数进行合法校验，因为这里已经有了发布的主题和发布内容的长度：

	 715     maxPayload := atomic.LoadInt64(&c.mpay)
	 716     if maxPayload > 0 && int64(c.pa.size) > maxPayload {
	 717         c.maxPayloadViolation(c.pa.size, maxPayload)
	 718         return ErrMaxPayload
	 719     }
	 720
	 721     if c.opts.Pedantic && !IsValidLiteralSubject(string(c.pa.subject)) {
	 722         c.sendErr("Invalid Subject")
	 723     }
	 
超长的非法的主题都返回失败。	 

## 处理消息
处理完参数后，开始处理发布的消息内容，代码在server/client.go的：

	1013 func (c *client) processMsg(msg []byte) {

这里首先会禁止发送"_SYS.>"	的主题的消息，然后做权限检查：

	1034     // Check if published subject is allowed if we have permissions in place.
	1035     if c.perms != nil {
	1036         allowed, ok := c.perms.pcache[string(c.pa.subject)]
	
检查逻辑和SUB里面的类似。

然后会优先去到订阅cache列表中，查看是否有缓存，如果没有则去svr.sl里面按照主题进行匹配：

	1083     if genid == c.cache.genid && c.cache.results != nil {
	1084         r, ok = c.cache.results[string(c.pa.subject)]
	1085     } else {
	1086         // reset
	1087         c.cache.results = make(map[string]*SublistResult)
	1088         c.cache.genid = genid
	1089     }
	1090
	1091     if !ok {
	1092         subject := string(c.pa.subject)
	1093         r = srv.sl.Match(subject)
	1094         c.cache.results[subject] = r

如果找到的话，同时也更新到cache中。然后准备对这些搜寻到的客户端进行消息转发

## 消息转发
消息转发是通过协议MSG进行的，MSG消息大概是这样：

	MSG <subject> <sid> [reply-to] <#bytes>\r\n[payload]\r\n

所以转发前先多消息进行组合：

	1118     // Scratch buffer..
	1119     msgh := c.msgb[:len(msgHeadProto)]
	1120
	1121     // msg header
	1122     msgh = append(msgh, c.pa.subject...)
	1123     msgh = append(msgh, ' ')
	
这里组合一个消息头，也就是第一个换行"\r\n"前面的内容。

然后依次对每个客户端进行消息发送：

	1146     for _, sub := range r.psubs {
	...
	1175         // Normal delivery
	1176         mh := c.msgHeader(msgh[:si], sub)
	1177         c.deliverMsg(sub, mh, msg)
	1178     }
	
这里	deliverMsg为：

	911 func (c *client) deliverMsg(sub *subscription, mh, msg []byte) {
	
	...
	 973     // Deliver to the client.
	 974     _, err := client.bw.Write(mh)
	 975     if err != nil {
	 976         goto writeErr
	 977     }
	 978
	 979     _, err = client.bw.Write(msg)
	 980     if err != nil {
	 981         goto writeErr
	 982     }
	...

这里的 bw是：

	  87 type client struct {
	  99     bw    *bufio.Writer

也就是bufio.Writer进行写数据。

看到这里，可能有同学会联想到前面的时序图，那就是gnatsd在处理客户端是，读入消息，然后处理，然后向多个客户端进行写，万一某个客户端，在Write
的时候塞住了，那么对这个客户端后续的消息处理就可能会延迟。

是的，在这个版本中，处理逻辑确实是这样。

## 总结
发布消息的过程，其实就是接受客户端发布的消息内容，然后根据主题找到订阅了这个主题的所有客户端，根据需要将消息写往这些客户端。
在这个版本（v1.0.0)写操作是直接调用的io的flush，而在v1.4.x里面，这里做了一个缓存，有另外一个专门的goroutine进行flush操作。


 	 

	 
	 

