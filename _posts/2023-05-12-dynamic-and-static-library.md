---
title: Dynamic Library Handling - Nitty Gritty of Relocation
date: 2023-05-12 11:33:00 +0800
categories: [Software]
tags: [software, os]
---

While most software engineers are aware of the basic concept of dynamic libraries and how they differ from static libraries. Not everyone knows the nitty gritty of how they actually work especially the implementation and implication details. I had to do some digging recently on an related problem, and hope the lesson learned can offer some help to you as well!      
\
For illustration purpose, I'll use a simple c++ program that compiled with gcc on Linux as example. The compiled target will be ELF files, I encourage you to read its spec [HERE](https://refspecs.linuxfoundation.org/elf/elf.pdf).  
\
First we write a very simple library function call that adds 1 to each element of a vector:
```c++
#include <mylib.h>
#include <iostream>

void vec_add(std::vector<int>& vec) {
    for (auto& e : vec) e = e + 1;
}
```
This file will be used for creating a library for our main executable to link. Then we write a main function that links the library:
```c++
#include <iostream>
#include <mylib.h>

int main() {
    std::vector<int> vec(10, 1);
    vec_add(vec);
    return 0;
}
```
For comparison purpose I'll create 2 flows, one for static and one for dynamic:  
* `main_dyn`,`libmylib.so` are dynamic executable and library.
* `main_static`, `libmylib_static.a` are static executable and library.

## Some Basics
* static library: code contained in them are linked directly into executable file at *compile time*. This results in a self-contained binary that includes all the necessary code within itself.  
* dynamic library: code contained in them are not linked directly into executable file, instead linker produces a record in executable for OS to recognize that it needs to load the library when the program starts executing.  

Dynamic libraries offers lots of flexibilities and reduces the *overall* executable size. They can be managed by OS such that they take one physical memory space but can be shared by many programs through being mapped at different virtual addresses, and allows easier library migration between versions.  
\
Static libraries on the other hand, are easier to manage, and offer better performance as they don't need the extra OS overhead to handle the library loading and symbol resolving at runtime.  

You can see the dynamic executable is smaller:
```bash
20288 main_dyn
21032 main_static
```

Depends on your background as a software engineer, you might have seen one more often than the other. If you write c/c++ programs on desktop OS environment, it's almost guaranteed your program used dynamic libraries. For example when you use `printf` or `std::cout` to print your data, the actual function are provided by `libc` library which comes with GNU/Linux. Any other common C/C++ standard function that you use but don't have to write your own implementation for, are similar story. But if you come from the world of Embedded System, static libraries are probably what you've dealt with the most, dynamic libraries are very rarely seen if not at all in firmware dev, especially in highly resource constrainted systems.

## Keys To Make Dynamic Library Work
Executable that links static libraries are easier to handle, as everything will be resolved at *link time*. When OS loads the program, it has all the code it needs.
Dynamic libraries are a bit tricky to handle. The fact that they are runtime loaded by OS has 2 major implications:
1. Code that contained within dynamic libraries need to be able to run at any random addresses.  
2. At executables' *link time* linker will not be able to determine their symbol addresses, thus requires a runtime resolving mechanism.  

The first issue needs to be handled during compile, by requiring compiler to generate *Position Independent Code*, which on gcc it's done through `-fPIC` flag.  
The second issue is handled by dynamic linker, at runtime it resolves the symbols with the help of GOT and PLT.

## Position Independent Code
What makes a piece of code that can be executed at any address? The most fundamental difference is that, all symbol references, including both code and data, cannot be done through hard-coded addresses. Relative addressing is instead required.  
\
This is achieved by applying `-fPIC` flag to the compiler. Some disadvantages that `-fPIC` brings are that, for one it will increase the binary size for the library itself, and for the other it'll also reduce the performance slightly, as besides the dynamic linker resolving time, changing from absolute address jumps to relative ones will takes extra instructions/computation to calculate the actual jump address.

Size Comparison:
```bash
6550 libmylib_static.a
16544 libmylib.so.1.0.0
```
So when you create dynamic libraries, make sure `-fPIC` is applied to all objects it contains, especially the ones it links through other static libraries. In which case compiler will just copy the object code it needs into target dynamic library, if those binary codes from static library source are not position independent, they may cause runtime failure.  
\
On the other hand, if you don't actually need dynamic libraries, try to avoid `-fPIC` flag, per the disadvantages.
## GOT and PLT
GOT stands for *Global Offset Table*. It is a *data* section that contains a table of offsets for external symbols.  
PLT stands for *Procedure Linkage Table*. This is a *code* section that contains the stubs for external function calls, it provides a level of indirection and helps to resolve the function address at runtime. Detail process will be discussed below.  

