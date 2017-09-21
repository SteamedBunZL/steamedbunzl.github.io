---
layout:     post
title:      "C语言学习笔记（一）指针"
date:       2017-09-19 12:00:00
author:     "ZhangLong"
catalog: true
---



因为平时能用到C的地方实在太少了，特此记录，重新自学一遍C语言，加深记忆

## 一、指针数组以及多级指针


#### 1.指针数组

直接上code

```c
int main(){
  int a*[10];//定义了一个指针数组，一共10个成员，其中每个成员都是int *类型
  printf("%d,%d\n",sizeof(a),sizeof(a[0]));//输出 40，4
  
  short *b[10];//定义了一个指针数组，一共10个成员，其中每个成员都是int *类型
  printf("%d,%d\n",sizeof(b),sizeof(b[0]));//输出 40，4
  
  //指针是地址，地址在同一个操作系统中是同样的大小的，比如4个字节，或者8个字节
  
  
}
```



#### 2.指向指针的指针

```c
int main(){
  int a = 10;
  int *p = &a;
  int **p = &p;//定义一个二级指针，指向一个一级指针的地址
  // 相当于 int **pp;  pp = &p;
  **pp = 100;//通过二级指针修改内存的值
  *p = 10;//千万不要这样写，这样相当于将p指向了编号为10的这块内存 这样写后，pp还是正常的指针，p就变成野指针
  printf("a = %d\n",a);//输出 100
  
  return 0;
}
```

如图所示关系



![image_point1](/img/image_point1.png)





#### 3.会带来更多的问题

```c
int main(){
  int a = 10;
  int *p = &a;
  int **p = &p;//定义一个二级指针，指向一个一级指针的地址
  // 相当于 int **pp;  pp = &p;
  **pp = 100;//通过二级指针修改内存的值
  *p = 10;//千万不要这样写，这样相当于将p指向了编号为10的这块内存 这样写后，pp还是正常的指针，p就变成野指针
  printf("a = %d\n",a);//输出 100
  
  int ***ppp = &pp;//定义了一个三级指针，指向一个二级指针的地址
  ****ppp = 0;
  printf("a = %d\n",);
  
  a = ppp;//等同于a = &pp
  printf("%d\n",***ppp);//19652
  
  printf("%d,%d,%d,%d\n",***ppp,&p,&pp,&ppp);//1375048,1375060,1375048,1375036
  return 0;
}
```

**C语言允许定义多级指针，但是指针级数过多，会增加代码的复杂性，考试的时候可能会考多级指针，但实际编程的时候最多用到3级，但3级指针也不常用，一级和二级指针是大量使用。**


## 二、指向多维数组的指针


```c
    int main(){
      int buf[2][3];      
      int *p[3];//指针数组
      
      int (*p)[3];//定义了一个指针，不是数组，注意这里。。。指向int [3]这种数据类型，指向二维数组的指针
      p = buff;//p指向了二维数组上的第一行
      p++;//指向了二维数组的第二行
      printf("%d\n",sizeof(p));// 输出 4 证明了这是一个指针而不是一个数组
      printf("%d,%d\n",p,p+1);//输出3276484，3276496  p+1 在内存中增加了12
      //加1是什么意思？位移了了1*sizeof(int [3])字节
      //printf("%d,%d\n,p,p+2"); 位移了2*sizeof(int [3])字节
      
      //怎么去定义他？
      int i;
      int j;
      for(i = 0;i<2;i++){
          for(j = 0;j<3;j++){
              printf("%d\n",p[i][j]);
              //printf("%d\n", *(*(p + i) + j)); 指针的方式
          }
      }
      
      return 0;
    }

```
![image_point2](/img/image_point2.png)



## 三、指针做为函数的参数

#### 1.把一维数组作为函数参数传递

```c
#include <studio.h>

void test(int n){
    n++;//值传递，形参再怎么改变不改变实参的值
}

void test1(int *n){
    (*n)++;
}

int main(){
  int i = 100;
  test(i);
  printf("i = %d\n",i);// 输出i = 100;
  test1(&i);
  printf("i = %d\n",i);// 输出i = 101;
  
  
  //经常用指针访问数组
  int a = 10;
  int *p = &a;
  *p = 100;
  printf("%d\n",*p);
  //很少直接用指针指向一个变量，然后通过*方式访问变量
  
  return 0;
}
```

![image_point3](/img/image_point3.png)

通过函数的指针参数可以间接实现形参修改实参的值，比如scanf("%d",&i)，所有需要在函数内部修改实参的值，都需要将指针做为函数的参数调用实现。

```c
void set_array(int buf[]){//数组名代表的数组的首地址
  printf("buf = %d\n",sizeof(buf));//输出 4 证明它是一个指针 所以形参等同于(int *buf)
  buf[0] = 100;
  buf[1] = 200;
  // *buff = 100;
  // buff++;
  // *buff = 200;
}

void print_array(int buf[],int n){
  int i;
  for(i = 0; i<n;i++){
      printf("buf[%d] = %d\n",i,buf[i]);
  }
}

int main(){
  int buf[] = {1,2,3,4,5,6,7,8,9,10,11,12};
  printf("buf = %d\n",sizeof(buf));//输出40 证明它是一个数组
  set_array(buf);//放的是数组的首地址，是个指针 等价于
  //set_array(&buf);
  print_array(buf,sizeof(buf)/sizeof(int));//注意这是标准写法
  return 0;
}
```

一维数组名作为函数的参数，当数组名作为函数参数时，C语言将数组名解释为指针

int buf[] == int *buf 一般用后者的写法

本质是指针，所以如果是数组是参数的时候，要传入数组的大小维度，因为在函数内部用sizeof是没意义的



