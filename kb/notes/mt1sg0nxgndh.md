# 一、MCU补充八股

## 01 单片机为什么需要置位清零而不是手动清零

- **应用场景**

单片机=={yellow}一个寄存器==里通常包含=={yellow}多个功能位==。

=={pink}修改某一位时，要避免影响其他==已经=={pink}配置好的位==。

- **底层实现原理**

1. =={green}置位 / 清零属于==**=={green}按位操作==**=={green}，只修改目标位，其他位保持不变==。
2. 如果手动把整个寄存器清零再写入，容易把其他控制位也一起改掉。
3. 很多寄存器里还有保留位、状态位、写 1 清零位，整寄存器直接写风险更大。
4. 所以实际开发里一般用 `|=` 和 `&=~` 做置位清零，保证修改更精确、更安全。

---

## 02 stm32 的启动过程

- **应用场景**

面试里常用来考察你对=={yellow}上电后程序是怎么跑起来==的是否真正理解。

也经常延伸到 BootLoader、向量表、中断初始化这些问题。

- **底层实现原理**

1. STM32 上电或复位后，=={pink}内核==先从=={pink}固定启动地址取出中断===={pink}向量表==中的=={pink}前两个值==。
2. 第=={yellow}{pink} 1 ==个值加载到 =={pink}MSP==，作为主堆栈指针；第 =={pink}2== 个值加载到 =={pink}PC==，跳到=={pink} ==`Reset_Handler`=={pink} ==执行。
3. `Reset_Handler` 里会完成=={green}运行时初始化==，比如把 Flash 中的 `.data` 拷贝到 RAM、把 `.bss` 清零。
4. 然后执行=={pink} ==`SystemInit()`=={pink}，==完成时钟、FPU、向量表偏移等=={green}底层初始化==。
5. 最后=={pink}进入 ==`main()`，此时 C 运行环境才算真正准备完成，用户程序开始执行。

- **延伸知识点**
- STM32 的 Boot 模式和启动地址是怎么选择的

