# Making a hello world binary "from scratch"

In this post, we'll progressively peel away layers of abstraction until writing an assembler to create a hello world (x86/x86-64) program (mainly as a way of briefly introducing various layers and tools for further exploration).

System setup (for semi-reproducibility)

```
$ gcc --version
gcc (GCC) 14.2.1 20240910
Copyright (C) 2024 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

$ gdb --version
GNU gdb (GDB) 15.1
Copyright (C) 2024 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
$ uname -a
Linux daniel 6.10.10-arch1-1 #1 SMP PREEMPT_DYNAMIC Thu, 12 Sep 2024 17:21:02 +0000 x86_64 GNU/Linux
$ ldd --version
ldd (GNU libc) 2.40
Copyright (C) 2024 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Written by Roland McGrath and Ulrich Drepper.
$ /lib64/ld-linux-x86-64.so.2 --version
ld.so (GNU libc) stable release version 2.40.
Copyright (C) 2024 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.
$ python --version 
Python 3.12.6
```

We'll start off pretty high level (in C).

```c
#include <stdio.h>

int main(void) {
    printf("Hello world\n");
}
```

```
$ gcc hello.c -o hello
$ ./hello
Hello world
```

Now, since `gcc` auto-links against `libc`, the only reason we need `<stdio.h>` is for the prototype of `printf`, so we can simplify the C a little as

```c
// See man 3 printf
int printf(const char *restrict format, ...);

int main(void) {
    printf("Hello world\n");
}
```

If we look at `glibc`'s source for `printf`, it ends up still using a number of abstractions[^1]. Interestingly, however, if we take a closer look at the generated code,

```
$ objdump --disassemble=main hello

hello:     file format elf64-x86-64


Disassembly of section .init:

Disassembly of section .plt:

Disassembly of section .text:

0000000000001139 <main>:
    1139:       55                      push   %rbp
    113a:       48 89 e5                mov    %rsp,%rbp
    113d:       48 8d 05 c0 0e 00 00    lea    0xec0(%rip),%rax        # 2004 <_IO_stdin_used+0x4>
    1144:       48 89 c7                mov    %rax,%rdi
    1147:       e8 e4 fe ff ff          call   1030 <puts@plt>
    114c:       b8 00 00 00 00          mov    $0x0,%eax
    1151:       5d                      pop    %rbp
    1152:       c3                      ret
```

