# Rust 中常见的有关生命周期的误解

_2019 年 05 月 01 日 · 预计阅读 30 分钟 · #rust · #生命周期_


**目录**
- [引言](#引言)
- [误解](#误解)
    - [1) `T` 只包含所有权类型](#1-t-只包含所有权类型)
    - [2) 如果 `T: 'static` 那么 `T` 直到程序结束为止都一定是有效的](#2-如果-t-static-那么-t-直到程序结束为止都一定是有效的)
    - [3) `&'a T` 和 `T: 'a` 是一回事](#3-a-t-和-t-a-是一回事)
    - [4) 我的代码里不含泛型也不含生命周期注解](#4-我的代码里不含泛型也不含生命周期注解)
    - [5) 如果编译通过了，那么我标注的生命周期就是正确的](#5-如果编译通过了那么我标注的生命周期就是正确的)
    - [6) 已装箱的 trait 对象不含生命周期注解](#6-已装箱的-trait-对象不含生命周期注解)
    - [7) 编译报错的信息会告诉我怎样修复我的程序](#7-编译报错的信息会告诉我怎样修复我的程序)
    - [8) 生命周期可以在运行时动态变长或变短](#8-生命周期可以在运行时动态变长或变短)
    - [9) 将独占引用降级为共享引用是 safe 的](#9-将独占引用降级为共享引用是-safe-的)
    - [10) 对闭包的生命周期省略规则和函数一样](#10-对闭包的生命周期省略规则和函数一样)
    - [11) `'static` 引用总能被强制转换为 `'a` 引用](#11-static-引用总能被强制转换为-a-引用)
- [总结](#总结)
- [讨论](#讨论)
- [温馨提示](#温馨提示)
- [拓展阅读](#拓展阅读)
- [译者注](#译者注)



## 引言

我曾经也抱有上述的这些误解，并且现在仍有许多初学者深陷其中。本文中我使用的术语可能并不那么官方，因此下面列出了一个表格，记录我使用的短语及其想表达的含义。

| 短语 | 意义 |
|-|-|
| `T` | 1) 所有可能类型的集合 _或_<br>2) 上述集合中的某一个具体类型 |
| 所有权类型 | 某些非引用类型，其自身拥有所有权 例如 `i32`, `String`, `Vec` 等等 |
| 1) 借用类型 _或_<br>2) 引用类型 | 引用类型，不考虑可变性 例如 `&i32`, `&mut i32` 等等 |
| 1) 可变引用 _或_<br>2) 独占引用 | 独占可变引用, 即 `&mut T` |
| 1) 不可变引用 _或_<br>2) 共享引用 | 可共享不可变引用, 即 `&T` |



## 误解

简单来讲，一个变量的生命周期是指一段时期，在这段时期内，该变量所指向的内存地址中的数据是有效的，这段时期是由编译器静态分析得出的，有效性由编译器保证。接下来我将探讨这些常见误解的细节。



### 1) `T` 只包含所有权类型

这更像是对泛型的误解而非对生命周期的误解，但在 Rust 中，泛型与生命周期的关系是如此紧密，以至于不可能只讨论其中一个而忽视另外一个。

当我刚开始学习 Rust 时，我知道 `i32`, `&i32`, 和 `&mut i32` 是不同的类型，同时我也知泛型 `T` 表示所有可能类型的集合。然而，尽管能分别理解这两个概念，但我却没能将二者结合起来。在当时我这位 Rust 初学者的眼里，泛型是这样运作的：

| | | | |
|-|-|-|-|
| **类型** | `T` | `&T` | `&mut T` |
| **例子** | `i32` | `&i32` | `&mut i32` |

其中 `T` 包全体所有权类型；`&T` 包括全体不可变引用；`&mut T` 包括全体可变引用；`T`, `&T`, 和 `&mut T` 是不相交的有限集。简洁明了，符合直觉，却完全错误。事实上泛型是这样运作的：

| | | | |
|-|-|-|-|
| **类型** | `T` | `&T` | `&mut T` |
| **例子** | `i32`, `&i32`, `&mut i32`, `&&i32`, `&mut &mut i32`, ... | `&i32`, `&&i32`, `&&mut i32`, ... | `&mut i32`, `&mut &mut i32`, `&mut &i32`, ... |

`T`, `&T`, 和 `&mut T` 都是无限集，因为你可以借用一个类型无限次。`T` 是 `&T` 和 `&mut T` 的超集。`&T` 和 `&mut T` 是不相交的集合. 下面有一些例子来验证这些概念：

```rust
trait Trait {}

impl<T> Trait for T {}

impl<T> Trait for &T {} // 编译错误

impl<T> Trait for &mut T {} // 编译错误
```

上述代码不能编译通过：

```rust
error[E0119]: conflicting implementations of trait `Trait` for type `&_`:
 --> src/lib.rs:5:1
  |
3 | impl<T> Trait for T {}
  | ------------------- first implementation here
4 |
5 | impl<T> Trait for &T {}
  | ^^^^^^^^^^^^^^^^^^^^ conflicting implementation for `&_`

error[E0119]: conflicting implementations of trait `Trait` for type `&mut _`:
 --> src/lib.rs:7:1
  |
3 | impl<T> Trait for T {}
  | ------------------- first implementation here
...
7 | impl<T> Trait for &mut T {}
  | ^^^^^^^^^^^^^^^^^^^^^^^^ conflicting implementation for `&mut _`
```

编译器不允许我们为 `&T` 和 `&mut T` 实现 `Trait`，因为这与我们为 `T` 实现的 `Trait` 发生了冲突，而 `T` 已经包括了 `&T` 和 `&mut T`. 因为 `&T` 和 `&mut T` 是不相交的，所以下面的代码可以通过编译：

```rust
trait Trait {}

impl<T> Trait for &T {} // 编译通过

impl<T> Trait for &mut T {} // 编译通过
```

