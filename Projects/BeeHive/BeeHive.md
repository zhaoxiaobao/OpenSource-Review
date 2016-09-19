# BeeHive
[官方的中文文档](https://github.com/alibaba/BeeHive/blob/master/README-CN.md)

> 几个月前刚开源的的项目，作者还在不断维护中。代码还没有细看，觉得还是值得花点时间阅读一下，官方文档写的比较详细，挖一下官方文档没有的东西。

<p align="center">
  <img src="https://raw.githubusercontent.com/zhaoxiaobao/OpenSource-Review/master/Projects/Images/BeeHive.png" alt="img"/>
</p>

# BHTimeProfiler
BeeHive 中通过BHTimeProfiler对象，对事件进行埋点监测。在测试环境通过saveTimeProfileDataIntoFile方法进行将打点时间放入txt文件中。

```ruby
- (void)saveTimeProfileDataIntoFile:(NSString *)fileName
{
    NSString *documentPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSString *filePath =  [documentPath stringByAppendingPathComponent:[fileName stringByAppendingPathExtension:@"txt"]];
    NSLog(@"TMTimeProfiler::SaveFilePath is %@", filePath);
    BOOL res=[[NSFileManager defaultManager] createFileAtPath:filePath contents:nil attributes:nil];
    if (!res) {
        return;
    }
    NSFileHandle *handle = [NSFileHandle fileHandleForWritingAtPath:filePath];
    for (NSString *eventName in self.identifiers)
    {
        CFTimeInterval current = [[self.timeDataDic objectForKey:eventName] doubleValue];
        NSString *output = [NSString stringWithFormat:@"%@ time stamp  %g and execute for  %g\n", eventName, current, (current - self.lastTime) * 1000];
        [handle writeData:[output dataUsingEncoding:NSUTF8StringEncoding]];
        self.lastTime = current;
    }

    [handle closeFile];
}

```
