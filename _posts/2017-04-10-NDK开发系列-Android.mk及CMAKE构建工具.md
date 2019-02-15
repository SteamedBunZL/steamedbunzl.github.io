---
layout:     post
title:      "NDK开发系列-Android.mk及CMAKE构建工具"
date:       2017-04-10 12:00:00
author:     "ZhangLong"
catalog: true
tags:
    - NDK
---



在开发NDK时，我们以前的看的项目基本都使用的Android.mk的方式去编译生成动态库的，现在Google已经支持并推荐了使用了cmake的方式，cmake是一套构建工具，并不是脚本，通过cmake会生成ninja脚本，然后通过脚 本再来编译和生成动态库



我们先来看一下在Android Studio中是如何进行配置和支持的吧，看下build.gradle

```groovy
defaultConfig{
		//指导我们的源文件编译
        externalNativeBuild{
            ndkBuild{
                //armeabi 是指的v5 v6这种版本，现在NDK 17不再支持生成armeabi了必须从armeabi-v7a开始
                abiFilters "armeabi-v7a"//,"x86" // 你希望编译你的c/c++ 源文件 编译出几种cpu(arm,x86)
            }
        }
    
    	//应该打包几种cpu
        //比如 集成了第三方库，第三方库中提供了 arm和 x86的 可以在此处指导只打包arm 生成的apk就只打包arm的so
        ndk{
            abiFilters "armeabi-v7a"
        }

}

//配置 native 的构建脚本路径
externalNativeBuild{
    ndkBuild{
        path "src/main/cpp/Android.mk" //和build.gradle同路径 如果在别的路径要修改
    }
}
```

我们看下Android.mk文件中的内容

```makefile
#源文件在的位置，宏函数 my-dir 返回当前目录(包含 Android.mk 文件本身的目录) 路径 意思就是把当前mk文件路径赋值给LOCAL_PATH
LOCAL_PATH := $(call my-dir)
$(info "${LOCAL_PATH}") #使用info可以输出LOCAL_PATH的值

#引入其他makefile文件，CLEAR_VARS 变量指向特殊 GNU Makefile,可为您清除许多 LOCAL_XXX变量，但不会清理LOCAL_PATH
include $(CLEAR_VARS)

#存储你要构建的的模块的名称 每个模块名称必须唯一 且不含任何空格
#如果模块名称开头已经是lib,则构建系统不会附加额外的前缀lib,而是按原样采用模块名称，并添加.so扩展名
LOCAL_MODULE := native-lib

#包含要构建的模块中的C/C++源文件列表 以空格分开
LOCAL_SRC_FILES := native-lib.cpp

#构建动态库
include $(BUILD_SHARED_LIBRARY)
```























