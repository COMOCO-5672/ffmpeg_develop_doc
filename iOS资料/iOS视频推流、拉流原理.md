# iOS视频推流、拉流原理

## 1、HLS协议：

简单讲就是把整个流分成一个个小的，基于 HTTP 的文件来下载，每次只下载一些，前面提到了用于 H5 播放直播视频时引入的一个 .m3u8 的文件，这个文件就是基于 HLS 协议，存放视频流元数据的文件。

每一个 .m3u8 文件，分别对应若干个 ts 文件，这些 ts 文件才是真正存放视频的数据，m3u8 文件只是存放了一些 ts 文件的配置信息和相关路径，当视频播放时，.m3u8 是动态改变的，video 标签会解析这个文件，并找到对应的 ts 文件来播放，所以一般为了加快速度，.m3u8 放在 web 服务器上，ts 文件放在 cdn 上。

.m3u8 文件，其实就是以 UTF-8 编码的 m3u 文件，这个文件本身不能播放，只是存放了播放信息的文本文件：

```
#EXTM3U                 m3u文件头#EXT-X-MEDIA-SEQUENCE   第一个TS分片的序列号#EXT-X-TARGETDURATION   每个分片TS的最大的时长#EXT-X-ALLOW-CACHE      是否允许cache#EXT-X-ENDLIST          m3u8文件结束符#EXTINF                 指定每个媒体段(ts)的持续时间（秒），仅对其后面的URI有效mystream-12.ts
```

## 2、HLS 的请求流程是：

1.http 请求 m3u8 的 url。
2.服务端返回一个 m3u8 的播放列表，这个播放列表是实时更新的，一般一次给出5段数据的 url。
3.客户端解析 m3u8 的播放列表，再按序请求每一段的 url，获取 ts 数据流。

### 2.1 关于HLS延迟原因：

hls 协议是将直播流分成一段一段的小段视频去下载播放的，所以假设列表里面的包含5个 ts 文件，每个 TS 文件包含5秒的视频内容，那么整体的延迟就是25秒。因为当你看到这些视频时，主播已经将视频录制好上传上去了，所以时这样产生的延迟。当然可以缩短列表的长度和单个 ts 文件的大小来降低延迟，极致来说可以缩减列表长度为1，并且 ts 的时长为1s，但是这样会造成请求次数增加，增大服务器压力，当网速慢时回造成更多的缓冲，所以苹果官方推荐的ts时长时10s，所以这样就会大改有30s的延迟;

### 2.2 数据采集原理：

下面将利用 ios 上的摄像头，进行音视频的数据采集，主要分为以下几个步骤：

- 音视频的采集，ios 中，利用 AVCaptureSession和AVCaptureDevice 可以采集到原始的音视频数据流。
- 对视频进行 H264 编码，对音频进行 AAC 编码，在 ios 中分别有已经封装好的编码库来实现对音视频的编码。
- 对编码后的音、视频数据进行组装封包；
- 建立 RTMP 连接并上推到服务端。
  ps：由于编码库大多使用 c 语言编写，需要自己使用时编译，对于 ios，可以使用已经编译好的编码库。

## 3、RTMP介绍：

Real Time Messaging Protocol（简称 RTMP）是 Macromedia 开发的一套视频直播协议。和 HLS 一样都可以应用于视频直播，区别是 RTMP 基于 flash 无法在 ios 的浏览器里播放，但是实时性比 HLS 要好。所以一般使用这种协议来上传视频流，也就是视频流推送到服务器。
对比：

- RTMP 首先就是延迟低，基于TCP的长链接，对于数据处理及时，收到即刻发送，推荐使用场景：即时互动。
- HLS 延迟高，短链接，原理是集合了一段时间的视频数据，切割ts片，逐个下载播放。优点是跨平台。

### 3.1 推流：

推流，就是将我们已经编码好的音视频数据发往视频流服务器中，一般常用的是使用 rtmp 推流，可以使用第三方库 librtmp-iOS 进行推流，librtmp 封装了一些核心的 api 供使用者调用，如果觉得麻烦，可以使用现成的 ios 视频推流sdk，也是基于 rtmp 的。具体说下：
也就是对编码好的音视频数据推到服务器上，这里我们又分为两类推流模式：手机端推流，服务器本地推流。就拿我上一家公司的电视直播来说，视频源是来自电视台的，需要通过ffmpeg命令来进行个推流，那么推流协议的话这里又分为了：HLS推流和rtmp推流，这里的取舍主要涉及到了是否需要及其实时的直播问题，也就是延迟20 30s是否接受，当然电视直播并不是主播实时互动，所以不需要使用实时流媒体协议的rtmp，所以通过ffmpeg -loglevel 这么一个命令将电视台给的视频进行各像nginx服务器的一个推流，那么我们就可以通过nginx服务器给的链接，配合我的第三方的直播框架，就可以实现个直播，这个是服务器本地的HLS协议的一个推流。当然如果我们要做一个没有延迟的比如实现各主播互动的一个直播，那么就是iOS客户端用rtmp协议的一个往nginx服务器的一个推流了。在iOS设备上进行各推流的话，是通过AVCaptureSession这么一个捕捉会话，指定两个AVCaptureDevice 也就是iOS的摄像头和麦克风，获取个原始视频和音频，然后需要进行个H.264的视频编码和AAC的音频编码，再将编码后的数据整合成一个音视频包，通过rmtp推送到nginx服务器。这里这些步骤，我们可以通过各第三方集成好的推流工具进行推流，这个工具有librtmp，和腾讯的GDLiveStreaming进行个推流。



原文链接：https://mp.weixin.qq.com/s?src=11&timestamp=1678859405&ver=4407&signature=kQhf-SSTbagt8wSRLirZVEAn4Z7le*p*PSZzTVEyG5DIHR0aY2ntdmVcLf9R*b2hcV04rJL07lwRtcW6hl9pNNHgl59TGPjSUu4My9Yo-gFiWJJUIoJlhv4i8Mll6W7e&new=1