1、gulp任务管理

npm start启动app，执行脚本：

"start": "gulp live"

该命令用启动gulp顺序任务组合live，位于gulpfile.js中，关于gulp相关点击这里

gulp.task('live', gulp.series('clean','lint','bundle:watch','html','css','resources','watch','browser'));

分别执行以下任务：

clean：清空OUTPUT_DIR目录；

lint：将lib目录下的js文件读取到流并进行一些格式化等操作；

bundle:watch：开启文件改变监控与绑定，该任务调用了gulpfile.js里面的bundle()函数，而bundle()函数里面主要是将上一步流中的js文件合并生成了mediasoup-demo-app.js文件，合并后的js文件有13.3M，这是一个大包的过程，便于一次性加载整个应用。

html：复制index.html至OUTPUT_DIR目录下；

css：读取stylus/index.styl文件，转化，重命名为mediasoup-demo-app.css输出到OUTPUT_DIR目录下；

在OUTPUT_DIR下创建resources目录；

watch：监控html, styl, resources...等一系列文件变化

browser：静态文件更新多端同步刷新配置

总结一下，app目录下执行npm start命令后其实就是将相关静态资源复制到server/public目录下，并且开启了文件刷新监控，多端同步，并自动打开浏览器访问访问配置的路径。然后所有的请求处理是由server来处理的，app是个静态资源服务器。

关于app启动时报的这4个参数错误：

Possible race condition: `window.SHOW_INFO` might be reassigned based on an outdated value of `window.SHOW_INFO`

该错误来源在app/lib/index.jsx中的run( )函数

if(info)

window.SHOW_INFO= true;if(throttleSecret)

window.NETWORK_THROTTLE_SECRET= throttleSecret;

然后自己加了几行代码，再启动发现错误越来越多，原来是Eslint代码风格检查的问题，并不会影响程序运行，代码风格规则配置在.eslintrc.js文件中，所以解决方法有3：

按配置的规则修改代码，参考这里，提交代码前先自己用 eslint 命令行检查修复一下对应的文件，例如 eslint file.js --fix(推荐)

干掉引起错误的规则，参考这里

选择忽略这个错误，比RoomClient.js文件报错了，那就在这个文件头部加上一行 /* eslint-disable */ ，表示此文件禁用代码检查警告，参考这里

2、RoomClient类

关于客户端房间的定义在app/lib/RoomClient.js模块中，该模块中引用了 mediasoup-client 模块，定义了RoomClient类，房间的相关属性，websocket事件，以及麦克风，摄像头，共享，聊天等交互操作。

加入房间后建立websocket连接客户端，用的是protoo-client 模块，这是一个专为群聊，会议设计的websocket模块，与一般的websocket client略有不同，除了websocket的open，close等事件之外还有request，response，notifaction事件，相关说明参考官方文档。

而server中用了与之对应的 protoo-server 模块，

3、Redux状态容器

除了上面的 gulp 外，RoomClient使用 Redux状态容器来记录 room 页面的状态参数，参见 Redux中文文档。

Redux的唯一数据源store作为一个全局变量，定义在RoomClient.js中，并且对外暴露了

static init(data)

{
store=data.store;

}

init 函数，作为类的静态成员来初始化store，该函数在app/lib/index.jsx中被调用，将store赋给当前window对象并初始化，这些js文件最后合而为一被用于房间页面的js依赖。

导致store状态改变的action 事件描述定义在 app/lib/redux下，而与之对应的改变store的reducer纯函数定义在 app/lib/reducers目录下，STATE.md正是当前所有状态的展示。

然后在RoomClient中用store.dispatch()函数调用相关action改变store状态树，例如：

store.dispatch(

stateActions.setRoomState('closed'));

4、WebRTC

webrtc才是核心，路漫漫其修远兮

5、屏幕共享的限制

mediasoup-demo client的屏幕捕捉是通过调用浏览器API实现的，只在Chrome和Firefox浏览器中可用。

6、动态页面-React部分

客户端页面涉及到React使用，简单了解一下,React官方文档.

index.jsx中使用：

import React from 'react';

import { render } from'react-dom';

import { Provider } from'react-redux';

render(

,

document.getElementById('mediasoup-demo-app-container')

);

Room.jsx，这是一个自定义的React的class组件，所有页面元素最后都被组合进这个组件中。

class Room extends React.Component

