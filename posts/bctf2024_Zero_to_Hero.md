---
usemathjax: true
---
# Zero to Hero (@ [b01lers CTF 2024](https://ctftime.org/event/2250))


This [puzzle](https://github.com/b01lers/b01lers-ctf-2024-public/tree/main/pwn/zero_to_hero) was a combination of two inspirations: a register wipe challenge that I encountered somewhere (maybe 
[picoGym](https://picoctf.org/index.html#picogym)?) that used the default`mmap`address and took great care to wipe even the stack(!), and 
an unintentional solve I once did via an`xmm0`leak.

The executable is stripped x86_64 ELF code with all protections enabled: full RELRO, stack canary, NX, PIE, FORTIFY. In 
addition, ASLR is of course enabled on the host. You do get arbitrary code execution for free - up to 512 bytes of your 
supplied code is copied to a fixed starting address 0x10000 and executed. However, the chal is seccomped, the only 
allowed syscall is`exit`=> you have to leak the flag one byte at a time.

Getting just`exit`might have felt anal but there was good reason for that. For example, with ability to`write`to stdout 
one can bruteforce ASLR in less than a minute :) [Syscalls do not segfault on bad address; rather, they return a 
meaningful error code.] The alternative would have been running the code in an emulator, which gives us much more 
flexibility in what limits to put on the supplied code - but that looked like too much work to implement.

Here are two [solvers](https://github.com/b01lers/b01lers-ctf-2024-public/tree/main/pwn/zero_to_hero/solve) from the 
public bctf2024 repo.


#### Step 0
get the docker image running - make sure to increase the timeout of the jail, e.g.,

```python
  ENV JAIL_MEM=10M JAIL_TIME=100000
```


#### Step 1
connect to the chal, attach`gdb`to the running`z2h`process, and set a breakpoint at the`jmp rax`instruction that follows the register 
wiping sequence. Now hunt for address leaks :)



#### Step 2

many x86_64 registers were zeroed but not all. For example, libraries frequently take advantage
of 16-byte read/write instructions, which go through SSE registers. Same was true here:

```
gef➤  i reg xmm6
  ...
  v2_int64 = {0x7fc04d602520, 0x3},
  ...
```

which pretty much looks like some library leak. Oddly, the address is just a little bit *beyond* the end of`libc.so`:

```
gef➤  i files
  ...
  Entry point: 0x55bfe332e440
  0x000055bfe332d318 - 0x000055bfe332d334 is .interp
  ...
  0x000055bfe3331060 - 0x000055bfe33310e8 is .bss  <-- STEP 5
  0x00007fc04d62a2a8 - 0x00007fc04d62a2c8 is .note.gnu.property in /lib64/ld-linux-x86-64.so.2
  ...
  0x00007fc04d659000 - 0x00007fc04d659190 is .bss in /lib64/ld-linux-x86-64.so.2  <-- STEP 4
  0x00007ffffcf86120 - 0x00007ffffcf86164 is .hash in system-supplied DSO at 0x7ffffcf86000
  ...
  0x00007ffffcf86a20 - 0x00007ffffcf86a3c is .altinstr_replacement in system-supplied DSO at 0x7ffffcf86000
  0x00007fc04d410350 - 0x00007fc04d410370 is .note.gnu.property in /lib/x86_64-linux-gnu/libc.so.6
  ...
  0x00007fc04d5fbd60 - 0x00007fc04d5fbff8 is .got in /lib/x86_64-linux-gnu/libc.so.6  <-- STEP 3
  ...
  0x00007fc04d5fd7a0 - 0x00007fc04d601660 is .bss in /lib/x86_64-linux-gnu/libc.so.6
gef➤
```  

I did not bother with finding out how it got there ;) but if you rerun, the offset is consistent => so we have a libc leak.

Note, if you send too much input,`xmm6`gets overwritten - but the challenge is still solvable using the alternative leak discussed 
at the end of this writeup.



#### Step 3 

leverage the libc leak it to an`ld.so`leak. The C library links against`ld.so`:

```
> nm -D libc.so.6 | grep " U"
                 U _dl_argv
                 U _dl_exception_create
                 U _dl_find_dso_for_object
                 U __libc_enable_secure
                 U _rtld_global
                 U _rtld_global_ro
                 U __tls_get_addr
                 U __tunable_get_val
```

so look at its GOT

```
gef➤  x/32xg 0x00007fc04d5fbd60
0x7fc04d5fbd60:	0xffffffffffffffa8	0xffffffffffffffa0
...
0x7fc04d5fbdf0:	0x00007fc04d658060	0x00007fc04d5fd440
...
gef➤  x/20si 0x00007fc04d658060
   0x7fc04d658060 <_rtld_global>:	nop
   0x7fc04d658061 <_rtld_global+1>:	xchg   ecx,eax
   0x7fc04d658062 <_rtld_global+2>:	rex.WRB sar BYTE PTR gs:[r15+0x0],0x0
   0x7fc04d658068 <_rtld_global+8>:	add    al,0x0
   ...
```

Yay, this is an `ld.so`leak.


#### Step 4

The dynamic linker/loader is statically linked but it did load the challenge binary. So look for any leftover info in its BSS:

```
gef➤  x/32xg 0x00007fc04d659000
0x7fc04d659000 <newname.10978>:	0x00007fc04d62a9d1	0x0000000000000000
...
0x7fc04d659050 <_dl_rtld_libname>:	0x000055bfe332d318	0x00007fc04d659000
...
```

Hurray - we are done, this is a `z2h` leak.


#### Step 5  

read off the flag starting at .bss + 0x20.


#### Alternative to Step 2

With the above solution settled, I was wondering whether there are any other avenues to close off besides limiting syscalls to 
`exit` only. In particular, I was curious what info might be lurking in memory near the stack canary`fs:0x28`. It is read-only area (= cannot 
be wiped) and it does have a good leak at`fs:0`:

```
gef➤  print $fs_base
$2 = 0x7fc04d603540    -> this is fs:0
gef➤  x/8xg $fs_base
0x7fc04d603540:	0x00007fc04d603540	0x00007fc04d603ea0
0x7fc04d603550:	0x00007fc04d603540	0x0000000000000000
0x7fc04d603560:	0x0000000000000000	0xf78b533f86595000
...
```

I.e., `[fs:0]` points to itself, and this is an address slightly beyond the end of libc but with a stable offset 0x77e0 to`libc.got` => so 
again a libc leak.

For half a minute I considered reworking the chal to spoil the`xmm`leak but in 
the end decided to leave both options.

**One small regret:** the vcpu we got in Google Cloud did not have Intel's CET tech (or it was inactive), so people 
could get by without using`endbr64`in their solution.

---

[back](/)
