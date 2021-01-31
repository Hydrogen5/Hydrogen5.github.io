---
title: Lisve开发小记0x01
date: 2019-07-19 15:35:55
tags: 
  - "Lisve"
  - "Rust"
cover: /img/ZvyVL6.png
---
# 进度
基本上搓出了Lisve的Parser基本框架
# 效果
给出如下Lisve代码
```lisp
(def add (x y)
    (+ x y))
(def mul (x y)
    (* x y))
(def addmul (x y)
    (double
    (add x y)
    (add x y)))
(addmul 5 8)
```
成功parse出所有token  
![](/img/ZvybTO.png)

# 实现细节 
必须要承认rust的enum与match真是太好用了，核心代码非常简单  
```rust
fn run_parser(it: &mut std::str::Chars) -> Box<Vec<Value>> {
    let mut tokens = Vec::new();
    while let Some(c) = it.next() {
        match c {
            '(' => tokens.push(Value::List(run_parser(it))),
            ')' => break,
            '"' => tokens.push(Value::Str(parse_str(it))),
            '0'...'9' => tokens.push(Value::Integer(parse_int(it, c))),
            ' ' | '\n' | '\t' | '\r' => (),
            _ => tokens.push(Value::Symbol(parse_symbol(it, c))),
        }
    }
    Box::from(tokens)
}
```
# 不足
这个parser并没能处理所有情况，比如没有浮点型也不能parse负数，延迟求值符号还没有添加，注释也没有处理
另外，在实现层面，完全没有错误处理，虽然是加个Result就完事的东西，但是还是懒  
基于parser实现的lisveformater已经在路上了