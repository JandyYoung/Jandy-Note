# 点亮一颗LED

## 主函数流程

首先将 PE7 设置为输出模式，然后进入循环模式：向引脚写入低电平信号并打印 `led on` 和 count 的值，等待500ms，向引脚写入高电平信号并打印 `led off`，等待500ms，将 count 加一。

## 所使用的RT-Thread资源

整个程序使用到了 RT-Thread 中的 PIN设备、内核线程，在程序中共调用了三个头文件，其作用如下：

| 头文件 | 作用 |
| ----- | ----- |
| rtthread.h | rtthread内核头文件，内核 API 的申明 |
| rtdevice.h | rtthread设备驱动框架的整合 |
| board.h | 针对stm32l4xx的头文件 |

整个程序使用到了如下 API：

| 名称 | 定义文件 | 作用 |
| ------ | ------ | -----|
| rt_pin_mode | pin.c | 定义引脚的工作模式 |
| rt_pin_write | pin.c | 向引脚写信号 |
| rt_kprintf | kseervicce.c | 向串口打印数据 |
| rt_thread_mdelay | thread.c | 延时函数，并将线程挂起 |

rt-thread提供了完善的设备框架，其真正的运行原理还需要对其进行分析。

## rt_pin_mode

线程执行的第一步便是对引脚的工作模式定义，其中 `rt_pin_mode` 的源码如下：

```
    void rt_pin_mode(rt_base_t pin, rt_base_t mode)
    {
        RT_ASSERT(_hw_pin.ops != RT_NULL);
        _hw_pin.ops->pin_mode(&_hw_pin.parent, pin, mode);
    }
```

代码简单，但是内部原理不简单。其主要原理便是对 `_hw_pin.ops` 进行断言判断是否为空，若不为空便对其对象进行设置引脚和模式 。那么有以下问题：
1. `_hw_pin` 是什么；
2. 为什么需要判断 `_hw_pin.ops` 里的值是否为空，他里面有什么；
3. `_hw_pin.ops` 中的 `pin_mode` 具体实质操作是什么；

为了解决第一个问题，查看了 `_hw_pin` 的定义，即为 `static struct rt_device_pin _hw_pin;`，那么可以知道 `_hw_pin` 为结构体 `rt_device_pin`，这里可以回答第一个问题，其中  `rt_device_pin` 结构体如下：

```
    struct rt_device_pin
    {
        struct rt_device parent;
        const struct rt_pin_ops *ops;
    };
```

其内部便是设备的内核对象和对引脚的操作接口，但是对于 `_hw_pin` 里具体内容还是一概不知，但是可以确定如果  `_hw_pin.ops` 内部为空则程序无法执行，这也可以回答第二个问题的前一半问题。

通过查找可以知道在 `pin.c` 文件中提供了 `rt_device_pin_register` 来对`_hw_pin` 进行初始化。在这里可以看到它的内核类型为 `RT_Device_Class_Miscellaneous` (该值在 `rt_device_class_type` 中定义)，设置了常用设备接口中的读、写和控制为 `_pin_read`、`_pin_write` 和 `_pin_control`。抱着好奇的心思去看看这三个真正的操作是怎么实现的： 

* _pin_read：先把 `pin` 进行断言判断不为 `RT_NULL`，确定 `status` 不为空和 `size` 与 `status` 的大小相同，执行 `pin->ops->pin_read`，返回 `size`大小。

* _pin_write：先判断 `pin` 进行断言判断不为 `RT_NULL`，确定 `status` 不为空和 `size` 与 `status` 的大小相同，执行 `pin->ops->pin_write`，返回 `size` 大小。

* _pin_control：先判断 `pin` 进行断言判断不为 `RT_NULL`，确定 `mode` 不为 `RT_NULL`，执行 `pin->ops->pin_mode`，返回0。

由上述可知还需知道 `ops` 中的操作，但是 `ops` 为输入形参，那么产生新的问题：

1. `rt_device_pin_register` 在程序中实质在哪里执行；
2. `rt_device_pin_register` 在执行的时候赋值进去的参数是什么；

带着上述问题重新查看程序，可以看到 `drv_gpio.c` 文件中的如下代码：

```
    int rt_hw_pin_init(void)
    {
        return rt_device_pin_register("pin", &_stm32_pin_ops, RT_NULL);
    }
```

然后可以知道在 `board.c` 文件中的 `rt_hw_board_init()`中调用了 `rt_hw_pin_init()` 函数，且必须定义 `BSP_USING_GPIO`，该定义在 `rtconfig.h` 文件中(rtthread的裁剪工作本质也是在该文件中对定义进行修改， menuconfig 工具中的配置也是对该文件的一种操作，其中这也是C语言中的一种小技巧，在附录中进行详细说明)，而 `rt_hw_board_init()` 在 rtthread 启动函数 ` rtthread_startup` 中便得到了执行。

而在这之中可以看到定义在 `drv_gpio.c` 文件中的 `_stm32_pin_ops` 的如下：

```
    const static struct rt_pin_ops _stm32_pin_ops =
    {
        stm32_pin_mode,
        stm32_pin_write,
        stm32_pin_read,
        stm32_pin_attach_irq,
        stm32_pin_dettach_irq,
        stm32_pin_irq_enable,
    };
```

上述API也是 `_hw_pin.ops` 的本质。



## 附录
对于代码的注释最常用的为 **”//“**  和 **”/*... */“**，在 RTThread 中使用了如下注释方法：

```
    #ifdef define_array
    func()
    #endif
```

如果 `define_array` 变量在之前定义了那么就执行 `func()` 程序，如果未定义则不执行，即相当于：

```
    #ifdef 0
    func()
    #endif
```

而上述方法也是一种很好的注释方法，因为 **”//“** 的注释会显得凌乱，而 **”/*... */“** 不支持嵌套。