---
title: 'PYNQ教程二:使用AXI总线实现PS与PL交互'
tags: PYNQ
categories: PYNQ教程
abbrlink: 855715585
---

ZYNQ架构分为处理器系统（PS）和可编程逻辑（PL）两大部分，PS运行在ARM Cortex A9处理器上，PL相当于FPGA，本文介绍如何使用AXI总线实现PS与PL之间的交互。

<!-- more -->

## 1 PL部分设计

PL设计可以使用Vivado HLS开发，或者Verilog开发，这里介绍使用Vivado HLS的方式。

设计一个简单的模块，代码如下：

{% box child:codeblock color:green %}
```c++
void test(float input[5],float num,float output[5]){
#pragma HLS INTERFACE s_axilite port=return
#pragma HLS INTERFACE s_axilite port=num
#pragma HLS INTERFACE m_axi depth=5 port=input offset=slave bundle=INPUT
#pragma HLS INTERFACE m_axi depth=5 port=output offset=slave bundle=OUTPUT

	for(int i=0;i<5;i++)
		output[i] = input[i] * num;
}
```
{% endbox %}

函数功能很简单，就是将输入数组乘上某一个数后交给输出数组。

**pragma解释：**

"port=return"表示配置块级协议，"s_axilite"表示将ap_start等信号放置于AXI从接口寄存器中。

其余几个都是配置端口协议，"m_axi"即配置为AXI主接口，"s_axilite"即配置为AXI从接口。因为主接口访问数据时需要知道数据所在的地址，"offset=slave"表示将地址放在AXI从接口寄存器中。"bundle"可以将接口合并，这里bundle各不相同，也就意味着生成两个AXI主接口。

<!-- ![ip](/pic/post4/ip.png) -->