we can see that the compiler actually ends up replacing the `printf` call with a `puts` call. Still, there is a decent amount of code run before we actually get to the eventual syscall where we write to stdout. From start to `_start`[^2], we run over 120,000 instructions (mostly related to loading the binary). From `_start` to `main`, we run over 200 more (`libc` setup code). From `main` to `puts`, there are over 600 (note, we didn't run with optimizations, but most of these instructions occur from the lazy[^3] dynamic linking of `puts`, anyway). Then, there are about 3,000 more instructions until the `syscall` (partially dynamic symbol resolution, partially the abstractions atop the raw `write` syscall (e.g., `FILE *`)).

```
# abbreviated output (removed duplication of function and shared library names)
$ gdb hello
GNU gdb (GDB) 15.1
(gdb) starti
Starting program: hello
Program stopped.
0x00007ffff7fe3dc0 in ?? () from /lib64/ld-linux-x86-64.so.2

# start to _start

(gdb) set $count = 0
(gdb) while $pc != _start
 >set $count = $count + 1
 >stepi
 >end
0x00007ffff7fe3dc3 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fd9f70 in __tunable_get_val () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fddf72 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fc96b0 in _dl_debug_state () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fe6c7f in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fc8480 in _dl_catch_exception () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7feb510 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fc8507 in _dl_catch_exception () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fc9880 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fd9f70 in __tunable_get_val () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7feb356 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fc8523 in _dl_catch_exception () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fc9d45 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fd9f70 in __tunable_get_val () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fd92fe in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7e56250 in ?? ()
0x00007ffff7fd51f1 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fd8840 in _dl_allocate_tls_init () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fd91d0 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7f015b0 in ?? ()
0x00007ffff7fd9f70 in __tunable_get_val () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7e2ef93 in ?? ()
0x00007ffff7fd9f70 in __tunable_get_val () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7e28687 in ?? ()
0x00007ffff7fe7136 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fc96b0 in _dl_debug_state () from /lib64/ld-linux-x86-64.so.2
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/libthread_db.so.1".
0x00007ffff7fe715a in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7dbdb40 in ?? () from /usr/lib/libc.so.6
0x00007ffff7dbc320 in *ABS*+0xacad0@plt () from /usr/lib/libc.so.6
0x00007ffff7f09230 in ?? () from /usr/lib/libc.so.6
0x00007ffff7fcb5b7 in ?? () from /lib64/ld-linux-x86-64.so.2
(gdb) p $count
$1 = 121469

# _start to main

(gdb) set $count = 0
(gdb) while $pc != main
 >set $count = $count + 1
 >stepi
 >end
0x0000555555555044 in _start ()
0x00007ffff7dbde40 in __libc_start_main () from /usr/lib/libc.so.6
0x00007ffff7dd71d0 in __cxa_atexit () from /usr/lib/libc.so.6
0x00007ffff7dd70e0 in ?? () from /usr/lib/libc.so.6
0x00007ffff7dbde73 in __libc_start_main () from /usr/lib/libc.so.6
0x0000555555555000 in _init ()
0x00007ffff7dbdef5 in __libc_start_main () from /usr/lib/libc.so.6
0x0000555555555130 in ?? ()
0x00007ffff7dbdf44 in __libc_start_main () from /usr/lib/libc.so.6
0x00007ffff7fdef30 in _dl_audit_preinit () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7dbdeb4 in __libc_start_main () from /usr/lib/libc.so.6
0x00007ffff7dbdd90 in ?? () from /usr/lib/libc.so.6
0x00007ffff7dd4e50 in _setjmp () from /usr/lib/libc.so.6
0x00007ffff7dd4d80 in __sigsetjmp () from /usr/lib/libc.so.6
0x00007ffff7dd4e00 in ?? () from /usr/lib/libc.so.6
0x0000555555555139 in main ()
(gdb) p $count
$2 = 249

# main to puts

(gdb) set $count = 0
(gdb) while $pc != puts
 >set $count = $count + 1
 >stepi
 >end
0x000055555555513a in main ()
0x0000555555555030 in puts@plt ()
0x0000555555555020 in ?? ()
0x00007ffff7fd9660 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7e18be0 in puts () from /usr/lib/libc.so.6
(gdb) p $count
$3 = 638

# puts to syscall

(gdb) set $count = 0
(gdb) while $pc != *write+18
 >set $count = $count + 1
 >stepi
 >end
0x00007ffff7e18be4 in puts () from /usr/lib/libc.so.6
0x00007ffff7dbc240 in *ABS*+0xac680@plt () from /usr/lib/libc.so.6
0x00007ffff7f07380 in ?? () from /usr/lib/libc.so.6
0x00007ffff7e18bfd in puts () from /usr/lib/libc.so.6
0x00007ffff7e241f0 in _IO_file_xsputn () from /usr/lib/libc.so.6
0x00007ffff7e235f0 in _IO_file_overflow () from /usr/lib/libc.so.6
0x00007ffff7e256c0 in _IO_doallocbuf () from /usr/lib/libc.so.6
0x00007ffff7e16250 in _IO_file_doallocate () from /usr/lib/libc.so.6
0x00007ffff7e240f0 in _IO_file_stat () from /usr/lib/libc.so.6
0x00007ffff7e9f910 in fstat64 () from /usr/lib/libc.so.6
0x00007ffff7e162b1 in _IO_file_doallocate () from /usr/lib/libc.so.6
0x00007ffff7e3ce50 in malloc () from /usr/lib/libc.so.6
0x00007ffff7e39640 in ?? () from /usr/lib/libc.so.6
0x00007ffff7fd9f70 in __tunable_get_val () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7e39849 in ?? () from /usr/lib/libc.so.6
0x00007ffff7e3d075 in malloc () from /usr/lib/libc.so.6
0x00007ffff7e3c6c0 in ?? () from /usr/lib/libc.so.6
0x00007ffff7e3a2d0 in __default_morecore () from /usr/lib/libc.so.6
0x00007ffff7eaea40 in sbrk () from /usr/lib/libc.so.6
0x00007ffff7ea5860 in brk () from /usr/lib/libc.so.6
0x00007ffff7eaeaf7 in sbrk () from /usr/lib/libc.so.6
0x00007ffff7ea5860 in brk () from /usr/lib/libc.so.6
0x00007ffff7eaeaac in sbrk () from /usr/lib/libc.so.6
0x00007ffff7e3a2e6 in __default_morecore () from /usr/lib/libc.so.6
0x00007ffff7e3b3ad in ?? () from /usr/lib/libc.so.6
0x00007ffff7e3a2d0 in __default_morecore () from /usr/lib/libc.so.6
0x00007ffff7eaea40 in sbrk () from /usr/lib/libc.so.6
0x00007ffff7e3a2e6 in __default_morecore () from /usr/lib/libc.so.6
0x00007ffff7e3b22b in ?? () from /usr/lib/libc.so.6
0x00007ffff7e3cf76 in malloc () from /usr/lib/libc.so.6
0x00007ffff7e3b6e0 in ?? () from /usr/lib/libc.so.6
0x00007ffff7e3d002 in malloc () from /usr/lib/libc.so.6
0x00007ffff7e162e4 in _IO_file_doallocate () from /usr/lib/libc.so.6
0x00007ffff7e25660 in _IO_setb () from /usr/lib/libc.so.6
0x00007ffff7e162fd in _IO_file_doallocate () from /usr/lib/libc.so.6
0x00007ffff7e25714 in _IO_doallocbuf () from /usr/lib/libc.so.6
0x00007ffff7e237a8 in _IO_file_overflow () from /usr/lib/libc.so.6
0x00007ffff7e23170 in _IO_do_write () from /usr/lib/libc.so.6
0x00007ffff7e242c8 in _IO_file_xsputn () from /usr/lib/libc.so.6
0x00007ffff7e257f0 in _IO_default_xsputn () from /usr/lib/libc.so.6
0x00007ffff7e24308 in _IO_file_xsputn () from /usr/lib/libc.so.6
0x00007ffff7e18c5a in puts () from /usr/lib/libc.so.6
0x000055555555514c in main ()
0x00007ffff7dbde08 in ?? () from /usr/lib/libc.so.6
0x00007ffff7dd7940 in exit () from /usr/lib/libc.so.6
0x00007ffff7dd76e0 in ?? () from /usr/lib/libc.so.6
0x00007ffff7dd74c0 in __call_tls_dtors () from /usr/lib/libc.so.6
0x00007ffff7dd790a in ?? () from /usr/lib/libc.so.6
0x00007ffff7fcb200 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7e2f9e0 in pthread_mutex_lock () from /usr/lib/libc.so.6
0x00007ffff7fcb27f in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7e31510 in pthread_mutex_unlock () from /usr/lib/libc.so.6
0x00007ffff7e313f0 in ?? () from /usr/lib/libc.so.6
0x00007ffff7fcb397 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00005555555550e0 in ?? ()
0x00007ffff7dd71e0 in __cxa_finalize () from /usr/lib/libc.so.6
0x00007ffff7e97260 in ?? () from /usr/lib/libc.so.6
0x00007ffff7dd72e4 in __cxa_finalize () from /usr/lib/libc.so.6
0x0000555555555108 in ?? ()
0x00007ffff7fc80f2 in ?? () from /lib64/ld-linux-x86-64.so.2
0x0000555555555154 in _fini ()
0x00007ffff7fcb3ee in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7dd7891 in ?? () from /usr/lib/libc.so.6
0x00007ffff7e25f70 in _IO_flush_all () from /usr/lib/libc.so.6
0x00007ffff7e28b10 in ?? () from /usr/lib/libc.so.6
0x00007ffff7e25fb3 in _IO_flush_all () from /usr/lib/libc.so.6
0x00007ffff7e235f0 in _IO_file_overflow () from /usr/lib/libc.so.6
0x00007ffff7e23170 in _IO_do_write () from /usr/lib/libc.so.6
0x00007ffff7e22280 in ?? () from /usr/lib/libc.so.6
0x00007ffff7e24150 in _IO_file_write () from /usr/lib/libc.so.6
0x00007ffff7ea4790 in write () from /usr/lib/libc.so.6
(gdb) p $count
$4 = 3356
(gdb) x/i $pc
=> 0x7ffff7ea47a2 <write+18>:   syscall
(gdb) ni
Hello world
0x00007ffff7ea47a4 in write () from /usr/lib/libc.so.6
```

Note that because most of these instructions happen in linking/loading/`libc`, using `-O3` doesn't substantially improve the instruction count (note that we can't immediately set the breakpoint on `*write+18` because `write` doesn't have a location until after `libc` is loaded (similarly, `puts`, on entry, ends up resolving to `puts@plt`)).

```
$ gcc -O3 hello.c -o hello 
$ gdb hello
GNU gdb (GDB) 15.1
(gdb) set $count = 0
(gdb) starti
Starting program: hello
Program stopped.
0x00007ffff7fe3dc0 in ?? () from /lib64/ld-linux-x86-64.so.2
(gdb) set $count = 0
(gdb) while $pc != *puts
 >stepi
 > set $count = $count + 1
 >end
0x0000555555555030 in puts@plt ()
(gdb) p $count
$1 = 121721
(gdb) while $pc != *write+18
 >set $count = $count + 1
 >stepi
 >end
0x00007ffff7ea4790 in write () from /usr/lib/libc.so.6
(gdb) p $count
$2 = 125710
(gdb) x/i $pc
=> 0x7ffff7ea47a2 <write+18>:   syscall
(gdb) ni
Hello world
0x00007ffff7ea47a4 in write () from /usr/lib/libc.so.6
```

Since most of the instructions happen from requiring `libc`, let's see what happens when we get rid of it. We want to replace the use of `puts` with a direct `write` syscall, and this will probably be easiest to do in assembly (while there are `write` and `syscall` functions, both are part of libc, so to do the syscall we'll need some assembly anyway). I'm writing on an AMD64/x86-64 machine, but you can find numbers/calling conventions in a number of places (e.g., <https://www.chromium.org/chromium-os/developer-library/reference/linux-constants/syscalls/>).

