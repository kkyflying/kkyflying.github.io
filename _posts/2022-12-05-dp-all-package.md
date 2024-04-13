---
layout: post
math: true
title:  "完全背包"
date:   2022-12-05 23:17:24 +0800
categories: [算法,动态规划]
tags: [动态规划]
mermaid : true
---


| 物品编号 | 重量w | 价值v |
| ---- | --- | --- |
| 0    | 1   | 15  |
| 1    | 3   | 20  |
| 2    | 4   | 30  |

背包的最大重量为4，每个物品可以装一次或者多次或者不装，问装满这个背包的最大价值是多少？
### 动态规划解法

#### 1. 确定dp数组的以及下标的含义
1.  i是物品的编号，j是背包重量
2.  dp[i][j]是当背包重量为j的时候，在[0,i]物品之间任取n个物品，所的到最大价值

#### 2. 确定递推公式

##### 不放入第i物品
1.  当背包重量为j并j>=weight(i),那么得出：
    ```dp[i][j]=dp[i-1][j]```
2.  当背包重量为j并j\<weight(i),那么实际上无法放入i物品，那么得出：
    ```dp[i][j]=dp[i-1][j]```

##### 放入第i物品
当背包重量为j并j>=weight(i)，同是因为同一种物品可以放入无限个，因此当重量为j的时候，能放入k个物品i,那就取得当放入几个i物品时候(也可以不放即k=0)，背包里获得最大值
```
k=j/weight(i)
dp[i][j] = max(
dp[i-1][j-0*weight[i]]+0*values[i] , 
dp[i-1][j-1*weight[i]]+1*values[i] ,
dp[i-1][j-2*weight[i]]+2*values[i] ,
....
dp[i-1][j-k*weight[i]]+k*values[i] 
)
```

#### 3. dp数组如何初始化
1.  当背包重量j为0的时候，什么物品都无法放入，价值为0,所以dp[i][0]=0
2.  当选取第0个物品的时候，只有当背包重量j大于等于weight(0)的时候才能把物品0放入，并重复放入k=j/weight(0)个物品，保证背包的价值最大
3.  初始化的结果如下

| 物品编号i\背包重量j | 0 | 1  | 2  | 3  | 4  |
| ----------- | - | -- | -- | -- | -- |
| 物品0         | 0 | 15 | 30 | 45 | 60 |
| 物品1         | 0 | 0  | 0  | 0  | 0  |
| 物品2         | 0 | 0  | 0  | 0  | 0  |

```
for(int j=1 ; j<=maxWeight ; j++){
        int k= j/weight[0];
        dp[0][j]= values[0]*k;
}
```

#### 4. 确定遍历顺序
还是先遍历物品再遍历重量,按照递推公式得到如下：
```cpp
    for(int i=1 ; i<num ; i++){
        for(int j=0 ; j<=maxWeight ;j++ ){
            if(j<weight[i]){
                dp[i][j]=dp[i-1][j];
            }else{
                 int k = j/weight[i];
                 int tempmax = 0;
                 for(int n=0;n<=k;n++){
                    int temp = dp[i-1][j- (n*weight[i]) ] + (n*values[i]);
                    if( tempmax < temp ){
                        tempmax=temp;
                    }
                 }
                 dp[i][j]=tempmax;
            }
        }
    }
```
从递推公式来看，01背包和完全背包都是依赖正上方和左上方的数据，只是完全背包需要对比放入不同数量的i物品时那个价值更大

#### 5. 举例推导dp数组

因为这个数据少，可以完全举例出来如下：

| 物品编号i\背包重量j | 0 | 1  | 2  | 3  | 4  |
| ----------- | - | -- | -- | -- | -- |
| 物品0         | 0 | 15 | 30 | 45 | 60 |
| 物品1         | 0 | 15 | 30 | 45 | 60 |
| 物品2         | 0 | 15 | 30 | 45 | 60 |

#### 6. 完整代码

```cpp
#include <iostream>
using namespace std;

#include <stdio.h>
#include <vector>
#include <math.h>

int maxWeight=4;
int num=3;
vector<int> weight={1,3,4};
vector<int> values={15,20,30};
vector<vector <int> > dp(weight.size(), vector<int>(maxWeight + 1, 0));//已经对全部dp[i][j]初始化为0
// vector<int> dp(num,0); //dp[j]q全部初始化为0

int main(){
    //init
    for(int j=1 ; j<=maxWeight ; j++){
        int k= j/weight[0];
        dp[0][j]= values[0]*k;
    }
    for(int i=1 ; i<num ; i++){
        for(int j=0 ; j<=maxWeight ;j++ ){
            if(j<weight[i]){
                dp[i][j]=dp[i-1][j];
            }else{
                 int k = j/weight[i];
                 int tempmax = 0;
                 for(int n=0;n<=k;n++){
                    //dp[i][j] = max( dp[i-1][j] , dp[i-1][j-n*weight(i)]+n*values(i) );
                    int temp = dp[i-1][j- (n*weight[i]) ] + (n*values[i]);
                    if( tempmax < temp ){
                        tempmax=temp;
                    }
                 }
                 dp[i][j]=tempmax;
            }
        }
    }
    printf("max=%d\n",dp[2][4]);
	return 0;
}
```

## 细节点
### 1. 遍历的顺序是否能交换？
答案：是可以交换的
原因：还是先看如下表格，dp[i][j]都是依赖正上方和左上方的数据来推导出，
1. 当先遍历物品再遍历背包重量，就是先从左到右再从上到下，符合上述dp[i][j]的推导
2. 当先遍历背包重量再遍历物品，就是先从上到下再从左到右，依旧是符合dp[i][j]的推导

| 物品编号i\背包重量j | 0 | 1  | 2  | 3  | 4  |
| ----------- | - | -- | -- | -- | -- |
| 物品0         | 0 | 15 | 30 | 45 | 60 |
| 物品1         | 0 | 15 | 30 | 45 | 60 |
| 物品2         | 0 | 15 | 30 | 45 | 60 |

### 2. 背包重量是否可以从大到小？
答案：不可以
原因：当背包重量从大到小，等于是从右到左，没有数据来满足dp[i][j]的推导