根据boot0和boot1的0/1组合决定
![image](data:image/png;base64,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NuspMVub7apaiYAIiIAIiIAIiEBdEJCYrYtmViVFoPIEVKIIiIAIiIAIVIKAxGwlKKsMERABERABERABEchPQCFlEJCYLQOekoqACIiACIiACIiACFSXgBOzCxcutPnz58s1g0GnTp3Eqxm8wv1L/Kp4v5XRbuF21LXa0fcB3dPqC74v5Dqqf6h/5OoXQT+0aEslsROzLU2sdCIgAiIgAiIgAiJQywRUt+QTkJhNfhvJQhEQAREQAREQAREQgTwEJGbzgJG3CFSegEoUAREQAREQARFoLgGJ2eYSU3wREAEREAEREIHqE5AFIrCUgMTsUhA6iIAIiIAIiIAIiIAIpI+AxGz62kwWV56AShQBERABERABEUgoAYnZhDaMzBIBERABERCBdBKQ1SJQWQISs5XlrdJEQAREQAREQAREQAQiJCAxGyFMZVV5AipRBERABERABESgvglIzNZ3+6v2IiACIiAC9UNANRWBmiQgMVuTzapKiYAIiIAIiIAIiEB9EJCYrY92rnwtVaIIiIAIiIAIiIAIVICAxGwFIKsIERABERABEShEQGEiIAItJyAx23J2SikCIiACIiACIiACIlBlAhKzVW6Ayhdf+yXefPPN9swzzxjHfLU9/PDD7fHHH3fxiIvjGn8c54XSE/bII4/YoEGDXBEcucYfjzPOOMPlzZHrQm6LLbawe+65JxOfNFzjXyidwkSgFgn4e8nfA/6a+4vzXHW+9NJL3f3DfYwjLuk5z+W4v7nPyYs8ffzwPce9SHryJ27YkQd5+fs+GF4oLBhP56UToH3ytWu4DWgz2o42DJfg24a24zwc3pxr0pMP5eVLhw3Yki9OONxfkyaXI9yXRZ7ECfr5sHo6RiZmAQlQ75588kn7xz/+kZnsPdSNN97YRo0aZY8++qgbfIhH5zzkkEN8lMyxWNxCHRs7aGSf2R577GG333673XTTTd4rEcdSufXt29euueYamzRpUobbrbfeatQrXJGouLVr185GjBhhjz32mD399NP24IMP2mGHHRYurqrXfiKivb3r1auXs4mj9/NHeBMIy379+tk222yTcVzjT3hzHIPo2WefbfDyZfp24chAx4CXL8+XXnrJFixY4ILbtm1rtHW3bt2MPOnjLkB/RKAOCHAvDR06NKumm2++ubu3PvnkE3vggQeywvzFsccea2PGjLFvv/3WXnzxRdt5553dPfXVV1/ZOeecY1OnTnVhxCHcp8t1ZN5gvPBjhS2NxL3IXIWNS710qBIB366M37Qv15jCOMt4SxtyncsRB72x7LLLGm748OFuTqXNg863v293H0b+5JEr77Af8YjPPEDYlltu6coq1ofOP/98Ny/5vkr/pa7ekRf2FKoncerJRSZmPbTXXnvNJkyYYG+88YZ1797dhg0bZj179nTBiKz//u//do00f/58mzhxok2ZMsXat29vNDoDkovY+KeUuLNnz3aimPLoMF9//bXhxzXuueeesx122MENcqeddpqzp6GhoTH35P1fiNuOO+5of/zjH22jjTayjz76yInKN99801ZddVU7/vjj7YADDshUKCpuZHjiiSfawIED7f333zeEc6tWrexXv/qV7bLLLgQnytHuxxxzjOtb/oYPHv2gUMxoBhkGNwY5L0wZNOhf9FEc54QhXhGctJ0fUP2gc++997qiOCKSJ0+enFl9Jb+wIz8SwBYhyzlH7h/O5USg1gkgGjp06JBVzXXXXdcJUzy5R8L3DeKSdITH6RAhl112mXFPzpkzJ86ilHeMBBB/jO8ffvhhzrmC8ZvimU9YvOHcO8IYz7lGs9D36I/kx3xBH+Ga+YF5gnjeMf8wH/l5ok2bNlnzQVDsMgeRjjw222wzQ6iPGzfOsJ0yK9HfKT9tLnIxS4Ofe+65dtVVV9ncuXNtndX4AwAAEABJREFU+eWXt86dOzsu+++/vxO2NCwC7MwzzzQECIMEQpRVNhqcyKXERSyPHj3aKI/Vw++//949jXONu/HGG2299dazrl27upXFRYsWkXUi3dSpU109cnFjpaJTp07uIQFurJbS0RGYyyyzjA0ZMsRxpWJRcdtggw2Mm+aLL76w6667zi655BLHkMmGlRLKqkV34IEHuocfv8JDf6RP+7qycstTMwMMjsGJgY+neMStn3D94MSRwYnV1z333DPnAEoZ3BOUwZFr74IPeITLiUCtEkAoIhLWX399txLLNQ/r3FcPP/ywW6h49dVXs47FWJCWB07uSwQHwgPR4dMx55xyyimZ8piLfDj3Lo643Jes6jIucI3jviY/8iV/RMyFF17oti8RRhy5eAj4dkU80r5cFyuJNkEc0o68HUYYIhBJx1zHNe3IeM9YzZhNWCFHXPKjX9BHvNgNp6FPBW395ptvjDL8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)

- 中断向量表为什么前两个元素是 MSP 和 Reset_Handler

---

## 03 STM32 定时器如何配置

- **应用场景**

STM32 定时器常用来做 =={pink}定时中断、PWM 输出、输入捕获、计数==。

=={yellow}本质==就是先让=={green}定时器按设定频率计数，再决定什么时候触发事件==。

- **底层实现原理**

1. 先=={pink}开启 TIM 外设时钟==，否则定时器寄存器不工作。
2. 配置=={pink}预分频器 PSC==和=={pink}自动重装值 ARR==，决定计数频率和溢出周期。
3. 根据=={green}需求选择模式==，比如 基本定时、中断、输出比较、PWM、输入捕获。
4. 如果要=={pink}定时中断==，还要配置=={yellow}DIER、NVIC==，让=={pink}更新事件能进入中断函数==。
5. =={green}最后使能定时器 CEN，计数器开始运行==，到达条件后产生更新事件或输出波形。
> PWM（Pulse Width Modulation，脉宽调制）
- **延伸知识点**
- PWM 频率和占空比怎么计算
- 输入捕获是怎么测频率和脉宽的

---

## 04 介绍一下 STM32 ARM 体系架构

- **应用场景**

面试里这个问题常用来考察你对 STM32 为什么能高效响应中断、执行代码、访问外设 的整体理解。

实际开发中，理解 ARM 体系架构也有助于看懂启动流程、异常机制和寄存器操作。

- **底层实现原理**

1. STM32 内核大多基于 ARM Cortex‑M 架构，属于面向微控制器的 RISC 体系，特点是指令精简、执行效率高。
2. Cortex‑M 采用**哈佛结构**思想，指令和数据访问通路分离，所以取指和访存效率更高。
3. 内核通过**总线矩阵**连接 Flash、SRAM 和各类外设，程序执行时本质上就是 CPU 不断取指、访存、控制外设寄存器。
4. ARM 架构里定义了=={yellow}寄存器组==、异常 / 中断机制、堆栈模型、权限级别**，STM32 则在这个基础上集成了 GPIO、USART、TIM、ADC 等片上外设。
5. 所以可以理解为：ARM Cortex‑M 提供通用计算核心，STM32 在外面封装了存储器和各种外设，最终形成完整单片机。

---

## 05 MCU 上如何设计用户态和内核态？如何保障操作系统的安全性？

- **应用场景**

在 MCU 上做 RTOS / 嵌入式 OS 时，如果既想让应用任务能跑，又不想它随便改内核数据、乱配外设，就要区分**用户态和内核态**。

常见在 车载、工业、医疗、安全设备 这类对隔离性和可靠性要求高的系统里。

- **底层实现原理**

1. MCU 上的用户态 / 内核态本质是利用 CPU 提供的**特权级**机制，比如 Cortex‑M 的 Thread mode + Handler mode、Privileged + Unprivileged。
2. 普通任务运行在用户态，只能访问被允许的代码区、数据区和外设；涉及中断控制、任务切换、时钟配置这类敏感操作时，必须通过 **SVC 系统调用**进入内核态。
3. 内核再配合 **MPU** 做内存保护，给不同任务划分独立的代码段、数据段、栈空间，非法访问时触发 MemManage/HardFault。
4. 这样可以防止应用任务误改内核、越界踩内存、随意访问关键外设，从而把 “功能错误” 尽量限制在任务自身。
5. 操作系统安全性本质靠三层：**特权隔离 + 内存隔离 + 异常兜底**，再配合栈溢出检测、参数检查、看门狗，提升系统可控性。

---

## 06 单片机中断的过程

- **应用场景**

中断用于**外设事件来了立即处理**，避免 CPU 一直轮询浪费时间。

常见场景有按键触发、串口收发、定时器到时、ADC 转换完成。

- **底层实现原理**

1. 外设产生中断请求后，先送到 **NVIC**，NVIC 判断该中断是否使能、优先级是否满足响应条件。
2. 如果可以响应，CPU 会暂停当前任务，并把当前现场自动压栈保存，主要包括 PC、LR、xPSR、R0‑R3、R12。
3. 然后 CPU 根据**中断向量表**取出对应中断服务函数入口地址，跳转执行 ISR。
4. ISR 中一般先判断中断源，再处理事件，最后清除中断标志，否则可能会重复进入中断。
5. ISR 执行结束后，CPU 通过异常返回恢复现场，回到之前被打断的位置继续执行。

---

## 08 大端和小端区别

- **应用场景**

大端小端描述的是**多字节数据在内存中的存放顺序**。

常见在 通信协议、寄存器解析、跨平台数据传输 里考察。

- **底层实现原理**

1. **大端**：高字节放低地址，低字节放高地址，符合人阅读数字的顺序。
2. **小端**：低字节放低地址，高字节放高地址，很多 MCU/CPU 都采用这种方式。
3. 比如 `0x12345678`，大端存放是 `12 34 56 78`，小端存放是 `78 56 34 12`。
4. 大小端只影响**内存中的字节排列**，不影响寄存器里看到的数值本身。
5. 所以问题本质不是 “值变了”，而是 “同一个值在内存里怎么排”。

---

## 09 51 和 32 架构的区别

- **应用场景**

51 和 32 位单片机的区别，本质是**老式 8 位 MCU 和现代 32 位 MCU 的能力差异**。

面试里一般想听你从 位宽、性能、资源、外设、开发方式 几个方面去对比。

- **底层实现原理**

1. **数据位宽不同**：51 是 8 位架构，一次处理 8 位数据更高效；32 单片机一般是 32 位架构，处理 32 位数据、地址和运算能力更强。
2. **存储空间不同**：51 的 RAM、Flash 一般较小，地址空间受限；32 位单片机资源更大，能跑更复杂的程序和协议栈。
3. **内核能力不同**：51 指令执行效率低，很多操作需要多条指令完成；32 位 MCU 多基于 ARM Cortex‑M，主频更高，中断、乘除法、堆栈管理都更强。
4. **外设和总线能力不同**：51 外设种类少、功能简单；32 位单片机通常带有更丰富的 TIM、ADC、DMA、SPI、I2C、USART、CAN、USB 等。
5. **开发方式不同**：51 更偏寄存器级和小程序控制；32 位单片机更偏模块化开发，常配合 HAL/LL、RTOS、中间件做复杂应用。

> 51 更简单直接，32 位单片机初始化更复杂，但功能更完整、扩展性更强。

- **延伸知识点**
- 哈佛结构和冯诺依曼结构的区别
- STM32 比 51 更适合跑 RTOS 的原因

---

## 10 32 如何实现 PWM 调速

- **应用场景**

32 位单片机一般通过 **PWM 改变电机两端的平均电压** 来实现调速。

常见于风扇、电机、小车、电调、舵机控制。

- **底层实现原理**

1. STM32 用**定时器**产生固定频率的 PWM 波，核心参数是 `PSC`、`ARR`、`CCR`。
2. 其中 `ARR` 决定周期，`CCR` 决定高电平时间，占空比 = `CCR / ARR`。
3. 电机由于有电感和机械惯性，看到的不是快速翻转的高低电平，而是一个**平均电压**。
4. 占空比越大，平均电压越高，电机转速通常越快。
5. 所以 PWM 调速本质就是：**频率基本固定，动态调整占空比**。

- **延伸知识点**
- PWM 频率过高或过低会有什么影响
- 电机调速为什么常配合 H 桥或驱动芯片

---

## 11 常见的 CPU 寄存器有哪些

- **应用场景**

CPU 寄存器是程序运行时最核心的硬件资源，用来**存数据、存地址、控流程**。

面试问这个，一般是想看你是否理解函数调用、中断、程序跳转这些底层过程。

- **底层实现原理**

1. 常见寄存器可分为 **通用寄存器、程序控制寄存器、状态寄存器、堆栈相关寄存器**。
2. 通用寄存器如 `R0‑R12`，主要用来存临时变量、函数参数、运算中间结果。
3. `PC` 保存下一条要执行的指令地址，`LR` 保存函数返回地址，`SP` 指向当前栈顶。
4. `PSR/xPSR` 用来保存条件标志和当前处理器状态，比如零标志、进位标志、中断状态。
5. 所以程序运行本质上就是 CPU 不断读写这些寄存器，完成数据处理、函数调用和流程切换。

> 结合底层可以这样理解：
> 
> 
> - x、y 传参时，常先放到 R0、R1
> - add () 执行时，结果可能放在某个通用寄存器
> - 返回时，结果一般放回 R0
> - 调函数时 LR 记录返回地址
> - 跳转执行靠 PC 改变
> - 如果发生中断，CPU 还会把部分寄存器自动压栈保护现场

- **延伸知识点**
- 函数调用时参数和返回值一般怎么传递
- 中断发生时哪些寄存器会自动压栈

---

## 12 STM32 位带操作

- **应用场景**

位带操作常用于 GPIO 某一位控制、标志位读写、临界位操作。

它的好处是可以像访问一个变量一样，直接访问寄存器中的某一位。

- **底层实现原理**

1. Cortex‑M 把一段 **位带区** 的每一位，映射到 **位带别名区** 的一个 32 位地址。
2. 这样访问别名区某个地址，本质上就是在访问原地址中的某一位。
3. 对别名地址写 `0/1`，就等价于把目标位清 0 或置 1；读别名地址，就能直接得到该位的值。
4. 这样避免了传统 **读‑改‑写** 的按位操作，代码更直观，也更适合单独控制某一位。
5. 所以位带本质就是：把 “按位访问” 转成 “按字访问”。

- **代码说明**

```csharp
#include "stm32f4xx.h"

/*
SRAM位带区:    0x20000000 ~ 0x200FFFFF
SRAM别名区:    0x22000000 ~ 0x23FFFFFF

外设位带区:    0x40000000 ~ 0x400FFFFF
外设别名区:    0x42000000 ~ 0x43FFFFFF
*/

/* 计算外设某一位的位带别名地址 */
#define BITBAND_PERI(addr, bitnum) \
  ((volatile unsigned int *)(0x42000000 + (((unsigned int)(addr) - 0x40000000) * 32) + ((bitnum) * 4)))

/* 把 PA5 的 ODR 第5位映射出来 */
#define GPIOA_ODR_Addr  ((unsigned int)&GPIOA->ODR)
#define PA5_ODR_BB      BITBAND_PERI(GPIOA_ODR_Addr, 5)

void LED_Init(void)
{
    /* 1. 开启 GPIOA 时钟 */
    RCC->AHB1ENR |= (1 << 0);

    /* 2. 配置 PA5 为输出模式 */
    GPIOA->MODER &= ~(3 << (5 * 2));
    GPIOA->MODER |=  (1 << (5 * 2));

    /* 3. 推挽输出 */
    GPIOA->OTYPER &= ~(1 << 5);
}

int main(void)
{
    LED_Init();

    while (1)
    {
        *PA5_ODR_BB = 1;  // PA5 输出高电平，只改 bit5
        for (volatile int i = 0; i < 500000; i++);

        *PA5_ODR_BB = 0;  // PA5 输出低电平，只改 bit5
        for (volatile int i = 0; i < 500000; i++);
    }
}
```

如果用普通方式控制某一位，一般写成：

```shell
GPIOA->ODR |=  (1 << 5);
GPIOA->ODR &= ~(1 << 5);
```

如果用位带后，就变成：

```markdown
*PA5_ODR_BB = 1;
*PA5_ODR_BB = 0;
```

本质区别是：

- 普通方式是 **按位运算改寄存器**
- 位带方式是 **直接把这一位当变量访问**

## 13 STM32 的 ADC 精度

- **应用场景**

ADC 精度决定了 **模拟量转数字量时能分多细**，直接影响电压、温度、电流等采样结果的分辨能力。

面试里一般既会问 **分辨率**，也会追问 **实际精度是否真的能达到标称值**。

- **底层实现原理**

1. STM32 ADC 常见分辨率有 **12 位**，部分型号还支持 **10/8/6 位** 可配置，少数高端型号可到更高精度。
2. 以 **12 位 ADC** 为例，表示可把输入电压范围分成 (2^{12}=4096) 个等级，数字范围通常是 **0~4095**。
3. 若参考电压是 `3.3V`，则 1 LSB 约等于 `3.3 / 4096 ≈ 0.8mV`，这叫**分辨率**。
4. 但分辨率不等于实际精度，实际还受参考电压误差、噪声、采样时间、线性误差、量化误差影响。
5. 所以面试里要区分：**标称精度看位数，实际精度看系统误差和采样环境。**

## 14 STM32 的时钟树

- **应用场景**

时钟树用来给 **CPU、总线、外设** 分配不同频率的时钟。

面试问它，本质是看你是否清楚 **系统时钟从哪来、怎么分给各模块。**

- **底层实现原理**

1. STM32 的时钟源一般有 **HSI、HSE、LSI、LSE**，高速时钟给系统和外设，低速时钟常给 RTC、看门狗。
2. 系统时钟 `SYSCLK` 可以选择直接来自 HSI/HSE，也可以先经过 **PLL 倍频** 后再作为主时钟。
3. `SYSCLK` 再往下分成 **AHB、APB1、APB2** 等总线时钟，分别供内核、内存、不同外设使用。
4. 不同外设再从总线时钟继续分频或单独选时钟源，比如 TIM、USART、ADC、RTC。
5. 所以时钟树本质就是：**时钟源选择 + 倍频 / 分频 + 分发到各模块。**

## 15 stm32 的启动方式

| 启动模式选择引脚 |  | 启动模式 | 说明 |
| --- | --- | --- | --- |
| BOOT1 | BOOT0 |  |  |
| X | 0 | 主闪存存储器 | 主闪存存储器被选为启动区域 |
| 0 | 1 | 系统存储器 | 系统存储器被选为启动区域 |
| 1 | 1 | 内置 SRAM | 内置 SRAM 被选为启动区域 |

(1) **用户闪存**：芯片内置的 Flash。正常的工作模式。

(2) **SRAM**：芯片内置的 RAM 区，就是内存。可以用于调试。

(3) **系统存储器**：芯片内部一块特定的区域，芯片出厂时在这个区域预置了一段 Bootloader，就是通常说的 ISP 程序。这个区域的内容在芯片出厂后没有人能够修改或擦除，即它是一个 ROM 区。启动的程序功能由厂家设置。

- **应用场景**

STM32 启动方式决定了 **复位后先从哪里取指执行程序。**

常见用于正常启动用户程序、下载程序、进入系统 BootLoader。

- **底层实现原理**

1. STM32 复位后，会先根据 **BOOT 引脚 / BOOT 配置位** 判断启动区域。
2. 常见启动位置有 **用户 Flash、System Memory、SRAM**。
3. 如果从 **用户 Flash** 启动，就执行我们烧录的应用程序。
4. 如果从 **System Memory** 启动，就进入芯片厂家固化好的 BootLoader，可通过串口、USB 等下载程序。
5. 所以启动方式本质就是：**上电后先把哪一段存储区映射到启动地址并开始执行。**

## 16 stm32 的外设和内存数量

- **应用场景**

这个问题本质是在问：**STM32 能提供多少资源给项目用。**

实际要结合具体型号看，因为不同系列的 Flash、RAM、外设数量差别很大。

- **底层实现原理**

1. STM32 的内存一般包括 **Flash、SRAM**，部分型号还带 **CCM、Backup SRAM、EEPROM 模拟区** 等。
2. 外设通常包括 **GPIO、USART、SPI、I2C、TIM、ADC、DAC、DMA、CAN、USB、RTC** 等，但具体数量因型号而异。
3. 比如低端型号资源少，适合简单控制；高端型号 Flash/RAM 更大，外设更多，适合复杂应用。
4. 所以不能笼统说 STM32 有多少内存、多少外设，必须先看 **具体芯片型号的数据手册。**
5. 面试里更好的回答是：**STM32 资源丰富，但不同系列差异大，选型要看存储需求和外设需求。**

## 17 arm 架构和 riscV 架构区别

- **应用场景**

ARM 和 RISC‑V 的区别，本质是 **成熟商业架构** 和 **开源指令集架构** 的区别。

面试里一般会从 **授权模式、生态、兼容性、灵活性** 这几个点回答。

- **底层实现原理**

1. ARM 是商业授权架构，厂商要向 ARM 购买授权后才能设计芯片；RISC‑V 是开源 ISA，任何厂商都可以基于标准指令集扩展实现。
2. ARM **生态更成熟**，工具链、芯片厂商、RTOS、中间件、调试支持都更完善；RISC‑V 这几年发展很快，但整体生态还在追赶。
3. ARM **兼容性和产品体系更完整**，从 Cortex‑M 到 Cortex‑A 覆盖 MCU、MPU、应用处理器；RISC‑V **灵活性更高**，厂商可按需求裁剪和自定义扩展。
4. ARM **标准化程度高**，不同厂商做出来的软件迁移通常更方便；RISC‑V 因为可扩展性强，不同实现之间差异可能更大。
5. 所以实际选型里，**ARM 更偏成熟稳定，RISC‑V 更偏开放灵活、成本和自主可控优势更明显。**

这段代码在 ARM 和 RISC‑V 上都能跑，区别不在 C 代码本身，而在：

- 编译器会生成不同的 **目标指令集**
- 芯片执行的是各自架构定义的机器指令
- 调用约定、寄存器命名、异常机制也会不同

比如本质上会变成：

```cpp
/* ARM: 生成 ARM/Thumb 指令 */
/* RISC‑V: 生成 RISC‑V 指令 */
```

所以它们的核心区别是 **底层 ISA 和生态体系不同，不是上层 C 写法不同。**

- 延申知识点
ISA、微架构、SoC 三者的区别
ARM Cortex‑M 和 RISC‑V MCU 在中断响应上的差异

## 18 交叉编译介绍一下，常见的交叉编译器

1. **跨架构**：主机与目标机的 CPU 架构不同（如 x86→ARM、x86→RISC‑V）。
2. **工具链专用**：需使用针对目标架构的专用编译工具链（包含编译器、链接器、汇编器等），而非主机原生工具（如 x86 的 `gcc` 无法直接编译 ARM 程序）。
3. **依赖适配**：编译时需链接目标机的库文件（如目标系统的 C 库 `libc`），而非主机的库。

> 常见交叉编译器：`arm‑linux‑gnueabihf‑gcc`、`aarch64‑linux‑gnu‑gcc`、`riscv64‑unknown‑linux‑gnu‑gcc`。

## 19 STM32 从 FLASH 启动，为什么是 0x08000000

- **应用场景**
这个问题常用来考察你是否理解 STM32 的存储器映射和启动机制。
本质不是 “为什么偏偏这个数字”，而是 “这个地址是谁规定的”。
- **底层实现原理**

1. `0x08000000` 是 STM32 片内 Flash 在内存映射中的起始地址，这是芯片硬件设计时就固定好的。
2. ARM 内核访问代码和数据，看到的是一张统一的**存储器地址映射表**，Flash、SRAM、外设都会被映射到不同地址段。
3. STM32 复位后虽然是从 `0x00000000` 开始取向量表，但芯片会把启动区域**重映射**到这个地址。
4. 当选择 Flash 启动时，本质上就是把 `0x08000000` 这一段 Flash 映射到 `0x00000000` 给内核取指。
5. 所以 `0x08000000` 是 Flash 的物理映射起始地址，而启动时 CPU 先访问的其实是被重映射后的启动地址空间。

可以这样理解：

- `0x08000000`：Flash 真正的存储器地址
- `0x00000000`：CPU 复位后默认取指的启动地址
- 启动时 STM32 会做一次**地址映射**，让 CPU 能先取到 Flash 里的向量表
- **延申知识点**
- STM32 启动时的地址重映射是怎么做的
- 为什么向量表可以搬到 SRAM 去运行

---

### 20 STM32 串口接收模式：对比轮询 / 中断 / DMA

- **应用场景**
STM32 串口接收常见有 **轮询、中断、DMA** 三种方式。
本质区别在于：**CPU 参与程度不同，实时性和效率也不同**。
- **底层实现原理**

1. **轮询**：CPU 一直检查接收标志位 `RXNE`，有数据就读寄存器；实现简单，但 CPU 会一直空转，效率最低。
2. **中断**：收到数据后串口触发中断，CPU 跳到 ISR 里读 `DR`；比轮询省 CPU，适合低速或不连续数据。
3. **DMA**：串口收到数据后，由 DMA 自动把 `DR` 搬到内存，CPU 不用逐字节参与；适合高速、大量、连续接收。
4. 轮询最简单但最占 CPU；中断实时性较好但频繁进中断有开销；DMA 效率最高，但配置最复杂。
5. 所以选型一般是：**少量数据用中断，连续大数据用 DMA，简单调试可用轮询**。

- **延申知识点**
- 串口空闲中断 + DMA 为什么适合不定长数据接收
- 环形缓冲区在串口接收里怎么用

## 21 STM32 分频设置是如何设置的，具体的接口？

- **应用场景**
STM32 分频设置主要用来 给内核、总线、外设分配合适时钟。
比如 CPU 跑高速，APB1 给低速外设，既满足性能又不超频。
- **底层实现原理**

1. STM32 的分频本质是对上一级时钟做**整数分频**，常见有 AHB 分频、APB1 分频、APB2 分频。
2. 配置入口本质都在 **RCC 时钟控制模块**，核心寄存器一般是 `RCC->CFGR`。
3. 其中 `HPRE` 控制 AHB 分频，`PPRE1` 控制 APB1 分频，`PPRE2` 控制 APB2 分频。
4. 配完后会得到 `HCLK`、`PCLK1`、`PCLK2`，分别供 CPU / 总线 / 外设使用。
5. 需要注意：**定时器时钟 在 APB 分频不为 1 时，时钟可能会 ×2**。

> 面试回答：
> 
> 
> STM32 分频主要通过 RCC 配置，寄存器级一般改 `RCC->CFGR` 的 `HPRE/PPRE1/PPRE2`，HAL 库一般用 `HAL_RCC_ClockConfig()` 接口。

- **延申知识点**
- PLL 倍频参数怎么计算
- 为什么 APB 分频后定时器时钟会翻倍

## 22 STM32 ADC 怎么配置的

- **应用场景**
STM32 的 ADC 用来把 **外部模拟电压转换成数字量**，常见于电压、电流、温度、传感器采样。
配置核心就是：**选通道、定采样时间、开转换、读结果**。
- **底层实现原理**

1. 先开启 GPIO 和 ADC 时钟，并把对应引脚配置为 **模拟模式**。
2. 再配置 ADC 的 **分辨率、采样时间、转换通道、转换顺序**。
3. 如果是单次采样，就软件触发一次转换；如果是连续采样，可打开连续转换模式。
4. 转换完成后可通过 **轮询 EOC、中断、DMA** 取走结果。
5. 所以 ADC 配置本质是：**模拟口准备 + ADC 参数配置 + 启动转换 + 读取结果**。

- **延申知识点**
- ADC 采样时间为什么会影响结果
- ADC 的轮询、中断、DMA 三种读取方式怎么选

## 23 STM32 Flash 怎么划分

- **应用场景**
STM32 的 Flash 划分主要关系到 程序存放、BootLoader/App 分区、参数掉电保存。
实际开发里经常会按区域划分成 **代码区、升级区、参数区**。
- **底层实现原理**

1. STM32 的 Flash 是按 **页（Page）或扇区（Sector）** 管理的，不是按字节独立擦除。
2. 不同系列划分方式不同：一般低容量型号多按页管理，像部分 F4/F7 常按扇区管理，而且各扇区大小可能不一样。
3. 程序下载时，代码通常从 Flash 起始地址开始放；如果要做 IAP，就会再单独预留 **BootLoader 区、App 区、参数存储区**。
4. Flash 的特点是 **读快、可掉电保存，但写之前通常要先擦除整个页 / 扇区**，所以划分时要考虑擦写粒度。
5. 所以 Flash 划分本质就是：**按芯片擦除单位管理存储空间，再按功能分配给代码和数据使用**。

- **延申知识点**
- STM32 不同系列 Flash 的 page 和 sector 有什么区别
- BootLoader 跳转 App 时为什么要重设 MSP 和向量表

---

## 24 STM32 系统死机 / 跑飞的常见原因及排查方法

### （一）硬件层面（基础故障，先排除硬件再查软件）

硬件问题是 “底层诱因”，若硬件不稳定，软件优化再完善也会死机，优先排查：

| 原因 | 典型现象 | 核心原理 |
| --- | --- | --- |
| 电源不稳定 | 电机 / 外设启动时死机、复位，低电压场景下概率性跑飞 | VDD/VCORE 纹波超标（STM32 要求 3.3V 纹波 < 100mV），电压跌落导致 MCU 寄存器乱码 |
| 晶振 / 时钟异常 | 系统时钟走偏、SysTick 延时错误，直接触发 HardFault | 晶振停振 / 频率偏移，或 RCC 配置错误（如 HSI 不稳定却作为系统时钟） |
| 虚焊 / 短路 / ESD 干扰 | 触碰电路板 / 工业环境下死机，PC 指针跳转到随机地址 | 虚焊导致引脚接触不良，ESD 冲击使寄存器 / 内存数据篡改 |

### （二）软件层面（最常见，占比 80% 以上）

软件问题多为 “内存操作 / 逻辑错误”，是嵌入式开发的核心坑点：

| 原因 | 典型现象 | 核心原理 |
| --- | --- | --- |
| 栈溢出（TOP1） | 任务执行一段时间后跑飞，HardFault，PC 指向栈区地址 | 任务栈 / 中断栈过小，函数调用 / 中断嵌套覆盖返回地地址（LR），PC 跳转到非法地址 |
| 内存越界 / 野指针 | 概率性死机，写数组越界覆盖 TCB / 寄存器，空指针解引用触发 HardFault | 数组越界写、malloc/free 碎片导致堆溢出，野指针访问 0x00000000 等非法地址 |
| 非对齐内存访问 | 直接触发 HardFault，报错地址为非 4/8 字节对齐地址 | ARM Cortex‑M 不支持非对齐访问（如 int 变量存在 0x20000001 地址） |
| RTOS 调度异常 | 任务卡死、调度器挂起，中断中调用非 FromISR API 导致死锁 | 优先级越界、中断优先级高于 SysCall 阈值，或未用 `xSemaphoreGiveFromISR` 等 API |
| 编译器优化问题 | O2/O3 优化后逻辑错误，变量被优化导致条件判断失效 | 未用 `volatile` 修饰易变变量（如外设寄存器、中断标志），被编译器优化掉 |
| 死循环 / 任务全阻塞 | CPU 占用 100%，看门狗超时复位，Idle 任务无法执行 | 所有任务阻塞（如等待不存在的信号量），或死循环无 `vTaskDelay` 让出 CPU |

### （三）中断 / 异常层面（难定位，需调试器辅助）

中断是 “实时性核心”，配置错误会直接导致系统崩溃：

| 原因 | 典型现象 | 核心原理 |
| --- | --- | --- |
| HardFault/NMI 异常 | 直接卡死，PC 跳转到 HardFault_Handler | 非法指令、访问非法地址、总线错误（如写只读寄存器、DMA 地址越界） |
| 中断优先级错误 | 中断不响应、响应后死机，RTOS 调度器崩溃 | 中断优先级高于 `configMAX_SYSCALL_INTERRUPT_PRIORITY`，调用 FreeRTOS API 触发异常 |
| 中断标志未清除 | 无限进入中断，CPU 卡死，串口 / 外设无响应 | ISR 中未清中断标志（如 USART_RXNE、EXTI_PR），导致中断无限触发 |
| 嵌套中断过深 | 中断中嵌套多级中断，触发栈溢出 | 高优先级中断嵌套低优先级中断，中断栈不足 |

### （四）外设交互层面（场景化故障）

外设驱动 / 交互错误易被忽视，尤其高频外设（DMA、I2C、SPI）

| 原因 | 典型现象 | 核心原理 |
| --- | --- | --- |
| DMA 传输异常 | DMA 传输后数据乱码，触发总线错误，HardFault | DMA 缓冲区非对齐、传输长度超出缓冲区，或外设时钟未使能 |
| 外设超时死等 | I2C/SPI 等待 ACK 超时，进入无限循环，CPU 卡死 | 外设无响应时未加超时机制，无限等待 `HAL_I2C_Master_Transmit` 返回 |
| Flash/EEPROM 写故障 | 写 Flash 时断电，程序区损坏，复位后跑飞 | Flash 写操作中电源中断，导致代码区数据篡改，PC 执行非法指令 |
| 总线冲突 | SPI 多主机冲突、I2C 地址冲突，总线锁死，外设无响应 | 多设备抢占总线，无仲裁机制导致数据传输错误，触发外设错误中断 |

---

## 25 如果 STM32 经过 ADC 模块采集数据，然后数据存放在哪里

- **考察场景**

STM32 用 ADC 采集模拟信号后，常用于 电压、电流、温度、传感器数据读取。

面试里这个问题本质是在问：转换结果先放哪，程序再从哪取。

- **底层实现原理**

1. ADC 完成一次转换后，结果会先存到 **ADC 数据寄存器 DR**。
2. 如果是 **轮询** 或 **中断** 方式，CPU 再去读取 DR，读出来后一般存到变量、数组或缓冲区里。
3. 如果是 **DMA** 方式，DMA 会自动把 DR 里的数据搬运到 RAM 中指定的数组或缓存区。
4. 所以 ADC 数据最开始一定是先到 ADC 外设寄存器，之后才会进入 CPU 变量 或 内存缓冲区。
5. 本质上就是：`ADC -> DR寄存器 -> CPU读取或DMA搬到RAM`。

> ADC 采样完成后，结果先存在 ADC 的 DR 数据寄存器里；如果是普通读取，再由 CPU 读到变量；如果是 DMA，则直接从 DR 搬到 RAM 缓冲区。

- **延申知识点**
- ADC 的 EOC 标志什么时候置位
- ADC + DMA 为什么适合连续采样

---

## 26 中断和异常对于 CPU 是同步还是异步

**考察场景**

考察中断、异常的触发来源，重点是同步异常和异步中断的区别。

**底层实现原理**

1. **中断一般是异步的**，由外设事件触发，比如串口接收、定时器溢出、GPIO 边沿。
2. **异常一般是同步的**，由当前执行指令触发，比如除零、非法指令、访问非法地址。
3. 同步异常和当前指令强相关，CPU 可以明确知道是哪条指令导致异常。
4. 异步中断和当前指令没有直接关系，只是在指令执行间隙被 CPU 响应。
5. STM32 Cortex‑M 中，`HardFault`、`MemManage`、`BusFault` 属于异常；`USART`、`TIM`、`EXTI` 属于外设中断。

---

## 27 STM32 的 Debug 方式有哪些？

**考察场景**

考察 STM32 常用调试手段，重点是在线调试、日志、硬件信号分析。

**底层实现原理**

1. 常用在线调试接口是 **SWD 和 JTAG**，STM32 最常用 SWD，占用引脚少。
2. 可以通过断点、单步、查看寄存器、查看内存、变量监控定位软件问题。
3. 日志调试常用 **UART、RTT、SWO**，适合不方便停机的运行态问题。
4. 外设时序问题常用逻辑分析仪或示波器，比如 SPI、I2C、UART、PWM。
5. 崩溃类问题重点看 fault 寄存器、栈现场、PC/LR、map 文件和反汇编。

**常见方式：**

1. SWD/JTAG：在线下载和断点调试
2. UART Log：串口打印日志
3. RTT：高速日志，不太影响实时性
4. SWO/ITM：单线调试输出
5. 逻辑分析仪：分析通信时序
6. 示波器：看电平、波形、毛刺

**延申知识点**

SWD 和 JTAG 区别

---

## 29 外部中断的触发方式有哪些？中断优先级配置需遵循什么原则？

**考察场景**

考察 EXTI 配置和 NVIC 优先级设计，重点是触发边沿和优先级不能乱配。

**底层实现原理**

1. 外部中断常见触发方式有**上升沿、下降沿、双边沿、电平触发**，STM32 EXTI 常用边沿触发。
2. 按键通常用下降沿或双边沿，但需要做消抖，避免一次按键触发多次中断。
3. 优先级配置要按实时性来，高实时外设优先级高，耗时处理优先级低。
4. 中断里只做快处理，比如清标志、读数据、发信号量，耗时逻辑放任务里。
5. 使用 FreeRTOS 时，调用 RTOS API 的中断优先级不能高于 `configMAX_SYSCALL_INTERRUPT_PRIORITY`。
6. 避免多个高优先级中断执行太久，否则会影响系统实时性和调度。

**延申知识点**

输入捕获如何测频率

PWM 频率和占空比怎么计算

---

## 31 STM32 中 GPIO 输入模式下，上拉 / 下拉电阻的作用是什么？

**考察场景**

考察 GPIO 输入稳定性，重点是为什么输入引脚不能悬空。

**底层实现原理**

1. GPIO 输入模式下，如果外部没有明确高低电平，引脚会悬空，读到的值不稳定。
2. 上拉电阻把引脚默认拉到高电平，适合按键按下接地的场景。
3. 下拉电阻把引脚默认拉到低电平，适合按键按下接高电平的场景。
4. 上拉 / 下拉提供默认状态，避免噪声导致误触发。
5. STM32 可以配置内部上拉 / 下拉，但阻值较大，抗干扰强场景可能需要外部电阻。

---

## 32 STM32 中内存泄漏怎么 debug

**考察场景**

考察嵌入式内存问题定位，重点是堆是否持续减少、谁申请后没释放。

**底层实现原理**

1. 先确认是否使用动态内存，比如 `malloc/free`、`new/delete`、RTOS 动态创建对象。
2. 周期打印剩余堆大小，观察是否持续下降。
3. 对 `malloc/free` 做封装，记录申请地址、大小、调用位置，释放时从表中删除。
4. 检查异常路径，比如函数中途 return、错误处理分支、任务退出前没有释放资源。
5. FreeRTOS 中重点看 `xPortGetFreeHeapSize()` 和 `xPortGetMinimumEverFreeHeapSize()`。
6. 嵌入式项目建议尽量静态分配，或使用固定大小内存池，减少泄漏和碎片。

---

## 33 STM32 出现 HardFault 如何定位问题

**考察场景**

考察 Cortex‑M fault 排查能力，重点是从栈中取出 PC/LR，再结合 map 和反汇编定位代码行。

**底层实现原理**

1. HardFault 常见原因有空指针、越界访问、栈溢出、函数指针错误、未对齐访问、非法地址访问。
2. 进入 HardFault 时，硬件会自动压栈 `R0‑R3、R12、LR、PC、xPSR`。
3. 关键是取出异常现场里的 `PC`，它通常指向出错指令附近。
4. 再查看 `CFSR、HFSR、BFAR、MMFAR` 等 fault 寄存器，判断是总线错误、内存管理错误还是用法错误。
5. 结合 `.map` 文件、反汇编或 IDE 的 call stack，把 PC 地址还原到具体函数和代码行。
6. 如果栈已破坏，要检查任务栈溢出、数组越界、DMA 写越界。

#### 延申知识点

1. Cortex‑M 自动压栈保存了哪些寄存器
发生异常进入中断 / 异常服务函数时，硬件自动压入栈的 8 个寄存器：
`R0`、`R1`、`R2`、`R3`、`R12`、`LR(R14)`、`PC(R15)`、`xPSR`。

> 注意：R4‑R11 需要软件手动入栈保存，硬件不会自动压栈。

1. **如何通过 PC 地址反查出错代码行**

获取 HardFault 异常栈里面的**PC 寄存器值**（出错指令的地址）。

使用编译生成的`.map`文件，根据 PC 地址找到对应的函数。

使用反汇编文件（`.lst`/`.dis`），按 PC 地址定位汇编指令。

在 IDE（Keil/STM32CubeIDE）中，根据反汇编映射到 C 语言源代码行；也可以在 IDE 输入 PC 地址跳转反汇编窗口定位。

> 补充：如果栈被破坏拿不到 PC，需要先排查栈溢出、内存越界。

#### 面试简短背诵版

> Cortex‑M 异常自动压栈：R0‑R3，R12，PC，LR，xPSR，R4~R11 需要软件保存。
> 
> 
> 通过 PC 反查代码：拿到 PC 值，结合 map 文件找函数，再看反汇编，映射到 C 源码行。=={yellow}=={yellow}=={pink}=={pink}=={pink}重点内容===={yellow}======