```asm
.globl _start
_start:
    mov $1, %rax /* write syscall number */
    mov $1, %rdi /* fd = 1 = stdout */
    lea text(%rip), %rsi /* const char *buf ("Hello world\n")--note that we use
    an rip-relative load so that we have position independent code */
    mov $text_end - text, %rdx /* size_t count */
    syscall
    /* Note: we should check the return value to see how much was written and
       adjust accordingly. Also, for simplicity, we've omitted the `exit`
       syscall, which causes the sgfault when it tries to interpret "Hello
       world\n" as an instruction. */
    

/* Note that this would normally appear in a different segment than the code */
text:
    .ascii "Hello world\n"
text_end:
```

Note that `gcc` can handle assembling and linking an assembly file (which is more convenient than using `as` and `ld`). Also, note for brevity, we've omitted many things (handling errors on the write, properly exiting, etc...).

```
$ gcc -o hello hello.S -nostdlib
$ ./hello                       
Hello world
[1]    167141 segmentation fault (core dumped)  ./hello
```

Not linking in and using the standard library saves us on almost every part we previously identified.

```
$ gdb hello
GNU gdb (GDB) 15.1
(gdb) starti
Starting program: hello
Program stopped.
0x00007ffff7fe3dc0 in ?? () from /lib64/ld-linux-x86-64.so.2
(gdb) set $count = 0
(gdb) while $pc != _start
 >set $count = $count + 1
 >stepi
 >end
0x00007ffff7fe3dc3 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fd9f70 in __tunable_get_val () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fddf72 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fc96b0 in _dl_debug_state () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fe6c7f in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fd9f70 in __tunable_get_val () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7feb356 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fd8840 in _dl_allocate_tls_init () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fd91d0 in ?? () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fc96b0 in _dl_debug_state () from /lib64/ld-linux-x86-64.so.2
0x00007ffff7fe715a in ?? ()
0x0000555555555000 in _start ()
(gdb) p $count
$1 = 35933
(gdb) display/i $pc
1: x/i $pc
=> 0x555555555000 <_start>:     mov    $0x1,%rax
(gdb) ni
0x0000555555555007 in _start ()
1: x/i $pc
=> 0x555555555007 <_start+7>:   mov    $0x1,%rdi
(gdb)
0x000055555555500e in _start ()
1: x/i $pc
=> 0x55555555500e <_start+14>:  lea    0x9(%rip),%rsi        # 0x55555555501e <text>
(gdb)
0x0000555555555015 in _start ()
1: x/i $pc
=> 0x555555555015 <_start+21>:  mov    $0xc,%rdx
(gdb)
0x000055555555501c in _start ()
1: x/i $pc
=> 0x55555555501c <_start+28>:  syscall
(gdb)
Hello world
0x000055555555501e in text ()
1: x/i $pc
=> 0x55555555501e <text>:       rex.W
(gdb)
Program received signal SIGSEGV, Segmentation fault.
0x000055555555501e in text ()
```

