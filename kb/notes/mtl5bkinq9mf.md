# 一、汇川

## =={pink}1. C 语言结构体字节对齐，如何消除填充 (padding)字节==

CPU 默认会按照**自然对齐**给结构体插入填充字节，提升内存访问效率；但有时候做协议、二进制报文、寄存器映射时，我们**不要 padding**，需要**紧凑结构体**。

> ⚠️重要警告：
> 
> 
> 1. 取消对齐会产生**非对齐访问**，ARM、Xtensa (ESP32) 非对齐访问有的会崩溃、有的性能暴跌；只适合：网络协议、flash 存储、二进制序列化，**不要用来定义普通运行时变量**。
> 2. 不同编译器语法不一样：GCC (GCC/ESP‑IDF)、MSVC。

## 1.GCC / ESP‑IDF / ARM‑GCC（最常用）

### =={yellow}方式 1：==`__attribute__((packed))` =={yellow}压缩属性==

```
// 整个结构体取消填充，紧凑排列
typedef struct __attribute__((packed)) {
    uint8_t  a;   // 1字节
    uint32_t b;   // 4字节
    uint16_t c;   // 2字节
}my_pkt_t;
// 总大小 = 1+4+2 =7字节，没有padding
```

=={yellow}也可以写在结构体后面==：

```
typedef struct {
    uint8_t  a;
    uint32_t b;
    uint16_t c;
} __attribute__((packed)) my_pkt_t;
```

> `__attribute__((packed))`=={yellow}：告诉编译器==**=={yellow}不要插入任何填充字节==**=={yellow}。==

### 风险点：非对齐访问！

### =={yellow}方式 2：修改对齐字节== `__attribute__((aligned(N)))`

`aligned(N)` 是设置**整个结构体的起始对齐要求**，不是消除 padding；
packed = 等价于 `aligned(1)`，按 1 字节对齐，不填充。

---

## =={pink}2. 裸机 C 语言实现简易消息队列==

裸机：无操作系统，=={yellow}单主循环 + 中断==，消息队列用于**中断给主循环传递数据**，=={yellow}核心是环形缓冲区==。

核心：**环形 FIFO 数组 + 头指针 head (写)、尾指针 tail (读)**，裸机多用于中断写、主循环读。

1. 开辟一块固定大小数组作为存储缓冲区；定义 head（写入位置）、tail（读出位置）两个变量。
2. 初始化：head=0，tail=0。
3. 判断满：head 向后走一步等于 tail，则队列满。
4. 判断空：head 等于 tail，则队列空。
5. **入队（写）**：队列没满，把数据放到 head 指向位置，head 下标 + 1，超过数组长度就绕回 0。

> 如果中断和主循环都修改同一指针，访问 head/tail 前后加临界区（关中断）。

1. **出队（读）**：队列不为空，读取 tail 位置的数据，tail 下标 + 1，超过数组长度绕回 0。
2. 使用约束：单生产者单消费者可以不用锁；多生产者 / 多消费者必须临界区保护共享指针；队列尽量短，不要存超大块数据。

---

## =={pink}3. 裸机实现临界区、原子操作==

裸机没有 OS，=={yellow}临界区本质：==**=={yellow}关闭全局中断==**=={yellow}，防止中断打断共享变量读写==。
### 原子操作两种方案

1. **=={yellow}临界区包裹==**=={yellow}：非原子的运算，包进临界区，裸机最通用。==

```
ENTER_CRITICAL();
g_val++;
EXIT_CRITICAL();
```

1. **=={yellow}硬件原子指令==**：Cortex‑M3/M4/M7 支持 LDREX/STREX 独占指令，实现无锁原子加减，CMSIS 提供：

```
#include "cmsis_gcc.h"
uint32_t val = __LDREXW(&g_var);
val++;
__STREXW(val, &g_var);
```

> =={green}裸机一般简单场景直接关中断临界区就够用。==

---

## 4. ESP32 WebServer
