---
title: 我们的brainfuck比天空更宽广
date: 2020-03-04 18:35:24
tags: "C++"
cover: /img/Zo7uKP.png
---
准备面试到大脑抽抽突然就放弃治疗了，大概应该没什么好怕的吧……于是就想写点玩具，最后打算拿*现代化*的*C plus plus*写一个*现代化*的带*JIT*的BrainFuck。  
## α
先大概说说思路，C++17添加了std::varivant，于是终于可以很愉快的写一些sum type，理想当然是好的，然而……
```cpp
std::visit([](auto&& arg) {
    using T = std::decay_t<decltype(arg)>;
    if constexpr (std::is_same_v<T, int>)
        std::cout << "int with value " << arg << '\n';
    else if constexpr (std::is_same_v<T, long>)
        std::cout << "long with value " << arg << '\n';
    else if constexpr (std::is_same_v<T, double>)
        std::cout << "double with value " << arg << '\n';
    else if constexpr (std::is_same_v<T, std::string>)
        std::cout << "std::string with value " << std::quoted(arg) << '\n';
    else 
        static_assert(always_false<T>::value, "non-exhaustive visitor!");
}, w);
// 以上代码来自https://zh.cppreference.com/w/cpp/utility/variant/visit
```
我的天哪，你们没有模式匹配的语言都得这么写代码吗？  

不过有总比没有好。由于没有构造子，想要把类型unwrap出来还得做些脏活。  

```cpp
// instrument.h
#ifndef _INSTRUMENT_H_
#define _INSTRUMENT_H_
struct plus_t { int x; };
struct minus_t { int x; };
struct incre_t { int x; };
struct decre_t { int x; };
loop_start_t { int end; };
struct loop_end_t { int start; };
struct input_t {};
struct output_t {};
using instrument_t = std::variant<plus_t, minus_t, incre_t, decre_t, loop_start_t, loop_end_t, input_t, output_t>;
#endif
```
接下来就需要对BF代码做点操作了，为了感受一下效率的变化，我们先不做任何优化
```cpp
// compile.cpp
std::vector<instrument_t> compile(const std::string &src)
{
    std::vector<instrument_t> ins;
    std::stack<int> call_stack;
    ins.reserve(src.size() + 0x30);
    int t;
    for (int i = 0; i < src.length(); i++)
    {
        switch (src[i])
        {
        case '+':
            ins.push_back(plus_t{1});
            break;
        ......
        ......
        case '[':
            ins.push_back(loop_start_t{19260817});
            call_stack.push(i);
            break;
        case ']':
            if (call_stack.empty())
                throw "unexpected right bracket";
            t = call_stack.top();
            call_stack.pop();
            ins.push_back(loop_end_t{t});
            ins[t] = loop_start_t{i};
            break;
        ......
        ......
        default:
            ins.push_back(plus_t{0});
            break;
        }
    }
    if (!call_stack.empty())
        throw "unclosed left bracket";
    return ins;
}
```
其实还是做了点有聊胜无的优化，比如`reserve`操作。  

开始和结束的解析说两句，维护一个调用栈，碰到`[`就把偏移入栈，同时记录一个假的终点偏移，我这里是随便选的一个无意义数字`19260817`，遇到`]`时出栈起点偏移，顺便把起点偏移里的假偏移改掉。  

同时神秘字符当成add0解析，因为反正会在日后被优化掉。  

最后作为*α*版本，满足敏捷开发的要求，先写个VM让它跑起来

```cpp
// bf_machine.cpp
class bf_machine
{
    using cube = unsigned short;
    using instrument_t = std::variant<plus_t, minus_t, incre_t, decre_t, loop_start_t, loop_end_t, input_t, output_t>;

public:
    bf_machine() : it(0),
                   cubes(0x3000, 0)
    {
    }
    ~bf_machine() {}

    void run(const std::vector<instrument_t> &instruments)
    {
        int ip(0);
        while (ip < instruments.size())
        {
            switch (instruments[ip].index())
            {
            case 0:
                plus(std::get<0>(instruments[ip]).x);
                break;
            ......
            ......
            }
            ++ip;
        }
    }

private:
    ......
    ......
    std::vector<cube> cubes;
    unsigned int it;
};
```
这个`std::variant`真是有够难用。  

