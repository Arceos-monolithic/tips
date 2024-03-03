## [issue](https://github.com/Arceos-monolithic/Starry/issues/15) 提到的无效系统调用问题

测试 cyclictest 需要开启一个叫 schedule 的 feature。具体来说，需要在根目录 Makefile 中讲 `FEATURES ?=` 一行改为 `FEATURES ?= schedule` 或者直接在 make 命令中添加。

在 riscv64 下指定测例为 `busybox sh ./cyclictest_testcode.sh"` 然后行下列命令：

```bash
./build_img riscv64
make run ARCH=riscv64 FEATURES=schedule
```

可得到正确输出

```
====== cyclictest NO_STRESS_P1 begin ======
WARN: stat /dev/cpu_dma_latency failed: No such file or directory
T: 0 (    9) P:99 I:1000 C:    108 Min:     60 Act: 9277 Avg: 8738 Max:   10963
====== cyclictest NO_STRESS_P1 end: success ======
====== cyclictest NO_STRESS_P8 begin ======
WARN: stat /dev/cpu_dma_latency failed: No such file or directory
T: 0 (   12) P:99 I:1000 C:    119 Min:     34 Act: 8774 Avg: 8014 Max:   11145
T: 1 (   13) P:99 I:1500 C:    107 Min:     34 Act:15420 Avg: 8699 Max:   15420
T: 2 (   14) P:99 I:2000 C:    102 Min:     65 Act:16243 Avg: 8484 Max:   16243
T: 3 (   15) P:99 I:2500 C:    101 Min:   4746 Act:15931 Avg: 8361 Max:   15931
T: 4 (   16) P:99 I:3000 C:    107 Min:     30 Act:15158 Avg: 8091 Max:   15158
T: 5 (   17) P:99 I:3500 C:    102 Min:     61 Act:15589 Avg: 7888 Max:   15589
T: 6 (   18) P:99 I:4000 C:    104 Min:     44 Act:15644 Avg: 7552 Max:   15644
T: 7 (   19) P:99 I:4500 C:    101 Min:     34 Act:10644 Avg: 8090 Max:   10943
====== cyclictest NO_STRESS_P8 end: success ======
====== start hackbench ======
Running in process mode with 10 groups using 40 file descriptors each (== 400 tasks)
Each sender will pass 100000000 messages of 100 bytes
Creating fdpair (error: Address family not supported by protocol)
====== cyclictest STRESS_P1 begin ======
WARN: stat /dev/cpu_dma_latency failed: No such file or directory
T: 0 (   26) P:99 I:1000 C:    111 Min:     32 Act: 8347 Avg: 8496 Max:   10943
====== cyclictest STRESS_P1 end: success ======
====== cyclictest STRESS_P8 begin ======
WARN: stat /dev/cpu_dma_latency failed: No such file or directory
T: 0 (   29) P:99 I:1000 C:    108 Min:     48 Act:11654 Avg: 8644 Max:   11654
T: 1 (   30) P:99 I:1500 C:    105 Min:     58 Act:12759 Avg: 8757 Max:   12759
T: 2 (   31) P:99 I:2000 C:    101 Min:   5311 Act:13014 Avg: 8756 Max:   13014
T: 3 (   32) P:99 I:2500 C:    101 Min:   4589 Act:12763 Avg: 8659 Max:   12763
T: 4 (   33) P:99 I:3000 C:    106 Min:     33 Act:12419 Avg: 8071 Max:   12419
T: 5 (   34) P:99 I:3500 C:    101 Min:   3149 Act:12633 Avg: 8094 Max:   12633
T: 6 (   35) P:99 I:4000 C:    102 Min:     42 Act:12686 Avg: 7851 Max:   12686
T: 7 (   36) P:99 I:4500 C:    103 Min:     52 Act: 9526 Avg: 7698 Max:   10051
====== cyclictest STRESS_P8 end: success ======
====== kill hackbench: success ======
```