**关键点回顾**
- `T` 是 `&T` 和 `&mut T` 的超集
- `&T` 和 `&mut T` 是不相交的集合


### 2) 如果 `T: 'static` 那么 `T` 直到程序结束为止都一定是有效的

**错误的推论**
- `T: 'static` 应该视为 _“`T` 有着 `'static`生命周期”_
- `&'static T` 和 `T: 'static` 是一回事
- 若 `T: 'static` 则 `T` 一定是不可变的
- 若 `T: 'static` 则 `T` 只能在编译期创建

让大多数 Rust 初学者第一次接触 `'static` 生命周期注解的代码示例大概是这样的：

```rust
fn main() {
    let str_literal: &'static str = "字符串字面量";
}
```

他们被告知说 `"字符串字面量"` 是被硬编码到编译出来的二进制文件当中去的，并在运行时被加载到只读内存中，所以它不可变且在程序的整个运行期间都有效，这也使其生命周期为 `'static`. 在了解到 Rust 使用 `static` 来定义静态变量这一语法后，这一观点还会被进一步加强。

```rust
static BYTES: [u8; 3] = [1, 2, 3];
static mut MUT_BYTES: [u8; 3] = [1, 2, 3];

fn main() {
   MUT_BYTES[0] = 99; // 编译错误，修改静态变量是 unsafe 的

    unsafe {
        MUT_BYTES[0] = 99;
        assert_eq!(99, MUT_BYTES[0]);
    }
}
```

关于静态变量
- 它们只能在编译期创建
- 它们应当是不可变的，修改静态变量是 unsafe 的
- 它们在整个程序运行期间有效

静态变量的默认生命周期很有可能是 `'static` , 对吧？所以可以合理推测 `'static` 生命周期也要遵循同样的规则，对吧？

确实，但 _带有_ `'static` 生命周期注解的类型和一个被 `'static` _约束_ 的类型是不一样的。后者可以于运行时被动态分配，能被安全自由地修改，也可以被 drop, 还能存活任意的时长。

区分 `&'static T` 和 `T: 'static` 是非常重要的一点。

`&'static T` 是一个指向 `T` 的不可变引用，其中 `T` 可以被安全地无期限地持有，甚至可以直到程序结束。这只有在 `T` 自身不可变且保证 _在引用创建后_ 不会被 move 时才有可能。`T` 并不需要在编译时创建。 我们可以以内存泄漏为代价，在运行时动态创建随机数据，并返回其 `'static` 引用，比如：

```rust
use rand;

// 在运行时生成随机 &'static str
fn rand_str_generator() -> &'static str {
    let rand_string = rand::random::<u64>().to_string();
    Box::leak(rand_string.into_boxed_str())
}
```

`T: 'static` 是指 `T` 可以被安全地无期限地持有，甚至可以直到程序结束。 `T: 'static` 在包括了全部 `&'static T` 的同时，还包括了全部所有权类型， 比如 `String`, `Vec` 等等。 数据的所有者保证，只要自身还持有数据的所有权，数据就不会失效，因此所有者能够安全地无期限地持有其数据，甚至可以直到程序结束。`T: 'static` 应当视为 _“`T` 满足 `'static` 生命周期约束”_ 而非 _“`T` 有着 `'static` 生命周期”_。 一个程序可以帮助阐述这些概念：

```rust
use rand;

fn drop_static<T: 'static>(t: T) {
    std::mem::drop(t);
}

fn main() {
    let mut strings: Vec<String> = Vec::new();
    for _ in 0..10 {
        if rand::random() {
            // 所有字符串都是随机生成的
            // 并且在运行时动态分配
            let string = rand::random::<u64>().to_string();
            strings.push(string);
        }
    }

    // 这些字符串是所有权类型，所以他们满足 'static 生命周期约束
    for mut string in strings {
        // 这些字符串是可变的
        string.push_str("a mutation");
        // 这些字符串都可以被 drop
        drop_static(string); // 编译通过
    }

    // 这些字符串在程序结束之前就已经全部失效了
    println!("i am the end of the program");
}
```

**关键点回顾**
- `T: 'static` 应当视为 _“`T` 满足 `'static` 生命周期约束”_
- 若 `T: 'static` 则 `T` 可以是一个有 `'static` 生命周期的引用类型 _或_ 是一个所有权类型
- 因为 `T: 'static` 包括了所有权类型，所以 `T`
  - 可以在运行时动态分配
  - 不需要在整个程序运行期间都有效
  - 可以安全，自由地修改
  - 可以在运行时被动态的 drop
  - 可以有不同长度的生命周期



### 3) `&'a T` 和 `T: 'a` 是一回事

这个误解是前一个误解的泛化版本。

`&'a T` 要求并隐含了 `T: 'a` ，因为如果 `T` 本身不能在 `'a` 范围内保证有效，那么其引用也不能在 `'a` 范围内保证有效。例如，Rust 编译器不会运行构造一个 `&'static Ref<'a, T>`，因为如果 `Ref` 只在 `'a` 范围内有效，我们就不能给它 `'static` 生命周期。

`T: 'a` 包括了全体 `&'a T`，但反之不成立。

```rust
// 只接受带有 'a 生命周期注解的引用类型
fn t_ref<'a, T: 'a>(t: &'a T) {}

// 接受满足 'a 生命周期约束的任何类型
fn t_bound<'a, T: 'a>(t: T) {}

// 内部含有引用的所有权类型
struct Ref<'a, T: 'a>(&'a T);

fn main() {
    let string = String::from("string");

    t_bound(&string); // 编译通过
    t_bound(Ref(&string)); // 编译通过
    t_bound(&Ref(&string)); // 编译通过

    t_ref(&string); // 编译通过
    t_ref(Ref(&string)); // 编译失败，期望得到引用，实际得到 struct
    t_ref(&Ref(&string)); // 编译通过

    // 满足 'static 约束的字符串变量可以转换为 'a 约束
    t_bound(string); // 编译通过
}
```

