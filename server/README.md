1、启动server

npm start启动服务，会执行脚本：

"start": "DEBUG=${DEBUG:='*mediasoup* *INFO* *WARN* *ERROR*'} INTERACTIVE=${INTERACTIVE:='true'} node server.js"
该命令设置了DEBUG环境变量，并启动server.js

2、SFU架构

参考：Android WebRTC完整入门教程04: 多人视频

多人视频有三种理论方案, 如下图所示, 从左到右分别是Mesh,SFU,MCU.



 SFU(Selective Forwarding Unit) 可选择转发单元, 有一个中心单元, 负责转发流. 每个人只跟中心单元建立一个连接, 上传自己的流, 并下载别人的流. 4个人的情况下, 每个人建立一个连接, 包括1个上传流和3个下载流. 此方案对客户端要求较高, 对服务端要求较高.

3、mediasoup相关角色

参考：官方文档

Producer：一个生产者表示将通过WebRTC传输被发送到mediasoup路由的一个音频或视频源，它是在定义媒体包传输方式之上创建的。

Consumer：一个消费者表示通过WebRTC传输正在从mediasoup路由被发送到客户端应用程序的音频或视频远程源，它是在定义媒体包传输方式之上创建的。

DataProducer：数据生产者表示将通过WebRTC传输发送到mediasoup路由器的数据源，它是在定义数据消息（SCTP）的传输方式之上创建的。

DataConsumer：数据生产者表示通过WebRTC传输从Mediasoup路由器传输到客户端应用程序的数据源。它是在定义数据消息（SCTP）的传输方式之上创建的。

4、server启动过程

npm start运行server.js文件时做了6件事

4.1、启动server的命令行交互服务，可以在server启动后再监听用户命令行输入并处理，定义的命令较多

4.2、启动client命令行交互服务，很少的几个命令

4.3、mediasoup核心C++ SFU服务以子进程的方式被nodejs加载启动，这里启动了2个子进程，用来执行相同的任务

[root@jxh server]# npm start

> mediasoup-demo-server@3.0.0 start /usr/local/src/node/mediasoup-demo/server
> DEBUG=${DEBUG:='*mediasoup* *INFO* *WARN* *ERROR*'} INTERACTIVE=${INTERACTIVE:='true'} node server.js

- config.mediasoup.numWorkers: 2
- process.env.DEBUG: *mediasoup* *INFO* *WARN* *ERROR*
- config.mediasoup.workerSettings.logLevel: warn
- config.mediasoup.workerSettings.logTags: [ 'info',
  'ice',
  'dtls',
  'rtp',
  'srtp',
  'rtcp',
  'rtx',
  'bwe',
  'score',
  'simulcast',
  'svc',
  'sctp' ]
  mediasoup-demo-server:INFO running 2 mediasoup Workers... +0ms
  mediasoup createWorker() +0ms
  mediasoup:Worker constructor() +0ms
  mediasoup:Worker spawning worker process: /usr/local/src/node/mediasoup-demo/server/node_modules/mediasoup/worker/out/Release/mediasoup-worker --logLevel=warn --logTag=info --logTag=ice --logTag=dtls --logTag=rtp --logTag=srtp --logTag=rtcp --logTag=rtx --logTag=bwe --logTag=score --logTag=simulcast --logTag=svc --logTag=sctp --rtcMinPort=40000 --rtcMaxPort=49999 +0ms
  mediasoup:Channel[pid:8092] constructor() +0ms

[opening Readline Command Console...]
type help to print available commands
cmd>   mediasoup:Worker worker process running [pid:8092] +24ms
  mediasoup createWorker() +26ms
  mediasoup:Worker constructor() +1ms
  mediasoup:Worker spawning worker process: /usr/local/src/node/mediasoup-demo/server/node_modules/mediasoup/worker/out/Release/mediasoup-worker --logLevel=warn --logTag=info --logTag=ice --logTag=dtls --logTag=rtp --logTag=srtp --logTag=rtcp --logTag=rtx --logTag=bwe --logTag=score --logTag=simulcast --logTag=svc --logTag=sctp --rtcMinPort=40000 --rtcMaxPort=49999 +0ms
  mediasoup:Channel[pid:8094] constructor() +0ms
  mediasoup:Worker worker process running [pid:8094] +63ms
  mediasoup-demo-server:INFO creating Express app... +91ms
  mediasoup-demo-server:INFO running an HTTPS server... +3ms
  mediasoup-demo-server:INFO running protoo WebSocketServer... +12ms
查看linux运行进程

root       8074   2823  0 16:56 pts/0    00:00:00 npm
root       8085   8074  0 16:56 pts/0    00:00:00 mediasoup-demo
root       8092   8085  0 16:56 pts/0    00:00:00 /usr/local/src/node/mediasou
root       8094   8085  0 16:56 pts/0    00:00:00 /usr/local/src/node/mediasou
对应上面启动日志中的PID为8092和8094两个进程，其PPID（父进程PID）都为8005，也就是mediasoup-demo主进程。

对应的启动文件为



该文件根据其上一级的Makefile文件在npm install时编译生成

启动的子进程数由server/config.js配置文件指定

numWorkers     : Object.keys(os.cpus()).length,
也就是说它是由当前机器的CPU核数决定的。



 我的虚拟机是2核的，因此程序启动了2个相同的子进程，对应代码位于server.js

async function runMediasoupWorkers()
{
    const { numWorkers } = config.mediasoup;

    logger.info('running %d mediasoup Workers...', numWorkers);

    for (let i = 0; i < numWorkers; ++i)
    {
        // mediasoup.createWorker(settings)：使用给定的设置创建一个新的工作进程
        const worker = await mediasoup.createWorker(
            {
                logLevel   : config.mediasoup.workerSettings.logLevel,
                logTags    : config.mediasoup.workerSettings.logTags,
                rtcMinPort : config.mediasoup.workerSettings.rtcMinPort,
                rtcMaxPort : config.mediasoup.workerSettings.rtcMaxPort
            });

        worker.on('died', () =>
        {
            logger.error(
                'mediasoup Worker died, exiting  in 2 seconds... [pid:%d]', worker.pid);

            setTimeout(() => process.exit(1), 2000);
        });

        mediasoupWorkers.push(worker);
    }
}
多个worker之间的架构图如下（来源于官网）

 

4.4、创建expressapp实例，server用的是express框架，定义了对http请求的相关路由处理

4.5、启动https server，启用证书，监听配置文件定义的端口

4.6、启动websocket server（protoo server）

4.7、启动每隔30s检测room状态的定时器