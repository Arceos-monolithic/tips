# UserRef 模块要求2

针对 syscall 分支的 commit：[update · Arceos-monolithic/Starry@15959d2 · GitHub](https://github.com/Arceos-monolithic/Starry/commit/15959d21b7148335d34a5c4b17bde8611a63d606)以及后续 commit。

以下三点优先级从高到低

## debug

先把 check 的问题调清楚，这里其实之前已经交流过了：

- 只改sys_open，它涉及用户给的字符串跑指导书里给的那些最基本的只有几行的测例，能不能跑通？不能就就地调

- 然后再加上 sys_read 和 write，能不能跑通？

- 然后就这三个去运行比赛的所有测例，如果遇到问题了，因为范围小也能一改一个准

- 再加上几个获取时间的syscall实现以便测试 <T> 泛型参数

- 最后再把syscall全改上

你这里已经全改成 UserRef 和 UserRefSlice 了，那就复制一份代码，改名成 UserRefDebug，然后 **把 checked 加回来，只在 sys_open里把 UserRef/UserRefSlice改成它**，按上面的步骤尝试运行，然后往后做

## 取消后续传入参数

```rust
    pub fn get_mut_ptr(&self, check_type: CheckType) -> Result<*mut T, SyscallError> {
        let is_ok = match check_type {
            CheckType::Lazy => self.manual_alloc_for_lazy_is_ok(),
            CheckType::TypeLazy => self.manual_alloc_type_for_lazy_is_ok(),
            CheckType::RangeLazy(end) => self.manual_alloc_range_for_lazy_is_ok(end),
        };

        if is_ok {
            Ok(self.addr.as_usize() as *mut T)
        } else {
            Err(SyscallError::EFAULT)
        }
    }
```

这里的类型 `check_type` 和数组范围 `end`不要后续传入，最好创建时就存到 struct 里。换句话说，一个 struct UserRef  时就应该知道自己是什么类型，包不包含范围参数，不需要用户每调用一次就传一次。

其他几个函数也类似。

## 用 ? 和 Result 特性

`manual_alloc_range_for_lazy_is_ok` 调用了一个返回 Result 的函数 `process.manual_alloc_for_lazy`，把它变成一个 bool。然后调用 `manual_alloc_range_for_lazy_is_ok` 的函数又试图把这个 bool 变成 Ok() 或者 Err()。这个写法从 Rust 来看太难受了，可以看看官方文档[问号表达式的用法](https://doc.rust-lang.org/1.30.0/book/second-edition/ch09-02-recoverable-errors-with-result.html)学一学，或者搜[其他中文文档](https://blog.csdn.net/linysuccess/article/details/124002592)。


