# flag

pwnable.kr attribution: this challenge comes from <https://pwnable.kr>, and I may reproduce parts of the challenge as visible from there for easier reference.

The intro here is

```
Papa brought me a packed present! let's open it.

Download : http://pwnable.kr/bin/flag

This is reversing task. all you need is binary
```

Downloading and running `flag` (which you should probably do under a vm/with appropriate security precautions), we get

```
$ wget 'http://pwnable.kr/bin/flag'
$ chmod +x flag
$ ./flag
I will malloc() and strcpy the flag there. take it.
```

Now, doing standard things, we don't get any useful output:

```
$ gdb flag
Reading symbols from flag...
(No debugging symbols found in flag)
(gdb) start
No symbol table loaded.  Use the "file" command.
(gdb) starti
Starting program: /home/daniel/Desktop/drebelsky.github.io/docs/notes/pwnable.kr/flag 

Program stopped.
0x000000000044a4f0 in ?? ()
(gdb) ni

Program received signal SIGTRAP, Trace/breakpoint trap.
0x000000000084a4f6 in ?? ()
(gdb) 

Program received signal SIGSEGV, Segmentation fault.
0x0000000000000672 in ?? ()
(gdb) 

Program terminated with signal SIGSEGV, Segmentation fault.
The program no longer exists.
```

```
$ objdump -d flag

flag:     file format elf64-x86-64

$ objdump -D flag

flag:     file format elf64-x86-64

$ readelf -S flag

There are no sections in this file.
```

However, looking at `strings flag`, we see some notable strings

```
$Info: This file is packed with the UPX executable packer http://upx.sf.net $
$Id: UPX 3.08 Copyright (C) 1996-2011 the UPX Team. All Rights Reserved. $
```

Then,

```
$ upx -d flag
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2024
UPX 4.2.4       Markus Oberhumer, Laszlo Molnar & John Reiser    May 9th 2024

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
    887219 <-    335288   37.79%   linux/amd64   flag

Unpacked 1 file.
$ gdb flag
GNU gdb (GDB) 15.1
Copyright (C) 2024 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-pc-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from flag...

(No debugging symbols found in flag)
(gdb) b strcpy
Breakpoint 1 at gnu-indirect-function resolver at 0x40c050
(gdb) r
Starting program: flag 
I will malloc() and strcpy the flag there. take it.

Breakpoint 1, 0x0000000000416b50 in __strcpy_sse2_unaligned ()
(gdb) x/s $rsi
0x496628:       "the flag is here, but it's probably more fun if you run it yourself"
```

To expand upon later:

* (which you should probably do under a vm/with appropriate security precautions)
* x86-64 System V calling convention
* `gdb` usage
* what `upx` is 