We can see them in the ELF's sections:
```bash
xinchen@xinchen-pc:~/code/relocation/build$ readelf -S main_dyn
There are 39 section headers, starting at offset 0xdaf8:

  [13] .plt              PROGBITS         0000000000001020  00001020
         00000000000000b0  0000000000000010  AX       0     0     16
  [25] .got              PROGBITS         0000000000003f68  00002f68
         0000000000000098  0000000000000008  WA       0     0     8
```

We can also examine the executable's relocatable symbols and see our `vec_add` function in the `.rela.plt` section. Specifically, `.rela.plt` are the symbols that goes through PLT stub indirection, and `.rela.dyn` contains all other symbols that needs relocation.
```bash
xinchen@xinchen-pc:~/code/relocation/build$ readelf -r main_dyn

Relocation section '.rela.dyn' at offset 0x870 contains 11 entries:
Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003d20  000000000008 R_X86_64_RELATIVE                    1260
000000003d28  000000000008 R_X86_64_RELATIVE                    138a
000000003d30  000000000008 R_X86_64_RELATIVE                    1220
000000004008  000000000008 R_X86_64_RELATIVE                    4008
000000003fd0  001100000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0
000000003fd8  000300000006 R_X86_64_GLOB_DAT 0000000000000000 __libc_start_main@GLIBC_2.34 + 0
000000003fe0  000c00000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTM[...] + 0
000000003fe8  000e00000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000003ff0  000f00000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCl[...] + 0
000000003ff8  001000000006 R_X86_64_GLOB_DAT 0000000000000000 _ZNSt8ios_base4In[...]@GLIBCXX_3.4 + 0
000000004010  000b00000001 R_X86_64_64       0000000000000000 __gxx_personality_v0@CXXABI_1.3 + 0

Relocation section '.rela.plt' at offset 0x978 contains 10 entries:
Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003f80  000100000007 R_X86_64_JUMP_SLO 0000000000000000 _ZSt17__throw_bad[...]@GLIBCXX_3.4 + 0
000000003f88  000200000007 R_X86_64_JUMP_SLO 0000000000000000 _ZSt20__throw_len[...]@GLIBCXX_3.4 + 0
000000003f90  000400000007 R_X86_64_JUMP_SLO 0000000000000000 _ZSt28__throw_bad[...]@GLIBCXX_3.4.29 + 0
000000003f98  000500000007 R_X86_64_JUMP_SLO 0000000000000000 __cxa_atexit@GLIBC_2.2.5 + 0
000000003fa0  000600000007 R_X86_64_JUMP_SLO 0000000000000000 _Znwm@GLIBCXX_3.4 + 0
000000003fa8  000700000007 R_X86_64_JUMP_SLO 0000000000000000 _ZdlPvm@CXXABI_1.3.9 + 0
000000003fb0  000800000007 R_X86_64_JUMP_SLO 0000000000000000 __stack_chk_fail@GLIBC_2.4 + 0
000000003fb8  000900000007 R_X86_64_JUMP_SLO 0000000000000000 _ZNSt8ios_base4In[...]@GLIBCXX_3.4 + 0
000000003fc0  000a00000007 R_X86_64_JUMP_SLO 0000000000000000 _Z7vec_addRSt6vec[...] + 0
000000003fc8  000d00000007 R_X86_64_JUMP_SLO 0000000000000000 _Unwind_Resume@GCC_3.0 + 0
```

