# n阶乘末尾有多少个0

这是一个经典的数学问题。阶乘末尾的0来自于因子 10 ，而 10=2×5 。在阶乘中，因子 2  的数量总是多于因子 5  的数量，因此末尾 0  的数量**等于 \*n\*!  中因子 5  的个数**。

**计算原理：**

- 每 5  个数贡献至少一个因子 5 （如 5,10,15... ）→ ⌊*n*/5⌋ 
- 每 25  个数额外贡献一个因子 5 （如 25,50...  包含 52 ）→ ⌊*n*/25⌋ 
- 每 125  个数再额外贡献一个 → ⌊*n*/125⌋ 
- 以此类推...

```c++
#include <iostream>
using namespace std;

long long countZeros(long long n) 
{
    long long count = 0;  // 记录因子5的总个数
    for (long long i = 5; i <= n; i *= 5)//每i*5个数额外贡献一个5，所以每层循环是i*5
    {
        count += n / i;//第一轮算每五个数的保底5个数
        				//第二轮算每25个数带来的额外5个数
        				//第三轮125......
        				//每轮累加
    }
    return count;
}

int main() 
{
    long long n;
    long long result = countZeros(n);    // 计算并输出结果
    cout << n << "! 末尾有 " << result << " 个0" << endl;
}

```