Before we go too much further, let's at least add an `exit` syscall so we don't get `SIGSGEV`

```asm
.globl _start
_start:
    mov $1, %rax
    mov $1, %rdi
    lea text(%rip), %rsi
    mov $text_end - text, %rdx
    syscall

    mov $60, %rax
    mov $0, %rdi
    syscall
    
text:
    .ascii "Hello world\n"
text_end:
```

Using some shell expansions to do a quick sanity check:

```
$ gcc -o hello{,.S} -nostdlib
$ ./hello                    
Hello world
$
```

Segfault eliminated. Now, let's break down the process a little further. Looking at the results of `strace gcc -o hello helo.S -nostdlib` (namely the `execve`s), we can see which subprograms are run. Primarily, we have `as` and `ld`[^4].

```
$ as hello.S -o hello.o
$ ld hello.o -o hello  
$ ./hello              
Hello world
```

Note that doing it manually like this has gotten rid of the dynamic linker overhead (i.e., it now only takes us 5 instructions to reach the `syscall`).

```
$ gdb hello
GNU gdb (GDB) 15.1
(gdb) starti
Starting program: hello

Program stopped.
0x0000000000401000 in _start ()
(gdb) disass $pc
Dump of assembler code for function _start:
=> 0x0000000000401000 <+0>:     mov    $0x1,%rax
   0x0000000000401007 <+7>:     mov    $0x1,%rdi
   0x000000000040100e <+14>:    lea    0x19(%rip),%rsi        # 0x40102e <text>
   0x0000000000401015 <+21>:    mov    $0xc,%rdx
   0x000000000040101c <+28>:    syscall
   0x000000000040101e <+30>:    mov    $0x3c,%rax
   0x0000000000401025 <+37>:    mov    $0x0,%rdi
   0x000000000040102c <+44>:    syscall
```