**关键点回顾**
- `T: 'a` 比 `&'a T` 更泛化，更灵活
- `T: 'a` 接受所有权类型，内部含有引用的所有权类型，和引用
- `&'a T` 只接受引用
- 若 `T: 'static` 则 `T: 'a` 因为对于所有 `'a` 都有 `'static` >= `'a`



### 4) 我的代码里不含泛型也不含生命周期注解

**错误的推论**
- 避免使用泛型和生命周期注解是可能的

这个让人爽到的误解之所以能存在，要得益于 Rust 的生命周期省略规则，这个规则能允许你在函数定义以及 `impl` 块中省略掉显式的生命周期注解，而由借用检查器来根据以下规则对生命周期进行隐式推导。
- 第一条规则是每一个是引用的参数都有它自己的生命周期参数
- 第二条规则是如果只有一个输入生命周期参数，那么它被赋予所有输出生命周期参数
- 第三条规则是如果是有多个输入生命周期参数的方法，而其中一个参数是 `&self` 或 `&mut self`, 那么所有输出生命周期参数被赋予 `self` 的生命周期。
- 其他情况下，生命周期必须有明确的注解

这里有不少值得讲的东西，让我们来看一些例子：

```rust
// 展开前
fn print(s: &str);

// 展开后
fn print<'a>(s: &'a str);

// 展开前
fn trim(s: &str) -> &str;

// 展开后
fn trim<'a>(s: &'a str) -> &'a str;

// 非法，没有输入，不能确定返回值的生命周期
fn get_str() -> &str;

// 显式标注的方案
fn get_str<'a>() -> &'a str; // 泛型版本
fn get_str() -> &'static str; // 'static 版本

// 非法，多个输入，不能确定返回值的生命周期
fn overlap(s: &str, t: &str) -> &str;

// 显式标注（但仍有部分标注被省略）的方案
fn overlap<'a>(s: &'a str, t: &str) -> &'a str; // 返回值的生命周期不长于 s
fn overlap<'a>(s: &str, t: &'a str) -> &'a str; // 返回值的生命周期不长于 t
fn overlap<'a>(s: &'a str, t: &'a str) -> &'a str; // 返回值的生命周期不长于 s 且不长于 t
fn overlap(s: &str, t: &str) -> &'static str; // 返回值的生命周期可以长于 s 或者 t
fn overlap<'a>(s: &str, t: &str) -> &'a str; // 返回值的生命周期与输入无关

// 展开后
fn overlap<'a, 'b>(s: &'a str, t: &'b str) -> &'a str;
fn overlap<'a, 'b>(s: &'a str, t: &'b str) -> &'b str;
fn overlap<'a>(s: &'a str, t: &'a str) -> &'a str;
fn overlap<'a, 'b>(s: &'a str, t: &'b str) -> &'static str;
fn overlap<'a, 'b, 'c>(s: &'a str, t: &'b str) -> &'c str;

// 展开前
fn compare(&self, s: &str) -> &str;

// 展开后
fn compare<'a, 'b>(&'a self, &'b str) -> &'a str;
```

如果你写过
- 结构体方法
- 接收参数中有引用的函数
- 返回值是引用的函数
- 泛型函数
- trait object(后面将讨论)
- 闭包（后面将讨论）

那么对于上面这些，你的代码中都有被省略的泛型生命周期注解。

**关键点回顾**
- 几乎所有的 Rust 代码都是泛型代码，并且到处都带有被省略掉的泛型生命周期注解



### 5) 如果编译通过了，那么我标注的生命周期就是正确的

**错误的推论**
- Rust 对函数的生命周期省略规则总是对的
- Rust 的借用检查器总是正确的，无论是技巧上还是语义上
- Rust 比我更懂我程序的语义

让一个 Rust 程序通过编译但语义上不正确是有可能的。来看看这个例子：

```rust
struct ByteIter<'a> {
    remainder: &'a [u8]
}

impl<'a> ByteIter<'a> {
    fn next(&mut self) -> Option<&u8> {
        if self.remainder.is_empty() {
            None
        } else {
            let byte = &self.remainder[0];
            self.remainder = &self.remainder[1..];
            Some(byte)
        }
    }
}

fn main() {
    let mut bytes = ByteIter { remainder: b"1" };
    assert_eq!(Some(&b'1'), bytes.next());
    assert_eq!(None, bytes.next());
}
```

`ByteIter` 是一个 byte 切片上的迭代器，简洁起见，我这里省略了 Iterator trait 的具体实现。这看起来没什么问题，但如果我们想同时检查多个 byte 呢？

```rust
fn main() {
    let mut bytes = ByteIter { remainder: b"1123" };
    let byte_1 = bytes.next();
    let byte_2 = bytes.next();
    if byte_1 == byte_2 {
        // 一些代码
    }
}
```

编译错误：

```rust
error[E0499]: cannot borrow `bytes` as mutable more than once at a time
  --> src/main.rs:20:18
   |
19 |     let byte_1 = bytes.next();
   |                  ----- first mutable borrow occurs here
20 |     let byte_2 = bytes.next();
   |                  ^^^^^ second mutable borrow occurs here
21 |     if byte_1 == byte_2 {
   |        ------ first borrow later used here
```

如果你说可以通过逐 byte 拷贝来避免编译错误，那么确实。当迭代一个 byte 数组上时，我们的确可以通过拷贝每个 byte 来达成目的。但是如果我想要将  `ByteIter` 改写成一个泛型的切片迭代器，使得我们能够对任意 `&'a [T]` 进行迭代，而此时如果有一个 `T`，其 copy 和 clone 的代价十分昂贵，那么我们该怎么避免这种昂贵的操作呢？哦，我想我们不能，毕竟代码都通过编译了，那么生命周期注解肯定也是对的，对吧？

错，事实上现有的生命周期就是 bug 的源头！这个错误的生命周期被省略掉了以至于难以被发现。现在让我们展开这些被省略掉的生命周期来暴露出这个问题。

