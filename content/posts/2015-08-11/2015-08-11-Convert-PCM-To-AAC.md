---
author: null
categories:
- Audio
date: '2015-08-11'
tags:
- Audio
title: Convert PCM To AAC
---

[reference](http://blog.csdn.net/poechant/article/details/7435054)

## 音频编解码·实战篇（1）PCM转至AAC（AAC编码）

- 作者：柳大·Poechant
- 博客：blog.csdn.net/poechant
- 邮箱：zhongchao.ustc@gmail.com
- 日期：April 7th, 2012

这里利用FAAC来实现AAC编码。

### 下载安装 FAAC

这里的安装过程是在 Mac 和 Linux 上实现的，Windows可以类似参考。

```
wget http://downloads.sourceforge.net/faac/faac-1.28.tar.gz
tar zxvf faac-1.28.tar.gz
cd faac-1.28
./configure
make
sudo make install
```

如果才用默认的 configure 中的 prefix path，那么安装后的 lib 和 .h 文件分别在/usr/local/lib和/usr/local/include，后面编译的时候会用到。

如果编译过程中发现错误：
```
mpeg4ip.h:126: error: new declaration ‘char* strcasestr(const char*, const char*)’
```

<!--more-->

解决方法：
从123行开始修改此文件mpeg4ip.h，到129行结束。 修改前：
```
#ifdef __cplusplus
extern "C" {
#endif
char *strcasestr(const char *haystack, const char *needle);
#ifdef __cplusplus
}
#endif
```

修改后：
```
#ifdef __cplusplus
extern "C++" {
#endif
const char *strcasestr(const char *haystack, const char *needle);
#ifdef __cplusplus
}
#endif
```

### FAAC API

#### Open FAAC engine

Prototype:
```
faacEncHandle faacEncOpen               // 返回一个FAAC的handle
(                   
    unsigned long   nSampleRate,        // 采样率，单位是bps
    unsigned long   nChannels,          // 声道，1为单声道，2为双声道
    unsigned long   &nInputSamples,     // 传引用，得到每次调用编码时所应接收的原始数据长度
    unsigned long   &nMaxOutputBytes    // 传引用，得到每次调用编码时生成的AAC数据的最大长度
);
```

#### Get/Set encoding configuration

获取编码器的配置：
```
faacEncConfigurationPtr faacEncGetCurrentConfiguration // 得到指向当前编码器配置的指针
(
    faacEncHandle hEncoder  // FAAC的handle
);
```

设定编码器的配置：
```
int FAACAPI faacEncSetConfiguration
(
    faacDecHandle hDecoder,         // 此前得到的FAAC的handle
    faacEncConfigurationPtr config  // FAAC编码器的配置
);
```

#### Encode

Prototype:
```
int faacEncEncode
(
    faacEncHandle hEncoder,     // FAAC的handle
    short *inputBuffer,         // PCM原始数据
    unsigned int samplesInput,  // 调用faacEncOpen时得到的nInputSamples值
    unsigned char *outputBuffer,// 至少具有调用faacEncOpen时得到的nMaxOutputBytes字节长度的缓冲区
    unsigned int bufferSize     // outputBuffer缓冲区的实际大小
);
```

#### Close FAAC engine

Prototype
```
void faacEncClose
(
    faacEncHandle hEncoder  // 此前得到的FAAC handle
);
```

### 流程

#### 做什么准备？

采样率，声道数（双声道还是单声道？），还有你的PCM的单个样本是8位的还是16位的？

#### 开启FAAC编码器，做编码前的准备

1. 调用faacEncOpen开启FAAC编码器后，得到了单次输入样本数nInputSamples和输出数据最大字节数nMaxOutputBytes；
2. 根据nInputSamples和nMaxOutputBytes，分别为PCM数据和将要得到的AAC数据创建缓冲区；
3. 调用faacEncGetCurrentConfiguration获取当前配置，修改完配置后，调用faacEncSetConfiguration设置新配置。

#### 开始编码

调用`faacEncEncode`，该准备的刚才都准备好了，很简单。

#### 善后

关闭编码器，另外别忘了释放缓冲区，如果使用了文件流，也别忘记了关闭。

### 测试程序

#### 完整代码

将PCM格式音频文件`/home/michael/Development/testspace/in.pcm`转至AAC格式文件`/home/michael/Development/testspace/out.aac`。

```
#include <faac.h>
#include <stdio.h>

typedef unsigned long   ULONG;
typedef unsigned int    UINT;
typedef unsigned char   BYTE;
typedef char            _TCHAR;

int main(int argc, _TCHAR* argv[])
{
    ULONG nSampleRate = 11025;  // 采样率
    UINT nChannels = 1;         // 声道数
    UINT nPCMBitSize = 16;      // 单样本位数
    ULONG nInputSamples = 0;
    ULONG nMaxOutputBytes = 0;

    int nRet;
    faacEncHandle hEncoder;
    faacEncConfigurationPtr pConfiguration; 

    int nBytesRead;
    int nPCMBufferSize;
    BYTE* pbPCMBuffer;
    BYTE* pbAACBuffer;

    FILE* fpIn; // PCM file for input
    FILE* fpOut; // AAC file for output

    fpIn = fopen("/home/michael/Development/testspace/in.pcm", "rb");
    fpOut = fopen("/home/michael/Development/testspace/out.aac", "wb");

    // (1) Open FAAC engine
    hEncoder = faacEncOpen(nSampleRate, nChannels, &nInputSamples, &nMaxOutputBytes);
    if(hEncoder == NULL)
    {
        printf("[ERROR] Failed to call faacEncOpen()\n");
        return -1;
    }

    nPCMBufferSize = nInputSamples * nPCMBitSize / 8;
    pbPCMBuffer = new BYTE [nPCMBufferSize];
    pbAACBuffer = new BYTE [nMaxOutputBytes];

    // (2.1) Get current encoding configuration
    pConfiguration = faacEncGetCurrentConfiguration(hEncoder);
    pConfiguration->inputFormat = FAAC_INPUT_16BIT;

    // (2.2) Set encoding configuration
    nRet = faacEncSetConfiguration(hEncoder, pConfiguration);

    for(int i = 0; 1; i++)
    {
        // 读入的实际字节数，最大不会超过nPCMBufferSize，一般只有读到文件尾时才不是这个值
        nBytesRead = fread(pbPCMBuffer, 1, nPCMBufferSize, fpIn);

        // 输入样本数，用实际读入字节数计算，一般只有读到文件尾时才不是nPCMBufferSize/(nPCMBitSize/8);
        nInputSamples = nBytesRead / (nPCMBitSize / 8);

        // (3) Encode
        nRet = faacEncEncode(
        hEncoder, (int*) pbPCMBuffer, nInputSamples, pbAACBuffer, nMaxOutputBytes);

        fwrite(pbAACBuffer, 1, nRet, fpOut);

        printf("%d: faacEncEncode returns %d\n", i, nRet);

        if(nBytesRead <= 0)
        {
            break;
        }
    }

    /*
    while(1)
    {
        // (3) Flushing
        nRet = faacEncEncode(
        hEncoder, (int*) pbPCMBuffer, 0, pbAACBuffer, nMaxOutputBytes);

        if(nRet <= 0)
        {
            break;
        }
    }
    */

    // (4) Close FAAC engine
    nRet = faacEncClose(hEncoder);

    delete[] pbPCMBuffer;
    delete[] pbAACBuffer;
    fclose(fpIn);
    fclose(fpOut);

    //getchar();

    return 0;
}
```

#### 编译运行

将上述代码保存为“pcm2aac.cpp”文件，然后编译：
```
g++ pcm2aac.cpp -o pcm2aac -L/usr/local/lib -lfaac -I/usr/local/include
```

运行：
```
./pcm2aac
```

然后就生成了out.aac文件了，听听看吧！~

### Reference

- [AudioCoding.com - FAAC](http://www.audiocoding.com/faac.html)
- [Dogfoot – 재밌는 개발](http://blog.tcltk.co.kr/?p=2130)