## 新问题：用户程序报错无法获取参数

如果在 `x86_64` 下运行类似命令：

```shell
./build_img
make run FEATURES=schedule
```

虽然内核没有报错，但用户程序的输出显示测试失败：

```
====== cyclictest NO_STRESS_P1 begin ======
unable to get scheduler parameters
Hangup
====== cyclictest NO_STRESS_P1 end: fail ======
====== cyclictest NO_STRESS_P8 begin ======
unable to get scheduler parameters
Hangup
====== cyclictest NO_STRESS_P8 end: fail ======
====== start hackbench ======
Running in process mode with 10 groups using 40 file descriptors each (== 400 tasks)
Each sender will pass 100000000 messages of 100 bytes
Creating fdpair (error: Address family not supported by protocol)
====== cyclictest STRESS_P1 begin ======
unable to get scheduler parameters
Hangup
====== cyclictest STRESS_P1 end: fail ======
====== cyclictest STRESS_P8 begin ======
unable to get scheduler parameters
Hangup
====== cyclictest STRESS_P8 end: fail ======
====== kill hackbench: success ======
```

查找 cyclictest 源码，发现 `unable to get scheduler parameters` 一句对应的是 `sched_getparam` 系统调用报错时的输出：

```c
    if (sched_setscheduler(0, SCHED_FIFO, &param)) {
        fprintf(stderr, "Unable to change scheduling policy!\n");
        fprintf(stderr, "either run as root or join realtime group\n");
        return 1;
    }
```

strace bash ./cyclictest_testcode.sh
但开启 `LOG=warn` 后内核输出显示，在输出报错语句时，用户程序只调用了 sched_getaffinity，并没有调 sched_getparam：

```
......
[  0.237253 0:10 syscall_entry::syscall:38] [syscall] id = ARCH_PRCTL, args = [4098, 5144, 67171115, 34, 18446744073709551615, 0], entry
[  0.237494 0:10 syscall_entry::syscall:48] [syscall] id = 158, args = [4098, 5144, 67171115, 34, 18446744073709551615, 0], return 0
[  0.237739 0:10 syscall_entry::syscall:38] [syscall] id = SET_TID_ADDRESS, args = [69313808, 5144, 67171115, 34, 0, 0], entry
[  0.237981 0:10 syscall_entry::syscall:48] [syscall] id = 218, args = [69313808, 5144, 67171115, 34, 0, 0], return 10
[  0.238267 0:10 syscall_entry::syscall:38] [syscall] id = SCHED_GETAFFINITY, args = [0, 128, 1073739920, 34, 28, 0], entry
[  0.238409 0:10 syscall_entry::syscall:48] [syscall] id = 204, args = [0, 128, 1073739920, 34, 28, 0], return 1
[  0.239194 0:10 syscall_entry::syscall:29] [syscall] id = WRITEV, args = [2, 1073740016, 2, 34, 1073740432, 0], entry
unable to get scheduler parameters
......
```

这就很奇怪了。

在我本机（x86）直接运行这个 `./cyclictest` 也会出现一样的报错信息 `unable to get scheduler parameters`。再使用 strace 命令在本机验证一下：

```bash
❯ strace -f bash ./cyclictest_testcode.sh
......
[pid   817] arch_prctl(ARCH_SET_FS, 0x7f5c77e09418) = 0
[pid   817] set_tid_address(0x7f5c77e0d510) = 817
[pid   817] sched_getaffinity(0, 128, [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19]) = 32
[pid   817] writev(2, [{iov_base="", iov_len=0}, {iov_base="unable to get scheduler paramete"..., iov_len=35}], 2unable to get scheduler parameters
) = 35
[pid   817] exit_group(1)               = ?
......
```

可以看到确实只调用了 `sched_getaffinity`，说明有可能是当初这个测例程序编译时的问题，使用的 libc 库没有链接到 `sched_getparam`。

