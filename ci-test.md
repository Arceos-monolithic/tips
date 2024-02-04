# CI-test for Starry

本文档用于说明 [Starry](https://github.com/Arceos-monolithic/Starry) 的 CI 测试的具体情况和需求。

## 分支情况

目前最新的分支是：`https://github.com/Arceos-monolithic/Starry/tree/x86_ZLMediaKit` 。它同时支持宏内核的 `RISC-V` 和 `x86_64`。这个分支应该也支持 `Unikernel` 模式的 `RISC-V/x86_64/aarch64` 架构，不过目前还没测试。

启动前需要根据 README 指引先去修改系统[启动时运行的程序](https://github.com/Arceos-monolithic/Starry/blob/x86_ZLMediaKit/ulib/axstarry/syscall_entry/src/test.rs#L314-L316)，主要有以下可用：

- `"busybox sh"` 是busybox 的终端
- `"./MediaServer -h"` 是我们最近在测试的一个大应用，可以先不涉及
- `"busybox sh ./test_all.sh"` 是我们期望系统通过的所有测例

此外，主分支支持宏内核的 `RISC-V` 和 `Unikernel` 模式的 `RISC-V/x86_64/aarch64` 架构。

## 对测试的大致期望

- 目前希望对架构（`RISC-V/x86_64/aarch64`）和文件系统（`fat32/ext4`）单独测试。

  - 杨老师，我们理解车载系统不需要 `RISC-V` 和 `fat32`，但因为项目是从这两个的组合起步的，因此加入这些测试能帮助我们分析问题是出在移植代码上还是功能代码上。

- 希望测试能更灵活，粒度更小
  目前的测试的主要代码在 `https://github.com/Arceos-monolithic/Starry/blob/x86_ZLMediaKit/testcases/testsuits-x86_64-linux-musl/test_all.sh` 包含 `libc-test` `busybox-test` `iozone` `lmbench` `netperf` 等等。
  还有一些零散测例用于测试一些零星的功能，[比如](https://github.com/Arceos-monolithic/Starry/blob/x86_64/ulib/axstarry/syscall_entry/src/test.rs#L315-L318)
  
  ```
  "./readlink_parent"
  "./prctl",
  "./test-vfork-exit-x86_64",
  "./test-vfork-exec-x86_64",
  ```
  
  期望能实现以下功能：
  
  - 每个测例集包含许多测例，经常是个别测例无法通过，希望脚本能具体区分个别测例。
  - 有些测例失败时会自行退出，但另一些测例可能导致系统崩溃（这通常是系统本身设计不完善的问题）。希望脚本能区分这种情况，并且在系统崩溃时自动重启 Qemu，继续运行接下来的测例。

- 完善“跳过测试”的功能。

  - 例如 `aarch64` 目前不支持宏内核，就可以跳过这一整块测试。否则按照上一条描述，脚本可能会对几百条失败测例都重启 Qemu 进行测试，浪费算力和时间


## 一些参考

- 一个同学做的简单 CI 版本。 `https://github.com/cg1937/starry_ext4/tree/main/.github`

- OS比赛的初赛测试。用 python 脚本检查逐行比对测例输出。`https://github.com/oscomp/testsuits-for-oskernel/tree/main/riscv-syscalls-testing`

- 另一个比赛内核的自动测试，使用 js 进行比对。
  - 代码仓库在 `https://github.com/scPointer/maturin/blob/master/.github/workflows/classroom.yml`。
  - 用于测试的脚本仓库在 `https://github.com/os-autograding/EvaluationScript`

- 我们本科OS课程实验的测试仓库 `https://github.com/LearningOS/rCore-Tutorial-Test-2023A`