```rust
struct ByteIter<'a> {
    remainder: &'a [u8]
}

impl<'a> ByteIter<'a> {
    fn next<'b>(&'b mut self) -> Option<&'b u8> {
        if self.remainder.is_empty() {
            None
        } else {
            let byte = &self.remainder[0];
            self.remainder = &self.remainder[1..];
            Some(byte)
        }
    }
}
```

感觉好像没啥用，我还是搞不清楚问题出在哪。这里有个 Rust 专家才知道的小技巧：给你的生命周期注解起一个更有含义的名字，让我们试一下：

```rust
struct ByteIter<'remainder> {
    remainder: &'remainder [u8]
}

impl<'remainder> ByteIter<'remainder> {
    fn next<'mut_self>(&'mut_self mut self) -> Option<&'mut_self u8> {
        if self.remainder.is_empty() {
            None
        } else {
            let byte = &self.remainder[0];
            self.remainder = &self.remainder[1..];
            Some(byte)
        }
    }
}
```

每个返回的 byte 都被标注为 `'mut_self`, 但是显然这些 byte 都源于 `'remainder`! 让我们来修复一下这段代码。

```rust
struct ByteIter<'remainder> {
    remainder: &'remainder [u8]
}

impl<'remainder> ByteIter<'remainder> {
    fn next(&mut self) -> Option<&'remainder u8> {
        if self.remainder.is_empty() {
            None
        } else {
            let byte = &self.remainder[0];
            self.remainder = &self.remainder[1..];
            Some(byte)
        }
    }
}

fn main() {
    let mut bytes = ByteIter { remainder: b"1123" };
    let byte_1 = bytes.next();
    let byte_2 = bytes.next();
    std::mem::drop(bytes); // 我们现在甚至可以把这个迭代器给 drop 掉！
    if byte_1 == byte_2 { // 编译通过
        // 一些代码
    }
}
```

现在我们再回过头来看看我们上一版的实现，就能看出它是错的了，那么为什么 Rust 会编译通过呢？答案很简单：因为这是内存安全的。

Rust 借用检查器对生命周期注解的要求只到能静态验证程序的内存安全为止。即便生命周期注解有语义上的错误，Rust 也能让程序编译通过，哪怕这样做为程序带来不必要的限制。

这儿有一个和之前相反的例子：在这个例子中，Rust 生命周期省略规则标注的生命周期是语义正确的，但是我们却在无意间使用了不必要的显式注解，导致写出了一个限制极其严格的方法。

```rust
#[derive(Debug)]
struct NumRef<'a>(&'a i32);

impl<'a> NumRef<'a> {
    // 我定义的泛型结构体以 'a 为参数，这意味着我也需要给方法的参数
    // 标注为 'a 生命周期，对吗？（答案：错）
    fn some_method(&'a mut self) {}
}

fn main() {
    let mut num_ref = NumRef(&5);
    num_ref.some_method(); // 可变借用 num_ref 直至其生命周期结束
    num_ref.some_method(); // 编译错误
    println!("{:?}", num_ref); // 同样编译错误
}
```

如果我们有一个带 `'a` 泛型参数的结构体，我们几乎不可能去写一个带 `&'a mut self` 参数的方法。因为这相当于告诉 Rust “这个方法将独占借用该对象，直到对象生命周期结束”。实际上，这意味着 Rust 的借用检查器只会允许在该对象上调用至多一次 `some_method`, 此后该对象将一直被独占借用并会因此变得不再可用。这种用例极其罕见，但是因为这种代码能够通过编译，所以那些对生命周期还感到困惑的初学者们很容易写出这种 bug. 修复这种 bug 的方式是去除掉不必要的显式生命周期注解，让 Rust 生命周期省略规则来处理它：

```rust
#[derive(Debug)]
struct NumRef<'a>(&'a i32);

impl<'a> NumRef<'a> {
    // 不再给 mut self 添加 'a 注解
    fn some_method(&mut self) {}

    // 上一行去掉语法糖后：
    fn some_method_desugared<'b>(&'b mut self){}
}

fn main() {
    let mut num_ref = NumRef(&5);
    num_ref.some_method();
    num_ref.some_method(); // 编译通过
    println!("{:?}", num_ref); // 编译通过
}
```

**关键点回顾**
- Rust 对函数的生命周期省略规则并不保证在任何情况下都正确
- 在程序的语义方面，Rust 并不比你懂
- 可以试试给你的生命周期注解起一个有意义的名字
- 试着记住你在哪里添加了显式生命周期注解，以及为什么要加



### 6) 已装箱的 trait 对象不含生命周期注解

之前我们讨论了 Rust _对函数_ 的生命周期省略规则。Rust 对 trait 对象也存在生命周期省略规则，它们是：
- 如果 trait 对象被用作泛型类型的一个类型参数，那么 trait 对象的的生命周期约束会依据该类型参数的定义进行推导
    - 若该类型参数有唯一的生命周期约束，则将这个约束赋给 trait 对象
    - 若该类型参数不止一个生命周期约束，则 trait 对象的生命周期约束需要显式标注
- 如果上面不成立，也就是说该类型参数没有生命周期约束，那么
    - 若 trait 定义时有且仅有一个生命周期约束，则将这个约束赋给 trait 对象
    - 若 trait 定义时生命周期约束中存在一个 `'static`, 则将 `'static` 赋给 trait 对象
    - 若 trait 定义时没有生命周期约束，则当 trait 对象是表达式的一部分时，生命周期从表达式中推导而出，否则赋予 `'static``

以上这些听起来特别复杂，但是可以简单地总结为一句话“一个 trait 对象的生命周期约束从上下文推导而出。”看下面这些例子后，我们会看到生命周期约束的推导其实很符合直觉，因此我们没必要去记忆上面的规则：

```rust
use std::cell::Ref;

trait Trait {}

