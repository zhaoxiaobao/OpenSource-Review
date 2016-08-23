# LFLiveKit-Review  

> 之前项目用到LFLiveKit，现在有空来写篇分析源码的文章。

## LFLiveKit架构图解
LFLiveKit工程文件已经比较清晰了，整理如下：
<p align="center">
  <img src="https://raw.githubusercontent.com/zhaoxiaobao/OpenSource-Review/master/Projects/Images/LFLiveKit.png" alt="img"/>
</p>

> 啧啧啧，这个图基本上就是来疯kit工程的目录图，个人觉得还是很👍的，你们可以follow这位作者[chenliming777](https://github.com/chenliming777)。往下🐱下，咱进入正题。统计一下LFLiveKit的代码行，在3664行左右。统计脚本如下：

```ruby
find . "(" -name "*.m" -or -name "*.mm" -or -name "*.cpp" -or -name "*.h" -or -name "*.rss" ")" -print | xargs wc -l
```

## 源码分析
### 从这初始化方法说起  

``` ruby

_session = [[LFLiveSession alloc] initWithAudioConfiguration:[LFLiveAudioConfiguration defaultConfiguration]
                                          videoConfiguration:[LFLiveVideoConfiguration defaultConfigurationForQuality:LFLiveVideoQuality_Medium2 landscape:NO]];

```
当创建这个session时候，做了这几件事情：
   1. 初始化音频配置

     ```ruby
     /// 默认音频配置
     + (instancetype)defaultConfiguration;
     +(instancetype)defaultConfigurationForQuality:(LFLiveAudioQuality)audioQuality;
     ```

   2. 初始化视频配置

   ```ruby
   /// 默认视频配置
    + (instancetype)defaultConfiguration;
   /// 视频配置(质量)
   +(instancetype)defaultConfigurationForQuality:(LFLiveVideoQuality)videoQuality;
   /// 视频配置(质量 & 是否是横屏)
   +(instancetype)defaultConfigurationForQuality:(LFLiveVideoQuality)videoQuality landscape:(BOOL)landscape;
   ```

   初始化session时候需要配置相关音视频参数，接下来我们来分析看看LFLiveKit是如何进行音视频采集的。

### 音视频采集
LFLiveKit音频采集是通过系统AVAudioUnit进行音频采集的，LFLiveKit对音频采集没有做什么特别的处理，之前遇到降噪算法处理，后来google之，发现iphone系统本身就对音频进行降噪处理。如果后期加入美声，变声之类对音频处理需要在LFAudioCapture中完善ing。如下贴出采集代码:

```ruby  
AVAudioSession *session = [AVAudioSession sharedInstance];
....
....
AudioComponentDescription acd;
  acd.componentType = kAudioUnitType_Output;
  acd.componentSubType = kAudioUnitSubType_RemoteIO;
  acd.componentManufacturer = kAudioUnitManufacturer_Apple;
  acd.componentFlags = 0;
  acd.componentFlagsMask = 0;
  self.component = AudioComponentFindNext(NULL, &acd);


```
> 这里介绍一下AVAudioSession，来自官方的解释是An audio session is a singleton object that you employ to set the audio context for your app and to express to the system your intentions for your app’s audio behavior. 就是说告诉系统应用程序使用音频的情况。这里可能就会涉及音频被其他程序占用的情况，比如有微信视频聊天抢占音频，音频恢复不过来。作者通过这两行代码来解决：

```
[[AVAudioSession sharedInstance] setCategory:AVAudioSessionCategoryPlayAndRecord error:nil];
[[AVAudioSession sharedInstance] setActive:YES error:nil];
```
LFLiveKit音频通过AURenderCallbackStruct进行回调处理采集音频数据，这里取出回调函数的参数
```ruby
static OSStatus handleInputBuffer(void *inRefCon,
                                  AudioUnitRenderActionFlags *ioActionFlags,
                                  const AudioTimeStamp *inTimeStamp,
                                  UInt32 inBusNumber,
                                  UInt32 inNumberFrames,
                                  AudioBufferList *ioData) {
                                    ......
                                    ......
                                    ..
                                  }

```
> 在handleInputBuffer通过[source.delegate captureOutput:source audioBuffer:buffers]代理，在LFLiveSession中将音频进行编码。


LFLiveKit视频采集是通过GPUImageVideoCamera进行视频采集的，美颜滤镜功能也是基于GPUImage，具体美颜算法还待深究。

```ruby
_videoCamera = [[GPUImageVideoCamera alloc] initWithSessionPreset:_configuration.avSessionPreset cameraPosition:AVCaptureDevicePositionFront];

```
加上filler通过GPUImageOut进行回调，这里涉及两个参数一个是output，一个是时间戳。

```ruby
#pragma mark -- Custom Method
- (void) processVideo:(GPUImageOutput *)output{
    __weak typeof(self) _self = self;
    @autoreleasepool {
        GPUImageFramebuffer *imageFramebuffer = output.framebufferForOutput;
        //CVPixelBufferRef pixelBuffer = [imageFramebuffer pixelBuffer];

        size_t width = imageFramebuffer.size.width;
        size_t height = imageFramebuffer.size.height;
        ///< 这里可能会影响性能，以后要尝试修改GPUImage源码 直接获取CVPixelBufferRef 目前是获取的bytes 其实更麻烦了
        if(imageFramebuffer.size.width == 360){
            width = 368;///< 必须被16整除
        }

        CVPixelBufferRef pixelBuffer = NULL;
        CVPixelBufferCreateWithBytes(kCFAllocatorDefault, width, height, kCVPixelFormatType_32BGRA, [imageFramebuffer byteBuffer], width * 4, nil, NULL, NULL, &pixelBuffer);

        if(pixelBuffer && _self.delegate && [_self.delegate respondsToSelector:@selector(captureOutput:pixelBuffer:)]){
            [_self.delegate captureOutput:_self pixelBuffer:pixelBuffer];
        }
        CVPixelBufferRelease(pixelBuffer);

    }
}

```
> 这里边需要特别关注一下CVPixelBufferRef全称是Core Video pixel buffer，其实说白了就是缓存的一帧图片。这是官方的解释A Core Video pixel buffer is an image buffer that holds pixels in main memory. Applications generating frames, compressing or decompressing video, or using Core Image can all make use of Core Video pixel buffers. 之前没接触过，所以mark下。

数据采集完毕之后通过CaptureDelegate代理在LFLiveSession中将缓存图片进行编码，编码涉及两重要的参数，一个pixelBuffer，一个timeStamp，其实就是采集时候那两个参数，接下来介绍编码。

### 音视频编码
如下在LFLiveSession两个代理方法：
```ruby
- (void)captureOutput:(nullable LFAudioCapture*)capture audioBuffer:(AudioBufferList)inBufferList{
    [self.audioEncoder encodeAudioData:inBufferList timeStamp:self.currentTimestamp];
}
- (void)captureOutput:(nullable LFVideoCapture*)capture pixelBuffer:(nullable CVImageBufferRef)pixelBuffer{
    [self.videoEncoder encodeVideoData:pixelBuffer timeStamp:self.currentTimestamp];
}
```
接着往下边看音频硬编码：
```ruby
- (void)encodeAudioData:(AudioBufferList)inBufferList timeStamp:(uint64_t)timeStamp{
    if (![self createAudioConvert]){
        return;
    }
    if(!aacBuf){
        aacBuf = malloc(inBufferList.mBuffers[0].mDataByteSize);
    }

    // 初始化一个输出缓冲列表
    AudioBufferList outBufferList;
    outBufferList.mNumberBuffers              = 1;
    outBufferList.mBuffers[0].mNumberChannels = inBufferList.mBuffers[0].mNumberChannels;
    outBufferList.mBuffers[0].mDataByteSize   = inBufferList.mBuffers[0].mDataByteSize; // 设置缓冲区大小
    outBufferList.mBuffers[0].mData           = aacBuf; // 设置AAC缓冲区
    UInt32 outputDataPacketSize               = 1;
    if (AudioConverterFillComplexBuffer(m_converter, inputDataProc, &inBufferList, &outputDataPacketSize, &outBufferList, NULL) != noErr){
        return;
    }
    LFAudioFrame *audioFrame = [LFAudioFrame new];
    audioFrame.timestamp = timeStamp;
    audioFrame.data = [NSData dataWithBytes:aacBuf length:outBufferList.mBuffers[0].mDataByteSize];

    char exeData[2];
    exeData[0] = _configuration.asc[0];
    exeData[1] = _configuration.asc[1];
    audioFrame.audioInfo =[NSData dataWithBytes:exeData length:2];
    if(self.aacDeleage && [self.aacDeleage respondsToSelector:@selector(audioEncoder:audioFrame:)]){
        [self.aacDeleage audioEncoder:self audioFrame:audioFrame];
    }
}
```
音频编码过程做了如下几件事：
    1. 将音频编码成AAC格式，编码样式在LFAudioFrame中，可编辑audioInfo在flv打包中AAC的header。
    2. 编码完成将数据上传。

> 通过[self.aacDeleage audioEncoder:self audioFrame:audioFrame];将打包好的音频数据上传，上传接下来分析。

接着往下边看视频硬编码：
```ruby
#pragma mark -- LFVideoEncoder
- (void)encodeVideoData:(CVImageBufferRef)pixelBuffer timeStamp:(uint64_t)timeStamp{
    if(_isBackGround) return;
    frameCount ++;
    CMTime presentationTimeStamp = CMTimeMake(frameCount, 1000);
    VTEncodeInfoFlags flags;
    CMTime duration = CMTimeMake(1, (int32_t)_configuration.videoFrameRate);
    NSDictionary *properties = nil;
    if(frameCount % (int32_t)_configuration.videoMaxKeyframeInterval == 0){
        properties = @{(__bridge NSString *)kVTEncodeFrameOptionKey_ForceKeyFrame: @YES};
    }
    NSNumber *timeNumber = @(timeStamp);
    VTCompressionSessionEncodeFrame(compressionSession, pixelBuffer, presentationTimeStamp, duration, (__bridge CFDictionaryRef)properties, (__bridge_retained void *)timeNumber, &flags);
}

```
> 视频编码直接通过VideoToolbox进行硬编，这个系统提供了支持，相对简易了许多。之前接触videocore，涉及ffmpeg软编码，不知道这两者效率怎么样。
VTCompressionSessionRef中有个VideoCallBack回调，代码如下，简易注释了下

```ruby
static void VideoCompressonOutputCallback(void *VTref, void *VTFrameRef, OSStatus status, VTEncodeInfoFlags infoFlags, CMSampleBufferRef sampleBuffer){

  //编码完了之后
  //回调....上传ing

}

```
这个方法代码有点多，放在gist上[戳下看看](https://gist.github.com/zhaoxiaobao/6322b0fb2fdc8467100284df4bf31253)。这个地方还有很多要说的，还在挖坑ing！！！


### 上传音视频数据
```ruby
#pragma mark -- EncoderDelegate
- (void)audioEncoder:(nullable id<LFAudioEncoding>)encoder audioFrame:(nullable LFAudioFrame*)frame{
    if(self.uploading) [self.socket sendFrame:frame];//<上传
}

- (void)videoEncoder:(nullable id<LFVideoEncoding>)encoder videoFrame:(nullable LFVideoFrame*)frame{
    if(self.uploading) [self.socket sendFrame:frame];//<上传
}

#pragma mark -- LFStreamTcpSocketDelegate
- (void)socketStatus:(nullable id<LFStreamSocket>)socket status:(LFLiveState)status{
    if(status == LFLiveStart){
        if(!self.uploading){
            self.timestamp = 0;
            self.isFirstFrame = YES;
            self.uploading = YES;
        }
    }
    dispatch_async(dispatch_get_main_queue(), ^{
        self.state = status;
        if(self.delegate && [self.delegate respondsToSelector:@selector(liveSession:liveStateDidChange:)]){
            [self.delegate liveSession:self liveStateDidChange:status];
        }
    });
}
```
> 上传数据之前用librtmp-iOS，后来改用pili-librtmp，疑似还有小bug，可能和服务器有关。

## 结尾
希望对大家帮助
