---
title: Rust的测试框架
date: 2023-01-21 09:16:18
tags:
- Rust
- Test
categories:
- Rust
---

测试（testing）是软件开发关键一环，重中之重，任何好的代码都离不开一个严格的测试，Rust也不例外。测试可以找出bug，可以证明某种情况下的正确性。本节介绍怎么在Rust测试框架下写测试代码。

<!--more-->



## 生成测试框架

在生成 library 库的时候，Rust一定会生成一个简单的测试模板，例如，我们建立一个名叫jacktest的库：

```rust
cargo new jacktest --lib
```

此时就会在src目录下生成lib.rs文件，里面就是一个简单的测试模板，简单的对 2+2 进行测试：

```rust
pub fn add(left: usize, right: usize) -> usize {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
}
```

 这里先注意到的是之前我们用过的一个语法 #[]，即表明增加某个属性（attribute），比如之前用的 #[derive(Debug)] 就是指这是一个派生（derive）属性，具体属性为Debug，即这个struct可以用于Debug，比如格式化打印观察里面的数据。

这里 #[test] 则表示it_works是一个 testing 函数，会被命令cargo test调起。执行cargo test后，会打印出it_works被执行的测试信息：

```rust
running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests jacktest

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

```

这样，我们就可以自己添加自己想要测试的内容了。这里不去讨论太多assert宏的使用，这个在官方文档花了大量篇幅介绍，这里不去一一介绍那些assert宏。关于assert宏的要说的有两点：

### assert宏的应用

第一，我们应该尽量选择合适的assert去体现测试原意，比如计算两个值是否相等，使用：

```rust
assert_eq!(a, b);
```

就比使用：

```rust
assert!(a == b);
```

要好。因为assert_eq出来的信息比assert出来的信息更能反映测试目的。

第二，assert后面可以打印一些format信息帮助我们在看测试结果的时候能看到更多有用的信息：

```rust
assert_eq!(a, b, "\nthe a.area = {}, b.area = {}", a.area(), b.area());
```



### panic

测试框架遇到panic会直接报错，但和平常的panic处理方法不同，测试框架里面的panic不会停止测试流程，只是单个测试会中断而已，例如我们在上一个测试代码中增加一个测试用例：

```rust
    #[test]
    fn it_will_panic() {
        panic!("something goes wrong!");
    }
```

测试框架还是会执行其它的测试用例：

```rust
running 2 tests
in partial_cmp of RectangcleArea, jack's area = 1000 and rose's area = 240
thread 'tests::it_will_panic' panicked at 'something goes wrong!', src/lib.rs:41:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
test tests::equal ... ok
test tests::it_will_panic ... FAILED
```

这是因为测试框架是用多线程来拉起测试用例的，某一个用例panic只会影响某一个线程，不会使整个测试框架进程都退出。

panic也可以format我们输出：

```rust
panic!("the value is {}", value);
```



### should_panic

有些时候panic就是我们所期望的，特别是我们需要测试一些失败用例的情况下，比如设计一个类，关联一个 1 到 100 的数字，如果输入不在这个范围内则panic：

```rust
struct Guess {
    value: i32,
}

impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 {
            panic!(
                "Guess value must be greater than or equal to 1, got {}.",
                value
            );
        } else if value > 100 {
            panic!(
                "Guess value must be less than or equal to 100, got {}.",
                value
            );
        }

        Guess { value }
    }
}
```

现在，如果输入200到new方法中，则理应在13行panic，这是我们期待的，但如果直接执行测试，那结果是不通过的，这时就需要should_panic宏来告诉测试框架，这个是我们期待的panic，可以认为通过。那么怎么告诉呢，其原理是检查panic宏打印的log是否包含should_panic中指定的字符串，如果包含，则认为如预期panic，测试用例通过：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected = "less than or equal to 100")]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```

上面的代码中，#[should_panic(expected = "less than or equal to 100")] 就是指定期待panic的地方，其中expect所等于的字符串，就是子串，只要逻辑代码中panic信息包含这个子串，那么认为测试通过而不是报失败。

```rust
running 1 test
test tests::greater_than_100 - should panic ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests jacktest

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

如果我们交换Guess函数中 if - else 语句的panic，此时panic信息自然就不是期待的那样了：

```rust
        if value < 1 {
            panic!(
                "Guess value must be less than or equal to 100, got {}.",
                value
            );
        } else if value > 100 {
            panic!(
                "Guess value must be greater than or equal to 1, got {}.",
                value
            );
        }
```

