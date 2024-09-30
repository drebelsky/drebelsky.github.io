# fd

pwnable.kr attribution: this challenge comes from <https://pwnable.kr>, and I may reproduce parts of the challenge as visible from there for easier reference.


This is the first level, and starts off relatively easily. It does also introduce us to the basic format for the website. First, if we click on the `fd` picture on <https://pwnable.kr/play.php>, we're greeted with basic instructions and hints (note the writeup link only works if you've correctly submitted the flag :)).

```
Mommy! what is a file descriptor in Linux?

* try to play the wargame your self but if you are ABSOLUTE beginner, follow this tutorial link:
https://youtu.be/971eZhMHQQw

ssh fd@pwnable.kr -p2222 (pw:guest)
```

Here, the first line tells us that we're probably do something related to file descriptors, and the last line tells us how to connect. When we connect, we can take a quick look around.

```
fd@pwnable:~$ ls -l
total 16
-r-sr-x--- 1 fd_pwn fd   7322 Jun 11  2014 fd
-rw-r--r-- 1 root   root  418 Jun 11  2014 fd.c
-r--r----- 1 fd_pwn root   50 Jun 11  2014 flag
```

Our end goal is to read the content of `flag`. Looking at the permissions, we can see that only the `fd_pwn` user and members of the `root` group will be able to read the flag file. Fortunately, the `fd` binary has the [setuid](https://en.wikipedia.org/wiki/Setuid) bit on, which means that the binary will effectively run as if our user id was `fd_pwn` (i.e., it will be able to read `flag`). Conveniently, here they provide us the source code in `fd.c`.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
	if(argc<2){
		printf("pass argv[1] a number\n");
		return 0;
	}
	int fd = atoi( argv[1] ) - 0x1234;
	int len = 0;
	len = read(fd, buf, 32);
	if(!strcmp("LETMEWIN\n", buf)){
		printf("good job :)\n");
		system("/bin/cat flag");
		exit(0);
	}
	printf("learn about Linux file IO\n");
	return 0;

}
```

We pass in the file descriptor (+ 0x1234 (=4660)) as the first argument to the program, and then, it reads (up to) 32 characters from that buffer, and if the value is `"LETMEWIN\n"`, we'll get the flag. We have a few options for how to approach this, but perhaps the simplest relies on the standard streams corresponding to 0 (`stdin`), 1 (`stdout`), and 2 (`stderr)`. Then, if we use 4660, we're asking it to read from `stdin`, so to solve

```
fd@pwnable:~$ ./fd 4660
LETMEWIN
good job :)
the flag is here, but it's probably more fun if you run it yourself
```

If we copy the flag (run on `pwnable.kr`) and put it in the `Flag? :` textbox on the website and auth, we will get the one point for the level.

To expand upon later:

* Using bash redirections
* partial `read`s
* which parts are entered in the solution