{
render()

{
const {
roomClient,

room,

me,

amActiveSpeaker,

onRoomLinkCopy

}= this.props;return(

...............

...............

componentDidMount()

{
const { roomClient }= this.props;

roomClient.join();

}

}

componentDidMount() 方法会在组件已经被渲染到 DOM 中后运行，类似于wondow.load()，此处在Room组件渲染完成后执行join()加入Room方法。

7、页面样式-Stylus

Room页面样式用 Stylus语法指定，静态文件位于 app/stylus/ 目录下，Room.styl中定义了自己的样式，Peers.styl中定义了参会者的样式，了解一下stylus语法后可按需求修改。

8、app启动失败解决方案

app在启动时可能出现如下错误，原因是watch文件超过系统允许配置导致的

Error: ENOSPC:System limit for number of file watchers reached

解决方案 这里。

9、协商应答的SDP信息

通过浏览器端输出日志可以发现，协商应答的sdp信息是在该函数中打印的

app/node_modules/mediasoup-client/lib/handlers/Chrome70.js/class SendHandler/stopSending()

async stopSending({ localId })

{
logger.debug('stopSending() [localId:%s]', localId);

const transceiver= this._mapMidTransceiver.get(localId);if (!transceiver)throw new Error('associated RTCRtpTransceiver not found');

transceiver.sender.replaceTrack(null);this._pc.removeTrack(transceiver.sender);this._remoteSdp.closeMediaSection(transceiver.mid);

const offer= await this._pc.createOffer();

logger.debug('stopSending() | calling pc.setLocalDescription() [offer:%o]', offer);

awaitthis._pc.setLocalDescription(offer);

const answer= { type: 'answer', sdp: this._remoteSdp.getSdp() };

logger.debug('stopSending() | calling pc.setRemoteDescription() [answer:%o]', answer);

awaitthis._pc.setRemoteDescription(answer);

}

这个sdp信息很关键，offer的太长了，应答sdp内容如下：

v=0o=mediasoup-client 10000 4 IN IP4 0.0.0.0s=-t=0 0a=ice-lite

a=fingerprint:sha-512 AA:99:03:C0:4E:DB:D6:BC:03:51:37:EF:40:00:09:34:99:43:71:CB:76:E8:CC:9E:3E:22:6F:BF:1E:44:0A:31:90:EE:ED:0F:E7:33:42:EF:0D:1E:F4:A4:04:67:4D:22:49:45:6C:8E:3D:FF:EA:6D:5C:07:D3:F4:E5:DF:BE:08a=msid-semantic: WMS *a=group:BUNDLE 0 1m=audio 7 UDP/TLS/RTP/SAVPF 111

c=IN IP4 127.0.0.1a=rtpmap:111 opus/48000/2a=fmtp:111 stereo=1;usedtx=1a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:mid

a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level

a=setup:active

a=mid:0a=recvonly

a=ice-ufrag:i5dt0ipouvpj58p6

a=ice-pwd:nqqq5j2hn4e4mtif07ffa4fiox7pby7g

a=candidate:udpcandidate 1 udp 1076302079 192.168.189.128 41774typ host

a=end-of-candidates

a=ice-options:renomination

a=rtcp-mux

a=rtcp-rsize

m=application 7 DTLS/SCTP 5000

c=IN IP4 127.0.0.1a=setup:active

a=mid:1a=ice-ufrag:i5dt0ipouvpj58p6

a=ice-pwd:nqqq5j2hn4e4mtif07ffa4fiox7pby7g

a=candidate:udpcandidate 1 udp 1076302079 192.168.189.128 41774typ host

a=end-of-candidates

a=ice-options:renomination

a=sctpmap:5000 webrtc-datachannel 262144m=video 0 UDP/TLS/RTP/SAVPF 96 97

c=IN IP4 127.0.0.1a=rtpmap:96 VP8/90000

a=rtpmap:97 rtx/90000

a=fmtp:96 x-google-start-bitrate=1000a=fmtp:97 apt=96a=rtcp-fb:96 goog-remb

a=rtcp-fb:96ccm fir

a=rtcp-fb:96nack

a=rtcp-fb:96nack pli

a=setup:active

a=mid:2a=inactive

a=ice-ufrag:i5dt0ipouvpj58p6

a=ice-pwd:nqqq5j2hn4e4mtif07ffa4fiox7pby7g

a=candidate:udpcandidate 1 udp 1076302079 192.168.189.128 41774typ host

a=end-of-candidates

a=ice-options:renomination

a=rtcp-mux

a=rtcp-rsize