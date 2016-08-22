# LFLiveKit-Review  

> 之前项目用到LFLiveKit，现在有空来写篇分析源码的文章。

## LFLiveKit架构图解
LFLiveKit工程文件已经比较清晰了，整理如下：
<p align="center">
  <img src="https://raw.githubusercontent.com/zhaoxiaobao/OpenSource-Review/master/Projects/Images/LFLiveKit.png" alt="img"/>
</p>

> 啧啧啧，这个图基本上就是来疯kit工程的目录图，个人觉得还是很👍的，你们可以follow这位作者[chenliming777](https://github.com/chenliming777)。往下🐱下，咱进入正题。

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

   创建session时候配置相关音视频参数~

### 开始推流
```ruby
- (void)startLive:(LFLiveStreamInfo *)streamInfo {
    if (!streamInfo) return;
    _streamInfo = streamInfo;
    _streamInfo.videoConfiguration = _videoConfiguration;
    _streamInfo.audioConfiguration = _audioConfiguration;
    [self.socket start];
}
```
> 推流建立streamInfo准备推流，