While we could change the linker script `ld` uses (see `man ld` (look for `-T`)), the language is somewhat obtuse, so we'll move to creating the ELF file directly. While doing so, we'll continue to ignore some details (e.g.,`PT_GNU_STACK`). While you're following along, you may want to bring up some ELF reference (e.g., <https://en.wikipedia.org/wiki/Executable_and_Linkable_Format> or <https://refspecs.linuxbase.org/elf/elf.pdf>).

Note that since GNU's `as` expects to output to an ELF file, getting just the assembly output is a little awkward (we use `objcopy` to treat the .text segment as its own ELF file). Note that the loadable segments must be page aligned (both in the file and in memory)---this took me some amount of debugging to realize, and in the process, I referenced [this blog post about crafting a 105 byte "Hello, world!" executable](https://nathanotterness.com/2021/10/tiny_elf_modernized.html); the end program ends up being remarkably similar, so I felt this was especially important to note.

```asm
START:
/* Elf header */
    /* e_ident */
        .byte 0x7f
        .ascii "ELF"  /* magic */
        .byte 2       /* 64 bit */
        .byte 1       /* little endian */
        .byte 1       /* ELF version (always 1) */
        .byte 0       /* Linux ABI */
        .byte 0       /* according to Wikipedia, ignored for statically linked
                         executables on Linux */
        .fill 7, 1, 0 /* padding */
    .hword 2                /* executable file */
    .hword 0x3E             /* AMD x86-64 */
    .long 1                 /* ELF version (always 1) */
    .quad 0x40000 + _start - START /* entry point */
    .quad PROG_HEAD - START /* start of program header table */
    .quad SECT_HEAD - START /* start of section header table */
    .long 0                 /* flags: this is probably fine to 0 out */
    .hword ELF_END - START  /* size of header */
    .hword 0x38             /* size of program header table entry */
    .hword 1                /* number of program header table entries */
    .hword 0x40             /* size of section header table entry */
    .hword 3                /* number of section header table entries */
    .hword 1                /* index of section table entry with section names */
ELF_END:

PROG_HEAD:
    .long 1                 /* loadable segment */
    .long 5                 /* readable and executable */
    .quad 0                 /* offset of segment in file */
    .quad 0x40000           /* virtual addr */
    .quad 0x40000           /* physical addr */
    .quad _start - START    /* file size */
    .quad _start - START    /* size in memory */
    .quad 0x1000            /* align to page boundary */

SECT_HEAD:
    /* null */
    .fill 0x40, 1, 0

    /* strtab */
    .long 1                   /* offset in shstrtab */
    .long 3                   /* string table */
    .quad 0x20                /* contains null-terminated strings */
    .quad 0
    .quad STRTAB - START      /* offset in file image */
    .quad STRTAB_END - STRTAB /* size */
    .long 0
    .long 0
    .quad 0
    .quad 0                   /* miscellaneous other things */

    /* .text */
    .long st_text - STRTAB    /* offset in shstrtab */
    .long 1                   /* string table */
    .quad 6                   /* alloc + exec */
    .quad 0x40000             /* vaddr */
    .quad _start - START      /* offset in file image */
    .quad text_end - _start   /* size */
    .long 0
    .long 0
    .quad 0x1000             /* align to page boundary */
    .quad 0

STRTAB:
    .byte 0
    .asciz ".strtab"
st_text:
    .asciz ".text"
STRTAB_END:


/* Note, we can get rid of the global directive */
_start:
    mov $1, %rax
    mov $1, %rdi
    lea text(%rip), %rsi
    mov $text_end - text, %rdx
    syscall

    mov $60, %rax
    mov $0, %rdi
    syscall
    
text:
    .ascii "Hello world\n"
text_end:
```

