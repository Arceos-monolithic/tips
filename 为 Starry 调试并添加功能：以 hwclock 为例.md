# 为 Starry 调试并添加功能：以 hwclock 为例

比方说我们在 Starry 打开的 busybox 终端输入 `busybox hwclock` 输出是一个非法时间点，这肯定是功能没实现，如何解决？

首先可以根据通过指导书给出的[打印 syscall 输出](https://scpointer.github.io/rcore2oscomp/docs/lab2/pos.html)和[strace](https://scpointer.github.io/rcore2oscomp/docs/lab3/lsdebug.html) 调试。

## 打印输出以后，看哪些 syscall 可能有问题

对于 hwclock 这个例子来说，我这的发现是 sys_open 想要打开的文件都没有

```bash
/ # busybox hwclock
sys_open: dir_fd=-100, path="/etc/adjtime", flags=0x8000, mode=0o666
sys_open: dir_fd=-100, path="/dev/rtc", flags=0x8000, mode=0o0
sys_open: dir_fd=-100, path="/dev/rtc0", flags=0x8000, mode=0o0
sys_open: dir_fd=-100, path="/dev/misc/rtc", flags=0x8000, mode=0o0
sys_open: dir_fd=-100, path="/etc/localtime", flags=0x88800, mode=0o11326444377
Sun Dec 31 00:00:00 1899  0.000000 seconds
sys_open: dir_fd=-100, path="/etc/passwd", flags=0x8000, mode=0o666
```

遇到不懂的 syscall 可以去[这个网址](https://jborza.com/post/2021-05-11-riscv-linux-syscalls/)查定义，但对于这个例子来说，应用问内核要打开这么多文件，结果要啥啥没有，这总能意识到是有问题的吧？

## 在本机 strace 试一下正常的结果是什么

我这里本机需要 sudo 权限才能执行，所以命令是 `sudo strace hwclock`，结果是

```bash
execve("/usr/sbin/hwclock", ["hwclock"], 0x7fff4b473a90 /* 13 vars */) = 0
brk(NULL)                               = 0x55e8686da000
arch_prctl(0x3001 /* ARCH_??? */, 0x7ffcb478dd80) = -1 EINVAL (Invalid argument)
......
close(3)                                = 0
openat(AT_FDCWD, "/dev/rtc0", O_RDONLY) = 3
access("/etc/adjtime", R_OK)            = -1 ENOENT (No such file or directory)
ioctl(3, RTC_UIE_ON)                    = 0
pselect6(4, [3], NULL, NULL, {tv_sec=10, tv_nsec=0}, NULL) = 1 (in [3], left {tv_sec=8, tv_nsec=978596485})
ioctl(3, PHN_NOT_OH or RTC_UIE_OFF)     = 0
ioctl(3, RTC_RD_TIME, {tm_sec=9, tm_min=19, tm_hour=9, tm_mday=16, tm_mon=0, tm_year=124, ...}) = 0
openat(AT_FDCWD, "/etc/localtime", O_RDONLY|O_CLOEXEC) = 4
newfstatat(4, "", {st_mode=S_IFREG|0644, st_size=561, ...}, AT_EMPTY_PATH) = 0
newfstatat(4, "", {st_mode=S_IFREG|0644, st_size=561, ...}, AT_EMPTY_PATH) = 0
read(4, "TZif2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 561
lseek(4, -342, SEEK_CUR)                = 219
read(4, "TZif2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 342
close(4)                                = 0
newfstatat(1, "", {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x8), ...}, AT_EMPTY_PATH) = 0
write(1, "2024-01-16 17:19:07.949168+08:00"..., 332024-01-16 17:19:07.949168+08:00
) = 33
close(-1)                               = -1 EBADF (Bad file descriptor)
close(3)                                = 0
dup(1)                                  = 3
close(3)                                = 0
dup(2)                                  = 3
close(3)                                = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```

中间很长被我省略了，是一些动态库的处理，但总之你可以找 **Starry 上syscall输出的前面那些文件**和**真正的输出（2024开头的时间戳**在哪

## 分析要加哪些东西

看上面的输出，有哪些在 Starry 那边也有的？

- `access("/etc/adjtime", R_OK) = -1` 说明这个文件在本机也找不到，那不管他

- `openat(AT_FDCWD, "/dev/rtc0", O_RDONLY) = 3`这个文件成功打开了，要管

- `ioctl(3, RTC_UIE_ON)` `ioctl(3, PHN_NOT_OH or RTC_UIE_OFF)` `ioctl(3, RTC_RD_TIME,...` 这里 3 是 fd，也就是说全在处理上面打开的 `/dev/rtc0`，怎么处理下面再说

- `openat(AT_FDCWD, "/etc/localtime", O_RDONLY|O_CLOEXEC) = 4` `read(4, "TZif2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 561` 这个文件也成功打开了

## 具体考虑怎么加

关于 `/dev/rtc0` 你可以搜到

- [rtc定时器配置ioctl 设置 RTC_AIE_OFF、RTC_RD_TIME、RTC_ALM_SET、RTC_AIE_ON-CSDN博客](https://blog.csdn.net/sunfanup/article/details/135531804) 实际使用的例子

- [rtc(4) - Linux manual page](https://www.man7.org/linux/man-pages/man4/rtc.4.html) 具体的规范

- `RTC_UIE_ON` 以及 `RTC_RD_TIME` 等等这些常量具体是什么？
  
  - 可以直接打印 Starry 上的输出，看 busybox 输入的是什么
  
  - 也可以去[规范页面]([rtc(4) - Linux manual page](https://www.man7.org/linux/man-pages/man4/rtc.4.html)) 提到的 `#include <linux/rtc.h>` 里找。Linux 的源码很容易搜到的
  
  - 也可以按照指导书里提的，去 musl-libc 的源码里搜。第三章教了大家如何看 libc 源码

- syscall 应该返回什么？看 Strace 的输出以及规范。Strace 说的 `ioctl(3, RTC_RD_TIME, {tm_sec=9, tm_min=19, tm_hour=9, tm_mday=16, tm_mon=0, tm_year=124, ...}) = 0` 里面一堆时间就是内核给用户程序的返回值。
  
  另一方面，规范里写了：
  
  ```
  RTC_RD_TIME
  Returns this RTC's time in the following structure:
  
    struct rtc_time {
        int tm_sec;
        int tm_min;
        int tm_hour;
        int tm_mday;
        int tm_mon;
        int tm_year;
        int tm_wday;     /* unused */
        int tm_yday;     /* unused */
        int tm_isdst;    /* unused */
    };
    The fields in this structure have the same meaning and
  ranges as for the tm structure described in gmtime(3).  A
  pointer to this structure should be passed as the third
  ioctl(2) argument.
  ```

- Starry 里怎么知道获取真正时间？Starry 里目前还真没法拿到真正的时钟，只有启动后的时间。但**只要实现一个从合理的常量时间（如2024/1/1)加上系统启动时间作为输出**，就已经算完成了90%了。后续**留个文档**，其他人把常量时间替换成真正的 hwc 就完事了，但你**不能因为在 Starry 中找不到 hwc 就卡在这不动**。

关于 `/etc/localtime`

- 可以搜到这是一个关于定义时区的文件

- 有空可以研究它的定义，但我们赶时间的话就**在你主机上 cat /etc/localtime，把输出复制一份直接放到 Starry 里**，也是可以的。

## 加到代码实现里

最简单的方案是使用 `if-else`，打开文件时判断如果是 `/dev/rtc0` 这样的字符串，就另外套一层 `if` 代码处理它。这能让代码第一次成功运行，但不是最终方案

之后，需要考虑它应该被塞到哪个功能里。以 Starry 为例，它们归 VFS 管。

- 上述的 `/dev/rtc0` 应该添加进 `axfs_devfs` 里

- `/etc/localtime` 可以看着办，比如你可以开机时塞一个这样的文件进去。在`ulib/axstarry/src/test.rs` 的 `fs_init` 函数开头就有一堆这样的东西，加一个就行。


