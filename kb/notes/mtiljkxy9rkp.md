# 一、UART

UART：**Universal Asynchronous Receiver/Transmitter 通用异步收发器**
- 异步：**没有共享时钟线**，收发双方预先约定波特率；
- 只有 TX、RX 两根信号线。
- 注意：UART 是物理协议逻辑；**USART = UART + 同步 SPI 模式**；USART 芯片外设可以配置成 UART (异步) 或者同步模式。
- 常用别名：串口；TTL‑UART；RS232/RS485 是**电平标准**，不是 UART 协议。
