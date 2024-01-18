# UserRef 模块要求

这是一个帮助内核更规范地处理用户传递过来的地址、结构的模块。就目前我们的想象而言，它应该拥有以下功能



## 地址检查

`UserRef` 需要替换掉所有 syscall 接口中所有 `manual_alloc_for_lazy` `manual_alloc_range_for_lazy` 以及类似形式的函数。

`UserRef` 应该在内核**第一次**想要访问内部这个地址时，调用 `manual_alloc_for_lazy` 等函数检查，在之后内核想要访问内部这个地址时不检查。这可以通过添加一个 bool 成员变量实现，也可以其他方式实现。

> “其他方式”：
> 
> 例如可以在访问时返回一个“检查过”的数据类型。比方说 path 是个 `UserRef<u8>` ，那内核调用 `let p = path.get()?;` 就会得到一个 `UserRefInited<u8>` 类型的 `p`，这个 `p` 后续怎么玩都不会再触发检查

因为检查可能出错，所以访问 `UserRef` 内部数据的**返回值应该设为 Option 或者 Result**。`UserRef` 当然也可以提供直接获取内部数据的接口，但在这种情况下，UserRef 应该让内核 **拿走** 数据，而不是复制一份。这可以通过支持一个从 `UserRef<T>` 到 `T` 的类型转换来实现，具体内容是：

- 这个 `UserRef` 被检查过吗？如否，则做一遍检查

- 然后，销毁 UserRef ，直接返回内部数据

## 独立模块

`UserRef` 自己是一个 Crate，不要再挂在 `modules/axprocess/src/lib.rs` 里面了。

## 多种参数类型

现在的实现是 `UserRef<T>`，带个泛型参数，有一定通用性但还不够。

- 数组类型：例如 `syscall_read` 中的 `pub fn syscall_read(fd: usize, buf: UserRef<u8>, count: usize)`。`path` 本身应该是个 `[u8]` 数组，它应该和 `count` 一起组成一个类似叫 `UserRefSlice<T>` 的结构。
  
  - 也就是说，数组类型的 `UserRef` 初始化时就要将起始地址和长度传入。这样在检查时也可以方便地直接调用 `manual_alloc_range_for_lazy`。**不要在里面再支持 slice_mut_with_len** 这样的东西了。初始化时是数组就是数组，是 struct 就是 struct。

## 函数名

impl 的函数名必须和原功能相同，不要画蛇添足。比如现在的

```rust
    #[inline]
    pub fn add_count(&self, count: usize) -> *mut T {
        unsafe{self.get_mut_ptr().add(count)}
    }
```

这个函数就应该叫 `add`，或者更进一步，可以加一个重载 `[]` 运算符的实现。其他的 `ptr_is_null` 等等这些也是同理。


