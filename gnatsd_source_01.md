# NATS 开源学习——0X01：代码目录结构

> NATS源码学习系列文章基于[gnatsd1.0.0](https://github.com/nats-io/gnatsd/tree/v1.0.0)。该版本于2017年7月13
> 日发布（[Release v1.0.0](https://github.com/nats-io/gnatsd/releases/tag/v1.0.0)）,在此之前v0.9.6是2016年12月
> 16日发布的,中间隔了半年。算是一个比较完备的版本，但是这个版本还没有增加集群支持。为什么选择这个版本呢？
> 因为一来这个版本比较稳定，同时也包含了集群管理和[Stream](https://github.com/nats-io/nats-streaming-server)
> 落地相关的逻辑，相对完善。

## 目录结构
下载了gnatsd1.0.0的代码后解压，大概是下面这样子：

	gnatsd-1.0.0 cz$ tree -L 1
	.
	├── Dockerfile // docker的镜像文件，现在docker换名后已经不再适用了
	├── LICENSE
	├── README.md
	├── ROADMAP.md
	├── TODO.md
	├── conf			// 配置文件解析
	├── logger		// 日志模块
	├── logos	
	├── main.go		// 程序入口
	├── scripts		// ci和跨平台交叉编译脚本
	├── server		// 服务核心代码（全部代码）
	├── test			// 测试用例
	├── util			// 工具目录
	└── vendor		// 依赖包
	
从上面可以看出，gnats的核心代码主要是在server目录下。一级目录下，只有conf/logger/server/util是包含go代码的。

这里的Dockerfile在现在的v1.4.x里面已经独立出来一个docker目录，文件也变成了新的“Dockerfile.alpine”文件。

Readme和License是一个正式开源项目必不可少的部分。

logos是NATS的logo图片 ![](./images/nats-server.png)

test目录包含了非常详细的测试用例代码，可以覆盖大部分逻辑代码

vendor目录是基于vendor放置的第三方依赖包，如果按照我们前一篇文章[NATS 开源学习——0X00：协议]()中介绍的“go mod”的
方式进行编译，则不需要这个目录了。

## main函数

所以我们就从main函数开始来看代码。

首先在main里面加个日志，然后编译运行一把。这样确保我们可以改代码并正常运行，方便后面打日志分析：

	 68 func main() {
	 69     println("gnatsd code for  cz")
	 70     // Server Options
	 71     opts := &server.Options{}

然后go build & ./gnatsd

	gnatsd-1.0.0 cz$ ./gnatsd
	gnatsd code for  cz
	[63927] 2019/03/20 10:43:24.573036 [INF] Starting nats-server version 1.0.0
	[63927] 2019/03/20 10:43:24.573404 [INF] Listening for client connections on 0.0.0.0:4222
	[63927] 2019/03/20 10:43:24.573415 [INF] Server is ready	 
这里看到日志里面会首先打印我们加的日志。在来看详细的main函数:

	192     // Create the server with appropriate options.
	193     s := server.New(opts)
	194
	195     // Configure the logger based on the flags
	196     s.ConfigureLogger()
	197
	198     // Start things up. Block here until done.
	199     if err := server.Run(s); err != nil {
	200         server.PrintAndDie(err.Error())
	201     }	
	
完整的main见[main.cpp]	(https://github.com/nats-io/gnatsd/blob/v1.0.0/main.go)。这里从其192行开始截出来一段。
在这一行之前都是对命令行参数选项的设置。说白了就是各种命令行参数和配置的设置，待组好好了完整的配置选项后，就开始调用
`s := server.New(opts)`创建一个server.Server对象。

接着通过 `server.Run(s)` 触发server.Server的Start()方法，从而开始启动一个服务。

## 辅助功能

main函数里面用到的server.Server会在后面的篇幅中介绍，主要是在server目录中。上面也看到了，除了这个目录包含go代码以为，还有
几个其他目录，比如conf/util/logger

### 1. conf
gnatsd自己实现了一套类似Yaml和JSON语法的配置文件系统，比较容易理解。但是其实这种自己实现一套配置文件语法，其实也有一定的弊端。
比如总有人觉得你的语法ugly，或者因为对语法不熟悉而配置错了。所以我更愿意去使用一套成熟标记语言来做配置文件，比如Yaml/JSON，当然
对于Golang肯定是首选Toml了。

所以这里我们不去仔细去看他的配置文件的实现，主要的逻辑在

	
	conf cz$ tree .
	.
	├── lex.go
	├── parse.go

这两个文件中，一个用来解析每个条目，基础结构是一个key:value的格式，一个用来解析具体含义。

所以在过代码的时候，直接将这些东西当成Toml的库或者JSON的库，知道能解析得到一个类似Key:Value的一系列键值对就可以了。

在main函数中:

	167     // Parse config if given
	168     if configFile != "" {
	169         fileOpts, err := server.ProcessConfigFile(configFile)
	170         if err != nil {
	171             server.PrintAndDie(err.Error())
	172         }
	173         opts = server.MergeOptions(fileOpts, opts)
	174     }

这里server.ProcessConfigFile在server/opts.go中，为：

	156 // ProcessConfigFile processes a configuration file.
	157 // FIXME(dlc): Hacky
	158 func ProcessConfigFile(configFile string) (*Options, error) {
	159     opts := &Options{ConfigFile: configFile}
	160
	161     if configFile == "" {
	162         return opts, nil
	163     }
	164
	165     m, err := conf.ParseFile(configFile)
	166     if err != nil {
	167         return nil, err
	168     }
	169
	170     for k, v := range m {
	171         switch strings.ToLower(k) {
	172         case "listen":
	173             hp, err := parseListen(v)
	174             if err != nil {
	175                 return nil, err
	176             }
	177             opts.Host = hp.host
	178             opts.Port = hp.port
	...
	
	}
	
这里，就是调用conf里的ParseFile解析一个文件到"map[string]interface{}"中，然后对每个Key做判断，赋值给响应的Options。

### 2. logger
我们上面运行gnatsd的时候，看到日志是直接打印到命令行终端的，这个是通过标准输出打印的。如果配置了日志打印选项，还可以打印到文件：

	-l, --log FILE                   File to redirect log output.
	-T, --logtime                    Timestamp log entries (default is true).
	-s, --syslog                     Enable syslog as log method.
	-r, --remote_syslog              Syslog server address.
	-D, --debug                      Enable debugging output.
	-V, --trace                      Trace the raw protocol.
	-DV                              Debug and Trace.

这里的logger实际上就是对标准输出、Golang提供的[log](https://golang.org/pkg/log/)包、Syslog作了一层封装，比如要打印到终端就是通过：

	 24 // NewStdLogger creates a logger with output directed to Stderr
	 25 func NewStdLogger(time, debug, trace, colors, pid bool) *Logger {

	 ...
	 
	 35
	 36     l := &Logger{
	 37         logger: log.New(os.Stderr, pre, flags),
	 38         debug:  debug,
	 39         trace:  trace,
	 40     }
	 ...
	 }

通过Golang的log包给一个标准输出作为输出目的地。

而存储到文件就是：

	 51 // NewFileLogger creates a logger with output directed to a file
	 52 func NewFileLogger(filename string, time, debug, trace, pid bool) *Logger {
	 53     fileflags := os.O_WRONLY | os.O_APPEND | os.O_CREATE
	 54     f, err := os.OpenFile(filename, fileflags, 0660)
	 55     if err != nil {
	 56         log.Fatalf("error opening file: %v", err)
	 57     }
		...
		
	 63
	 64     pre := ""
	 65     if pid {
	 66         pre = pidPrefix()
	 67     }
	 68
	 69     l := &Logger{
	 70         logger: log.New(f, pre, flags),
	 71         debug:  debug,
	 72         trace:  trace,
	 73     }	 
	 
	 ...
	 
先创建一个文件，然后使用Golang的log包进行日志的存储。

这里封装，主要通过选项来控制了日志输出目的地和日志等级。

### 3. util

在1.0.0这个版本中，utils里面主要是TLS的配置和一个秘密生成工具:

	util cz$ tree .
	.
	├── mkpasswd.go
	├── tls.go
	├── tls_pre17.go
	└── tls_pre18.go
	
这里通过编译选项	:

	  1 // Copyright 2017 Apcera Inc. All rights reserved.
	  2 // +build go1.7,!go1.8
	  3
	  4 package util
 
 来做了Go1.7和Go1.8的区分，因为二者在系统库“crypto/tls"有些不同的地方。
 
 而在现在的1.4.x版本中，都采用了新的tls也就没有在这里做区分，仅留下了一个秘密制作工具：mkpasswd。这个和服务器逻辑没有直接关系，我们也可以
 选择忽略。
 
## 总结
 
这一篇，我们介绍gnatsd代码的目录的大致作用，并从main函数入手，gnatsd程序首先做命令行参数解析，然后读取配置文件，开始运行server.Server的Start
函数启一个服务。对于和MQ逻辑无关的conf/logger/util目录我们大概说了下实现，但是为了聚集，我们不用去看其具体实现，后续篇幅开始解毒这个关键的
server.Server。
	 
	 

