---
title: UINPUT 事件注入
date: 2016-01-16 14:08:12
categories:
- Android
tags:
- uinput
- linux
description: 初识 Linux 中 uinput 模块
---

简介
---

**uinput** 是 **Linux Kernel** 的一个模块，是向用户层提供输入接口的子系统。我们可以通过 **uinput** 在 */dev/input/* 中创建一个虚拟字符设备并注入事件。本文主要讲解如何使用 **uinput** 实现键盘事件的注入。

创建设备
---

1.**uinput** 通常位于 */dev/uinput* 或者 */dev/input/uinput* ，终端执行 `adb shell ls -al /dev/uinput` ，可以看到 

> crw-rw---- system  system  10, 223 1970-01-01 08:00 uinput

该文件为系统权限的字符设备，作为一个普通应用程序没有办法访问，所以在访问 **uinput** 之前，先要提权。终端执行 `adb shell su && chmod 666 /dev/uinput` 更改字符设备的访问权限，再次执行 `adb shell ls -al /dev/uinput` 可以看到

> crw-rw-rw- system  system  10, 223 1970-01-01 08:00 uinput

<!--more-->

2.提权后便可以使用 **uinput** 了，以 **只读** 和 **非阻塞** 模式打开它

```cpp
#include <linux/input.h>
#include <linux/uinput.h>
#include <fcntl.h>
#include <stdlib.h>
#include <stdio.h>
...

int fd = open("/dev/uinput", O_WRONLY | O_NONBLOCK);
if (fd < 0) {
    fd = open("/dev/input/uinput", O_WRONLY | O_NONBLOCK);
    if (fd < 0) {
        ...
        return -1;
    }
}
```

3.打开后需要配置 **uinput** 哪些类型的输入事件会用到，这些事件定义在 **linux/input.h** 文件中

| 宏       |     释义 |
| :-------- | :--------|
| define EV_SYN 0x00 |   同步事件 |
| define EV_KEY 0x01 |   键盘按下和释放事件 |
| define EV_REL 0x02 |   相对坐标事件 |
| define EV_ABS 0x03 |   绝对坐标事件 |


对于键盘事件，进行如下配置（允许发送 key 和 syn 两种类型的事件）
```cpp
ret = ioctl(fd, UI_SET_EVBIT, EV_KEY);
ret = ioctl(fd, UI_SET_EVBIT, EV_SYN);
```

配置哪些按键可以输入（上下左右方向键）
```cpp
// 允许输入方向键上、下、左、右
ret = ioctl(fd, UI_SET_KEYBIT, KEY_UP);
ret = ioctl(fd, UI_SET_KEYBIT, KEY_DOWN);
ret = ioctl(fd, UI_SET_KEYBIT, KEY_LEFT);
ret = ioctl(fd, UI_SET_KEYBIT, KEY_RIGHT);
```

4.上面已经配置了一些基本特性，接下来介绍结构体 `uinput_user_dev `，它定义在 **linux/uinput.h** 头文件中

```cpp
#define UINPUT_MAX_NAME_SIZE 80
struct uinput_user_dev {
    char name[UINPUT_MAX_NAME_SIZE];
    struct input_id id;
        int ff_effects_max;
        int absmax[ABS_MAX + 1];
        int absmin[ABS_MAX + 1];
        int absfuzz[ABS_MAX + 1];
        int absflat[ABS_MAX + 1];
};
```

其中有几个比较重要的字段:
- name 要创建的虚拟设备名称
- id 内部结构体，描述设备的 usb 类型，厂商，产品，版本
- absmin and absmax 整型数组，描述鼠标或触屏事件的阈值

将虚拟设备的信息写入 `fd` ，虚拟设备命名为 **uinput-sample**
```cpp
struct uinput_user_dev dev;

memset(&dev, 0, sizeof(dev));
snprintf(dev.name, UINPUT_MAX_NAME_SIZE, "uinput-sample"); // 虚拟设备名称
dev.id.bustype = BUS_USB; // udb 类型
dev.id.vendor = 0x1;
dev.id.product = 0x1;
dev.id.version = 1;

write(fd, &dev, sizeof(dev)); // 写入
```

5.最后，创建配置好的虚拟设备
```cpp
ret = ioctl(fd, UI_DEV_CREATE)
```
终端执行 `adb shell getevent` 你会看到虚拟设备 **uinput-sample** 已经创建成功:  **/dev/input/event1**

> add device 1: /dev/input/event1
>   name:     "uinput-sample" 
>   
> add device 2: /dev/input/event6
>   name:     "eventserver-Joystick"
>   
> add device 3: /dev/input/event5
>   name:     "eventserver-Mouse"
>   
> add device 4: /dev/input/event0
>   name:     "aml_keypad"


***

事件注入
---
创建好了设备，开始注入事件，这里还需介绍一个结构体 `input_event` ，它定义在 **linux/input.h** 中，其中几个重要的字段：

- `type`:  event 的类型 (EV_KEY, EV_ABS, EV_REL, ...),
- `code`:  如果 type 是 EV_KEY 类型，则是 key code；若type是 EV_ABS 或者 EV_REL 类型，则是 x、y轴
- `value`:   如果 type 是 EV_KEY 类型，则 1 (press) 0 (release)；若type是 EV_ABS 或者 EV_REL 类型，则为 x，y 坐标


声明写入事件的方法
```cpp
int writeEvent(int fd,int type,int code,int value){
    struct input_event ev;
    memset(&ev, 0, sizeof(ev));
    ev.type = type;
    ev.code = code;
    ev.value = value;
    ssize_t bytes;// 写入文档的字节数（成功）；-1（出错）
    bytes = write(fd, &ev, sizeof(struct input_event));
    return (int) bytes;
}
```

写入一个键盘方向键事件  **KEY_UP** 
```cpp
writeEvent(fd,EV_KEY,KEY_UP,1) // KEY_UP press
writeEvent(fd,EV_KEY,KEY_UP,0) // KEY_UP release
```

将写入的事件同步到虚拟设备
```cpp
writeEvent(fd,EV_SYN,SYN_REPORT,0)
```

***

销毁设备
---
注入了事件之后需要销毁虚拟设备，调用下面的函数，再次执行 `adb shell getevent` 你会看到设备已被移除
```cpp
int destroy(int fd) {
    if (fd != -1) {
        int ret = ioctl(fd, UI_DEV_DESTROY);
        return ret;
    }
    return -1;
}
```

实现 uinput 事件注入步骤就是：**创建虚拟设备->注入事件->销毁设备**，上文只是介绍了键盘事件的注入，其实也可以注入鼠标事件，触摸事件等，接下来的文章会介绍到，请大家多多关注。

**参考：**
https://source.android.com/devices/input/input-device-configuration-files.html
http://thiemonge.org/getting-started-with-uinput
https://www.kernel.org/doc/Documentation/input/multi-touch-protocol.txt
http://bitmath.org/code/mtdev/
http://stackoverflow.com/questions/4386449/send-touch-event-from-adb-to-a-device
http://blog.csdn.net/droidphone/article/details/8434768