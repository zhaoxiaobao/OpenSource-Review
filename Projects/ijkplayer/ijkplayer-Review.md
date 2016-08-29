# ijkplayer深入分析
> ijkplayer 涉及的平台比较多，本文主要以iOS平台下，视频解码为例，进行整个项目的分析。ijkplayer工程比较大，而且还有不少涉及C语言的东西，先了解各大概吧。主要目的想搞懂ijkplayer在iOS平台下运行机制和视频解码流程。

如下这个鱼骨图简单的描述了IJKMediaPlayer工程下各个文件的依赖关系：
<p align="center">
  <img src="https://raw.githubusercontent.com/zhaoxiaobao/OpenSource-Review/master/Projects/Images/IJKMediaPlayer.png" alt="img"/>
</p>

## 起步
ijkplayer通过IJKFFMoviePlayerController完成播放器的初始化，初始化时候涉及两个参数，一个是url，另一个是IJKFFOptions。通过IJKFFOptions参数，完成对解码相关配置，比如max-fps，min-frames等等，IJKFFOptions深究下去是ffmpeg的一些相关配置。optionsByDefault的默认配置如下：
```ruby
[options setPlayerOptionIntValue:30     forKey:@"max-fps"];
[options setPlayerOptionIntValue:0      forKey:@"framedrop"];
[options setPlayerOptionIntValue:3      forKey:@"video-pictq-size"];
[options setPlayerOptionIntValue:0      forKey:@"videotoolbox"];
[options setPlayerOptionIntValue:960    forKey:@"videotoolbox-max-frame-width"];
[options setFormatOptionIntValue:0                  forKey:@"auto_convert"];
[options setFormatOptionIntValue:1                  forKey:@"reconnect"];
[options setFormatOptionIntValue:30 * 1000 * 1000   forKey:@"timeout"];
[options setFormatOptionValue:@"ijkplayer"          forKey:@"user-agent"];

```
这个都是在ff_ffplaye_options.h中封装的，ijk在ffplayer基础上搞起来的，所以深究ijk，其实也是深究ffmpeg。ijk在初始化过程中，通过OpenGL进行视图渲染，封装在ijksdl中。关于ijksdl内容后续添加。初始化完成后，我们来探究ijk是如何来播放的。
## 播放
在IJKFFMoviePlayerController中有一个play的方法，用于基础的播放功能。
```ruby
- (void)play
{
    if (!_mediaPlayer)
        return;

    [self setScreenOn:_keepScreenOnWhilePlaying];

    [self startHudTimer];
    ijkmp_start(_mediaPlayer);
}

```
其中通过ijkmp_start调用ijkplayer.c中的ijkmp_start函数
```ruby
int ijkmp_start(IjkMediaPlayer *mp)
{
    assert(mp);
    MPTRACE("ijkmp_start()\n");
    pthread_mutex_lock(&mp->mutex);
    int retval = ijkmp_start_l(mp);
    pthread_mutex_unlock(&mp->mutex);
    MPTRACE("ijkmp_start()=%d\n", retval);
    return retval;
}
```
pthread_mutex_lock和pthread_mutex_unlock用于多线程并发控制，互斥锁。贴下这个player的结构体：
```ruby
struct IjkMediaPlayer {
    volatile int ref_count;
    pthread_mutex_t mutex;
    FFPlayer *ffplayer; //这个在ff_ffplay_def.h中进行了封装,ijk是将播放器通过c语言封装起来FFPlayer，然后再移植到苹果和安卓的平台。

    int (*msg_loop)(void*);
    SDL_Thread *msg_thread;//ijksdl_thread.h中对SDL线程进行封装
    SDL_Thread _msg_thread;

    int mp_state;
    char *data_source;
    void *weak_thiz;

    int restart;
    int restart_from_beginning;
    int seek_req;
    long seek_msec;
};
```
往下边继续追踪代码，就到了视频解码相关的代码了，ijk是通过ffmpeg解码的，在[ff_ffplay_def.h](https://gist.github.com/zhaoxiaobao/b1143c9bdd9b3f92b2675e451dd32ae0)中。

> 其实这里边涉及很多ffmpeg方面的知识，以后接触更多流媒体的知识，再深挖下去，现在在这个解码的地方打住，待跟进ing。

## IJKMediaPlayer
   IJKMediaPlayer中包含一个IJKAudioKit用于处理系统音频，也借鉴于kxmovie。

## ijkmedia
>  ijk核心主要集中在这个目录下了
