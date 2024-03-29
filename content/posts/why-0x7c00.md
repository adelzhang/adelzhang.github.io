---
title: "0x7c00"
slug: "0x7c00"
date: 2014-10-18
---

`BIOS`完成自检(`POST`,Power-on self-test)后，将MBR(Master Boot Record)加载到
Ox7c00位置，然后跳转到该地址执行。MBR即硬盘的第一个扇区(0柱面，0磁头，1扇区），
硬盘每个扇区有512字节，MBR必须以`0xAA55`为结尾，`BIOS`检测最后两个字节来判断是
否是合法的MBR。

那么现在问题来了：

- 为什么`BIOS`会将MBR加载到0x7c00，而不是其他位置，比如说0x0？
- 为什么选择`0xAA55`作为MBR标志？

注意到
    
    0x7c00 = 0x8000 - 0x400 
           = 32KB - 1024B

可见0x7c00并不是随意选择的地址。

其实这跟第一台IBM PC和DOS有关。BIOS选择0x7c00主要有以下原因：

- 预留最多的地址空间给DOS
- DOS操作系统最小内存要求是32KB
- 8086/8088使用0x0 - 0x3FF作为中断向量表，紧接着BIOS数据区
- 启动扇区总共为512字节，还需要额外空间作为栈或数据区

参考内存分布图，在低地址空间已被占用，内存至少32KB的情况下，在高地址腾出1024字
节加载MBR是一个合理的决定。

    +--------------------- 0x0
    | Interrupts vectors
    +--------------------- 0x400
    | BIOS data area
    +--------------------- 0x5??
    | OS load area
    +--------------------- 0x7C00
    | Boot sector
    +--------------------- 0x7E00
    | Boot data/stack
    +--------------------- 0x7FFF
    | (not used)
    +--------------------- (...)

至于为什么使用0xAA55作为MBR标志，考虑到x86使用的是小尾序，0xAA55在内存中会记录
成

    01010101 10101010

看着这么整齐可爱的模样，只能说是出于偏爱的原因。

Ref
===

[1] [why bios loads MBR into 0x7c00 in x86?](http://www.glamenv-septzen.net/en/view/6)


-EOF-
