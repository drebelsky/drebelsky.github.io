# random

pwnable.kr attribution: this challenge comes from <https://pwnable.kr>, and I may reproduce parts of the challenge as visible from there for easier reference.

Doing a quick look around as always,

```
random@pwnable:~$ ls -l
total 20
-r--r----- 1 random_pwn root     49 Jun 30  2014 flag
-r-sr-x--- 1 random_pwn random 8538 Jun 30  2014 random
-rw-r--r-- 1 root       root    301 Jun 30  2014 random.c
```

`random.c` contains

```c
#include <stdio.h>

int main(){
        unsigned int random;
        random = rand();        // random value!

        unsigned int key=0;
        scanf("%d", &key);

        if( (key ^ random) == 0xdeadbeef ){
                printf("Good!\n");
                system("/bin/cat flag");
                return 0;
        }

        printf("Wrong, maybe you should try 2^32 cases.\n");
        return 0;
}
```

Note, if we read the `man` page for `rand(3)`, we see

```
DESCRIPTION
    ...
       If no seed value is provided, the rand() function is automatically seeded with a value of 1.
```

So, since there's no seeding we should have a deterministic "random" value. While we could just iterate through all 4 billion possible values, it's a little more polite to the `pwnable.kr` server to just figure out what value to use.

```
random@pwnable:~$ gdb random
(gdb) disass main
Dump of assembler code for function main:
   0x00000000004005f4 <+0>:     push   %rbp
   0x00000000004005f5 <+1>:     mov    %rsp,%rbp
   0x00000000004005f8 <+4>:     sub    $0x10,%rsp
   0x00000000004005fc <+8>:     mov    $0x0,%eax
   0x0000000000400601 <+13>:    callq  0x400500 <rand@plt>
   0x0000000000400606 <+18>:    mov    %eax,-0x4(%rbp)
   0x0000000000400609 <+21>:    movl   $0x0,-0x8(%rbp)
   0x0000000000400610 <+28>:    mov    $0x400760,%eax
   0x0000000000400615 <+33>:    lea    -0x8(%rbp),%rdx
   0x0000000000400619 <+37>:    mov    %rdx,%rsi
   0x000000000040061c <+40>:    mov    %rax,%rdi
   0x000000000040061f <+43>:    mov    $0x0,%eax
   0x0000000000400624 <+48>:    callq  0x4004f0 <__isoc99_scanf@plt>
   0x0000000000400629 <+53>:    mov    -0x8(%rbp),%eax
   0x000000000040062c <+56>:    xor    -0x4(%rbp),%eax
   0x000000000040062f <+59>:    cmp    $0xdeadbeef,%eax
   0x0000000000400634 <+64>:    jne    0x400656 <main+98>
   0x0000000000400636 <+66>:    mov    $0x400763,%edi
   0x000000000040063b <+71>:    callq  0x4004c0 <puts@plt>
   0x0000000000400640 <+76>:    mov    $0x400769,%edi
   0x0000000000400645 <+81>:    mov    $0x0,%eax
   0x000000000040064a <+86>:    callq  0x4004d0 <system@plt>
   0x000000000040064f <+91>:    mov    $0x0,%eax
   0x0000000000400654 <+96>:    jmp    0x400665 <main+113>
   0x0000000000400656 <+98>:    mov    $0x400778,%edi
   0x000000000040065b <+103>:   callq  0x4004c0 <puts@plt>
   0x0000000000400660 <+108>:   mov    $0x0,%eax
   0x0000000000400665 <+113>:   leaveq 
   0x0000000000400666 <+114>:   retq   
End of assembler dump.
(gdb) b *0x400606
Breakpoint 1 at 0x400606
(gdb) r
Starting program: /home/random/random 

Breakpoint 1, 0x0000000000400606 in main ()
(gdb) p/x $eax
$1 = 0x6b8b4567
(gdb) p/d $eax ^ 0xdeadbeef
$2 = 3039230856
```

So, we can retrieve the flag

```
random@pwnable:~$ ./random
3039230856
Good!
the flag is here, but it's probably more fun if you run it yourself
```

To expand upon later:

* prngs