#### 2.把二维数组名作为函数参数传递

```c
void print_buf(const int (*p)[3],int a,int b){//将二维数组做为函数参数传递的时候，定义的指针类型，这个3一定要和定义数组的地方的列数对应
  //通过const标示，不需要修改参数的值
  int i;
  int j;
  for(i = 0;i<2;i++){
      for(j = 0;j<3;j++){
          printf("p[%d][%d] = %d\n",i,j,p[i][j]);
      }
  }
}

int main(){
    printf_buf(buf,sizeof(buf)/sizeof(buf[0]),sizeof(buf[0])/sizeof(int));//分别代表二维数组的行数和列数
}
```

#### 3.const 关键字保护数组内容

```c
void mystrcat(char *s1,const char *s2){//第一个参数是要被修改，第二个参数不被修改，用于保护，会报错
  int len = 0;
  while(s2[len]){
    len++;
  }
  while(*s1){
    s1++;
  }
  int i;
  for(i = 0;i<len;i++){
    *s1 = *s2;
    s1++;
    s2++;
  }
}

int main(){
   char s1[10] = "abc";
  char s2[10] = "bcd"
    mystrcat(s1,s2);//abcbcd
  
}
```

定义了const的指针参数，只读，不能修改



#### 4.将指针作为函数的返回值

```c
char *mystr(const char *s,char c){
    while(*s){
      	if(*s == c)
          return s;
        s++;
    }
  	return NULL;
}

int main(){
  char str[100] = "hello world";
  char *s = strchr(str,'0');
  printf("s = %s\n",s);
  mystr(str,'0');//返回 o world 如果没有就返回<null>空
  return 0;
}
```



## 四、函数指针



#### 1.函数指针的定义

```c
int add(int a,int b){
    return a+b;
}

int max(int a,int b){
    return a>b?a:b;
}

void print_array(int *s,int n){
    int i;
  	for(i = 0;i<n;i++){
        printf("%d\n",s[i]);
    }
}

int main(){
  int (*p)(int,int);//定义了一个指向函数的指针，可以指向两个参数，都是int,返回值也是Int这种类型的函数,在这里p是不能指向print_array函数的
  
  int status = 0;
  scanf("%d",&status);
  //p = add;//直接与函数的名字，代表函数的地址，将add这个函数的地址复制给指针变量p
  if(status ==1)
    p = max;
  else
    p = add;
  
  int i = 0;
  //编译器在编译i = p(5,7)的时候，根本就不知道要调用什么代码，运行期才知道
  i = add(5,7);//i = p(5,7);通过指针变量间接的调用指针指向的函数
  printf("i = %d\n",i);
  
  p = print_array;//类型不周的赋值，这种语句在C++中是编译通不过的
  
  void (*p1)(int *,int);
  void (*p)(int,char *);//定义了一个指向参数为int,char* 返回值为void的函数指针
  void *p(int,char *);//声明了一个函数，函数的名字叫p,函数的返回值是void *,函数的参数分别是int 和char* 注意和上面的区别
  
  //定义一个参数为int*返回值为int *的指向函数的指针
  int *(*p)(int *);
  
  
  return 0;
}
```

在回调函数和运行期动态绑定的时候，大量的用到了指向函数的指针



#### 2.把指向函数的指针做为函数的参数

```c
int add(int a,int b){
    return a+b;
}

int max(int a,int b){
    return a>b?a:b;
}

void func(int(*p)(int,int),int a,int b){//第一个参数是指向函数的指针
    return p(a,b);//通过指向函数的指针调用一个函数
}

int main(){
	int i = func(add,6,9);//add函数在这里就叫回调函数，android中的点击事件
  printf("%d\n",i);
  return 0;
}

```



## 五、内存操作函数



#### 1.memset

```c
int main(){
  int buf[1024] = {0};//这个方法只能用于定义数组同时初始化内容
  buf[0] = 8;
  buf[1] = 9;
  buf[4] = 8;
  
  //想将这个buf再一次初始化为0?
  //不能用buf[10] = {0}
  int i;
  for(i = 0;i<1024;i++){
      buf[i] = 0;
  }
  
  memset(buf,0,sizeof(buf));//第一个参数是要设置的内存地址；第二个参数是要设置的值；第三个参数是内存大小单位是字节
  //将一块内存初始化为0最常见的方法
  
  
  
  return 0;
}
```

memset需要#include\<string.h>

![image_point4](/img/image_point4.png)

#### 2.memcpy

```c
int main(){
  int buf1[10] = {1,2,3,4,5,6,7,8,9,10};
  int buf2[10];
  memcpy(buf2,buf1,sizeof(buf1));//将buf1的内存内内容全部拷贝到buf2,拷贝大小为第三个参数：字节
 
  return 0;
}
```



#### 3.memmove

```c
int main(){
  int buf1[10] = {1,2,3,4,5,6,7,8,9,10};
  int buf2[10];
  memmove(buf2,buf1,sizeof(buf1));//和memcpy结果一致，区别是什么？
}
```

![image_point5](/img/image_point5.png)

#### 4.区别到底在哪？

建议，使用memcpy的时候，一定要确保内存没有重叠区域，但是没有提memmove

什么叫内存重叠区域？

```c
int main(){
    int buf[10] = {1,2,3,4,5,6,7,8,9,10};
  
  int *start = buf + 3;
  int *end = buf + 5;
  
  memcpy(start,end,16);//拷贝时不建义内存重叠
  
  int i;
  for(i = 0;i<10;i++){
      printf("buf[%d] = %d\n",i,buf[i]);
  }
}
```

![image_point6](img/image_point6.png)



