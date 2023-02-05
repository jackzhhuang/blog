---
title: 如何构造自定义错误类型和向下转化
date: 2023-02-05 15:02:44
tags:
categories:
- Rust
---

Rust 所有的错误都是实现了 std::error::Error 这个trait，因此我们只要也实现这个 trait 就可以自定义自己的错误类型了。

<!--more-->

## 实现自定义错误

那么怎么实现 Error 这个 trait呢？实际上并不需要实现这个 trait 的太多方法，只要有 Debug 这个 trait 就行了：

```rust
#[derive(Debug)]
struct MyError(u32, String);

impl Error for MyError {
}
```

这里我们定义了 MyError，绑定了 u32 和 String 两个变量，用于记录错误码和错误信息。下面的 impl Error for MyError 我们不实现任何方法，Debug trait 都已经有了。

接下来我们有必要实现 std::fmt::Display 这个 trait，因为大多数标准库的错误都支持 to_string 方法，方便日后我们打印错误 log：

```rust
impl std::fmt::Display for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "error code = {}, error message = {}", self.0, self.1)
    }
}

impl MyError {
   fn make_my_error(error_code: u32, error_message: &str) -> Box<dyn Error> {
        Box::new(MyError(999, error_message.to_string()))
   } 
}
```

上面的代码中，除了实现 Display trait，我们还提供了一个静态方法，方便生成错误对象，不用每次去敲Box::new  这种代码。这样我们的自定义错误就做好了，可以写个函数测试一下：

```rust
fn some_error_happen() -> Result<(), Box<dyn Error>> {
    Err(MyError::make_my_error(999, "some error happen"))
}
```



## 理解统一错误处理

前面说了，标准库的所有专属错误都是实现了 std::error::Error ，因此，我们也应该遵守这个规则，方便则任何时候都可以统一的进行错误处理。

此外，标准库大多数时候也都是使用 Result\<T, E\> 来表示方法或者函数返回值，正常则放在 Ok(T)，异常则放在 Err(E)，这里的 E 一般就是 Box\<dyn Error\>。因为，异常需要脱离函数范围在外部处理，所以错误对象需要 Box 存放，然后各个不同的库都有自己的错误定义，统一的是都实现 std::error::Error trait，因此 Box 需要处理的是 dyn Error，即运行时 dispatch。我们以后也应该遵守这个守则，方便统一进行错误处理，即返回值应该是：Result\<X, Box\<dyn Erro\>\> 。



## 向下转化

前面说我们返回出去的时候是 Error 这个 trait，那么，我们怎么获得具体的实际类呢？这就涉及到一个向下转化的问题。有两种方法，一种是直接用原始指针进行强行转化，这个是属于 unsafe 的过程，不建议这么干：

```rust
fn main() -> Result<(), Box<dyn Error>>{
    let result = some_error_happen();
    if let Err(e) = &result {
        unsafe {
            println!("in raw, error code: {}, error message: {}", 
                    (*(e.deref() as *const dyn Error as *const MyError)).0,
                    (*(e.deref() as *const dyn Error as *const MyError)).1);
        } 
        let down_obj = e.downcast_ref::<MyError>();
        if let Some(e) = down_obj {
            println!("in main, error code: {}, error message: {}", e.0, e.1);
        }
    }

    Ok(())
}
```

上面的代码中，第 5 到 第 7 行使用了 原始指针的方式去进行向下转化。e.deref() 先获得 &T，也即&Error，然后 as 为原始指针，这里就会产生 unsafe 代码，因此这几行都要包含在 unsafe 中，再从 \*const Error 转为\*const MyError，即两次 as，第一次转为原始指针，第二次转为向上转化的指针。这些直接错做原始指针的行为都是 unsafe的，毕竟，向上转为是有可能失败的。

因此，Box 提供了安全的 downcast 方法，如果转化失败，则返回None，否则返回 Some，当然其内部也是用 unsafe 实现的，但我们不必感知，应该推荐用这种方法进行向上转化，如第 11行。
