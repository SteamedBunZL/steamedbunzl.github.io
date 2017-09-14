---
layout:     post
title:      "结合Volley和Logger设计自己的DebugLog日志输出工具"
date:       2017-09-10 12:00:00
author:     "ZhangLong"
catalog: true
---


兵马未动良草先行，一个优秀的程序怎么能没有一个方便好用的日志工具做辅助呢，很多初学者封装的日志工具大多数是类似这样的

```java
public class DebugLog{
	...
  	public static void i(String tag,String message){
      	if(DEBUG)
      		Log.i(tag,message);
  	}
  	...
}
```

只是进行了简单的封装，这里不去探究是否去应对反编译时日志内容泄漏问题，使用统一的日志工具，毕竟更加方便和快捷，而且有利于代码的整洁和可读性，感觉在编译期去掉or保留各有千秋，不是本文探讨的内容。

看了volley的源码，感觉volley对日志输出这块算是比较有特点的了，volley的日志输出，会输出当前调用的线程名称和调用方法及类的名称，输出信息相对简单的日志要丰富的多，在我们查看日志时反馈的信息量相对较大，可以更加方便我们进行分析和探究。技术上也很简单，过滤调用栈中有classname和methodname，并整合到日志输出中来，而且采用C的方式进行内容输出，习惯后比自己去在内容里不停的拼接String和各种类型数据要爽的多

```java
    
	public static String TAG = "DebugLog";	

	private String buildMessage(String format, Object... args) {
        String msg = (args == null) ? format : String.format(Locale.US, format, args);
        StackTraceElement[] trace = new Throwable().fillInStackTrace().getStackTrace();

        String caller = "<unknown>";
        for (int i = 2; i < trace.length; i++) {
            Class<?> clazz = trace[i].getClass();
            if (!clazz.equals( DebugLog.class)) {
                String callingClass = trace[i].getClassName();
                callingClass = callingClass.substring(callingClass.lastIndexOf('.') + 1);
                callingClass = callingClass.substring(callingClass.lastIndexOf('$') + 1);

                caller = callingClass + "." + trace[i].getMethodName();
                break;
            }
        }
        return String.format(Locale.US, "[%s] %s: %s", Thread.currentThread().getName() + "-" + Thread.currentThread().getId(), caller, msg);
    }

 	public void i(String format, Object... args) {
        if (DEBUG) {
            Log.v(TAG, buildMessage(format, args));
        }
    }
```

调用的方法很简单

```java
DebugLog.i("输出的内容 %s","日志");
```

volley是一个网络框架，为了记录request - response 的过程，使用了MarkerLog的方式，持有一个集合去记录过程中想要打的点，最后在流程结束时进行日志输出，整个流程每一步使用的时间，和线程信息都输出出来了，这个思路非常的值得借鉴，尤其是复杂流程的调用，这样流程会非常清晰明了，对这里有兴趣的同学，可以具体去参见volley源码。

上面说的都是volley日志的优点，如果真的完美无缺就不会有下面的Logger什么事了

1.首先volleylog对异常日志的输出不够完善和方便

2.volley里的tag必须指定死，虽然可以开放设置方法，但是如果一旦进行set的话，在不同的线程里，这么使用会有问题，所有的日志tag都会改变，这不利于企业级项目团队开发，因为如果不同的人使用，，有的人习惯用类名设置tag，有的人习惯用自己的设计的名字。。。所以这应对不了扩展的需求。

针对以上两点，看了下Logger，Logger也是一个不错的日志框架，但是对于我们日常使用显得略为的沉重，里面加入了很多日志输出的策略，这至少对于我们应用来是没必要的，Logger中的采用了链式的调用，可以每次输出时指定不同的tag，这个不正是我们想要的么，看看怎么做到的。

封装一个Printer类，我们的主要输出逻辑都在这里

```java
	public static String TAG = "DebugLog";//设置一个默认的tag

	ThreadLocal<String> localTag = new ThreadLocal<>();//关键点，提供只使用一次的tag

	/**
	 * 对外提供的更换tag的方法
	 */
	public Printer t(String tag){
        if (tag!=null)
            localTag.set(tag);
        return this;
    }

	/**
     * 获取tag
     */
	private String getTag(){
        String tag = localTag.get();
        if (tag!=null){
            localTag.remove();
            return tag;
        }
        return TAG;
    }

	//结合volley的输出内容形式去输出日志
	public void i(String format,Object... args){
        if (DEBUG)
            Log.i(getTag(),buildMessage(format,args));
    }
```

这里用到了线程本地变量去存储tag 真的是非常好的设计思想，每次使用后remove 如果取不到就用公有的tag 可以有效的避免因多线程调用导致的问题。

这样我们外层DebugLog要再进行一次封装

```java
    private static Printer printer = new Printer();

    public static Printer t(String tag){
       return printer.t(tag);
    }

	public static void i(String format,Object... args){
        printer.i(format,args);
    }

	...
```

使用起来也非常方便

```java
DebugLog.t("tag").i("输出内容 %s","日志");
```

看起来顺眼多了，另外对于异常信息，借鉴了Logger，把throwable转化为string 方便volley的输出格式

```java
 	public String getStackTraceString(Throwable tr) {
        if (tr == null) {
            return "";
        }

        // This is to reduce the amount of log spew that apps do in the non-error
        // condition of the network being unavailable.
        Throwable t = tr;
        while (t != null) {
            if (t instanceof UnknownHostException) {
                return "";
            }
            t = t.getCause();
        }

        StringWriter sw = new StringWriter();
        PrintWriter pw = new PrintWriter(sw);
        tr.printStackTrace(pw);
        pw.flush();
        return sw.toString();
    }
```

简单好用的日志工具就介绍到这里了，完整的代码见这里。