title: iOS Music Background Play And Lock Screen Image
categories: IOS
tags:
  - Audio
date: 2015-08-11 22:13:28
---

[reference](http://blog.csdn.net/zsk_zane/article/details/47320621)
## iOS 音乐后台播放及锁屏信息显示
实现音乐的后台播放，以及播放时，可以控制其暂停，下一首等操作，以及锁屏图片歌曲名等的显示 
此实例需要真机调试，效果图如下： 

![](http://img.blog.csdn.net/20150806175143383)

工程下载：[github工程下载](https://github.com/Nongchaozhe/MusicRemoteControl)

### 实现步骤： 

1) 首先修改info.plist

![](http://img.blog.csdn.net/20150806174938474)

2) 其次引入两个需要的框架

```
#import <AVFoundation/AVFoundation.h>
#import <MediaPlayer/MediaPlayer.h>
```

3) 设置播放器及后台播放

```
- (void)viewDidLoad {
    [super viewDidLoad];
//    设置后台播放
    [[AVAudioSession sharedInstance] setCategory:AVAudioSessionCategoryPlayback error:nil];

//    设置播放器
    NSURL *url = [NSURL fileURLWithPath:[[NSBundle mainBundle] pathForResource:@"那些花儿" ofType:@"mp3"] ];
    _player = [[AVPlayer alloc] initWithURL:url];
    [_player play];
    _isPlayingNow = YES;

    //后台播放显示信息设置
    [self setPlayingInfo];
}

#pragma mark - 接收方法的设置
- (void)remoteControlReceivedWithEvent:(UIEvent *)event {
    if (event.type == UIEventTypeRemoteControl) {  //判断是否为远程控制
        switch (event.subtype) {
            case  UIEventSubtypeRemoteControlPlay:
                if (!_isPlayingNow) {
                    [_player play];
                }
                _isPlayingNow = !_isPlayingNow;
                break;
            case UIEventSubtypeRemoteControlPause:
                if (_isPlayingNow) {
                    [_player pause];
                }
                _isPlayingNow = !_isPlayingNow;
                break;
            case UIEventSubtypeRemoteControlNextTrack:
                NSLog(@"下一首");
                break;
            case UIEventSubtypeRemoteControlPreviousTrack:
                NSLog(@"上一首 ");
                break;
            default:
                break;
        }
    }
}
```

<!-- more -->

4) 设置后台播放时显示的东西，例如歌曲名字，图片等

```
- (void)setPlayingInfo {
//    <MediaPlayer/MediaPlayer.h>
    MPMediaItemArtwork *artWork = [[MPMediaItemArtwork alloc] initWithImage:[UIImage imageNamed:@"pushu.jpg"]];

    NSDictionary *dic = @{MPMediaItemPropertyTitle:@"那些花儿",
                          MPMediaItemPropertyArtist:@"朴树",
                          MPMediaItemPropertyArtwork:artWork
                          };
    [[MPNowPlayingInfoCenter defaultCenter] setNowPlayingInfo:dic];
}
```

5) 远程控制设置

```
- (void)viewDidAppear:(BOOL)animated {
//    接受远程控制
    [self becomeFirstResponder];
    [[UIApplication sharedApplication] beginReceivingRemoteControlEvents];
}

- (void)viewDidDisappear:(BOOL)animated {
//    取消远程控制
    [self resignFirstResponder];
    [[UIApplication sharedApplication] endReceivingRemoteControlEvents];
}
```