{% image https://9258ce6.webp.li/post4/ip.png fancybox:true %}

上面就是HLS生成的IP核。在HLS中打开solution1/impl/ip/drivers/*_v1_0/src，找到一个以hw.h结尾的文件，打开后可以看到AXI从接口寄存器信息（也可以使用SDK的方式，但是我觉得这样更方便）。

<!-- ![hwh](/pic/post4/hwh.png) -->

{% image https://9258ce6.webp.li/post4/hwh.png fancybox:true %}



## 2 Vivado Block Design

<!-- ![zynq](/pic/post4/zynq.png) -->

{% image https://9258ce6.webp.li/post4/zynq.png fancybox:true %}

可以看到，ZYNQ有两个AXI_GP主接口、两个AXI_GP从接口、四个AXI_HP从接口和一个AXI_ACP从接口，这些接口的区别可以参考下面的视频：

{% link https://www.bilibili.com/video/BV1FD4y1E79q/?spm_id_from=333.999.0.0 AXI接口讲解 %}

在这里，由于HLS生成的IP核有一个AXI从接口和两个AXI主接口，因此这里ZYNQ至少使用一个AXI主接口和一个AXI从接口，然后使用AXI Interconnect或AXI SmartConnect实现多主多从之间的连接。这里可以使用Vivado的自动连接功能。

<!-- {% image /pic/post4/bd.png fancybox:/pic/post4/bd.png %} -->

{% image https://9258ce6.webp.li/post4/bd.png fancybox:true %}

上图是一种连接方式，也可以使用AXI_GP从接口，不一定要用AXI_HP从接口。



## 3 PYNQ PS端代码

**PS主接口和PL从接口：**

根据pynq.io官方的描述："In an overlay, peripherals connected to the AXI General Purpose ports will have their registers or address space mapped into the system memory map. With PYNQ, the register, or address space of an IP can be accessed from Python using the *MMIO* class."

原网址如下：

{% link https://pynq.readthedocs.io/en/latest/pynq_libraries/mmio.html MMIO %}

翻译过来就是：对于连接至PYNQ AXI_GP主接口的IP，PS可以在Python代码中使用MMIO类访问它们。

**PS从接口和PS主接口：**

同样根据官方的说法："IP connected to the AXI Master (HP or ACP ports) has access to PS DRAM. Before IP in the PL accesses DRAM, some memory must first be allocated (reserved) for the IP to use and the size, and address of the memory passed to the IP. An array in Python, or Numpy, will be allocated somewhere in virtual memory. The physical memory address of the allocated memory must be provided to IP in the PL. "（这里第一句话感觉应该不是AXI Master而是AXI Slave才对）

原网址如下：

{% link https://pynq.readthedocs.io/en/latest/pynq_libraries/allocate.html Allocate %}

翻译过来就是：连接至PYNQ从接口的IP可以访问DDR内存（DRAM），不过PS需要预先分配出一段内存，并将这段内存的物理地址传递给PL的IP核。



**接下来开始代码实现过程：**

（1）烧写底层设计的bit流文件：

{% box child:codeblock color:green %}
```python
from pynq import Overlay
ol = Overlay('ps_pl.bit')

ol?
```
{% endbox %}

”ol?“可以查看overlay的信息：

<!-- ![ip_block](/pic/post4/ip_block.png) -->

{% image https://9258ce6.webp.li/post4/ip_block.png fancybox:true %}

可以看到，识别到hls生成的IP核”test_0“，该IP核被封装为一个MMIO类对象，可以通过"ol.test_0?"查看其详细信息，mmio类内部有"write"和"read"函数实现读写功能。

（2）使用allocate函数分配一段内存，该内存可以当作一个numpy数组来使用，同时device_address属性为其物理地址：

{% box child:codeblock color:green %}
```python
from pynq import allocate
import numpy as np

input_buffer = allocate(shape=(5,),dtype=np.float32)
input_buffer.flush()
output_buffer = allocate(shape=(5,),dtype=np.float32)
output_buffer.flush()
```
{% endbox %}

（3）将内存地址传递给PL的IP核：
{% box child:codeblock color:green %}

```python
test_ip = ol.test_0
test_ip.write(0x10,input_buffer.device_address)
test_ip.write(0x20,output_buffer.device_address)
```
{% endbox %}

前面hw.h文件显示的AXI从接口寄存器地址中，0x10和0x20分别是输入和输出的地址，因此这里将PS分配的内存的物理地址分别写入到这两个地方中。

（4）写入输入数据：

{% box child:codeblock color:green %}
```python
inputs = np.array([1,2,3,4,5],dtype=np.float32)
np.copyto(input_buffer,inputs)

import struct
num = struct.unpack('<I',struct.pack('<f',3))[0]
test_ip.write(0x18,num)
```
{% endbox %}

这里num不能直接写入3，因为HLS定义了num是float类型。但是也不能写入3.0，否则会报下面的错误：

{% note color:red ValueError:&nbsp;Data&nbsp;type&nbsp;must&nbsp;be&nbsp;int&nbsp;or&nbsp;bytes. %}

因为MMIO类的"write"函数不能写入float类型数据。所以这里使用struct做了一个变换，从float类型转换成int类型，struct的用法可参考：

{% link https://zhuanlan.zhihu.com/p/387421751 struct函数解析 %}


（5）启动IP核工作，等待其工作结束：

{% box child:codeblock color:green %}
```python
import time

test_ip.write(0x0,1)
while(test_ip.read(0x0)&0x2 == 0):
    time.sleep(0.001)
```
{% endbox %}

0x0位置是控制信号，写入1则ap_start置1，IP核开始工作。而第2位为ap_done，因此这里采用轮询的方式，等待第2位为1则IP核工作结束。

（6）获取输出结果：

{% box child:codeblock color:green %}
```python
outputs = np.zeros([5])
np.copyto(outputs,output_buffer)

print(outputs)
```
{% endbox %}

结果如下：

<!-- ![result](/pic/post4/result.png) -->

{% image https://9258ce6.webp.li/post4/result.png fancybox:true %}

