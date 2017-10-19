---
layout:     post
title:      "C语言复习笔记（二）内存"
date:       2017-09-29 12:00:00
author:     "ZhangLong"
catalog: true
---





## 内存管理



#### 1.作用域

一个C语言变量的作用域可以是代码块  作用域，函数作用域或者文件作用域。

a.c 文件

```c
int age = 10;//定义了一个全局变量
```

main.c文件

```c
extern int age;//有一个变量，类型是int,名字是age,已经在其他文件上定义了,这里就直接使用了

void set_age(int n){
  age = n;
}

void get_age(){
  printf("age = %d\n",age);
}

int main(){
  set_age(20);
  get_age();
  
  return 0;
}
```

出现在{}之外的变量，就是全局变量





**1.1 auto 自动变量**

auto 写不写是一样的，不写auto c语言默认变量都是auto,还有一个signed

这个变量是不需要管的，这个变量什么时候在内存中出现，什么时候消失，在代码块中的一般都是auto





**1.2 register寄存器变量**

```c
int main(){
  register int i = 0;//建议，如果有寄存器空闲，那么这个变量就放到寄存器里面使用,会增加效率
  int *p = &i;//错误！！！对于一个register变量，是不能取地址操作的
  return 0;
}
```



**1.3 代码块作用域的静态变量**

```c
void mystatic(){
  static int a = 0;//静态变量，只初始化一次（这句话只执行一次），而且程序运行期间，静态变量一直存在
  printf("a = %d\n",a);
  i++;
}

int main(){
  int i;
  for(i = 0;i<10;i++){
    mystatic();//a 如果没有static a都是 0 加上static 后数值 1，2，3，4
  }
}
```

静态变量是指内存位置在程序执行期间一直不改变的变量，*一个代码块内存部的静态变量只能被这个代码块内部访问*



**1.4 代码块作用域外的静态变量**

a.c

```c
static int age = 10;//一旦全局变量定义 static ,意思是这个变量只是在定义这个变量的文件内部全局有效，在main.c中是无效的，这个是有别于全局变量
```

代码块之外的静态变量在程序执行期间一直存在，但只能被定义这个变量的文件访问



**1.5 全局函数和静态函数**

在C语言中函数默认都是全局的，使用关键字static 可以将函数声明为静态

加了static 只能在当前文件中使用

a.c

```c
int test(){//全局函数
  
}
```

main.c 要使用 test();

```c
int test();//也可以 extern int test(); 但是变量是不可以去掉extern使用的
int age;//有两个含义，一个含义是声明一个变量，还有一个含义，定义一个变量

int main(){
  test();
  return 0;
}
```





#### 2.内存四区



![image_memory1](/img/image_memory1.png)



**代码区**

代码区code，程序被操作系统加载到内存时，所有可执行代码都加载到代码区，也叫代码段，这块内存是不可以在运行期间修改的。

```c
int test(){
  int a; //这个不是代码
  a = 0; //这个是在代码区，注意
  a = 5; 
}
```

**静态区**

所有的全局变量以及程序中的静态变量都存储到静态区

```c
int c = 0;//在静态区

int test(int a,int b){
    printf("%d,%d",&a,&b)
}

int main(){//函数在代码区里,函数名称代表函数地址
  static int d = 0;//也在静态区
  （auto）int a = 0;//在栈区
  int b = 0;
  
  printf("%d,%d,%d,%d,%d",&a,&b,&c,main,&d);
  return 0;
}
```



**栈区**

栈stack是一种先进后出的内存结构，所有的自动变量，函数的形参都是由编译器自动释放出栈中，当一个自动变量超出其作用域时，自动从栈上弹出。

对于自动变量，什么时候入栈，什么时候出栈，是不需要程序控制的，由C语言编译器实现



错误写法

```c
int *geta(){//函数的返回值是一个指针
    int a  = 100;
  return &a;
}

int main(){
    int *p = geta();//这里得到一个临时栈变量的地址，这个地址在函数geta调用完成之后已经无效了
  *p = 100;
  printf("%d\n",*p);
  return 0;
}
```



