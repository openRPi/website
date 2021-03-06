真正的GPIO内核驱动
=========================

上一章，我们简单介绍了GPIO驱动的原理————操作相应的GPIO控制寄存器。在用户态程序中，我们通过 ``/dev/mem`` 接口访问真实的物理地址，这实际上就是实现用户空间驱动的一种方法。

有时编写一个所谓的用户空间设备驱动对比钻研内核是一个明智的选择，用户空间驱动的好处在于：

* 完整的C库可以链接，驱动可以进行许多奇怪的任务，而不用依靠外面的程序（实现使用策略的工具程序，常常随着驱动自身发布）。
* 程序员可以在驱动代码上运行常用的调试器，而不必调试一个运行中的内核的弯路。
* 如果一个用户空间驱动挂起了，你可以简单地杀死它。用户空间驱动出现问题不可能挂起整个系统，除非被控制的硬件真的疯掉了。
* 用户内存是可交换的，不像内核内存。这样一个不常使用却有很大一个驱动的设备不会占据别的程序可以用到的RAM，除了在它实际在用时。一个精心设计的驱动程序仍然可以如同内核空间驱动一样允许对设备的并行存取。
* 如果你必须编写一个封闭源码的驱动，用户空间的选项使你容易辨明不明朗的许可的情况和改变的内核接口带来的问题。

但是，用户空间的设备驱动有几个缺点，最重要的是：

* 中断在用户空间无法使用，在某些平台上有对这个限制的解决方法，例如在IA32体系结构上的vm86系统调用。  
* 只可能通过内存映射 ``/dev/mem`` 来使用DMA，而且只有特权用户可以这样做。  
* 存取I/O端口只能在调用 ``ioperm`` 或者 ``iopl`` 只有，此外，不是所有的平台都支持这些系统调用，而存取 ``/dev/port`` 可能太慢而无效率，这些系统调用和设备文件都要求特权用户。
* 响应时间慢，因为需要上下文切换在用户和硬件之间传递消息和动作。
* 更坏的是，如果驱动已经被交换到硬盘，响应时间会长到不可接受，使用 ``mlock`` 系统调用可能会有帮助，但是你需要经常锁住许多内存页，因为一个用户空间程序依赖大量的库代码，``mlock`` 也限制在授权用户上。
* 最重要的设备不能在用户空间处理，包括网络接口和块设备。

因此，一般的建议是，当你开始处理新的没有用过的硬件时，通过开发用户空间驱动，你可以学习去管理你的硬件，不必担心挂起整个系统，一旦你完成了，再把它封装成内核模块（*什么是内核模块？*）

在本章的内容中，我们假设读者已经掌握了某种硬件，并在用户空间成功驱动了它。是时候把驱动程序封装成内核模块了。

在前几章的例程中，我们能在树莓派上完成从编译到运行的所有过程，因为我们只用到了树莓派上的gcc编译器。但是，编译内核模块，同时需要gcc编译器和内核源代码树，后者树莓派上并未提供，所以，我们必须交叉编译内核模块（*什么是交叉编译？*）。

在此之前，下面几个准备工作需要先完成，否则我们将无法交叉编译自己的内核模块：

1. `建立树莓派交叉编译环境 <附录-建立树莓派交叉编译环境.html>`_
#. `交叉编译新内核和驱动模块 <附录-交叉编译新内核和驱动模块.html>`_

现在，我们可以着手封装自己的内核模块了，这次我们再次点亮一盏LED（伟大的LED实验！）。实验例程位于 https://github.com/openRPi/gpio/tree/master/kernel_module/blink