那现在来跑个速度吧，看看这个未经雕琢的BrainFuck跑`mandelbrot`要多久。  

![](/img/2020_3_4_A.png)  

553.33s，这也太菜了。   

不过O2优化之后，达到了42.17s，甚至不到原来十分之一，快多了。  

到这里，*α*完美实现，非常的软件工程。  

~~所以我为什么想不开放着Rust不写来写C++我快吐了~~  

___
## β
上回说到，这个*现代化*的BrainFuck已经可以跑了，但是效果非常不优秀，所以得做点微小的优化。  

显然耗时最严重的地方在于连续的重复指令，那直接压缩起来就可以了。  

```cpp
template <typename T, size_t Ty>
instrument_t fold(std::vector<instrument_t>::const_iterator &it)
{
    int now = std::get<Ty>(*it).x;
    ++it;
    while (it->index() == Ty)
    {
        now += std::get<Ty>(*it).x;
        ++it;
    }
    --it;
    return T{now};
}
std::vector<instrument_t> optimize(const std::vector<instrument_t> &code)
{
    std::vector<instrument_t> ins;
    std::stack<int> call_stack;
    int t;
    int program(0);
    ins.reserve(code.size());
    auto it = code.begin();
    while (it != code.end())
    {
        switch (it->index())
        {
        case 0:
            ins.push_back(fold<plus_t, 0>(it));
            break;
        case 1:
            ins.push_back(fold<minus_t, 1>(it));
            break;
        case 2:
            ins.push_back(fold<incre_t, 2>(it));
            break;
        case 3:
            ins.push_back(fold<decre_t, 3>(it));
            break;
        case 4:
            ins.push_back(loop_start_t{19260817});
            call_stack.push(program);
            break;
        case 5:
            t = call_stack.top();
            call_stack.pop();
            ins.push_back(loop_end_t{t});
            ins[t] = loop_start_t{program};
            break;
        default:
            ins.push_back(*it);
        }
        ++program;
        ++it;
    }
    return ins;
}
```
在c++里想做点编译器运算还真难，这个template写的非常不如意，感觉根本没有少写多少代码。  

这个fold所做的无非就是把连续的+-><拼起来，至于><><><><><><>这种反复横跳的基本上处理不了，不过正常大概也不会有这样的代码吧。其实要解决也可以，但是这个template还是过于难用，想依靠类型计算还得多些不少东西，真的做出来估计就是stl里的一坨traits那样。等我再学点template看看能不能把这一坨屎改掉。

那么这点优化有多少效果呢？

| 无优化  | fold优化 | fold优化O2 |
| ------- | -------- | ---------- |
| 553.33s | 193.50s  | 14.48s     |

这个速度总算看的过眼了，~~快速迭代大成功~~。

___

## γ

~~BF道，堂堂连载！~~

先稍微说说我对JIT的看法吧，这个东西看着挺高端的，~~不知为何让人联想到方舟编译器~~。将虚拟机的字节码转成机械码跑在真正的CPU上，这个说法确实唬人，不过稍微了解之后，这不就是写Shellcode吗？

必要的知识早在一年之前就已备齐，剩下的写就行了。

我本想这么干，但是这确实太累了，手算jmp偏移什么的，最重要的还是不可移植。

调库的话LLVM不太熟，文档只翻过几页，而且我也不想装环境。于是选了DynASM。

STL里没有相关的库，所以大概要写一些很不C++的代码了。

我日你妈我手写shellcode了我不管什么可移植性了

```
/*
    rcx : aPtr
    r12 : getchar
    r13 : putchar
    mov rcx, rdi ==== save aPtr -> H\x89\xf9
    add byte ptr [rcx], n ==== plus -> \x80\x01n
    sub byte ptr [rcx], n ==== minus -> \x80)n
    add rcx, n ==== incre -> H\x83\xc1n
    sub rcx, n ==== decre -> H\x83\xe9n
    cmp byte ptr [rcx], 0 ; jnz x; \x809\x00 ==== jmp to end  -> \x0f\x85____ 
    cmp byte ptr [rcx], 0 ; jnz x; \x809\x00 ==== jmp to start -> \x0f\x84____ 
    mov r15, rcx ; mov rax, r12 ; call rax ; mov rcx, r15 ; mov byte ptr [rcx], al ==== input
    mov r15, rcx ; mov dl, byte ptr [rcx] ; mov rax, r13 ; call rax ; mov rcx, r15 ==== output
    */
```