栈不会很大，一般都是以K为单位的

栈溢出

当栈空间已满，但还往栈内存压变量，这个就叫栈溢出

```c
int main(){
  char array[1024 *1024 *100] = {0}; //100M 定义一个超大的数组一定会栈溢出
  array[0] = 'a';
  pritnf("%s\n",array)
  return 0;
}
```



对于一个32位操作系统，最大管理内存4G内存，其中1G给操作系统自己用的，剩下的都是给用户的，一个用户程序理论上可以使用3G的内存空间。如果使用这么大的空间——堆



**堆区**

堆heap和栈一样，也是一种在程序运行过程中可以随时修改的内存区域，但没有栈那样先进后出的顺序。

堆是一个大容器，它的容量要远远大于栈，但是在C语言中，堆内存空间的申请和释放需要手动通过代码来完成。



```c
void print_array(int *p,int n){
    int i;
  for(i = 0;i<n;i++){
      printf("p[%d] = %d\n",i,p[i]);
  }
}

int *geta(){
    int a = 0;
  	return &a;//错误，不能将一个栈变量的地址通过函数的返回值返回
}

int *geta1(){
    int *p = malloc(sizeof(int));//申请了一个堆空间
  	return p;//合法
}

int *geta2(){
    static int a = 0;//合法的
  return &a;//静态区的地址可以得到，但是不能free malloc和free成对出现
}

int main(){
  int array[10] = {0};//栈数组
  
  //堆数组
   int *p = (int *)malloc(sizeof(int)*10);//在堆中间申请内存,在堆中申请了10个int这么大的空间
  char *p1 = malloc(sizeof(char)*10);//在堆中申请了10个char这么大的空间
  
  memset(p,0,sizeof(int)*10);//把内存清空了，注意sizof(int)*10
  
  int i;
  for(i = 0;i<10;i++){
      p[i] = i;
  }
  print_array(p,10);
 
  
  free(p);//释放通过malloc分配的堆内存,一块内存不能free两次
  free(p1);//释放通过malloc分想的堆内存
  
  return 0;
}
```

```c
void getheap(int *p){
  	printf("p = %p\n",&p);
    p = malloc(sizeof(int)*10);
  
}//getheap 执行完成以后，p就消失了，导致他指向的具体堆空间的地址的编号也随之消失

void getheap1(int **p){
    *p = malloc(sizeof(int)*10);
}

int main(){
    int *p = NULL;
  	printf("p = %p\n",&p);//和getheap中的地址不同
   	//p = malloc(sizeof(int)*10); 这样是正常的
  	getheap(p); //这样会崩溃，实参没有任何改变，还是空指针
  
  	getheap1(&p);//程序正确，得到了堆内存的地址
  	p[0] = 1;//没法赋值
  	p[1] = 2;
  	
  	pritnf("p[0] = %d,p[1] = %d\n",p[0],p[1]);
  	free(p);
  	return 0;
}
```

为什么会崩溃？ getheap内存模型如下图

![image_memory2](/img/image_memory2.png)

getheap1内存模型如下

![image_memory3](/img/image_memory3.png)



#### 3.堆、栈和内存映射



每个线程都有自己的专属的栈（stack），先进后出（LIFO）

栈的最大尺寸固定，超出则引起栈溢出

变量离开作用范围后，栈上的数据会自动释放

堆上的内存必须手工释放（C/C++），除非语言执行环境支持GC

栈还是堆？

​	明确知道数据占用多少内存

​	数据很小

​	大量内存

​	不确定需要多少内存

```c
int main(){
  int i = 0;
  scanf("%d",&i);
  //int array[i];//定义数组时，必须是常量，而不能是变量
  int *array = malloc(sizeof(int)*i);//在堆中当动态创建一个Int数组
  
  free(array);
}
```



![image_memory3](/img/image_memory4.png)



![image_memory3](/img/image_memory5.png)





#### 4.堆的分配和释放

操作系统在管理内存的时候，最小单位不是字节，而是内存页



```c
int main(){
    while(1){
        int *p = malloc(1024);
      getchar();
    }
  return 0;
}
```





