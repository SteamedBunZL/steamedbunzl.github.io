---
layout:     post
title:      "C语言复习笔记（三）结构体，联合体，枚举与typedef"
date:       2017-10-01 12:00:00
author:     "ZhangLong"
catalog: true
---





一、结构体

1.定义结构体struct和初始化

之前的数据类型都比较简单，复杂数据类型可以用结构体来表达

```c
#include <stdio.h>
#include <string.h>

struct student{
    char name[100];
  int age;
  int sex;
};//说明了一个结构体的数据成员类型。

int main(){
  struct student st;//定义了一个结构体的变量，名字叫st;
  //struct student st = {"dddd",30,0};定义一个结构，同时初始化
  //struct student st = {0};定义一个结构体变量，名字叫st,同时将所有成员的值初始化为0
  
  //st.age = 20;
  //st.sex = 0;
  //strcpy(st.name,"刘德华")；
  
  struct student st = {.age = 20,.name = "ssss",.sex = 20};
  
  printf("name = %s",st.name);
  printf("age = %d\n",st.age);
  printf("sex = %d\n",st.sex);
  return 0;
  
}
```



2.访问结构体成员

.操作符



3.结构体的内存对齐模式

```c
struct A{
    int a;
};

int main(){
    struct A a;
  printf ("%d\n",sizeof(a));
  return 0
}
```

