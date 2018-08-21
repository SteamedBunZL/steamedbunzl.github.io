---
layout:     post
title:      "xcode配置ffmpeg库"
date:       2017-8-21 12:00:00
author:     "ZhangLong"
catalog: true
tags:
    - Android
    - ffmpeg
---



不光是ffmpeg库，第三方库的配置皆是如此



## 1.安装ffmpeg

首先，可以从ffmpeg的官网下载ffmpeg，或者使用最简单的方式利用包安装器brew 来安装 ffmpeg

`brew install ffmpeg`

安装成功后的路径为 `/usr/local/Cellar/ffmpeg`



## 2.配置c/c++项目

这时打开xcode 创建c/c++项目 

![WX20180821-145421@2x](/img/WX20180821-145421@2x.png)

在搜索栏输入search 找到头文件和库的搜索路径

#### 2.1 导入头文件

在 图中`3`处把 `/usr/local/Cellar/ffmpeg/4.0.2/include   `头文件的路径导入 `Header Search Paths`处



#### 2.2 导入静态库

![WX20180821-150227@2x](/img/WX20180821-150227@2x.png)

按图中操作，把 `/usr/local/Cellar/ffmpeg/4.0.2/lib` 路径下的静态库，直接拖进 `Link Binary With Libraries`下，这样就把静态库导入成功了



## 3.打印ffmpeg的的版本号

已经成功的导入了 ffmpeg的头文件和静态库，我们来打印ffmpeg的版本号	

```c
#include <stdio.h>
#include <libavcodec/avcodec.h>

int main(int argc, const char * argv[]) {
    // insert code here...
    printf("ffmpeg version is %s\n",av_version_info());
    return 0;
}
```

可以看到输出

```
ffmpeg version is 4.0.2
```

大功告成！