写了这么一大坨汇编，累死我了，然后就是慢慢造~~payload~~，~~ShellCode~~，JIT

~~好烦啊忘写返回调用约定了干脆让它直接崩溃结束进程好了反正也跑完了而且跑完shellcode还要返回？我从来没见过这样的需求~~

好吧，抛弃了所谓可移植性之后，这个程序的鲁棒性变得非常惊人，Windows下面跑不了，wsl下面居然也跑不了，只能在虚拟机里跑一跑，同时，惊人的事情再一次发生了，由于虚拟机的环境实在太老了，里面的g++编译不了这个项目……

所以这个东西最后是在Windows下编写，wsl里编译，虚拟机里运行的，~~达成了一段佳话~~。

```cpp
void (*bf_jit(std::vector<instrument_t> instruments))(unsigned char *, int(void), int(int))
{
    std::stack<int> call_stack;
    int t, t2;
    int program = 0;
    int ip(0);
    unsigned char *code = new unsigned char[0x300000];
    memcpy(code, "H\x89\xf9I\x89\xf4I\x89\xd5", 9);
    program += 9;
    while (ip < instruments.size())
    {
        switch (instruments[ip].index())
        {
        case 0:
            t = std::get<0>(instruments[ip]).x;
            code[program++] = '\x80';
            code[program++] = '\x01';
            code[program++] = static_cast<unsigned char>(t);
            break;
        case 1:
            t = std::get<1>(instruments[ip]).x;
            code[program++] = '\x80';
            code[program++] = ')';
            code[program++] = static_cast<unsigned char>(t);
            break;
        case 2:
            t = std::get<2>(instruments[ip]).x;
            code[program++] = 'H';
            code[program++] = '\x83';
            code[program++] = '\xc1';
            code[program++] = static_cast<unsigned char>(t);
            break;
        case 3:
            t = std::get<3>(instruments[ip]).x;
            code[program++] = 'H';
            code[program++] = '\x83';
            code[program++] = '\xe9';
            code[program++] = static_cast<unsigned char>(t);
            break;
        case 4:
            memcpy(&code[program], "\x80\x39\x00\x0f\x84\xde\xad\xbe\xef", 9);
            program += 9;
            call_stack.push(program);
            break;
        case 5:
            t = call_stack.top();
            memcpy(&code[program], "\x80\x39\x00\x0f\x85", 5);
            program += 5;
            t -= (program + 4);
            memcpy(&code[program], reinterpret_cast<unsigned char *>(&t), 4);
            program += 4;
            t = call_stack.top();
            t2 = program - t;
            memcpy(&code[t - 4], reinterpret_cast<unsigned char *>(&t2), 4);
            call_stack.pop();
            break;
        case 6:
            memcpy(&code[program], "I\x89\xcfL\x89\xe0\xff\xd0L\x89\xf9\x88\x01", 13);
            program += 13;
            break;
        case 7:
            memcpy(&code[program], "I\x89\xcfH1\xd2\x8a\x11H\x89\xd7L\x89\xe8\xff\xd0L\x89\xf9", 19);
            program += 19;
            break;
        }
        ++ip;
    }
    code[program++] = '\xc3';
    auto buf = mmap((void *)0x200000, program + 0x500, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    memcpy(buf, code, program);
    mprotect(buf, program + 0x500, PROT_READ | PROT_EXEC);
    return reinterpret_cast<void (*)(unsigned char *, int (*)(), int (*)(int))>(buf);
}
```

这就是最后的成果，手拼机械码，对自己的毅力产生了一丝敬意。

那么就是最后的效果了

![](/img/image-20200305173715749.png)

1.53s，这还是虚拟机的屑CPU。O2优化什么的也不想测了，意义不大。

Okay，这次摸出来的东西还算满意，虽说还有优化空间，不过我实在太懒了。

项目全部代码均已上传GitHub，虽然并不会有人来看……