Note that the section headers aren't strictly necessary for correct execution, but they do make the end result play better with the standard tools.

```
$ as hello.S -o hello.o                     
$ objcopy -O binary -j .text hello.o hello
$ chmod +x hello
$ ./hello
Hello world
$ objdump -d hello

hello:     file format elf64-x86-64


Disassembly of section .text:

0000000000040000 <.text>:
   40000:       48 c7 c0 01 00 00 00    mov    $0x1,%rax
   40007:       48 c7 c7 01 00 00 00    mov    $0x1,%rdi
   4000e:       48 8d 35 19 00 00 00    lea    0x19(%rip),%rsi        # 0x4002e
   40015:       48 c7 c2 0c 00 00 00    mov    $0xc,%rdx
   4001c:       0f 05                   syscall
   4001e:       48 c7 c0 3c 00 00 00    mov    $0x3c,%rax
   40025:       48 c7 c7 00 00 00 00    mov    $0x0,%rdi
   4002c:       0f 05                   syscall
   4002e:       48                      rex.W
   4002f:       65 6c                   gs insb (%dx),%es:(%rdi)
   40031:       6c                      insb   (%dx),%es:(%rdi)
   40032:       6f                      outsl  %ds:(%rsi),(%dx)
   40033:       20 77 6f                and    %dh,0x6f(%rdi)
   40036:       72 6c                   jb     0x400a4
   40038:       64                      fs
   40039:       0a                      .byte 0xa
```