// 展开前
type T1 = Box<dyn Trait>;
// 展开后，Box<T> 没有对 T 的生命周期约束，所以推导为 'static
type T2 = Box<dyn Trait + 'static>;

// 展开前
impl dyn Trait {}
// 展开后
impl dyn Trait + 'static {}

// 展开前
type T3<'a> = &'a dyn Trait;
// 展开后，&'a T 要求 T: 'a, 所以推导为 'a
type T4<'a> = &'a (dyn Trait + 'a);

// 展开前
type T5<'a> = Ref<'a, dyn Trait>;
// 展开后，Ref<'a, T> 要求 T: 'a, 所以推导为 'a
type T6<'a> = Ref<'a, dyn Trait + 'a>;

trait GenericTrait<'a>: 'a {}

// 展开前
type T7<'a> = Box<dyn GenericTrait<'a>>;
// 展开后
type T8<'a> = Box<dyn GenericTrait<'a> + 'a>;

// 展开前
impl<'a> dyn GenericTrait<'a> {}
// 展开后
impl<'a> dyn GenericTrait<'a> + 'a {}
```

一个实现了 trait 的具体类型可以被引用，因此它们也会有生命周期约束，同样其对应的 trait 对象也有生命周期约束。你也可以直接对引用实现 trait, 引用显然是有生命周期约束的：

```rust
trait Trait {}

struct Struct {}
struct Ref<'a, T>(&'a T);

impl Trait for Struct {}
impl Trait for &Struct {} // 直接为引用类型实现 Trait
impl<'a, T> Trait for Ref<'a, T> {} // 为包含引用的类型实现 Trait
```

总之，这个知识点值得反复理解，新手在重构一个使用 trait 对象的函数到一个泛型的函数或者反过来时，常常会因为这个知识点而感到困惑。来看看这个示例程序：

```rust
use std::fmt::Display;

fn dynamic_thread_print(t: Box<dyn Display + Send>) {
    std::thread::spawn(move || {
        println!("{}", t);
    }).join();
}

fn static_thread_print<T: Display + Send>(t: T) {
    std::thread::spawn(move || {
        println!("{}", t);
    }).join();
}
```

这里编译器报错：

```rust
error[E0310]: the parameter type `T` may not live long enough
  --> src/lib.rs:10:5
   |