Finally, let's take a look at the disassembly difference between the static binary and a dynamic library when a plt stub is involved:  
In static executable:
```
    12a1:       48 89 c7                mov    %rax,%rdi
    12a4:       e8 58 09 00 00          call   1c01 <_Z7vec_addRSt6vectorIiSaIiEE>
// actual code
0000000000001c01 <_Z7vec_addRSt6vectorIiSaIiEE>:
    1c01:	f3 0f 1e fa          	endbr64 
    1c05:	55                   	push   %rbp
    1c06:	48 89 e5             	mov    %rsp,%rbp
    1c09:	48 83 ec 40          	sub    $0x40,%rsp
    1c0d:	48 89 7d c8          	mov    %rdi,-0x38(%rbp)
```
In dynamic executable:
```
    12c1:	48 89 c7             	mov    %rax,%rdi
    12c4:	e8 97 fe ff ff       	call   1160 <_Z7vec_addRSt6vectorIiSaIiEE@plt>
// plt stub
0000000000001160 <_Z7vec_addRSt6vectorIiSaIiEE@plt>:
    1160:	f3 0f 1e fa          	endbr64 
    1164:	f2 ff 25 55 2e 00 00 	bnd jmp *0x2e55(%rip)        # 3fc0 <_Z7vec_addRSt6vectorIiSaIiEE@Base>
    116b:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
```
## Dynamic Loader/Linker
When an executable has the need for dynamic library linking, program linker will write a record in the ELF image to specify the path of the dynamic linker it wants to handle such process.  
This information is written in the `.interp` section:
```bash
  readelf -e main_dyn
  INTERP         0x0000000000000318 0x0000000000000318 0x0000000000000318
                   0x000000000000001c 0x000000000000001c  R      0x1
       [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```
Ok, so it says `ld-linux-x86-64.so.2` is the one it wants. But what does it actually do?  
In general, a dynamic linker provides the following functions:
* Examines the ELF's *dynamic* segment to see what other dynamic library dependencies the program needs.
* Finds and maps these libraries.
* Perform runtime symbol relocations.

We can see what the program's *dynamic* segment contains:
```bash
nchen@xinchen-pc:~/code/relocation/build$ readelf -d main_dyn 

Dynamic section at offset 0x2d38 contains 31 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libmylib.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libstdc++.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [libgcc_s.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000001d (RUNPATH)            Library runpath: [/home/xinchen/code/relocation/build]
 0x000000000000000c (INIT)               0x1000
 0x000000000000000d (FINI)               0x1c24
 0x0000000000000019 (INIT_ARRAY)         0x3d20
 0x000000000000001b (INIT_ARRAYSZ)       16 (bytes)
 0x000000000000001a (FINI_ARRAY)         0x3d30
 0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x3b0
 0x0000000000000005 (STRTAB)             0x588
 0x0000000000000006 (SYMTAB)             0x3d8
 0x000000000000000a (STRSZ)              532 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000015 (DEBUG)              0x0
 0x0000000000000003 (PLTGOT)             0x3f68
 0x0000000000000002 (PLTRELSZ)           240 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0x978
 0x0000000000000007 (RELA)               0x870
 0x0000000000000008 (RELASZ)             264 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000000000001e (FLAGS)              BIND_NOW
 0x000000006ffffffb (FLAGS_1)            Flags: NOW PIE
 0x000000006ffffffe (VERNEED)            0x7c0
 0x000000006fffffff (VERNEEDNUM)         3
 0x000000006ffffff0 (VERSYM)             0x79c
 0x000000006ffffff9 (RELACOUNT)          4
 0x0000000000000000 (NULL)               0x0
```

