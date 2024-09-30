# collision

pwnable.kr attribution: this challenge comes from <https://pwnable.kr>, and I may reproduce parts of the challenge as visible from there for easier reference.

First, note that we're in a relatively similar setup as [fd](./fd.html), in terms of source/flag/suid/etc...

```
col@pwnable:~$ ls -l
total 16
-r-sr-x--- 1 col_pwn col     7341 Jun 11  2014 col
-rw-r--r-- 1 root    root     555 Jun 12  2014 col.c
-r--r----- 1 col_pwn col_pwn   52 Jun 11  2014 flag
```

`col.c` contains

```c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
	int* ip = (int*)p;
	int i;
	int res=0;
	for(i=0; i<5; i++){
		res += ip[i];
	}
	return res;
}

int main(int argc, char* argv[]){
	if(argc<2){
		printf("usage : %s [passcode]\n", argv[0]);
		return 0;
	}
	if(strlen(argv[1]) != 20){
		printf("passcode length should be 20 bytes\n");
		return 0;
	}

	if(hashcode == check_password( argv[1] )){
		system("/bin/cat flag");
		return 0;
	}
	else
		printf("wrong passcode.\n");
	return 0;
}
```

Some basic things to note are that we'll put our password as the first argument, and that the password should be 20 characters long. `int`s on the platform are 32 bits, [little-endian](https://en.wikipedia.org/wiki/Endianness). The ["hash"](https://en.wikipedia.org/wiki/Hash_function) treats the password as 5 consecutive `int`s and returns their sum. (Note also that while [two's complement](https://en.wikipedia.org/wiki/Two%27s_complement) isn't required until C23 or so, `int`s here are (and overflow works as you'd expect based on that)). We want the hashcode to be equal to `0x21DD09EC`. That is, basically, our goal is to find five integers such that the low 32 bits of their sum is equal to `0x21DD09EC`. Ideally, each byte of each integer is between 30 and 126 (inclusive) since these are the easily inputable ASCII characters (technically, we can put any byte except 0 into part of the argument, it's just less convenient). We'll start with 0x20202020 (4 spaces, arbitrary, but also at the bottom of the range we set up) for each of the numbers and work from least to most significant byte.

```python3
# byte 1
   5 * 0x20 = 0xA0 # current sum
0xEC - 0xA0 = 0x4C # EC is desired sum, 4C is delta we need to make up
0x20 + 0x4C = 0x6C ('l')

# byte 2
0x09 + 0x100 - 0xA0 = 0x69 # delta needed (note we add 0x100 to avoid a
                           # negative, which works because of carrying)
        0x20 + 0x69 = 0x89 # greater than 0x7E (126 ('~')), so we need to split it
        0x89 - 0x7E = 0x0b
        0x20 + 0x0B = 0x2B ('+')

# byte 3
0xDD - 0xA1 = 0x3C # note A1, not A0 since we carry a one from the 2 byte
0x20 + 0x3C = 0x5C ('\\')

# byte 4
0x21 + 0x100 - 0xA0 = 0x81
        0x20 + 0x81 = 0xA1
        0xA1 - 0x7E = 0x23
        0x20 + 0x23 = 0x43 ('C')
```

So, every byte is ' ' (0x20) except one least-significant 'l', a '+' and '~' in the second least-significant, a '\\' in the second most-significant, and a 'C' and '~' in the most significant.

```
  1234
1:l+\C
2: ~ ~
3:    
4:    
5:    
```

So, we should be able to get the passcode from running

```
col@pwnable:~$ ./col 'l+\C ~ ~            '
```

To expand upon later:

* why is it called "collision"?
* shell expansion and quoting rules
