---
layout: post
math: true
title:  "结构体对齐"
date:   2022-04-22 22:29:22 +0800
categories: [C/C++]
tags: [C/C++]
---

## 简单介绍
1. 结构体的占用大小在大多数情况下都不会直接等于各个成员大小的总和,因为会有一个对齐内存的操作
2. 为了移植原因,这是在不同的平台下,如果没有对齐可能会抛出异常
3. 为了提高CPU的读取效率,因为CPU一般都是内存读取都是以N的整数倍,如果没有对齐,就可能需要读取两次才能将数据读完,而对齐后,可以一次性读完数据,提高效率

## 对齐规则

### 规则1
```cpp
struct Data{
    char c;
    short s;
    int i;
};

int main(){
    int c=sizeof(char);
    int s=sizeof(short);
    int i=sizeof(int);
    printf("sizeof(char)=%d\n",c); 
    printf("sizeof(short)=%d\n",s); 
    printf("sizeof(int)=%d\n",i); 
    printf("sizeof(Data)=%ld\n",sizeof(Data)); 
    return 0;
}
```
执行结果
```
sizeof(char)=1
sizeof(short)=2
sizeof(int)=4
sizeof(Data)=8
```
在结构体中有分别有char,short,int三个基本类型的成员,按照其大小直接相加是7,但是在执行结构来看却是8,那么这里可以引出对齐的**规则1**

**<u>结构体中的成员按照定义顺序依次置于内存中，但不一定是紧密排列。从结构体首地址开始依次将成员放入内存时， 会根据成员自身的对齐大小,放在其整数倍的地址上</u>**

要知道结构体首地址偏移量为0,以下是结构体Data的地址占位,结构体Data中第一个成员的类型是char,对其大小为1,地址为0,按照规则1,第二个成员的类型是short,对齐大小是2,那么地址为2,符合其自身的对齐大小,放在其整数倍的地址结构上,那么刚好第三个成员的类型是ini,对齐大小是4,那么地址为4也符合规则1
那么综上分析,就是因为short的地址偏移一位,没有从1开始,导致结构体大小比预想中的大

| 0    | 1   | 2   | 3   | 4   | 5   | 6   | 7   |
| ---  | --- | --- | --- | --- | --- | --- | --- |
| char |     |short|     | int |     |     |     |

### 规则2
那么改动结构Data的成员顺序
```cpp
struct Data{
    int i;
    short s;
    char c;
};

int main(){
    int c=sizeof(char);
    int s=sizeof(short);
    int i=sizeof(int);
    printf("sizeof(char)=%d\n",c); 
    printf("sizeof(short)=%d\n",s); 
    printf("sizeof(int)=%d\n",i); 
    printf("sizeof(Data)=%ld\n",sizeof(Data)); 
    return 0;
}
```
现在在已知规则1的情况下,预测下Data的大小为7,三个成员的地址如下

| 0   | 1   | 2   | 3   | 4   | 5   | 6   | 7   |
| --- | --- | --- | --- | --- | --- | --- | --- |
| int |     |     |     |short|     | char|     |

执行结果
```
sizeof(char)=1
sizeof(short)=2
sizeof(int)=4
sizeof(Data)=8
```
从结果上来看,结构体Data的大小依旧为8,并非如同预测那般为7,那么这里可以引出**规则2**

**<u>如果结构体的大小不是所有成员中最大对齐大小的整数倍，则结构体对齐到最大成员对齐大小的整数倍，填充放置到结构体末尾</u>**

那么在结构体Data中,所有成员中最大的对齐大小为4,按照成员顺序计算出大小后7,并非为4的整数倍,所以加1,结构体大小为8,正好为4的倍数

### 规则3
修改结构体Data, 新增结构体SubData,根据**规则1**和**规则2**,可以算出结构体SubData大小为24,成员中最大对齐数是8

```cpp
struct SubData{
    char c;
    double d;
    int i;
};

struct Data{
    char c;
    short s;
    short s1;
    short s2;
    SubData sub;
};

int main(){
    int c=sizeof(char);
    int s=sizeof(short);
    int i=sizeof(int);
    int d=sizeof(double);
    printf("sizeof(char)=%d\n",c); 
    printf("sizeof(short)=%d\n",s); 
    printf("sizeof(int)=%d\n",i); 
    printf("sizeif(double)=%d\n",d);
    printf("sizeif(SubData)=%ld\n",sizeof(SubData));
    printf("sizeof(Data)=%ld\n",sizeof(Data)); 
    return 0;
}
```

执行结果
```
sizeof(char)=1
sizeof(short)=2
sizeof(int)=4
sizeif(double)=8
sizeif(SubData)=24
sizeof(Data)=32
```
结构体SubData的大小和预想的一样,然后按照规则1,把成员都排列出来,引出**规则3**

| 0   | 2   | 4   | 6   | 8  | 16   |  24 | ... |
| --- | --- | --- | --- | ---| ---  | --- | --- |
| char|short|short|short|char|double| int | ... |

**<u>如果结构体A中嵌套了另外一个结构体B为成员C,那么成员C的对齐数就是结构体B中的成员里最大对齐数</u>**

其实**规则3**可以换一种更好理解的说法,直接把结构体B中的成员按顺序直接添加到结构体A中成员C的地方,再按照**规则1**和**规则2**计算即可

在结构体中出现数组的情况下也是类似的思路,按照这个数组在结构体中的位置,把数组每个成员直接按顺序放在结构体中按照**规则1**和**规则2**计算即可