And we can examine the process of a dynamic linker in work in gdb using our program.
When the program start executing, before the dynamic linker is called, our frame looks like:
```
► 0x7ffff7fe32b0 <_start>               mov    rdi, rsp
  0x7ffff7fe32b3 <_start+3>             call   _dl_start                <_dl_start> 
```
And our memory mapping is following:
```
pwndbg> info proc mappings
process 383273                                 
Mapped address spaces:

        Start Addr           End Addr       Size     Offset  Perms  objfile
    0x555555554000     0x555555555000     0x1000        0x0  r--p   /home/xinchen/code/relocation/build/main_dyn
    0x555555555000     0x555555556000     0x1000     0x1000  r-xp   /home/xinchen/code/relocation/build/main_dyn
    0x555555556000     0x555555557000     0x1000     0x2000  r--p   /home/xinchen/code/relocation/build/main_dyn
    0x555555557000     0x555555559000     0x2000     0x2000  rw-p   /home/xinchen/code/relocation/build/main_dyn
    0x7ffff7fbd000     0x7ffff7fc1000     0x4000        0x0  r--p   [vvar]
    0x7ffff7fc1000     0x7ffff7fc3000     0x2000        0x0  r-xp   [vdso]
    0x7ffff7fc3000     0x7ffff7fc5000     0x2000        0x0  r--p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7fc5000     0x7ffff7fef000    0x2a000     0x2000  r-xp   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7fef000     0x7ffff7ffa000     0xb000    0x2c000  r--p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7ffb000     0x7ffff7fff000     0x4000    0x37000  rw-p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffffffde000     0x7ffffffff000    0x21000        0x0  rw-p   [stack]
0xffffffffff600000 0xffffffffff601000     0x1000        0x0  --xp   [vsyscall]
```
At this point besides our program's text and data, only the dynamic loader is mapped.  
\
We can then set a break point at our main function, which is after the the dynamic linker performs some of its tasks:
```
pwndbg> info proc mappings 
process 383273
Mapped address spaces:

        Start Addr           End Addr       Size     Offset  Perms  objfile
    0x555555554000     0x555555555000     0x1000        0x0  r--p   /home/xinchen/code/relocation/build/main_dyn
    0x555555555000     0x555555556000     0x1000     0x1000  r-xp   /home/xinchen/code/relocation/build/main_dyn
    0x555555556000     0x555555557000     0x1000     0x2000  r--p   /home/xinchen/code/relocation/build/main_dyn
    0x555555557000     0x555555558000     0x1000     0x2000  r--p   /home/xinchen/code/relocation/build/main_dyn
    0x555555558000     0x555555559000     0x1000     0x3000  rw-p   /home/xinchen/code/relocation/build/main_dyn
    0x555555559000     0x55555557a000    0x21000        0x0  rw-p   [heap]
    0x7ffff7a46000     0x7ffff7a4b000     0x5000        0x0  rw-p   
    0x7ffff7a4b000     0x7ffff7a59000     0xe000        0x0  r--p   /usr/lib/x86_64-linux-gnu/libm.so.6
    0x7ffff7a59000     0x7ffff7ad5000    0x7c000     0xe000  r-xp   /usr/lib/x86_64-linux-gnu/libm.so.6
    0x7ffff7ad5000     0x7ffff7b30000    0x5b000    0x8a000  r--p   /usr/lib/x86_64-linux-gnu/libm.so.6
    0x7ffff7b30000     0x7ffff7b31000     0x1000    0xe4000  r--p   /usr/lib/x86_64-linux-gnu/libm.so.6
    0x7ffff7b31000     0x7ffff7b32000     0x1000    0xe5000  rw-p   /usr/lib/x86_64-linux-gnu/libm.so.6
    0x7ffff7b32000     0x7ffff7b5a000    0x28000        0x0  r--p   /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7b5a000     0x7ffff7cef000   0x195000    0x28000  r-xp   /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7cef000     0x7ffff7d47000    0x58000   0x1bd000  r--p   /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7d47000     0x7ffff7d4b000     0x4000   0x214000  r--p   /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7d4b000     0x7ffff7d4d000     0x2000   0x218000  rw-p   /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7d4d000     0x7ffff7d5a000     0xd000        0x0  rw-p   
    0x7ffff7d5a000     0x7ffff7d5d000     0x3000        0x0  r--p   /usr/lib/x86_64-linux-gnu/libgcc_s.so.1
    0x7ffff7d5d000     0x7ffff7d74000    0x17000     0x3000  r-xp   /usr/lib/x86_64-linux-gnu/libgcc_s.so.1
    0x7ffff7d74000     0x7ffff7d78000     0x4000    0x1a000  r--p   /usr/lib/x86_64-linux-gnu/libgcc_s.so.1
    0x7ffff7d78000     0x7ffff7d79000     0x1000    0x1d000  r--p   /usr/lib/x86_64-linux-gnu/libgcc_s.so.1
    0x7ffff7d79000     0x7ffff7d7a000     0x1000    0x1e000  rw-p   /usr/lib/x86_64-linux-gnu/libgcc_s.so.1
    0x7ffff7d7a000     0x7ffff7e14000    0x9a000        0x0  r--p   /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.30
    0x7ffff7e14000     0x7ffff7f24000   0x110000    0x9a000  r-xp   /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.30
    0x7ffff7f24000     0x7ffff7f93000    0x6f000   0x1aa000  r--p   /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.30
    0x7ffff7f93000     0x7ffff7f9e000     0xb000   0x218000  r--p   /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.30
    0x7ffff7f9e000     0x7ffff7fa1000     0x3000   0x223000  rw-p   /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.30
    0x7ffff7fa1000     0x7ffff7fa4000     0x3000        0x0  rw-p   
    0x7ffff7fb6000     0x7ffff7fb7000     0x1000        0x0  r--p   /home/xinchen/code/relocation/build/libmylib.so.1.0.0
    0x7ffff7fb7000     0x7ffff7fb8000     0x1000     0x1000  r-xp   /home/xinchen/code/relocation/build/libmylib.so.1.0.0
    0x7ffff7fb8000     0x7ffff7fb9000     0x1000     0x2000  r--p   /home/xinchen/code/relocation/build/libmylib.so.1.0.0
    0x7ffff7fb9000     0x7ffff7fba000     0x1000     0x2000  r--p   /home/xinchen/code/relocation/build/libmylib.so.1.0.0
    0x7ffff7fba000     0x7ffff7fbb000     0x1000     0x3000  rw-p   /home/xinchen/code/relocation/build/libmylib.so.1.0.0
    0x7ffff7fbb000     0x7ffff7fbd000     0x2000        0x0  rw-p   
    0x7ffff7fbd000     0x7ffff7fc1000     0x4000        0x0  r--p   [vvar]
    0x7ffff7fc1000     0x7ffff7fc3000     0x2000        0x0  r-xp   [vdso]
    0x7ffff7fc3000     0x7ffff7fc5000     0x2000        0x0  r--p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7fc5000     0x7ffff7fef000    0x2a000     0x2000  r-xp   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7fef000     0x7ffff7ffa000     0xb000    0x2c000  r--p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7ffb000     0x7ffff7ffd000     0x2000    0x37000  r--p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7ffd000     0x7ffff7fff000     0x2000    0x39000  rw-p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffffffde000     0x7ffffffff000    0x21000        0x0  rw-p   [stack]
0xffffffffff600000 0xffffffffff601000     0x1000        0x0  --xp   [vsyscall]
```
Now we have all our dynamic libraries mapped into the program's memory view, and we are ready to execute our main function.