9  | fn static_thread_print<T: Display + Send>(t: T) {
   |                        -- help: consider adding an explicit lifetime bound...: `T: 'static +`
10 |     std::thread::spawn(move || {
   |     ^^^^^^^^^^^^^^^^^^
   |
note: ...so that the type `[closure@src/lib.rs:10:24: 12:6 t:T]` will meet its required lifetime bounds
  --> src/lib.rs:10:5
   |
10 |     std::thread::spawn(move || {
   |     ^^^^^^^^^^^^^^^^^^
```

很好，编译器告诉了我们怎样修复这个问题，让我们修复一下。

```rust
use std::fmt::Display;

fn dynamic_thread_print(t: Box<dyn Display + Send>) {
    std::thread::spawn(move || {
        println!("{}", t);
    }).join();
}

fn static_thread_print<T: Display + Send + 'static>(t: T) {
    std::thread::spawn(move || {
        println!("{}", t);
    }).join();
}
```

现在它编译通过了，但是这两个函数对比起来看起来挺奇怪的，为什么第二个函数要求 `T` 满足 `'static` 约束而第一个函数不用呢？这是个刁钻的问题。事实上，通过生命周期省略规则，Rust 自动在第一个函数里推导并添加了一个 `'static` 约束，所以其实两个函数都含有 `'static` 约束。Rust 编译器实际看到的是这个样子的：

```rust
use std::fmt::Display;

fn dynamic_thread_print(t: Box<dyn Display + Send + 'static>) {
    std::thread::spawn(move || {
        println!("{}", t);
    }).join();
}

fn static_thread_print<T: Display + Send + 'static>(t: T) {
    std::thread::spawn(move || {
        println!("{}", t);
    }).join();
}
```

关键点回顾
- 所有 trait 对象都含有自动推导的生命周期



### 7) 编译器的报错信息会告诉我怎样修复我的程序

**错误的推论**
- Rust 对 trait 对象的生命周期省略规则总是正确的
- Rust 比我更懂我程序的语义

这个误解是前两个误解的结合，来看一个例子：

```rust
use std::fmt::Display;

fn box_displayable<T: Display>(t: T) -> Box<dyn Display> {
    Box::new(t)
}
```

报错如下：

```rust
error[E0310]: the parameter type `T` may not live long enough
 --> src/lib.rs:4:5
  |
3 | fn box_displayable<T: Display>(t: T) -> Box<dyn Display> {
  |                    -- help: consider adding an explicit lifetime bound...: `T: 'static +`
4 |     Box::new(t)
  |     ^^^^^^^^^^^
  |
note: ...so that the type `T` will meet its required lifetime bounds
 --> src/lib.rs:4:5
  |
4 |     Box::new(t)
  |     ^^^^^^^^^^^
```

好，让我们按照编译器的提示进行修复。这里我们先忽略一个事实：返回值中装箱的 trait 对象有一个自动推导的 `'static` 约束，而编译器是基于这个没有显式说明的事实给出的修复建议。

```rust
use std::fmt::Display;

fn box_displayable<T: Display + 'static>(t: T) -> Box<dyn Display> {
    Box::new(t)
}
```

现在可以编译通过了，但这真的是我们想要的吗？可能是，也可能不是，编译器并没有提到其他修复方案，但下面这个也是一个合适的修复方案。

```rust
use std::fmt::Display;

fn box_displayable<'a, T: Display + 'a>(t: T) -> Box<dyn Display + 'a> {
    Box::new(t)
}
```

这个函数所能接受的实际参数比前一个函数多了不少！这个函数是不是更好？确实，但不一定必要，这取决于我们对程序的要求与约束。上面这个例子有点抽象，所以让我们看一个更简单明了的例子：

```rust
fn return_first(a: &str, b: &str) -> &str {
    a
}
```

报错：

```rust
error[E0106]: missing lifetime specifier
 --> src/lib.rs:1:38
  |
1 | fn return_first(a: &str, b: &str) -> &str {
  |                    ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `a` or `b`
help: consider introducing a named lifetime parameter
  |
1 | fn return_first<'a>(a: &'a str, b: &'a str) -> &'a str {
  |                ^^^^    ^^^^^^^     ^^^^^^^     ^^^
```

这个错误信息推荐我们给所有输入输出都标注上同样的生命周期注解。如果我们这么做了，那么程序将通过编译，但是这样写出的函数过度限制了返回类型。我们真正想要的是这个：

```rust
fn return_first<'a>(a: &'a str, b: &str) -> &'a str {
    a
}
```

**关键点回顾**
- Rust 对 trait 对象的生命周期省略规则并不保证在任何情况下都正确
- 在程序的语义方面，Rust 并不比你懂
- Rust 编译错误的提示信息所提出的修复方案并不一定能满足你对程序的需求



### 8) 生命周期可以在运行时动态变长或变短

**错误的推论**
- 容器类可以在运行时交换其内部的引用，从而改变自身的生命周期
- Rust 借用检查器能进行高级的控制流分析

这个编译不通过：

```rust
struct Has<'lifetime> {
    lifetime: &'lifetime str,
}

fn main() {
    let long = String::from("long");
    let mut has = Has { lifetime: &long };
    assert_eq!(has.lifetime, "long");

    {
        let short = String::from("short");
        // “转换到” 短的生命周期
        has.lifetime = &short;
        assert_eq!(has.lifetime, "short");

        // “转换回” 长的生命周期（实际是并不是）
        has.lifetime = &long;
        assert_eq!(has.lifetime, "long");
        // `short` 变量在这里 drop
    }

    // 编译失败， `short` 在 drop 后仍旧处于 “借用” 状态
    assert_eq!(has.lifetime, "long");
}
```

报错：

```rust
error[E0597]: `short` does not live long enough
  --> src/main.rs:11:24
   |
11 |         has.lifetime = &short;
   |                        ^^^^^^ borrowed value does not live long enough
...
15 |     }
   |     - `short` dropped here while still borrowed
16 |     assert_eq!(has.lifetime, "long");
   |     --------------------------------- borrow later used here
```

下面这个还是报错，报错信息也和上面一样：

```rust
struct Has<'lifetime> {
    lifetime: &'lifetime str,
}

fn main() {
    let long = String::from("long");
    let mut has = Has { lifetime: &long };
    assert_eq!(has.lifetime, "long");

    // 这个代码块逻辑上永远不会被执行
    if false {
        let short = String::from("short");
        // “转换到” 短的生命周期
        has.lifetime = &short;
        assert_eq!(has.lifetime, "short");

        // “转换回” 长的生命周期（实际是并不是）
        has.lifetime = &long;
        assert_eq!(has.lifetime, "long");
        // `short` 变量在这里 drop
    }

    // 还是编译失败， `short` 在 drop 后仍旧处于 “借用” 状态
    assert_eq!(has.lifetime, "long");
}
```

生命周期必须在编译时被静态确定，而且 Rust 借用检查器只会做基本的控制流分析，所以它假设每个 `if-else` 块和 `match` 块的每个分支都能被执行，然后选出一个最短的生命周期赋给块中的变量。一旦一个变量被一个生命周期约束了，那么它将 _永远_ 被这个生命周期所约束。一个变量的生命周期只能缩短，而且所有的缩短时机都在编译时确定。

**关键点回顾**
- 生命周期在编译时被静态确定
- 生命周期在运行时不能被改变
- Rust 借用检查器假设所有代码路径都能被执行，所以总是选择尽可能短的生命周期赋给变量



### 9) 将独占引用降级为共享引用是 safe 的

**错误的推论**
- 通过重借用引用内部的数据，能抹掉其原有的生命周期，然后赋一个新的上去

你可以将一个独占引用作为参数传给一个接收共享引用的函数，因为 Rust 将隐式地重借用独占引用内部的数据，生成一个共享引用：

```rust
fn takes_shared_ref(n: &i32) {}

fn main() {
    let mut a = 10;
    takes_shared_ref(&mut a); // 编译通过
    takes_shared_ref(&*(&mut a)); // 上面那行去掉语法糖
}
```

这在直觉上是合理的，因为将一个独占引用转换为共享引用显然是无害的，对吗？令人讶异的是，这并不对，下面的这段程序不能通过编译：

```rust
fn main() {
    let mut a = 10;
    let b: &i32 = &*(&mut a); // 重借用为不可变引用
    let c: &i32 = &a;
    dbg!(b, c); // 编译失败
}
```

报错如下：

```rust
error[E0502]: cannot borrow `a` as immutable because it is also borrowed as mutable
 --> src/main.rs:4:19
  |
3 |     let b: &i32 = &*(&mut a);
  |                     -------- mutable borrow occurs here
4 |     let c: &i32 = &a;
  |                   ^^ immutable borrow occurs here
5 |     dbg!(b, c);
  |          - mutable borrow later used here
```

代码里确实有一个独占引用，但是它立即重借用变成了一个共享引用，然后自身就被 drop 掉了。但是为什么 Rust 好像把这个重借用出来的共享引用看作是有一个独占的生命周期呢？上面这个例子中，允许独占引用直接降级为共享引用是没有问题的，但是这个允许确实会导致潜在的内存安全问题。

```rust
use std::sync::Mutex;

struct Struct {
    mutex: Mutex<String>
}

impl Struct {
    // 将 self 的独占引用降级为 str 的共享引用
    fn get_string(&mut self) -> &str {
        self.mutex.get_mut().unwrap()
    }
    fn mutate_string(&self) {
        // 如果 Rust 允许独占引用降级为共享引用，那么下面这一行代码执行后，
        // 所有通过 get_string 方法返回的 &str 都将变为非法引用
        *self.mutex.lock().unwrap() = "surprise!".to_owned();
    }
}

fn main() {
    let mut s = Struct {
        mutex: Mutex::new("string".to_owned())
    };
    let str_ref = s.get_string(); // 独占引用降级为共享引用
    s.mutate_string(); // str_ref 失效，变成非法引用，现在是一个悬垂指针
    dbg!(str_ref); // 当然，实际上会编译错误
}
```

这里的关键点在于，你在重借用一个独占引用为共享引用时，就已经落入了一个陷阱：为了保证重借用得到的共享引用在其生命周期内有效，被重借用的独占引用也必须保证在这段时期有效，这延长了独占引用的生命周期！哪怕独占引用自身已经被 drop 掉了，但独占引用的生命周期却一直延续到共享引用的生命周期结束。

使用重借用得到的共享引用是很难受的，因为它明明是一个共享引用但是却不能和其他共享引用共存。重借用得到的共享引用有着独占引用和共享引用的缺点，却没有二者的优点。我认为重借用一个独占引用为共享引用的行为应当被视为 Rust 的一种反模式。知道这种反模式是很重要的，当你看到这样的代码时，你就能轻易地发现错误了：

```rust
// 将独占引用降级为共享引用
fn some_function<T>(some_arg: &mut T) -> &T;

struct Struct;

impl Struct {
    // 将独占的 self 引用降级为共享的 self 引用
    fn some_method(&mut self) -> &self;

    // 将独占的 self 引用降级为共享的 T 引用
    fn other_method(&mut self) -> &T;
}
```

尽管你可以在函数和方法的声明里避免重借用，但是由于 Rust 会自动做隐式重借用，所以很容易无意识地遇到这种情况。

```rust
use std::collections::HashMap;

type PlayerID = i32;

#[derive(Debug, Default)]
struct Player {
    score: i32,
}

fn start_game(player_a: PlayerID, player_b: PlayerID, server: &mut HashMap<PlayerID, Player>) {
    // 从 server 中得到 player, 如果不存在就创建一个默认的 player 并得到这个新创建的。
    let player_a: &Player = server.entry(player_a).or_default();
    let player_b: &Player = server.entry(player_b).or_default();

    // 对得到的 player 做一些操作
    dbg!(player_a, player_b); // 编译错误
}
```

上面这段代码会编译失败。这里 `or_default()` 会返回一个 `&mut Player`，但是由于我们添加了一个显式的类型标注，它会被隐式重借用成 `&Player`。而为了达成我们真正的目的，我们不得不这样做：

```rust
use std::collections::HashMap;

type PlayerID = i32;

#[derive(Debug, Default)]
struct Player {
    score: i32,
}

fn start_game(player_a: PlayerID, player_b: PlayerID, server: &mut HashMap<PlayerID, Player>) {
    // 因为编译器不允许这两个返回值共存，所有这里直接丢弃这两个 &mut Player
    server.entry(player_a).or_default();
    server.entry(player_b).or_default();

    // 再次获取 player, 这次我们直接拿到共享引用，避免隐式的重借用
    let player_a = server.get(&player_a);
    let player_b = server.get(&player_b);

    // 对得到的 player 做一些操作
    dbg!(player_a, player_b); // 现在能编译通过了
}
```

难用，而且很蠢，但这是我们为了内存安全这一信条所做出的牺牲。

**关键点回顾**
- 尽量避免重借用一个独占引用为共享引用，不然你会遇到很多麻烦
- 重借用一个独占引用并不会结束其生命周期，哪怕它自身已经被 drop 掉了



### 10) 对闭包的生命周期省略规则和函数一样

这更像是 Rust 的陷阱而非误解

尽管闭包可以被当作是一个函数，但是并不遵循和函数同样的生命周期省略规则。

```rust
fn function(x: &i32) -> &i32 {
    x
}

fn main() {
    let closure = |x: &i32| x;
}
```

报错：

```rust
error: lifetime may not live long enough
 --> src/main.rs:6:29
  |
6 |     let closure = |x: &i32| x;
  |                       -   - ^ returning this value requires that `'1` must outlive `'2`
  |                       |   |
  |                       |   return type of closure is &'2 i32
  |                       let's call the lifetime of this reference `'1`
```

去掉语法糖后，我们得到的是：

```rust
// 输入的生命周期应用到了输出上
fn function<'a>(x: &'a i32) -> &'a i32 {
    x
}

fn main() {
    // 输入和输出有它们自己各自的生命周期
    let closure = for<'a, 'b> |x: &'a i32| -> &'b i32 { x };
    // 注意：上一行并不是合法的语句，但是我们需要它来描述我们目的
}
```

出现这种差异并没有什么好处。只是在闭包最初的实现中，使用的类型推断语义与函数不同，而现在将二者做一个统一将是一个 breaking change, 因此现在已经没法改了。那么我们怎么显式地标注一个闭包的类型呢？我们有以下几种方案：

```rust
fn main() {
    // 转换成 trait 对象，但这样是不定长的，所以会编译错误
    let identity: dyn Fn(&i32) -> &i32 = |x: &i32| x;

    // 可以分配到堆上作为替代方案，但是在这里堆分配感觉有点蠢
    let identity: Box<dyn Fn(&i32) -> &i32> = Box::new(|x: &i32| x);

    // 可以不用堆分配而直接创建一个 'static 引用
    let identity: &dyn Fn(&i32) -> &i32 = &|x: &i32| x;

    // 上一行去掉语法糖 :)
    let identity: &'static (dyn for<'a> Fn(&'a i32) -> &'a i32 + 'static) = &|x: &i32| -> &i32 { x };

    // 这看起来很完美，但可惜不符合语法
    let identity: impl Fn(&i32) -> &i32 = |x: &i32| x;

    // 这个也行，但也不符合语法
    let identity = for<'a> |x: &'a i32| -> &'a i32 { x };

    // 但是 "impl trait" 可以作为函数的返回值类型
    fn return_identity() -> impl Fn(&i32) -> &i32 {
        |x| x
    }
    let identity = return_identity();

    // 上一个解决方案的泛化版本
    fn annotate<T, F>(f: F) -> F where F: Fn(&T) -> &T {
        f
    }
    let identity = annotate(|x: &i32| x);
}
```

我想你应该注意到了，在上面的例子中，如果对闭包应用 trait 约束，闭包会和函数遵循同样的生命周期省略规则。

这里没有什么现实的教训或见解，只是说明一下闭包是这样的。

**关键点回顾**
- 每个语言都有其陷阱 🤷


### 11) `'static` 引用总能被强制转换为 `'a` 引用

我之前有过这样的代码：

```rust
fn get_str<'a>() -> &'a str; // 泛型版本
fn get_str() -> &'static str; // 'static 版本
```

一些读者联系我，问这两者之间是否有实际的差异。我一开始并不确定，但一番研究过后遗憾地发现，是的，这二者确实有差异。

通常在使用值时，我们能用 `'static` 引用直接代替一个 `'a` 引用，因为 Rust 会自动把 `'static` 引用强制转换为 `'a` 引用。直觉上这很合理，因为在一个对生命周期要求比较短的地方用一个生命周期比较长的引用绝不会导致任何内存安全问题。下面的这段代码通过编译，和预期一致：

```rust
use rand;

fn generic_str_fn<'a>() -> &'a str {
    "str"
}

fn static_str_fn() -> &'static str {
    "str"
}

fn a_or_b<T>(a: T, b: T) -> T {
    if rand::random() {
        a
    } else {
        b
    }
}

fn main() {
    let some_string = "string".to_owned();
    let some_str = &some_string[..];
    let str_ref = a_or_b(some_str, generic_str_fn()); // 编译通过
    let str_ref = a_or_b(some_str, static_str_fn()); // 编译通过
}
```

然而当引用作为函数类型签名的一部分时，强制类型转换并不生效。所以下面这段代码不能通过编译：

```rust
use rand;

fn generic_str_fn<'a>() -> &'a str {
    "str"
}

fn static_str_fn() -> &'static str {
    "str"
}

fn a_or_b_fn<T, F>(a: T, b_fn: F) -> T
    where F: Fn() -> T
{
    if rand::random() {
        a
    } else {
        b_fn()
    }
}

fn main() {
    let some_string = "string".to_owned();
    let some_str = &some_string[..];
    let str_ref = a_or_b_fn(some_str, generic_str_fn); // 编译通过
    let str_ref = a_or_b_fn(some_str, static_str_fn); // 编译错误
}
```

报错如下：

```rust
error[E0597]: `some_string` does not live long enough
  --> src/main.rs:23:21
   |
23 |     let some_str = &some_string[..];
   |                     ^^^^^^^^^^^ borrowed value does not live long enough
...
25 |     let str_ref = a_or_b_fn(some_str, static_str_fn);
   |                   ---------------------------------- argument requires that `some_string` is borrowed for `'static`
26 | }
   | - `some_string` dropped here while still borrowed
