都是个人理解，错了正常。

~~本文尝试从逆向角度来学习rust特性。~~

# 1.交叉编译

因为我用的是macOS arm，但是我不熟悉arm汇编，所以需要交叉编译到x64架构来分析rust。

```shell
rustup target list | grep win
rustup target add x86_64-apple-darwin
cargo build --target x86_64-apple-darwin
```

# 2.所有权

**那么什么类型才具有所有权？**

**“Rust 堆上对象还有一个特殊之处，它们都拥有一个所有者，因此受所有权规则的限制：当赋值时，发生的是所有权的转移（只需浅拷贝栈上的引用或智能指针即可。”**

**简而言之，当一个对象实现了Copy Trait的时候，就不会主动发生所有权的转移，而实现Copy Trait的对象一般都分配在栈上。 **

**比如也可以把使用Box::new把基本类型分配到堆上，让它能主动发生所有权的转移。**

rust实现了垃圾回收，但是没有使用gc，而是全面使用一种新的概念**所有权**（cpp里面好像有？），所有权的检查在编译期进行，所以从生成的汇编代码来看是看不到所有权的存在的。

牢记以下3个概念：

1. Rust 中每一个值都被一个变量所拥有，该变量被称为值的所有者
2. 一个值同时只能被一个变量所拥有，或者说一个值只能拥有一个所有者
3. 当所有者(变量)离开作用域范围时，这个值将被丢弃(drop)

所有权是为了实现垃圾回收以及避免野指针和数据竞争。

```rust
fn main() {
    let s1 = String::from("hello"); //s1是指针，指向分配在堆上的String。
    println!("s1.as_ptr = {:p}", s1.as_ptr());
    println!("&s1 = {:p}", &s1);
    let s2 = s1; //所有权move
    println!("s2.as_ptr = {:p}", s2.as_ptr());
    println!("&s2 = {:p}", &s2);
    println!("----------");
    let s3 = &s2;
    println!("&s3 = {:p}", &s3);
    println!("s3 = {:p}", s3);
    println!("s3.as_ptr = {:p}", s3.as_ptr());

    let s4 = &s2;
    println!("&s4 = {:p}", &s4);
    println!("s4 = {:p}", s4);
    println!("s4.as_ptr = {:p}", s4.as_ptr());
}
```

```
s1.as_ptr = 0x60000346c050
&s1 = 0x16f23aab8
s2.as_ptr = 0x60000346c050
&s2 = 0x16f23ab60
----------
&s3 = 0x16f23ac38
s3 = 0x16f23ab60
s3.as_ptr = 0x60000346c050
&s4 = 0x16f23ad10
s4 = 0x16f23ab60
s4.as_ptr = 0x60000346c050
```

![s1 moved to s2](./rust.assets/v2-3ec77951de6a17584b5eb4a3838b4b61_1440w.jpg)

根据第二条规则，**一个值同时只能被一个变量所拥有，或者说一个值只能拥有一个所有者**。即一个地址只能被一个指针指向。`let s2 = s1`时`&s1 = 0x16f23aab8 -> &s2 = 0x16f23ab60`，同时在编译期s1不能再被访问，即**所有权的转移**。

# 3.借用

```rust
fn main() {
    let s1 = String::from("hello"); //s1是指针，指向分配在堆上的String。
    println!("s1.as_ptr = {:p}", s1.as_ptr());
    println!("&s1 = {:p}", &s1);
    let s2 = s1; //所有权move
    println!("s2.as_ptr = {:p}", s2.as_ptr());
    println!("&s2 = {:p}", &s2);
    println!("----------");
    let s3 = &s2;	//借用
    println!("&s3 = {:p}", &s3);
    println!("s3 = {:p}", s3);
    println!("s3.as_ptr = {:p}", s3.as_ptr());

    let s4 = &s2;
    println!("&s4 = {:p}", &s4);
    println!("s4 = {:p}", s4);
    println!("s4.as_ptr = {:p}", s4.as_ptr());
}
```

```
s1.as_ptr = 0x60000346c050
&s1 = 0x16f23aab8
s2.as_ptr = 0x60000346c050
&s2 = 0x16f23ab60
----------
&s3 = 0x16f23ac38
s3 = 0x16f23ab60
s3.as_ptr = 0x60000346c050
&s4 = 0x16f23ad10
s4 = 0x16f23ab60
s4.as_ptr = 0x60000346c050
```

![&String s pointing at String s1](./rust.assets/v2-fc68ea4a1fe2e3fe4c5bb523a0a8247c_1440w.jpg)

根据第二条规则，**一个值同时只能被一个变量所拥有，或者说一个值只能拥有一个所有者**。即一个地址只能被一个指针指向。但是为什么`s3 = 0x16f23ab60 -> s3 = 0x16f23ab60`，却能同时指向`0x16f23ab60`呢？那是因为s指向的s1实现了Copy Trait。呵呵，如果多个指针**不能**同时指向一个对象的话，那么所有类型的操作都会是**深拷贝**。

**但是这样又发生了一个问题，多个指针指向一个对象可能会导致数据竞争，所以rust还有一套借用规则。**

总的来说，借用规则如下：

1. 同一时刻，你只能拥有要么一个可变引用, 要么任意多个不可变引用
2. 引用必须总是有效的

因为这套规则的限制，rust的引用和c的引用不同，所以称之为`借用（Borrowing）`

# 4.生命周期

还没想好怎么写，暂略。

# 5.宏展开

安装nightly版本以及cargo-expand。

```bash
rustup toolchain install nightly
cargo +nightly install cargo-expand
cd dir
rustup override set nightly
cargo expand
```

```rust
fn main() {
    let v = vec![1, 2, 3];
}
```

宏展开

```rust
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2021::*;
#[macro_use]
extern crate std;
fn main() {
    let v = <[_]>::into_vec(#[rustc_box] ::alloc::boxed::Box::new([1, 2, 3]));
}
```