### Lazy Binding vs. Eager Binding
Dynamic linker helps resolves the dynamic symbols, but when does it do that? Imagine you have a complex program that has lots of dynamic library dependencies, having the dynamic linker to go through everything before our `main` function even gets a chance to run is obviously not desired.  
* Eager Binding: All symbols resolved before `main`.
* Lazy (Deferred) Binding: Symbols are resolved upon the first time they are encountered.

Eager Binding increases the pre-main handling time, but provides guarantees that related issues will be exposed before `main` gets executed. For example GDB does eager binding when it loads a program.  
Lazy Binding is the default behavior for the most though, for performance gains it brings. And once the symbols are resolved upon first encounter, the info will be saved for following use.  

## Complete Relocation Process at Runtime

The complete process are following:  
*Note: this is omitting other details and put focus on relocation*  
* OS loads the ELF and looks up its information, including its type, program headers, dynamic linker path, etc.
* OS allocates virtual memory pages for the programs' segments, loads the dynamic linker and pass the control to it.
* Dynamic linker examines the libraries dependencies, loads them into the program memory.
* Dynamic linker sets up GOT, and may optionally resolve the dynamic symbols if eager binding is specified.
* Dynamic linker returns the control back to the program.
* Execution reaches external function call (`vec_add` in our example).
* The function call is directed to PLT stub, which is passed to dynamic linker if it's the first time this function is called.
* Dynamic linker does its work (details not discussed here), and pops the GOT entry for the symbol. (Lazy binding)
```
(gdb) info address vec_add
Symbol "vec_add" is at 0x555555557fc0 in a file compiled without debugging.
(gdb) x/gx 0x555555557fc0
0x555555557fc0 <_Z7vec_addRSt6vectorIiSaIiEE@got.plt>:  0x00007ffff7fb7239
(gdb) disass 0x00007ffff7fb7239
Dump of assembler code for function _Z7vec_addRSt6vectorIiSaIiEE:
   0x00007ffff7fb7239 <+0>:     endbr64
   0x00007ffff7fb723d <+4>:     push   %rbp
   0x00007ffff7fb723e <+5>:     mov    %rsp,%rbp
```
At this point, GOT entry for our `vec_add` is resolved and filled with actual address `0x00007ffff7fb7239` which we can confirm is indeed pointing to the function we want.
* The function in the dynamic library get executed, once finished return to caller in main.
* All following calls to `vec_add` still goes through PLT stub, but since the address is already resolved, it directly jumps to the actual `vec_add` without going through the dynamic linker resolving process again.
