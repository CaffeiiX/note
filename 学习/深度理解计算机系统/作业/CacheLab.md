## Part A: Writing a Cache Simulator

## Part B: Optimizing Matrix Transpose
### 题目描述
在`trans.c`文件中实现矩阵的转置，其中$A$代表矩阵，$A_{ij}$ 代表第$i$行，第$j$列，实现$A_{ij} = A^T_{ji}$
```C
char trans_desc[] = "Simple row-wise scan transpose";
void trans(int M, int N, int A[N][M], int B[M][N]);
```
文件中提供的原生的转置函数的效率一般，需要实现一个类似的函数`transpose_submit`
```C
char transpose_submit_desc[] = "Transpose submission"; 
void transpose_submit(int M, int N, int A[N][M], int B[M][N]);
```
注意：不要改变“Transpose submission”的描述，自动评分器搜索此字符串以确定要评估效率的转置函数。
规则：
1. 最多只能定义12个int的局部变量
2. 不允许通过使用任何 long 类型的变量或使用任何位技巧将多个值存储到单个变量来回避前面的规则。
3. 你的转置函数可能不适用递归
4. 如果选择使用帮助程序函数，则在帮助程序函数和顶级转置函数之间，堆栈上的局部变量一次不得超过 12 个。例如，如果你的转置声明了 8 个变量，然后你调用了一个使用 4 个变量的函数，该函数调用了另一个使用 2 的函数，那么堆栈上将有 14 个变量，你将违反规则。
5. 你的转置可能不会修改原矩阵
6. 不允许在代码中使用malloc定义数组
参数采用的是$s=5,E=1,b=5$


### 优化思路
行优先读取有助于内部优化，但是对于转置而言，其中一个数组是通过行优先的方式进行读取，那么转置结果将通过列优先进行写入。

