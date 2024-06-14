---
layout: post
math: true
title:  "不同路径"
date:   2024-06-13 22:56:28 +0800
categories: [算法,动态规划]
tags: [动态规划]
mermaid : true
---

## 不同路径1
从一个m*n网格的最左上角开始，每次只能向下或者向右移动一步，试图达到网格的最右下角，问：共有多少条不同的路？

![](/img/diff-path_1.svg)

从上的网格来看，水平方向为i，竖直方向为j，到格子(i,j)的位置的路线是由正上方（i,j-1）和左边的（i-1，j）的和

1. 确定dp数组以及下标的定义

    (0,0)是起点，（i,j）是终点，dp[i][j]从(0,0)到(i,j)有多少条路径

2. 确定递推公式

    dp[i][j]=dp[i][j-1] + dp[i-1][j]  

3. dp数组初始化

    当在dp[0][0]的时候，就是只有一条路，而到dp[0][j]和dp[i][0]的时候都是只有一条路，因为d[0][0]=dp[i][0]=dp[0][j]=1

4. 确定遍历顺序

    因为需要依赖正上方和左边的数据，所以要先遍历i再j，但是初始化的时候，最边上已经有数据了，所以先i后j或者先j后i都是可以的

5. 举例推导dp数组

| 1 | 1 | 1 | 1 | 1 | 1 | 1 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| 1 | 3 | 6 | 10 | 15 | 21 | 28 |

```cpp
#include <iostream>
using namespace std;

#include <stdio.h>
#include <vector>
#include <math.h>

#include "time.h"

//g++ -std=c++11 path2.cpp ; ./a.out

int M=3;
int N=7;
vector< vector<int> > dp ( M , vector<int>( N,0) );

int main(){

    for ( int i = 0; i < M; i++){
        dp[i][0] = 1;
    }

    for ( int j = 0; j < N; j++){
        dp[0][j] = 1;
    }

    for(int i = 1 ; i < M ; i++){
        for(int j = 1 ; j < N ; j++){
            dp[i][j]=dp[i][j-1] + dp[i-1][j];
        }
    }

    for(int i = 0 ; i < M ; i++){
        for(int j = 0 ; j < N ; j++){
            printf("%d\t",dp[i][j]);
        }
        printf("\n");
    }



}

```

上面的代码可以优化成滚动数组的版本，第一行的的数据是一开始就明确 dp[0][j]=1 第一列的数据dp[i][0]=1,而dp[i][j]仅需要正上方和左边的数据，满足上一层的数据可以重复使用，直接拷贝到当前层，并且不需要历史记录, 优化后的代码如下：

```cpp
#include <iostream>
using namespace std;

#include <stdio.h>
#include <vector>

int M=3;
int N=7;
vector< int > dp ( N , 1 );

int main(){

    for(int i=1 ; i < M ; i++){
        for(int j=1 ; j< N; j++){
            dp[j]=dp[j]+dp[j-1];
            printf("%d\t",dp[j]);
        }
        printf("\n");
    }
    printf("%d\n",dp[N-1]);
}
```

## 不同路径2
从一个m*n网格的最左上角开始，每次只能向下或者向右移动一步，试图达到网格的最右下角，当中间有障碍的时候，不可通行，问：共有多少条不同的路？

![](/img/diff-path_2.svg)


基本上和上面的思路一直，但是要多个判断，

1. 当(i,j)是障碍的时候，那么dp[i][j]=0
2. 还有障碍是有可能出现在(0,j)和(i,0)的位置上，那么在初始化的时候，假如(0,2)是障碍，那么dp[0][j>=2]=0，初始化的值，例如：

| 开始1 | 1 | 障碍0 | 0 |
| --- | --- | --- | --- |
| 1 |  |  |  |
| 1 |  |  | 结束 |

```cpp
#include <iostream>
using namespace std;

#include <stdio.h>
#include <vector>

//g++ -std=c++11 path5.cpp ; ./a.out

int M=3;
int N=4;
vector< vector<int> > dp ( M , vector<int>( N,0) );

//这个写法只是为了更加直观一下
int refer[3][4] = {
    {0,0,1,0} , 
    {0,1,0,0} , 
    {0,0,0,0} ,
};
//优化写法：
//vector< vector<int> > refer ( M , vector<int>( N,0) );
//refer[0][2] = 1;
//refer[1][1] = 1;

int main(){

    for ( int i = 0; i < M; i++){
        if(refer[i][0]==1){
            break;
        }
        dp[i][0] = 1;
    }
    
    for ( int j = 0; j < N; j++){
        if(refer[0][j]==1){
            break;
        }
        dp[0][j] = 1;
    }

    for(int i = 1 ; i < M ; i++){
        for(int j = 1 ; j < N ; j++){
            if(refer[i][j]==1){
                dp[i][j]=0;
            }else{
                dp[i][j]=dp[i][j-1] + dp[i-1][j];
            }
        }
    }

    for(int i = 0 ; i < M ; i++){
        for(int j = 0 ; j < N ; j++){
            printf("%d\t",dp[i][j]);
        }
        printf("\n");
    }
}
```

当然也可以优化成滚动数组的做法

1. 要初始化dp[j]，根据障碍来判断
2. 每一次往下一层计算前，都要判断dp[0]是1或0。因为最左边的列只要中间出现一个障碍，后面的都应该初始化为0，但是滚动数组无法保存，只能每次都判断dp[0]的值

```cpp
#include <iostream>
using namespace std;

#include <stdio.h>
#include <vector>


//g++ -std=c++11 path5.cpp ; ./a.out

int M=3;
int N=4;
// vector< vector<int> > dp ( M , vector<int>( N,0) );
vector< int > dp ( N , 0 );

int refer[3][4] = {
    {0,0,1,0} , 
    {1,0,0,0} , 
    {0,0,0,0} ,
};


int main(){

    for ( int j = 0; j < N; j++){
        if(refer[0][j]==1){
            break;
        }
        dp[j] = 1;
    }
    
    int i_f = 0;
    for(int i=0 ; i<M ; i++){
        if(refer[i][0]==1){
            i_f = i;
            break;
        }
    }

    for(int i = 1 ; i < M ; i++){
        if(i_f>0 && i_f<=i){
            dp[0]=0;
        }else{
            dp[0]=1;
        }
        printf("%d\t",dp[0]);
        for(int j = 1 ; j < N ; j++){
            if(refer[i][j]==1){
                dp[j]=0;
            }else{
                dp[j]=dp[j]+dp[j-1];
            }
            printf("%d\t",dp[j]);
        }
        printf("\n");
    }

}
```