## 重新编译生成 cyclictest

下面尝试重新编译一份 cyclictest，在本机试试能否跑通

首先查看原本提供的测例程序的参数

```bash
❯ file ./cyclictest
./cyclictest: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), static-pie linked, not stripped
❯ ./cyclictest --version
./cyclictest: unrecognized option: version
cyclictest V 1.00
Usage:
cyclictest <options>

-a [NUM] --affinity        run thread #N on processor #N, if possible
                           with NUM pin all threads to the processor NUM
-A USEC  --aligned=USEC    align thread wakeups to a specific offset
-b USEC  --breaktrace=USEC send break trace command when latency > USEC
-B       --preemptirqs     both preempt and irqsoff tracing (used with -b)
......
```

发现是静态+PIE编译，版本是 1.00。到[官方维护仓库](https://git.kernel.org/pub/scm/utils/rt-tests/rt-tests.git) 上下了一份 1.00 的代码，加上 `-static-pie` 编译后得到：

```
./cyclictest: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, BuildID[sha1]=7ab0d84e570e04cd84bcb4404a1d71624d0f339d, for GNU/Linux 3.2.0, not stripped
```

还想把 `version 1 (GNU/Linux)` 改成原版的 `SYSV` 编译，试了很多种方法没有成功。可能是比赛官方测例的编译器比较奇怪，但已经不可考了。

## 测试新的 cyclictest

输入以下命令在本机测试

```
❯ bash ./cyclictest_testcode.sh
====== cyclictest NO_STRESS_P1 begin ======
Unable to change scheduling policy!
either run as root or join realtime group
====== cyclictest NO_STRESS_P1 end: fail ======
====== cyclictest NO_STRESS_P8 begin ======
Unable to change scheduling policy!
either run as root or join realtime group
====== cyclictest NO_STRESS_P8 end: fail ======
====== start hackbench ======
Running in process mode with 10 groups using 40 file descriptors each (== 400 tasks)
Each sender will pass 100000000 messages of 100 bytes
====== cyclictest STRESS_P1 begin ======
Unable to change scheduling policy!
either run as root or join realtime group
====== cyclictest STRESS_P1 end: fail ======
====== cyclictest STRESS_P8 begin ======
Unable to change scheduling policy!
either run as root or join realtime group
====== cyclictest STRESS_P8 end: fail ======
Signal 2 caught, longjmp'ing out!
longjmp'ed out, reaping children
sending SIGTERM to all child processes
signaling 400 worker threads to terminate
Time: 1.023
====== kill hackbench: success ======

~/2024/rt-tests-1.0
❯ sudo bash ./cyclictest_testcode.sh
[sudo] password for scpointer: 
====== cyclictest NO_STRESS_P1 begin ======
# /dev/cpu_dma_latency set to 0us
T: 0 (18237) P:99 I:1000 C:   1000 Min:      7 Act:   51 Avg:   72 Max:     329
====== cyclictest NO_STRESS_P1 end: success ======
====== cyclictest NO_STRESS_P8 begin ======
# /dev/cpu_dma_latency set to 0us
T: 0 (18253) P:99 I:1000 C:    998 Min:      3 Act:  224 Avg:   45 Max:    2157
T: 1 (18254) P:99 I:1500 C:    667 Min:      8 Act:   27 Avg:   55 Max:     351
T: 2 (18255) P:99 I:2000 C:    500 Min:      4 Act:   44 Avg:   77 Max:     377
T: 3 (18256) P:99 I:2500 C:    400 Min:      8 Act:   32 Avg:   51 Max:     265
T: 4 (18257) P:99 I:3000 C:    334 Min:      3 Act:   29 Avg:   58 Max:     343
T: 5 (18258) P:99 I:3500 C:    286 Min:      7 Act:   28 Avg:   50 Max:     466
T: 6 (18259) P:99 I:4000 C:    250 Min:      5 Act:   41 Avg:   59 Max:     428
T: 7 (18260) P:99 I:4500 C:    223 Min:      8 Act:   54 Avg:   64 Max:     388
====== cyclictest NO_STRESS_P8 end: success ======
====== start hackbench ======
Running in process mode with 10 groups using 40 file descriptors each (== 400 tasks)
Each sender will pass 100000000 messages of 100 bytes
====== cyclictest STRESS_P1 begin ======
# /dev/cpu_dma_latency set to 0us
T: 0 (18685) P:99 I:1000 C:    998 Min:      3 Act:   10 Avg:   16 Max:    1187
====== cyclictest STRESS_P1 end: success ======
====== cyclictest STRESS_P8 begin ======
# /dev/cpu_dma_latency set to 0us
T: 0 (18729) P:99 I:1000 C:    999 Min:      3 Act:    6 Avg:   13 Max:     975
T: 1 (18730) P:99 I:1500 C:    667 Min:      4 Act:    9 Avg:   16 Max:     556
T: 2 (18731) P:99 I:2000 C:    500 Min:      4 Act:    6 Avg:   15 Max:     931
T: 3 (18732) P:99 I:2500 C:    400 Min:      4 Act:    6 Avg:   15 Max:     471
T: 4 (18733) P:99 I:3000 C:    333 Min:      5 Act:    6 Avg:   21 Max:     905
T: 5 (18734) P:99 I:3500 C:    286 Min:      4 Act:   12 Avg:   17 Max:     546
T: 6 (18735) P:99 I:4000 C:    249 Min:      4 Act:    9 Avg:   15 Max:     262
T: 7 (18736) P:99 I:4500 C:    222 Min:      5 Act:    6 Avg:   24 Max:    2331
====== cyclictest STRESS_P8 end: success ======
Signal 2 caught, longjmp'ing out!
longjmp'ed out, reaping children
sending SIGTERM to all child processes
signaling 400 worker threads to terminate
Time: 3.107
====== kill hackbench: success ======
```

第一次测试时 `bash ./cyclictest_testcode.sh` 的报错信息 `Unable to change scheduling policy!` 只是权限问题，加了 `sudo` 之后就好了。第二次的输出证明刚编译的这个 cyclictest 没有问题。

## 在 Starry 中运行新测例

重新生成镜像，加 `LOG=warn` 运行后输出

```
......
[  0.238421 0:8 syscall_entry::syscall:21] [syscall] id = MMAP, args = [0, 1048576, 3, 34, 4294967295, 0], entry
[  0.238778 0:8 syscall_entry::syscall:48] [syscall] id = 9, args = [0, 1048576, 3, 34, 4294967295, 0], return 4096
[  0.240225 0:8 syscall_entry::syscall:21] [syscall] id = MPROTECT, args = [5513216, 16384, 1, 96, 5344, 3], entry
[  0.240928 0:8 syscall_entry::syscall:48] [syscall] id = 10, args = [5513216, 16384, 1, 96, 5344, 3], return 0
[  0.241701 0:8 syscall_entry::syscall:29] [syscall] id = OPENAT, args = [4294967196, 5287030, 0, 0, 8, 1], entry
[  0.242952 0:8 axruntime::lang_items:5] panicked at ulib/axstarry/syscall_entry/src/syscall.rs:43:9:
unknown syscall id: 239
```

查表可知这个 syscall 是内核还没实现的 `get_mempolicy`。新加一个 syscall 直接返回 0，再次运行后，卡在另一个 syscall 上：

```
[  0.275971 0:8 syscall_entry::syscall:21] [syscall] id = MMAP, args = [0, 266240, 0, 131106, 4294967295, 0], entry
[  0.276283 0:8 syscall_entry::syscall:48] [syscall] id = 9, args = [0, 266240, 0, 131106, 4294967295, 0], return 1052672
[  0.276514 0:8 syscall_entry::syscall:21] [syscall] id = MPROTECT, args = [1056768, 262144, 3, 18446744073709551552, 4294967295, 0], entry
[  0.276940 0:8 syscall_entry::syscall:48] [syscall] id = 10, args = [1056768, 262144, 3, 18446744073709551552, 4294967295, 0], return 0
[  0.277455 0:8 syscall_entry::syscall:38] [syscall] id = SIGPROCMASK, args = [0, 5295200, 1073739456, 8, 261760, 64], entry
[  0.277706 0:8 syscall_entry::syscall:48] [syscall] id = 14, args = [0, 5295200, 1073739456, 8, 261760, 64], return 0
[  0.278128 0:8 axruntime::lang_items:5] panicked at ulib/axstarry/syscall_entry/src/syscall.rs:43:9:
unknown syscall id: 435, args = [1073739120, 88, 4370928, 8, 1316416, 1073739391]
```

这次是缺 clone3。这个 syscall 还是挺常用的，不过 Starry 目前只有 clone 没有 clone3。

因为这次本机输出是正常的，所以可以用 strace 看看应该返回什么：

```
sudo strace -f -o log bash ./cyclictest_testcode.sh
grep -n clone3 log
......(这里有非常多输出，总之第285行就已经看到对应位置了)
head -500 log > log2
```

之后就能找到对应的调用了：

```
1513  mmap(NULL, 8392704, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7f58d397a000
1513  mprotect(0x7f58d397b000, 8388608, PROT_READ|PROT_WRITE) = 0
1513  rt_sigprocmask(SIG_BLOCK, ~[], [ALRM], 8) = 0
1513  clone3({flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, child_tid=0x7f58d417a910, parent_tid=0x7f58d417a910, exit_signal=0, stack=0x7f58d397a000, stack_size=0x7ffe80, tls=0x7f58d417a640} <unfinished ...>
1514  rseq(0x7f58d417afe0, 0x20, 0, 0x53053053 <unfinished ...>
1513  <... clone3 resumed> => {parent_tid=[1514]}, 88) = 1514
```

## 实现 clone3 之后

简单写了一版 sys_clone3 放在内核里。

```
/// 创建子进程的新函数，所有信息保存在 CloneArgs
pub fn syscall_clone3(
    args: *const CloneArgs,
    size: usize,
) -> SyscallResult {
    assert!(size >= size_of::<CloneArgs>());

    let curr_process = current_process();

    let args = match curr_process.manual_alloc_type_for_lazy(args) {
        Ok(_) => unsafe { &*args },
        Err(_) => return Err(SyscallError::EFAULT),
    };

    let clone_flags = CloneFlags::from_bits(args.flags as u32).unwrap();

    let stack = if args.stack == 0 {
        None
    } else {
        Some(args.stack as usize)
    };
    #[cfg(feature = "signal")]
    let sig_child = SignalNo::from(args.exit_signal as usize & 0x3f) == SignalNo::SIGCHLD;

    warn!("stack size  {}", args.stack_size);
    if let Ok(new_task_id) = curr_process.clone_task(
        clone_flags,
        stack,
        args.parent_tid as usize,
        args.tls as usize,
        args.child_tid as usize,
        #[cfg(feature = "signal")]
        sig_child,
    ) {
        Ok(new_task_id as isize)
    } else {
        return Err(SyscallError::ENOMEM);
    }
}
```

功能与 sys_clone 相同，只是子进程信号与 Flags 被拆开了，而且参数多传了一个新进程用户栈的大小在 CloneArgs 里。再次运行后，出现如下报错

```
====== cyclictest NO_STRESS_P1 begin ======
WARN: stat /dev/cpu_dma_latency failed: No such file or directory
cyclictest: allocatestack.c:193: advise_stack_range: Assertion `freesize < size' failed.
```

这个报错的文件并不在 cyclictest 里，而是在 glibc 里。查 glibc 代码，发现这条出错很可能是用户程序爆栈了。再次检查 strace 信息（就是上面同一段）：

```
1513  mmap(NULL, 8392704, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7f58d397a000
1513  mprotect(0x7f58d397b000, 8388608, PROT_READ|PROT_WRITE) = 0
1513  rt_sigprocmask(SIG_BLOCK, ~[], [ALRM], 8) = 0
1513  clone3({flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, child_tid=0x7f58d417a910, parent_tid=0x7f58d417a910, exit_signal=0, stack=0x7f58d397a000, stack_size=0x7ffe80, tls=0x7f58d417a640} <unfinished ...>
1514  rseq(0x7f58d417afe0, 0x20, 0, 0x53053053 <unfinished ...>
1513  <... clone3 resumed> => {parent_tid=[1514]}, 88) = 1514
```

发现本机上运行时 clone3 向内核报告的栈大小是 `stack_size=0x7ffe80`与之对应的是上面 mmap 的 8392704（即 `0x801000`）和 mprotect 的 8388608（即`0x800000`）。

与之相对的，加 LOG=warn 输出后，发现在 Starry 上 clone3 只报告了 261760 大小的内存。为什么用户程序没要够内存呢？继续往上找 prlimit，发现本机上运行时内核告诉应用可以开 `0x800000` 这么大：

```
1512  prlimit64(0, RLIMIT_STACK, NULL, 
{rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
```

而 Starry 中的实现为：

```rust
/// 设置任务资源限制
///
/// pid 设为0时，表示应用于自己
pub fn syscall_prlimit64(
    pid: usize,
    resource: i32,
    new_limit: *const RLimit,
    old_limit: *mut RLimit,
) -> SyscallResult {
    // 当pid不为0，其实没有权利去修改其他的进程的资源限制
    let curr_process = current_process();
    if pid == 0 || pid == curr_process.pid() as usize {
        match resource {
            RLIMIT_STACK => {
                if old_limit as usize != 0 {
                    unsafe {
                        *old_limit = RLimit {
                            rlim_cur: TASK_STACK_SIZE as u64,
                            rlim_max: TASK_STACK_SIZE as u64,
                        };
                    }
                }
            }
    ......
```

其中常量 `TASK_STACK_SIZE` 的大小为 262144（即 `0x40000`），这和输出情况是相对应的。

尝试在 `modules/axconfig/defconfig.toml` 调整这个常量的大小，然后 `make clean` 刷新，但似乎还是出现同样的问题。

检查报错前的 syscall调用，发现有一个要求巨大的 mmap：

```
[  1.292807 0:9 syscall_entry::syscall:21] [syscall] id = MMAP, args = [0, 134217728, 0, 16418, 4294967295, 0], entry
[  1.337106 0:9 syscall_entry::syscall:48] [syscall] id = 9, args = [0, 134217728, 0, 16418, 4294967295, 0], return 9773056
[  1.337420 0:9 syscall_entry::syscall:21] [syscall] id = MUNMAP, args = [9773056, 57335808, 57335808, 16418, 4294967295, 0], entry
[  1.338231 0:9 syscall_entry::syscall:48] [syscall] id = 11, args = [9773056, 57335808, 57335808, 16418, 4294967295, 0], return 0
[  1.338403 0:9 syscall_entry::syscall:21] [syscall] id = MUNMAP, args = [134217728, 9773056, 57335808, 16418, 4294967295, 0], entry
[  1.338711 0:9 syscall_entry::syscall:48] [syscall] id = 11, args = [134217728, 9773056, 57335808, 16418, 4294967295, 0], return 0
[  1.338853 0:9 syscall_entry::syscall:21] [syscall] id = MPROTECT, args = [67108864, 135168, 3, 16418, 4294967295, 0], entry
[  1.339188 0:9 syscall_entry::syscall:48] [syscall] id = 10, args = [67108864, 135168, 3, 16418, 4294967295, 0], return 0
[  1.339947 0:9 syscall_entry::syscall:29] [syscall] id = WRITE, args = [2, 5565760, 89, 67111136, 0, 0], entry
```

对应到正常运行的本机 strace 那边，报错位置应该是 madvise 退出之前。

目前这个 bug 的具体原因还没查到

## 寻找比赛的 cyclictest 版本

相对于自己编译的 x86，还有个参照物就是比赛提供的 cyclictest 。[最近一次修改记录](https://github.com/oscomp/testsuits-for-oskernel/commit/ff418c617de498ce03114f9507e5bdabae7843b9#diff-1a164516a5d5dee23b4ac85be69447409d06f4dc96493763787c585a96428b4c)里，Makefile 里开头是 riscv64 也就算了，仔细一看不同测例居然还是各种不同版本的 gcc

本机改成 musl（而不是gnu）编译后，发现报错

```
❯ make      
x86_64-linux-musl-gcc -D VERSION=1.0 -c src/cyclictest/cyclictest.c -Wall -Wno-nonnull -static -fPIC -O2 -D_GNU_SOURCE -Isrc/include -o bld/cyclictest.o
src/cyclictest/cyclictest.c:59: warning: "sigev_notify_thread_id" redefined
   59 | #define sigev_notify_thread_id __sev_fields._tid
      | 
In file included from src/cyclictest/cyclictest.c:22:
/home/scpointer/x86_64-linux-musl-native/include/signal.h:195: note: this is the location of the previous definition
  195 | #define sigev_notify_thread_id __sev_fields.sigev_notify_thread_id
      | 
src/cyclictest/cyclictest.c: In function ‘timerthread’:
src/cyclictest/cyclictest.c:59:44: error: ‘union <anonymous>’ has no member named ‘_tid’
   59 | #define sigev_notify_thread_id __sev_fields._tid
      |                                            ^
src/cyclictest/cyclictest.c:1010:23: note: in expansion of macro ‘sigev_notify_thread_id’
 1010 |                 sigev.sigev_notify_thread_id = stat->tid;
      |                       ^~~~~~~~~~~~~~~~~~~~~~
make: *** [Makefile:91: bld/cyclictest.o] Error 1
```

看起来似乎这个测例集就不能用 musl 编译。比赛仓库里它是怎么编译出来的？再往回找 commit，发现[比赛那边自己还改过 cyclictest 源码](https://github.com/oscomp/testsuits-for-oskernel/commit/651cb12f66085d72152dab2d241758f5aac961f8#diff-3e76e95dfc29b9ebe6dcda68a15670738f44a8c732aff8d61199070caa8c27aa)。

按比赛的方式修改源码以及 Makefile 后再次尝试编译 cyclictest，主要是修改了上面报错的 signal 选项，改为新增 `signal.h` 实现，以及**用 musl 替换掉 gnu**。期间遇到了一个新问题：`pthread_attr_setaffinity_np` 在我的本地的 `x86_64-linux-musl-gcc ver 11.2.1` 中不存在，需要替换成 `pthread_setaffinity_np`。修改后又得到了和比赛完全一直的文件结构：

```
❯ file ./cyclictest
./cyclictest: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), static-pie linked, not stripped
```

在本机和 Starry 运行后，得到了熟悉的结果 `unable to get scheduler parameters`。至此，**我们已经找到了比赛 cyclictest 的编译方式，但它编出来是错误的**。不过这个错误仅存在于 x86+musl 的组合上，它是2024年1月加的，x86的测例还没有实际用到赛中。

总结一下目前 cyclictest 测例本身的情况：

- riscv+musl：编译正常，Starry也能通过

- x86+musl：编译后的可执行文件 `cyclictest`不对，在 x86 本机和 Starry 内部都无法获取正确输出

- x86+gnu：编译正常，在 x86 本机可以通过测试。在 Starry 内部由于对 glibc 的支持不足，会出现 glibc 库中的报错，还待进一步修复
