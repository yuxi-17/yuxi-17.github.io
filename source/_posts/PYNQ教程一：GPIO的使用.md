---
mathjax: true
title: 'PYNQ教程一:GPIO的使用'
tags: PYNQ
categories: PYNQ教程
abbrlink: 1649035944
---

GPIO通常用于读写寄存器，与外部设备的交互等场景。本文从PS GPIO和AXI GPIO两方面分别介绍它们的配置方法，以及如何使用PYNQ编写Python代码实现数据交互。

<!-- more -->

## 1 PS GPIO

### 1.1 PS GPIO的配置

PS GPIO分为MIO和EMIO，其中MIO共54bit，EMIO共64bit。

其使用方法可以参考以下的网站：

{% link https://pynq.readthedocs.io/en/latest/pynq_package/pynq.gpio.html pynq官方文档 %}

{% note pynq.io网站上文档非常全面介绍了Pynq的各种函数，后面的AXI_GPIO和Arduino_IO都可以在这里找到相关的函数解析 color:cyan %}

{% link https://blog.csdn.net/qq_35712169/article/details/106038000 MIO和EMIO的使用 %}

总的来说，MIO无需配置，其位置已经固定，可以将其配置为GPIO口，但是需要注意的是其与其它接口复用（比如USB、ETH等），可能存在冲突导致无法使用的情况，而且位置固定后反而不方便使用，所以我个人更推荐使用EMIO的方式。

EMIO没有固定的引脚，其配置方法较简单，打开ZYNQ7 IP核配置页面，然后按下面的路径打开：

<!-- ![emio1](/pic/post3/emio1.png) -->

{% image https://9258ce6.webp.li/post3/emio1.png fancybox:true %}

打开后拉到最下面就可以看到GPIO的配置，这里可以选择使用的EMIO的位数：

<!-- ![emio2](/pic/post3/emio2.png) -->

{% image https://9258ce6.webp.li/post3/emio2.png fancybox:true %}

配置完后可以看到IP核多了一个GPIO_0接口，然后可以将这些端口绑定到特定的引脚上。

<!-- ![zynq7_ip](/pic/post3/zynq7_ip.png) -->

{% image https://9258ce6.webp.li/post3/zynq7_ip.png fancybox:true %}

为了方便测试读写功能，可以将数据传输给其它的IP核，并获取其输出。这里可以使用“Utility Vector Logic”构建一个与门，输入输出大小为2。

<!-- ![and_gate](/pic/post3/and_gate.png) -->

{% image https://9258ce6.webp.li/post3/and_gate.png fancybox:true %}

GPIO_O只有一个，而与门有两个输入，因此这里需要通过两个“slice”IP核将64位数据分割开来，slice0配置如下，取输入的0-1位输出。

<!-- ![slice](/pic/post3/slice.png) -->

{% image https://9258ce6.webp.li/post3/slice.png fancybox:true %}

slice1配置类似，取输入2-3位输出。而后将这两个slice的输出连接到与门。

<!-- ![and_in](/pic/post3/and_in.png) -->

{% image https://9258ce6.webp.li/post3/and_in.png fancybox:true %}

与门的输出并不能直接与GPIO_I连接。前面提到，EMIO只有64位，这里GPIO_O和GPIO_I显然是共用的，如果将与门的输出连接到GPIO_I，那么其结果将会发给低位，也就是0-1位，但是前面0-1位已经被GPIO_O用于输出了，当然如果0-1位配置为“inout”，那就没问题。但是后续的Python代码中PS GPIO的某一位只能被配置为“in”或“out”，而不是“inout”。因此这里需要加一个偏移项，将与门的输出传给GPIO_I的4-5位，从而避免该冲突，这可以通过“Constant”和“Concat”IP核实现偏移功能。

<!-- ![and_out](/pic/post3/and_out.png) -->

{% image https://9258ce6.webp.li/post3/and_out.png fancybox:true %}

如图所示，“Constant”设置了4位，值为0，通过“Concat“将两者拼接起来，组成6位数据，与门的输出在最高两位。

在Synthesis中，会出现以下警告，但是无关紧要，如果不想出现这样的警告，可以将EMIO的位数设置为6位。

<!-- ![warning](/pic/post3/warning.png) -->

{% image https://9258ce6.webp.li/post3/warning.png fancybox:true %}

### 1.2 PS GPIO Python代码

<!-- ![ps_gpio_python](/pic/post3/ps_gpio_python.png) -->

{% image https://9258ce6.webp.li/post3/ps_gpio_python.png fancybox:true %}

首先烧写bit流文件，然后导入GPIO库，配置0-3位为输出，4-5位为输入，通过”read“和”write“函数实现GPIO的读写功能。

{% note 前面的内容参考了下面的视频，所以不太理解的也可以查看下面的视频讲解 color:cyan %}

{% link https://youtu.be/a5NnLozPEI0?si=TiMc5d3H6u_wqcK1 youtube视频讲解 %}

### 1.3 Arduino的GPIO

<!-- ![arduino](/pic/post3/arduino.png) -->

{% image https://9258ce6.webp.li/post3/arduino.png fancybox:true %}

使用Arduino的IO时，同样不需要配置，Python代码也比较简单，可以参考以下网站：

{% link https://rkblog.dev/posts/python/using-gpio-usb-and-hdmi-pynq-z2-board/ Arduino IO %}


## 2 AXI GPIO

### 2.1 AXI GPIO配置

AXI GPIO的配置相比于EMIO来说要简单一点，且可以同时使用多个AXI GPIO，灵活性很强。

IP核配置可以参考以下文章：

{% link https://blog.csdn.net/u011565038/article/details/138836478 AXI GPIO IP核配置 %}

我通常启用双通道，一个All Outputs，一个All Inputs，这样就不需要管Default Tri State Value这一栏了。

<!-- ![axi_gpio](/pic/post3/axi_gpio.png) -->

{% image https://9258ce6.webp.li/post3/axi_gpio.png fancybox:true %}

连接也很简单，先手动将AXI GPIO的输出与输入和非门连接，接下来使用Vivado的自动连接就行了。

<!-- ![axi_gpio_design](/pic/post3/axi_gpio_design.png) -->

{% image https://9258ce6.webp.li/post3/axi_gpio_design.png fancybox:true %}

### 2.2 AXI GPIO Python代码

<!-- ![axi_gpio_python](/pic/post3/axi_gpio_python.png) -->

{% image https://9258ce6.webp.li/post3/axi_gpio_python.png fancybox:true %}

首先烧写bit流文件，获取overlay信息：

<!-- ![ol](/pic/post3/ol.png) -->

{% image https://9258ce6.webp.li/post3/ol.png fancybox:true %}

得到AXI GPIO IP核的名字后，即可使用AxiGPIO函数将该IP核实例化为一个类对象，而后根据Pynq提供的函数实现读写功能。

也可以参考下面的官方例程：

{% link https://github.com/Xilinx/PYNQ_Workshop/blob/master/Session_4/2_axi_gpio.ipynb AXI GPIO官方案例 %}



## 写在最后

可以看到，文中引用了很多其它网站，毕竟在最初的学习中，也是广泛搜索了各种网站，然后才慢慢明白。所以我精选出这些我觉得有用的网站，可以省去广泛查找的过程，同时也更加全面。当然，有一定基础的看一下我写的也大概明白了。虽然不是很难，但是硬件的东西，别人说一下可能几分钟就知道了，如果自己去找资料看文档，可能要花费几个小时甚至于几天的时间。