Getting rid of the section headers gives us this

```asm
START:
/* Elf header */
    /* e_ident */
        .byte 0x7f
        .ascii "ELF"  /* magic */
        .byte 2       /* 64 bit */
        .byte 1       /* little endian */
        .byte 1       /* ELF version (always 1) */
        .byte 0       /* Linux ABI */
        .byte 0       /* according to Wikipedia, ignored for statically linked
                         executables on Linux */
        .fill 7, 1, 0 /* padding */
    .hword 2                /* executable file */
    .hword 0x3E             /* AMD x86-64 */
    .long 1                 /* ELF version (always 1) */
    .quad 0x40000 + _start - START /* entry point */
    .quad PROG_HEAD - START /* start of program header table */
    .quad SECT_HEAD - START /* start of section header table */
    .long 0                 /* flags: this is probably fine to 0 out */
    .hword ELF_END - START  /* size of header */
    .hword 0x38             /* size of program header table entry */
    .hword 1                /* number of program header table entries */
    .hword 0x40             /* size of section header table entry */
    .hword 0                /* number of section header table entries */
    .hword 0                /* index of section table entry with section names */
ELF_END:

PROG_HEAD:
    .long 1                 /* loadable segment */
    .long 5                 /* readable and executable */
    .quad 0                 /* offset of segment in file */
    .quad 0x40000           /* virtual addr */
    .quad 0x40000           /* physical addr */
    .quad _start - START    /* file size */
    .quad _start - START    /* size in memory */
    .quad 0x1000            /* align to page boundary */

/* Note, we can get rid of the global directive */
_start:
    mov $1, %rax
    mov $1, %rdi
    lea text(%rip), %rsi
    mov $text_end - text, %rdx
    syscall

    mov $60, %rax
    mov $0, %rdi
    syscall
    
text:
    .ascii "Hello world\n"
text_end:
```

```
$ as hello.S -o hello.o                    
$ objcopy -O binary -j .text hello.o hello 
$ chmod +x hello
$ ./hello
Hello world
$ objdump -D hello

hello:     file format elf64-x86-64
```

