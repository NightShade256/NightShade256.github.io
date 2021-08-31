---
layout: default
title: A Chip-8 interpreter, ...for the Game Boy?
tags: emulation gameboy chip-8
---

# A Chip-8 interpreter, ...for the Game Boy?

For the past couple of days, I was busy writing a Chip-8 interpreter, called Cobalt. While that in itself might not be
remarkable, what is though, is that Cobalt was written for the Game Boy (DMG) in pure assembly.

I believe it is the second Chip-8 interpreter, behind Optix's GB-8, which runs on the Game Boy Color and the first
that runs on the Game Boy.

## Introduction

Before I start talking about Game Boy or Chip-8 specific details, it is only natural I give a small introduction of 
both the systems.

Quoting from Wikipedia,

```
CHIP-8 is an interpreted programming language, developed by Joseph Weisbecker. It was initially used on the COSMAC
VIP and Telmac 1800 8-bit microcomputers in the mid-1970s. CHIP-8 programs are run on a CHIP-8 virtual machine. It
was made to allow video games to be more easily programmed for these computers.
```

Chip-8 was never a real _physical_ system, but a programming language made for ease of game development. It supported
a _tiny_ 64 x 32 screen and 2 drawing colours. It features a unique drawing method which uses the XOR bitwise 
operation.

The Game Boy on the other hand is theoretically much more powerful, with a Sharp SM83 CPU (not an 8080 or Z80)
clocked at roughly 1 MHz (the internal clock is timed to 4 Mhz but the CPU always takes 4 cycles to do anything
meaningful). It features a custom graphics chip called the PPU which uses a unique tile-based rendering mechanism
similar to the NES.

## Starting the Implementation

The biggest hurdle actually came before any code was ever written!

The Game Boy is quite puny compared to modern computers. A cheap Raspberry Pi is thousands of times faster than
than the Game Boy ever was. This also means that using the least amount of CPU cycles and memory is of utmost 
importance. This in effect severely restricts the performance of stack based languages like C.

The only realistic option here was to use raw assembly, with an assembler like `RGBDS` or `WLA-DX`. I went with
`RGBDS` since that is what most projects use.

This was a huge challenge since I didn't have much prior experience writing assembly for any particular platform.

## Graphics

The Game Boy can comfortably fit even twice the size of the Chip-8 display in its viewport, but in practice it turned
out to be extremely hard.

The Game Boy uses a tile based rendering system. A group of 8 x 8 pixels is called a *tile*. Tiles are then grouped
together in VRAM to produce images. Each row of a tile is represented by two bytes making a tile sixteen bytes in 
size. The first byte of a row contains the top bit of the colour of the pixel, and the second byte contains the
bottom bit.

```ascii
For Example:

10101010 01010101 -> 10 01 10 01 10 01 10 01
```

This makes dynamic drawing with pixel-granularity a bit of a hassle. Another problem is that a Chip-8 sprite can span 
up to four complete tiles making drawing even harder. What's more is that Chip-8 expects all draw calls to instantly 
take effect, while the Game Boy only supports accessing VRAM at certain fixed times.

One might then naively then perform the following calculation,

``` ascii
Chip-8 Display -> 8 x 4 tiles -> 32 tiles * 16 bytes -> 512 bytes
```

and conclude that the Chip-8 VRAM needs to be `512` bytes in size. This is actually false. We can do better. Instead 
of keeping the second byte around, we can change the `BGP` register to map `10` colour to White and `00` colour to
black reducing the byte count to half! This means `256` bytes are enough to represent the whole of Chip-8 screen.

The problem now is that the programmer can only modify the VRAM in VBlank and HBlank (and OAM Search). All other times
the VRAM is locked by the PPU and writing to it during that time yields no effect. The emulated `CLS` instruction
alone takes `2560` M-cycles which is more than the whole period of VBlank. Therefore it is impossible to run the
emulation loop in just those times.

So, one solution is to keep a *shadow* VRAM in Work RAM and render everything there, and then just copy over the 
shadow VRAM data to the actual VRAM at VBlank. Sounds great right? Well there is a problem. VBlank lasts for only 
around `1140` M-cycles but it takes around `10` M-cycles to just copy *one* byte of data, thus making it *impossible* 
to copy the buffer to VRAM in VBlank.

At first, I couldn't think of a solution, but then I looked over to the Game Boy Colour for some inspiration. The Game
Boy Colour provides two DMA options to quickly transfer bytes from one memory region to the other. One of these is
called HDMA (HBlank DMA). When HDMA is initiated `16` bytes of data is transferred every HBlank to the specified
memory location in just `8` M-cycles. The execution of the rest of the program is suspended while this copy takes
place. The Game Boy on the other hand does not have any such DMA option (except the OAM DMA which is quite useless
in this particular case).

So I wondered, what if I emulated the HDMA transfer? The answer is that it works! HBlank lasts for around `51` 
M-cycles (assuming rendering without sprites and scrolling), making it possible to transfer about `2-3` bytes of data 
per HBlank. There are `144` scan lines in one frame, but copying `256` bytes only actually requires `128` HBlank(s) 
if `2` bytes are copied each time!

I quickly started writing out the implementation. The Game Boy provides us with the LCD STAT interrupt which you can 
setup to trigger at the start of HBlank. Using that I was able to write a simple working implementation in no time. 
It worked as expected and was reasonably fast.

## Input

Chip-8 is unique in the respect that it accepts input using a keypad with a total of `16` keys. The Game Boy only has 
`8` physical keys (which includes the A, B, Select, Start buttons). This is the point where there is no easy solution.
You can implement a scheme where pressing two-buttons would correspond to a unique Chip-8 key, but this is a lot more
difficult to implement with no real benefit since a majority of Chip-8 games don't actually utilize the full 16 keys.

So I just mapped the four Game Boy DPAD keys to four Chip-8 keys and then left it as is, and it works for a majority
of games. Even if a game doesn't work, you can always modify the source to change the key mapping.

## Ending

In conclusion, I think it was a fun journey to undertake. It really sheds some light on how powerful modern computers
are and how we take that power for granted without thinking about what goes on underneath.

If you want to check out Cobalt, you can do so [here](https://github.com/NightShade256/Cobalt/).

Cheers!