此时执行测试，走入if value > 100分支中，但提示信息和should_panic不一样，那么会被判定为不期待的panic，测试失败：

```rust
note: panic did not contain expected string
      panic message: `"Guess value must be greater than or equal to 1, got 200."`,
 expected substring: `"less than or equal to 100"`

failures:
    tests::greater_than_100

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```



## 控制测试框架

### 测试框架的参数

测试框架参数分成两种，一种用于过滤测试代码用例（比如执行或者不执行某些测试用例），一种是用来控制测试框架行为的（比如用例的执行方式，怎么打印信息等）。

两种参数用 -- 隔离。例如：

```rust
cargo test 测试用例过滤参数 -- 控制测试框架行为参数
```



### 控制测试用例行为

#### 控制测试线程数量

之前也提到过，测试框架使用的是多线程拉起各个测试用例的，也就是测试用例是并行运行的，一般情况下不会有什么问题，但如果比如两个用例同时写一个文件那么久会出现不可预知的后果，这时就需要我们告诉测试框架，使用单线程去执行用例，这个是控制测试框架行为的，所以放在 -- 的右边：

```rust
cargo test -- --test_threads=1
```

这样就不会有并发运行测试了。



#### 在测试流程中打印print信息

假设我们现在有一个矩形面积类：

```rust
#[derive(Debug)]
struct RectangcleArea {
    height: u32,
    width: u32,
    name: String,
}

impl RectangcleArea {
    fn area(&self) -> u32 {
        self.height * self.width
    }   
}
```

我们需要测试哪个矩形的面积哪个面积大，那么我们必须先实现PartialEq（需要实现eq方法）和PartialOrd（需要实现partial_cmp方法）这两个trait：

```rust
impl PartialEq for RectangcleArea {
    fn eq(&self, other: &RectangcleArea) -> bool {
        println!("in eq of RectangcleArea, {}'s area = {} and {}'s area = {}", 
                 self.name, self.area(), other.name, other.area());

        self.area() == other.area()
    }    
}

impl PartialOrd for RectangcleArea {
    fn partial_cmp(&self, other: &RectangcleArea) -> Option<std::cmp::Ordering> {
        println!("in partial_cmp of RectangcleArea, {}'s area = {} and {}'s area = {}", 
                 self.name, self.area(), other.name, other.area());

        if self.area() > other.area() {
            return Some(std::cmp::Ordering::Greater);
        }
        return Some(std::cmp::Ordering::Less);
    }
}
```

 但我们在进行测试的时候，发现println!并没有打印出来：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn large() {
        let a = RectangcleArea {
            height: 100,
            width: 10,
            name: "jack".to_string(),
        };
        let b = RectangcleArea {
            height: 120,
            width: 2,
            name: "rose".to_string(),
        };
        
        assert!(a > b, "\nthe a.area = {}, b.area = {}", a.area(), b.area());
    }
}
```

 这是因为cargo test不会打印出标准输入输出，为了打印print信息，控制测试框架行为，需要这样：

```rust
cargo test -- --nocapture
```

或者这样：

```rust
cargo test -- --show-output
```

这样，我们在代码中print出来的数据就能看到了。



### 过滤测试用例

#### 运行指定用例

在默认情况下，如果只是运行cargo test那么所有被标记为 #[test] 属性的用例都会被执行，但偶尔我们只想运行其中某个用例，此时就可以直接用名字来指定某个用例，例如增加一个这么个用例：

```rust
    #[test]
    fn eq() {
        let a = RectangcleArea {
            height: 100,
            width: 10,
            name: "jack".to_string(),
        };
        let b = RectangcleArea {
            height: 10,
            width: 100,
            name: "rose".to_string(),
        };
        
        assert_eq!(a, b, "\nthe a.area = {}, b.area = {}", a.area(), b.area());
    }
```

如果我们不想执行large测试用例，只想执行eq测试用例，那么：

```rust
cargo test eq -t- --show-output
```

因为是过滤，所以eq写在 -- 的左边。表示执行以eq为开头的用例。这样large用例就不会被拉起。

如果想运行多个，那么用前缀匹配即可，例如增加一个eq_fail：

```rust
   #[test]
    #[should_panic(expected = "not equal!")]
    fn eq_fail() {
        let a = RectangcleArea {
            height: 100,
            width: 10,
            name: "jack".to_string(),
        };
        let b = RectangcleArea {
            height: 100,
            width: 20,
            name: "rose".to_string(),
        };
        
        if a.area() != b.area() {
            panic!("the area of a{} and b{} are not equal!", a.area(), b.area());
        }
    }
