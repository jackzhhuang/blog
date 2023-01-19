---
title: Rust的生命周期（lifetime）
date: 2023-01-19 16:19:54
toc: true
published: false
tags:
- Rust
categories:
- Rust
---

生命周期，即lifetime是Rust最独有的一个特性，早期并没有这个特性，但后来为了辅助Rust的编译器检查生命周期是否合法，也为了调用方方便确认函数或者方法对生命周期的要求就加上去了，也许在未来，这个特性会被优化掉，谁知道呢。做为学习者，我们还是要把这些细节知识补充一下的。

<!--more-->

## 生命周期是什么

生命周期（lifetime）是annotation，即一个标记。仅此而已。不管我们怎么做，生命周期就是一个标记，它没有改变任何东西，尤其不会改变对象的生命周期，尽管我们把它叫做生命周期。所以，遇到生命周期时，记住，它只是一个标记，相当于给编译器看的注释，用于给它提示对象的生命周期的。



## 函数的生命周期

为什么我们要关注生命周期呢，看看下面这个例子，找出一句话的第一个单词：

```rust
fn find_first_word(s: &str) -> &str {
    let words: Vec<&str> = s.split(&[' ', ',', '.']).collect();
    if words.len() > 0 {
        return words.get(0).unwrap();
    }
    s
}
```

可以看到这个find_first_word函数入参是一个引用，出参也是一个引用，这么做完全没问题，因为如果出参引用的是入参，那么出参的生命周期和入参一样，如果出参是内部new出来的，那么如果出现悬空指针，那么编译器是可以知道的，所以此时不需要生命周期标记。但如果有两个参数就不一样了，比如返回最长的那个字符串引用：

```rust
fn find_longest(s1: &str, s2: &str) -> &str {
    if s1.len() >= s2.len() {
        return s1;
    }
    return s2;
}
```

这里，find_longest函数可能返回s1也可能返回s2，那么其返回值的生命周期到底是s1还是s2呢？这个是要靠运行时才能知道的，由于不知道到底返回哪个，编译器自然也无法知道返回值的生命周期，那么无法编译使用这个函数的地方的代码，因为Rust必须在编译时就精确知道各个变量的生命周期，这样才能避免悬空指针，比如下面这个例子：

```rust
    let s2 = "nootherword".to_string();
    let longest_string;
    {
        let s1 = "hi, the first word of this line is ".to_string();
        longest_string = find_longest(&s1, &s2);
    }

    println!("the longest string is {}", longest_string);
```

显然，现在函数find_longest会返回s1，那么longest_string将会引用s1，但s1在离开大括号后就没有了，于是第 8 行的print代码显然在打印一个悬空指针。

如果没有生命周期标记，Rust编译器无法知道第 5 行longest_string拿到的返回值生命周期到底有多长，也就很难（实际上是可以发现的，可能未来会优化）发现悬空指针问题。

我们加上生命周期后如下：

```rust
fn find_longest<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() >= s2.len() {
        return s1;
    }
    return s2;
}
```

这里和模板很像，首先在函数名的简括号里面生命一个生命周期，以 ' 开头，这个是生命周期语法的要求，后面的 a 则是一般约定俗成的写法，一般使用小写字母，短的单词表示某个某个生命周期，唯一比较特殊的是 'static 这个生命周期，这个放后面讲。

声明完生命周期后，后面引用的时候在&符号后面加上这个'a就表示这个引用的生命周期是'a，那么上面的代码中，s1和s2以及返回值这三个引用的生命周期都是'a，即表示，他们这三个变量中，最小的生命周期必须覆盖其它两个变量的使用范围。

前一个例子中，s1的生命周期是最小的，它只能活在大括号内，而s2和longest_string在可以存活在大括号外面，所以，s1的生命周期最小，它必须覆盖完s2和longest_string的使用范围。但很明显，出了大括号后，我们访问了longest_string，此时s1已经被销毁，不满足上述要求，因此编译器报错。修复它当然很简单，延长s1的生命周期范围即可：

```rust
    let s2 = "nootherword".to_string();
    let s1 = "hi, the first word of this line is ".to_string();
    let longest_string = find_longest(&s1, &s2);

    println!("the longest string is {}", longest_string);
```



## struct的生命周期

和函数一样，struct也有生命周期问题，比如：

```rust
struct Person {
    name: &str,
    age: u32
}
```

此时，name这个字段是个引用，那么为了避免有悬空指针出现，Person的对象的生命周期小于等于name引用的对象的生命周期，否则，如果Person的对象还存在，但name所引用的对象已经销毁，那么就会出现悬空指针了。于是为了说明这一点，我们加上生命周期标记，让编译器也让使用方知道，我们必须保证Person的对象生命周期小于等于name所引用的对象的生命周期：

```rust
struct Person<'a> {
    name: &'a str,
    age: u32
}
```

和函数类似，在Person后的尖括号内声明生命周期'a，name的引用符号&后加上'a，这样就表示Person的对象和这个name字段引用的对象它们的生命周期必须保证Person对象能在name引用对象之前销毁。

那么，如果我们给这个Person加上方法，是否也需要跟着都加上生命周期呢？

```rust
impl<'a> Person<'a> {
   fn get_age(&self) -> u32 {
       self.age 
   } 
}
```

答案是需要的，即使不管是入参还是返回值，都没有去碰name这个引用，返回值也不是引用，为什么需要加上'a声明呢？

因为Rust其实是把生命周期和泛型看成差不多一样的对待，大家也看到了，生命周期的声明和模板的声明一样，都在一个地方，他们都是在标记带有某个类型或者声明周期的struct，也就是说，既然我们声明了struct Person<'a>，那么Person\<'a\>就是一个叫做“生命周期为'a的struct，struct名叫Person”的类型，那么，给这个类型添加方法，也需要带上'a，简单说也就是Person\<'a\>是一个整体，就好像模板Person\<T\>一样。

那么返回name这个引用呢？

```rust
impl<'a> Person<'a> {
   fn get_age(&self) -> u32 {
       self.age 
   } 

   fn get_name(&self, pre: &str) -> &str {
    println!("pre = {}", pre);
    self.name
   }
}
```

可以看到，依然顺利通过编译，但get_name如果改成可能某种情况下会返回pre或者self.name：

```rust
   fn get_name(&self, pre: &str) -> &str {
        println!("pre = {}", pre);
        if pre == "hello" {
            return pre;
        }
        self.name
   }
```

就会报类似前面的错误了。这是为什么呢？这里就需要Rust的声明周期检查三个规则来解释了。



## Rust编译器检查生命周期的三个规则



## 生命周期和模板混用