Now, let's see if we can replace our use of `as` and `objcopy`. While I was initially tempted to hand assemble the file, I realized that there would be no way to distinguish this from me just writing the bytes as seen by, e.g., `xxd`. So, instead, we'll write a quick-n-dirty/ugly assembler in Python, just to demonstrate the basics of what the assembly process involves (in particular, to avoid discussing lexing/parsing we'll embed using Python types and the assembler will support the bare minimum we need; we'll also abuse Python's `globals`). To figure out how to encode an instruction, we'll be referencing [AMD64 Architecure Programmer's Manual Volume 3: Genral-Purpose and System Instructions](https://www.amd.com/content/dam/amd/en/documents/processor-tech-docs/programmer-references/24594.pdf).

```python
from os import chmod

def byte(val):
    return bytearray([val])

def fill(size):
    return bytearray(size)

def hword(val):
    return val.to_bytes(2, "little")

def long(val):
    return val.to_bytes(4, "little")

def quad(val):
    return val.to_bytes(8, "little")


# Note that we flipped to 32 bit registers becase
# 1. they have a simper encoding
# 2. every value we load easily fits in 32 bits 
# 3. MOVing or LEAing to a 32 bit register auto zeros the upper 32 bits
# page 18
EAX = 0b000
EDX = 0b010
ESI = 0b110
EDI = 0b111

def mov(val, reg):
    # Page 235: note we only support mov imm32, reg32 (MOV reg32, imm32 in the manual)
    return bytearray([0xB8 + reg]) + val.to_bytes(4, "little")

def syscall():
    # Page 475
    return bytearray([0x0F, 0x05])

def lea_rip(offset, reg):
    # Page 213 (lea)
    # Page 25 (Rip-Relative Addressing): mod = 00; r/m = 101
    # Page 17 (ModRM Byte)
    modrm = (reg << 3) | 0b101
    return bytearray([0x8D, modrm]) + offset.to_bytes(4, "little")

def ascii(s):
    return s.encode()

# Some int initial value for labels
_start = START = PROG_HEAD = SECT_HEAD = ELF_END = text = text_end = cur = 0

def program():
    return [
        "START",
        byte(0x7f),
        ascii("ELF"),
        byte(2),
        byte(1),
        byte(1),
        byte(0),
        byte(0),
        fill(7),
        hword(2),
        hword(0x3E),
        long(1),
        quad(0x40000 + _start - START),
        quad(PROG_HEAD - START),
        quad(SECT_HEAD - START),
        long(0),
        hword(ELF_END - START),
        hword(0x38),
        hword(1),
        hword(0x40),
        hword(0),
        hword(0),
        "ELF_END",
        "PROG_HEAD",
        long(1),
        long(5),
        quad(0),
        quad(0x40000),
        quad(0x40000),
        quad(_start - START),
        quad(_start - START),
        quad(0x1000),
        "_start",
        mov(1, EAX),
        mov(1, EDI),
        lea_rip(text - cur, ESI),
        "cur",
        mov(text_end - text, EDX),
        syscall(),
        mov(60, EAX),
        mov(0, EDI),
        syscall(),
        "text",
        ascii("Hello world\n"),
        "text_end",
    ]

# Set labels
offset = 0
for inst in program():
    if isinstance(inst, str):
        globals()[inst] = offset
    else:
        offset += len(inst)

# Write label adjusted program
with open("hello", "wb") as out:
    for inst in program():
        if not isinstance(inst, str):
            out.write(inst)

# Ensure executable
chmod("hello", 0o755)
```

```
$ python hello.py
$ ./hello
Hello world
```

Without digging into the kernel, that's about as far as we can go.

[^1]: See, e.g., [printf.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=stdio-common/printf.c;h=f4e14c400220e8a2c4b9a25293327045edff6af6;hb=3d1aed874918c466a4477af1da35983ab036690e) and [vprintf-internal.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=stdio-common/vfprintf-internal.c;h=771beca9bf71f4c817800fb44c45c19ec1e3a9d3;hb=3d1aed874918c466a4477af1da35983ab036690e)
[^2]: instruction counting code from <https://stackoverflow.com/a/21639842>
[^3]: `puts` doesn't get a resolved value in the PLT until the first time it is dynamically called (i.e., not when the process starts up)
[^4]: it also used `/usr/lib/gcc/x86_64-pc-linux-gnu/14.2.1/collect2`, but we won't need that
