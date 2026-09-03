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

## 2. 裸机 C 语言实现简易消息队列

裸机：无操作系统，=={yellow}单主循环 + 中断==，消息队列用于**中断给主循环传递数据**，=={yellow}核心是环形缓冲区==。

3.
