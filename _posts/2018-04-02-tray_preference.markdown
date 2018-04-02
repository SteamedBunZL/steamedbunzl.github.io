---
layout:     post
title:      "跨进程存取TrayPreference源码分析"
date:       2018-04-2 12:00:00
author:     "ZhangLong"
catalog: true
tags:
    - Android
---



之前为了解决SharedPreference多进程读取的问题，公司采取的解决方案是使用了ContentProvider去访问SP，这两天看到了github上有一个TrayPreference开源项目，也是为了解决多进程读取问题的，感到有兴趣就了解了一下下，从实现原理，架构，到具体的细节做了下大概的分析。

## 原理

开始很好奇它是怎么实现的，最后发现实现的根本在于，Tray使用了数据库来替代传统的SP，使用ContentProvider去访问数据库，从而解决了多进程的问题

## 结构

![img](/img/image_tray.png)



如图所示是TrayPreference 类关系图，核心功能访问接口使用了PreferenceAccessor，具体的业务使用了PreferenceStorage接口，通过抽象类Preferences把它们整合到一起，存储类的TrayItem，在TrayPreference大量使用了泛型编程，可以在不同的实现层上分别耦合需要的逻辑类

**纵向上**

AppPrferences—>TrayPrferences—>AbstractPrference—>Preferences—>PreferenceAccessor

ContentProviderStorage—>TrayStorage—>PrferenceStorage

**横向上**

分别依赖，在每一层上特有的业务逻辑

具体的功能最终都使用了TrayProviderHelper这个类来实现



## 核心功能类介绍

**PreferenceAccessor.java**

核心接口，AppPreferences实现此接口，此接口规定了一系列存取的方法

PreferenceStorage.java

核心接口，规定了基础的数据存取规则，上述PreferenceAccessor中存取都是借由此接口实现类去具体实现功能的

**Preferences.java**

抽象类，实现PreferenceAccessor，并且耦合PreferenceStorage接口，使两个功能接口关联上，把具体功能的实现导向PreferenceStorage接口的实现类

**ContentProviderStorage.java**

PreferenceStorage的实现类，具体依赖于TrayProviderHelper来实现，最后都指向TrayProviderHelper的persist、remove、query等方法

**TrayProvderHelper.java**

这里就是一些直接操作数据库的一些，增、删、改、查的操作了

**AppPreferences.java**

面向用户的Tray的核心功能类，写入/读取的核心类

```java
AppPreferences mTrayPreferences = new AppPreferences(this);
mTrayPreferences.put(KEY_MULTIPROCESS_COUNTER_SERVICE_WRITE, mCount);//写入
int trayCount = mTrayPreferences.getInt(KEY_MULTIPROCESS_COUNTER_SERVICE_READ, 0);//读取
```

由于底层使用了ContentProvider存取数据，可以支持写入和读取数据发生在不同的进程，并且没有进程数据同步的问题

**SharedPreferencesImport.java**

可以把原来旧有的SharedPreferences里的数据，导入到Tray的数据库中

```java
final String data = "SOM3 D4T4 " + (System.currentTimeMillis() % 100000);
mSharedPreferences.edit().putString(SHARED_PREF_KEY, data).apply();//使用原生SharedPreference写入数据

final SharedPreferencesImport sharedPreferencesImport =new SharedPreferencesImport(this, SampleActivity.SHARED_PREF_NAME,SampleActivity.SHARED_PREF_KEY, TRAY_PREF_KEY);
mImportPreference.migrate(sharedPreferencesImport);//把
```

**TrayUri.java**

采用Builder模式，这里在拼uri时获取Authority的方式比较好，看TrayContract.java

```java
 @NonNull
    private static synchronized String getAuthority(@NonNull final Context context) {
        if (sAuthority != null) {
            return sAuthority;
        }

        checkOldWayToSetAuthority(context);

        // read all providers of the app and find the TrayContentProvider to read the authority
        final List<ProviderInfo> providers = context.getPackageManager()
                .queryContentProviders(context.getPackageName(), Process.myUid(), 0);
        if (providers != null) {
            for (ProviderInfo provider : providers) {
                if (provider.name.equals(TrayContentProvider.class.getName())) {
                    sAuthority = provider.authority;
                    TrayLog.v("found authority: " + sAuthority);
                    return sAuthority;
                }
            }
        }

        // Should never happen. Otherwise we implemented tray in a wrong way!
        throw new TrayRuntimeException("Internal tray error. "
                + "Could not find the provider authority. "
                + "Please fill an issue at https://github.com/grandcentrix/tray/issues");
    }
```

通过读取所有的providers，然后找到TrayContentProvider去获取authority，而TrayContentProvider则使用Manifest静态注册的方式

```java
 <provider
            android:name=".provider.TrayContentProvider"
            android:authorities="${applicationId}.tray"
            android:exported="false"
            android:multiprocess="false" />
```

这里会跟随包名而改变，采用了包名 + tray的authority