```

同样运行之前的命令，此时eq开头的测试用例都会被拉起：

```rust
running 2 tests
test tests::eq ... ok
test tests::eq_fail - should panic ... ok

successes:

---- tests::eq stdout ----
in eq of RectangcleArea, jack's area = 1000 and rose's area = 1000

---- tests::eq_fail stdout ----
thread 'tests::eq_fail' panicked at 'the area of a1000 and b2000 are not equal!', src/lib.rs:54:13
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


successes:
    tests::eq
    tests::eq_fail

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 0.00s
```



#### 忽略某些用例

有些用例很特殊，比如可能某个版本会出错，此时我们需要暂时忽略这些用例，那么可以用ignore属性忽略掉，比如：

```rust
    #[test]
    #[ignore = "no testing anymore"]
    fn large() {
      ...
```

此时即使使用cargo test也不会拉起large用例了。



#### 运行所有用例（包括已忽略的）

如果想运行所有用例即使是被 igore 的呢？那么一种方法当然是删掉 ignore 属性，一种是使用控制测试框架行为的参数，把被 ignored 的用例 include 进来：

```rust
cargo test -- --show-output --include-ignored
```

当然，如果只想运行被 ignored 的用例，那么就指定使用 ignored 用例即可：

```rust
cargo test -- --show-output --ignored
```



## 测试用例结构

### 测试代码与二进制文件

当某个mod被声明test的时候，编译二进制文件讲不会包含测试代码，测试代码和生产用的代码是被分别编译成不同的产物的，这样即节省了编译时间，也减少了测试代码对生产代码的影响：

```rust
cfg(test)]
mod tests {
    use super::*;
  ...
```



### 测试私有方法

可以看到我们一直以来都是在测试非pub的函数，那么方法一样，我们测试的所有函数和方法都不是mod test里面的，也没有声明为pub，但实际我们一直在测试，所以Rust是允许 test 越过访问权限来测试的。

```rust
use super::*;
```

Test 框架中use supper也说明它引入了所有其它mod，无需担心是否有pub属性。



## 集成测试

前面说的都是单元测试，所谓单元测试，即对某一个模块的某一个函数或者方法进行测试，而集成测试，则是对某一个模块所导出的接口进行测试，这个模块可能依赖多个其它模块。和单元测试不同，单元测试是和生产代码（放在src目录下）放在一起的，只是都集中放在mod tests里面，并标记 cfg(test)] ，集成测试则是放在和生产代码目录src同级目录下，名叫tests，这些名字都是固定的。cargo会根据目录名而知道，tests下放的每一个文件都是一个testing crate（Rust的基础模块单元）。集成测试不需要再把测试代码放进mod tests里面，只要保证在tests目录下即可。

例如我们把之前的RectangleArea的测试代码移动到tests目录下，最后目录结构如下：

```rust
rectangle
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── lib.rs
│   └── rectangle_area.rs
└── tests
    └── test_rectangle_area.rs
```

其中lib.rs导出rectangle_area：

```rust
pub mod rectangle_area;
```

 而内容如下：

```rust
use rectangle::rectangle_area::RectangcleArea;

#[test]
fn test_area_eq_large() {
    let a = RectangcleArea {
        height: 100,
        width: 30,
        name: String::from("jack"),
    };
    let b = RectangcleArea {
        height: 40,
        width: 50,
        name: String::from("jack"),
    };

    assert!(a > b, "a.area() = {} is not larger than b.area() = {}", a.area(), b.area());
}
```

 可以看到，测试框架因为被移出src目录，需要使用use把rectangle目录引入，而且，由于不再是单元测试，如果是私有的字段或者方法，将需要声明为pub，否则tests是无法使用的。

运行cargo build将会编译库文件，运行cargo test将会执行tests目录下的测试模块，例如：

```rust
cargo test --test test_rectangle_area -- --nocapture
```

即运行 test_rectangle_area.rs的测试用例，且打印print信息。如果不指定test_rectangle_area，那么tests中所有的测试用例都会被拉起执行，而且如果其中某一个失败，会中断所有的测试流程。

这里似乎不能像python test那样指定到某个函数进行测试，而且对于执行文件（main.rs）是不能使用测试框架的，因为Rust官方认为，执行文件的逻辑应该是很少的，大部分逻辑应该都在库文件中，执行文件只需要简单的运行起来就能测试了。