```

很难说这是不是 Rust 的一个陷阱，把 `for<T> Fn() -> &'static T` 强制转换成 `for<'a, T> Fn() -> &'a T` 并不是一个像把 `&'static str` 强制转换为 `&'a str` 这样简单直白的情况。前者是类型之间的转换，后者是值之间的转换。

**关键点回顾**
- `for <'a，T> fn（）->＆'a T` 签名的函数比 `for <T> fn（）->＆'static T` 签名的函数要更灵活，并且泛用于更多场景



## 总结

- `T` 是 `&T` 和 `&mut T` 的超集
- `&T` 和 `&mut T` 是不相交的集合
- `T: 'static` 应当视为 _“`T` 满足 `'static` 生命周期约束”_
- 若 `T: 'static` 则 `T` 可以是一个有 `'static` 生命周期的引用类型 _或_ 是一个所有权类型
- 因为 `T: 'static` 包括了所有权类型，所以 `T`
    - 可以在运行时动态分配
    - 不需要在整个程序运行期间都有效
    - 可以安全，自由地修改
    - 可以在运行时被动态的 drop
    - 可以有不同长度的生命周期
- `T: 'a` 比 `&'a T` 更泛化，更灵活
- `T: 'a` 接受所有权类型，内部含有引用的所有权类型，和引用
- `&'a T` 只接受引用
- 若 `T: 'static` 则 `T: 'a` 因为对于所有 `'a` 都有 `'static` >= `'a`
- 几乎所有的 Rust 代码都是泛型代码，并且到处都带有被省略掉的泛型生命周期注解e
- Rust 生命周期省略规则并不保证在任何情况下都正确
- 在程序的语义方面，Rust 并不比你懂
- 可以试试给你的生命周期注解起一个有意义的名字
- 试着记住你在哪里添加了显式生命周期注解，以及为什么要
- 所有 trait 对象都含有自动推导的生命周期
- Rust 编译错误的提示信息所提出的修复方案并不一定能满足你对程序的需求
- 生命周期在编译时被静态确定
- 生命周期在运行时不能被改变
- Rust 借用检查器假设所有代码路径都能被执行，所以总是选择尽可能短的生命周期赋给变量
- 尽量避免重借用一个独占引用为共享引用，不然你会遇到很多麻烦
- 重借用一个独占引用并不会结束其生命周期，哪怕它自身已经被 drop 掉了
- 每个语言都有其陷阱 🤷
- `for <'a，T> fn（）->＆'a T` 签名的函数比 `for <T> fn（）->＆'static T` 签名的函数要更灵活，并且泛用于更多场


## 拓展阅读

- [Sizedness in Rust](./sizedness-in-rust.md)
- [Learning Rust in 2019](./learning-rust-in-2019.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](./too-many-brainfuck-compilers.md)

