---
layout: page
title: "Writing Your First Gameboy Game"
permalink: /writing-your-first-gameboy-game
---

_Originally posted on [Medium](https://medium.com/@ulrikdamm/writing-your-first-gameboy-game-4ea62c76db29)_

The Gameboy was (and still is) a pretty great device. Not only from a gamers perspective, but also from the perspective of a programmer, who wants to learn more about how computers work on a more fundamental level. Assembly language, CPU opcodes, memory mapped I/O, all these things that are fundamental to how computers work, but we don’t really get to use much in our everyday work, since we’re working on a much higher level of abstraction now. But it can be fun and useful to know how our everyday devices work under the hood. Combined with that, retro computing and games are always fun!

As I said, the Gameboy is a perfect device for this. Released in 1989, it has a Z80-based 8-bit CPU (not too different from the Intel 8080, which you might know of) with a 16-bit address space and a clock speed of 1 MHz, along with around 16kb of RAM. The executables comes on cartridges, which can be anywhere from 32kb to multiple megabytes in size. The cartridges itself is just a piece of hardware, so it can also have extra RAM, battery-powered RAM for game saves, or even more crazy things, like a camera. And it’s easy to find emulators for to run and debug your programs.

In this article, we’ll be writing a simple Gameboy game, which just displays a sprite on the screen. That’s not much of a game, but there’ll be a lot to learn to get to that point. We’ll be using an assembler I’ve written in Swift, which you can find on Github for macOS and Linux. The article will first explain how the Gameboy CPU works, then how you write assembly programs, then how graphics works on the Gameboy, and finally, we’ll be combining all those pieces into a working program. Let’s get started with learning a bit about how the Gameboy works!

# The CPU and the address space

There are two pieces that are central to understanding how the Gameboy, and really any computing system, works: The CPU, which takes instructions and executes them, and the address space, which is used to access the different hardware components. When an instruction is run on the CPU, it modifies the state of the system somehow; either the internal state of the CPU, or the state of some hardware component, which is accessed through memory addresses. The Gameboy has a 16-bit address space, which means that you can access memory addresses from 0 to 65535 ($ffff in hexadecimal). The hardware of the Gameboy has mapped each of these addresses to some hardware; it could be the cartridge ROM, the sound chip, or some general purpose RAM.

When you compile your assembly code, it’s turned into a binary, which is a series of opcodes. An opcode is a number, representing one instruction the CPU can perform. As an example, the opcode $00 means NOP, which is a no-operation. $82 is an ADD instruction, which adds two numbers together. Your binary is a long list of these opcodes, which are placed in the ROM on the cartridge. The cartridge ROM is memory mapped to the first half of the Gameboy’s memory space, meaning all addresses from $0000 to $7fff (These values are easier to speak of in hex numbers. A 16-bit value is a 4-cipher hex number). This means that if you read any address between $0000 and $7fff, it’s going to read one byte from the cartridge. Note that the address space is 16-bit, meaning that each address to a memory location is between $0000 and $ffff, but the values that are stored in those locations are 8-bit, so between $00 and $ff.

The Gameboy CPU keeps its state in internal registers. Each register is a 8- or 16-bit value, which is stored directly inside the CPU, and it therefore very fast to read and write, since it doesn’t have to talk to any other hardware component through the address bus. The Gameboy has eight 8-bit registers and two 16-bit registers:

8-bit registers: A, F, B, C, D, E, H, L

16-bit registers: SP (stack pointer), PC (program counter)

The SP register keeps track of the size of the stack (we’ll get to that later), and the PC register keeps track of where in the code the CPU are currently executing. These two registers are 16-bit, just like the address space. This is because they’re pointing to a location in the address space. The PC register is literally pointing at the memory location it’s going to execute next, and you jump around in your code by changing the value of PC.

All the 8-bit registers, expect for F, are general purpose registers, and you can read and write values to them as you please. For example, if you want to add two numbers together, you can put the first value into the A register, the second value into the B register, and then call the ADD A, B instruction, which will read the value of A, read the value of B, add those two together, and place the result in A. The F register, the Flags register, is a special register that contain single-bit flags, which are set from different instructions. If you’re wondering why the ordering is a bit strange, that’s because some instructions use two 8-bit registers as a single 16-bit register, in case you want to be able to, say, add numbers that are bigger than 255 ($ff). These 16-bit combination-registers are called AF, BC, DE and HL.

When the Gameboy boots up, the first thing you see is the Gameboy logo. After that, it will hand control of the entire system to your code, by setting PC to $100. This is the entry point of your program. The CPU runs in cycles, and each of these cycles will read the byte pointed to by PC, then increase PC to point at the next byte in memory, and then actually execute the instruction represented by the byte value it just read. The available instructions for a CPU and the mapping between CPU instructions and their opcodes is the instruction set. The Gameboy has 499 instructions in total, you can see a reference of all of them here. For example, you can see that the NOP instruction has the opcode $00, which means that if the CPU reads a $00 when fetching the instruction, it will just pause for a while, and then move on to the next instruction.

So let’s write the first lines of assembly code to handle the entry point for our game.

# Writing assembly programs

In the programming languages you’re used to, there are usually things like types, control flow like if and while loop, functions that take arguments and return values, etc. Assembly, being a low level language, doesn’t have any of this. Assembly is really just a readable layer on top of the raw machine bytes. So instead of writing $00 into a hex editor, you write the mnemonic for that instruction, which is NOP. Assembly code is just a long list of these instructions. The only tool you have other than writing instruction mnemonics is labels, which are names you can give to series of instructions. You use the labels to jump in your code. So, where you in C-like languages might have a function like this:

```c
void stopCPU() {
 halt()
}
```

In assembly, this would simply be

```
stopCPU:
 halt
 ret
```

(HALT is an instruction which stops the CPU temporarily. RET returns to the caller by restoring PC. Also note that the assembly is case-insensitive, so while I’ll be using UPPERCASE when describing instructions and registers in the text, I’m using lowercase in the code itself)

You might also have variables and functions that return values in your usual high-level language. In assembly, you always need to put your values somewhere. This can either be in RAM, or in one of the general purpose registers. So, where you would usually write something like this:

```c
int add_5(int value) {
 return value + 5;
}
```

In assembly, this would be

```
add_5:
 ld A, B
 add A, 5
 ret
```

(LD is a load instruction, copies a value from one register to another)

This takes the input value through the B register, and returns a value by modifying the A register. Typically you will write which registers are used in a documentation comment above the label:

```
# Add 5 to a value
# B: input value
# A: output value
add_5:
 ld a, B
 add a, 5
 ret
```

When you want to call this “function”, you use the call instruction:

```
ld b, 10
call add_5
```

CALL will take current value in PC (the pointer to the next instruction to get executed), and save it on the stack. The stack is an area in memory where data can be stored and retrieved. It’s controlled by the stack pointer (SP), and every time a value is pushed or popped from the stack, SP will increase or decrease. You can push and pop values manually with PUSH <register> and POP <register>. In this case, PC is pushed to the stack, so it’s saved in memory, and then modified to point to the add_5 label, which is the location in memory of the first instruction under that label. The CPU will then run instructions from there, until it hits the RET instruction, which will pop the old PC value back from the stack, which is pointing to the instruction after the CALL, and continue running where it left off.

It’s also possible to modify PC directly without saving the value to the stack. This is a jump (or goto), and is used for control flow like loops. Take this high-level loop:

```c
for (int i = 0; i < 10; i++) {
 // code goes here
}
```

What this code is doing is running the same piece of code 10 times. It will go through the body of the loop once, then increase i, check the condition, and if it holds, jump back to the top, and perform the loop one more time. In assembly, we do the same, just without all the fancy abstractions:

```
ld a, 10
loop:
 # code goes here
 dec a
 jp nz, loop
```

This will load 10 into register A, run through the loop body, then decrement a by one and jump to the loop label if A is not zero (nz = not zero). The JP instruction can take a label, or a condition and a label (or a direct memory address to jump to, but don’t do that unless you know what you’re doing). The conditions will check the bits in the F (flags) register. One of these flags are the Zero flag, which is set when an arithmetic instruction performs a calculation that results in a value of zero. In this case, if DEC sets A to zero (which it will on the last iteration, when A is decremented from 1 to 0), it will set the Zero flag, and the JP instruction will not jump back the beginning.

So that’s basically what you need to know about assembly. The rest is just knowing all the different instructions you have at your disposal. Here’s some common ones that you’ll probably need:

- `NOP`: doesn’t nothing, just pauses the CPU for one cycle
- `ADD <r1>, <r2>`: Adds the value of register r1 and r2, and stores the result in r1. For example, ADD A, B will add A and B and store the result in A. r2 can also be a direct value, like ADD A, 5.
- `SUB <r1>, <r2>`: Like add, but subtracts values instead. When a value goes below zero, it will wrap back around to 255.
- `LD <r1>, <r2>`: Copies the value of r2 into r1, discarding the old value in r1. r2 can also be a direct value.
- `JP [condition], <target>`: Jumps to the target label or memory address if the condition holds. If there’s no condition, it will just jump there. The condition can be Z (Zero flag is set), NZ (Zero flag is not set), C (Carry flag is set) or NC (Carry flag is not set).
- `CALL [condition], <target>`: Like jump, but will save the value of PC to the stack before jumping. Can also have a condition.
- `RET`: Returns from a CALL by restoring PC from the stack.
- `PUSH <r1>`: Pushes r1 to the stack. r1 can also be a direct value.
- `POP <r1>`: Pops the most recently pushed value off the stack, and puts it in r1.
- `INC <r1>`: Increases the value in r1 by one
- `DEC <r1>`: Decreases the value in r1 by one
- `AND <r1>`: Performs a logical AND operation on A and r1, storing the result in A. Logical AND takes each bit in each value and returns 1 if both bits are set.
- `OR <r1>`: Performs a logical OR operation on A and r1, storing the result in A. Logical OR takes each bit in each value and returns 1 if any of the two bits are set.
- `XOR <r1>`: Performs a logical OR operation on A and r1, storing the result in A. Logical OR takes each bit in each value and returns 1 if *only* one of the bits are set.

For a complete list of instructions, you need to find a Gameboy programming manual.

Now, with that introduction to assembly and the most common instructions, let’s get to actually writing some code for the Gameboy!

## Beginning your program

As I said earlier, the Gameboy will start running your code from memory location $100. How do we put some code at exactly that location? We do that by writing a label, and giving it an origin. This tells the assembler to put the code for that label at that location in the ROM.

```
[org(0x100)]
start:
 # Code goes here
```

Now the code under that label will be run when our game starts. And actually, the first thing we need to do is to jump somewhere else. The Gameboy has specific parts of the ROM reserved for information about the game cartridge, like its ROM size, the game title, etc., and that information is stored from location $104 to $14f, which means that we only have 4 bytes available before the game information starts! Luckily, this is just enough to jump away. The JP instruction opcode is one byte, and that is followed by the memory location it wants to jump to, which is two bytes (a 16-bit address). Because hardware is weird, it’s also recommended that you put a NOP as your first instruction, and with that, we’ve used our 4 bytes of space.

```
[org(0x100)]
start:
 nop
 jp game_init
[org(0x150)]
game_init:
```

This will jump to a game_init label, which we defined just underneath. If we had defined the label without an origin, the assembler would just put it right after the start routine in memory, and this is exactly what we didn’t want to happen, since that’s reserved, so we put the game_init label at $150, which is just after the reserved memory ends.

So now we have an entry point for our program. We should try to run it, just to confirm that it’s not completely blowing up. One quick thing we’ll need before that, is to have our binary be the correct size. The minimum size of a Gameboy cartridge ROM is 32kb ($8000 bytes), so we’ll have to produce a binary of that length. To do that, we’ll make a label at position $7fff, the last byte in our ROM, and place a value of $00 there.

```
[org(0x7fff)] pad: db 0x00
```

DB is not a Gameboy instruction. You won’t find it anywhere in the instruction set. This is a special instruction for the assembler. It means “define byte”, and let’s you put values directly in your binary. Remember that the binary is just a list of numbers. Some are instruction opcodes, some are game information, some could be sprite and sound resources. And they can also just be numbers that you define, like we did here.

Now we can try save it as game.asm and compile it. You can get the assembler by just cloning it and running swift build.

```
> git clone git@github.com:ulrikdamm/Assembler.git
> cd Assembler
> swift build
```

And then just run the binary from where you’ve put your assembly source.

```
> Assembler/.build/debug/Assembler game.asm
```

This should produce a binary called game.gb, which you can try to run in an emulator. If everything went well you should have a blank screen. Progress!
Loading graphics into memory

To display graphics on the screen, we need a piece of graphic to display, and to put this into the Gameboy’s VRAM. In the address space, the ROM on the Cartridge is mapped to the $0000 — $7fff range. Just after that, from $8000 to $a000, is the VRAM, which is memory built in to the Gameboy, where you can put your sprite and tile data. That data will ship as part of your ROM, so we’ll have to place it somewhere in our binary, and then copy that data into the VRAM area. Let’s start by creating the sprite we want to display, and put it in the ROM. Sprites are stored as 8x8 pixel tiles, so I’ve made this 8x8 pixel smiley we can display.

```
________
__#__#__
__#__#__
________
#______#
_######_
________
________
```

The screen on the Gameboy can show four different “colors” at the same time. I’m putting “colors” in quotes, because the Gameboy had a monochrome display, so what I really mean is four shades of grey. This means that each pixel of our sprite needs two bits of information, which can represent a value from 0 to 3. We won’t be using other colors than black and white for this example, so we’ll just use 0–0 to represent white and 1–1 (3) to represent black. Which colors are represented by which values is something we can specify ourselves by setting a palette, which we will do later. Now, we need to convert our sprite into binary data. The sprite is stored in 16 bytes: 8 pixels x 8 pixels x 2 bits per pixel. It’s stored in lines, so the first byte is the first bit of the first line; the second byte is the second bit of the first line; the third byte is the first bit of the second line, etc.

```
00000000 $00
00000000 $00 — first line
00000000 $00
00000000 $00 — second line
00100100 $24
00100100 $24 — third line
00000000 $00
00000000 $00 — forth line
10000001 $81
10000001 $81 — fifth line
01111110 $7e
01111110 $7e — sixth line
00000000 $00
00000000 $00 — seventh line
00000000 $00
00000000 $00 — eighth line
```

I hope you get the idea. So the data for our smiley sprite is

```
$00 $00 $00 $00 $24 $24 $00 $00 $81 $81 $7e $7e $00 $00 $00 $00
```

To put this data in our binary, we can just use the DB command again, to make the assembler just put these numbers directly in the ROM.

```
smiley_sprite: db 0x00, 0x00, 0x00, 0x00, 0x24, 0x24, 0x00, 0x00, 0x81, 0x81, 0x7e, 0x7e, 0x00, 0x00, 0x00, 0x00
```

There we go. Now to write a routine for copying these bytes into the correct VRAM address. For this, we’ll use a special register value. Remember back to the 8-bit general purpose registers, there were two of them named H and L, which combined to form the 16-bit general purpose register HL. Well, when we have a 16-bit value, we can use this as a pointer in our memory, so the Gameboy have some handy instructions for treating the value pointed to by HL as a register. So say that we want to put the value $ff into the RAM at address $8100. We can use LD to put the number $8100 into HL, put $ff into A, and then say LD [HL], A to load the value $ff into the memory address $8100. Or the other way around, if we want to load the contents of the memory address into A, we just say LD A, [HL]. Using a value in [square brackets] generally means that the value is a pointer, and it should operate on the byte being pointed to. If you don’t want to use HL, you can also just say LD [0x8100], A.

That’s very handy, but it doesn’t even stop there. A very typical operation to do is to loop over a lot of memory, copying bytes, just as we want to do. You use LD [HL], A for this, by loading the start address into HL, and then increasing HL for each loop iteration. Well there’s a special syntax for making that even easier: LD [HL+], A, which will load A into the address pointed to by HL, and then increasing HL by one afterwards. And of course, you can also do LD [HL-], A, if you’re going to the other way.

So let’s use this, and what we learned about loops before, to write some code that copies our graphics data into memory at location $9000 (which is where the background tile data is stored).

```
ld hl, 0x9000 + 16
ld de, smiley_sprite
ld b, 16
copy_loop:
 ld a, [de]
 inc de
 ld [hl+], a
 dec b
 jp nz, copy_loop
```

Let’s go through this. First we put $9000 into HL. We’ll use HL as the target pointer. Then we put smiley_sprite into DE. Remember that this was the name of the label where we defined our sprite data, so when we use this in a statement, it will become the memory location of that label. So after this, DE will be pointing to the first byte of our sprite in the ROM. Then we load 16 into register D, which is the length of the data we want to copy. After that, we start the loop by putting in a label, which we are able to jump back to. The structure should look familiar to the loop we looked at before: after the label there is the loop body, and after that we decrement D and jump back to the label if D is not zero. Inside the loop body, we transfer one byte from the address pointed to by BC to the address pointed to by HL, and increase both of them by one (using the special syntax [HL+] for HL, but this doesn’t exist for BC). The way to copy a byte between two memory addresses is to first load it into the A register, and then load from A to the target location.

Why can’t we just transfer directly from [BC] to [HL] I hear you ask? That’s because that instruction doesn’t exist in the instruction set. If you look in the instruction set reference you’ll find LD A, [BC], LD [HL], A, etc., but no LD [HL], [BC]. This is simply just a hardware limitation. If you try to write that, you’ll get an error from the assembler.

You might also wonder about LD HL, 0x9000 + 16. Typically when you want to add two numbers together, you have to do it through the A register. In this case though, we can figure out the result at build-time, so the assembler will be smart enough to know that this really means LD HL, 0x9010. We can write it this way to keep it more human-readable. Why we use this exact memory location we’ll get to in the next section.

So that’s it, after this, our sprite has been loaded into the VRAM, and can be displayed on the screen. To do that, let’s talk a bit about how the tile and sprite data are laid out in the VRAM, and how you turn on the display.

## Turning on the display

There are three ways to display things on the Gameboy display: the background, which is a 32x32 tile layer which can be scrolled, the window, which is a layer above the background which can also be scrolled around, and then a layer of individually movable sprites. For this example, we’ll just be using the background layer. We’ll set the first tile of the background to point to the sprite we loaded into memory. Which sprites are displayed in the background is just a list of numbers in memory from either $9800-$9bff or $9c00-$9fff (you can choose between the two locations with the LCDC I/O register, which we’ll get to in a bit). The value for each tile is an offset into the sprite data which is located from either $8800-$97ff or $8000-$8fff (Also selectable via LCDC). As you can see in our sprite loading code, we copied the sprite data to $9000 + 16, which is the location for the second sprite. Sprite #0 is located at $9000 when we have chosen the $8800-$97ff range for sprite data, since this is index with signed numbers, which means -128 to 127, with 0 being in the middle, at $9000. So we’re using tile index 1 for our sprite.

Now we need to specify in the background tile map. We’re using the $9800-$9bff range, which is unsigned-indexed. This a bit easier, since we only display one sprite, and thereby only have to set one byte, the first byte of this memory, with the value of the sprite index, which is 1.

```
ld hl, 0x9800
ld [hl], 1
```

As you can see, the instruction set does support loading a direct value into [HL], so we don’t need to load it into A first. Handy! If we wanted to use BC instead of HL, we would have to load 1 into A, and then load A into [BC].

Now we need to set the palette data. We talked about this earlier; this is how we define the “colors” of our sprites. For our sprite data, we used 0–0 (0) to mean white and 1–1 (3) to mean black. The palette data is an I/O register in memory. This means that it’s a special byte in the upper memory ($ff## addresses) which configures the Gameboy in certain ways when you modify it. The background and window palette is located at $ff47, and has the following bit layout:

- Bit 0–1: Color for value 0
- Bit 2–3: Color for value 1
- Bit 4–5: Color for value 2
- Bit 6–7: Color for value 3

Each color is defined as a 2-bit value. We will set it to a value that means 0 is the lightest, 3 is the darkest, and the two others are a value in between.

```
ld hl, 0xff47
ld [hl], 0b1110_0100
```

(You can use the 0b-prefix to specify binary numbers, which is often easier to handle than hex codes for bit fields. You can also place underscores to make the number more readable.)

With that done, the last thing we need to do is to specify which memory locations we want to use for our tile and sprite data, and turn on the display. This information is stored in the LCDC (LCD Control) I/O register at location $ff40. It has the following bit layout:

- Bit 0: Background and window display on/off
- Bit 1: Sprite layer on/off
- Bit 2: Sprite size (0: 8x8, 1: 8x16)
- Bit 3: Background tile map select (0: $9800-$9bff, 1: $9c00-$9fff)
- Bit 4: Background and window tile data select (0: $8800-$97ff, 1: $8000-$8fff)
- Bit 5: Window display on/off
- Bit 6: Window tile map select (0: $9800-$9bff, 1: $9c00-$9fff)
- Bit 7: Display on/off

For this example we want the following options set:

- Background and window display: on (1)
- Sprite layer: off (0)
- Sprite size: don’t care (0)
- Background tile map select: $9800-$9bff (0)
- Background and window tile data select: $8800-$97ff (0)
- Window display: off (0)
- Window tile map select: don’t care (0)
- Display: on (1)

So for the byte representing the LCDC options, we need to set bit number 0 and bit number 7, and load that value into the correct memory location.

```
ld hl, 0xff40
ld [hl], 0b1000_0001
```

And that’s it! The display is turned on, the background layer is turned on, it’s reading the tile map from $9800-$9bff, where the first byte is 1, which means that it will read that sprite from $9010, where we’ve loaded our smiley sprite into.
Now all we’re missing is to tell the Gameboy to stop running. If we just left the code like this, the PC would just keep on increasing, and it would just read garbage instructions from memory. We stop the execution with a very simple statement:

```
end: jp end
```

This will trap the PC forever, since it will always just jump to the same location (we could also have used the HALT instruction, but let’s keep things simple for now).

Let’s put it all together!

```
[org(0x100)]
start:
 nop
 jp game_init
[org(0x150)]
game_init:
 # Copy the sprite data into VRAM
 ld hl, 0x9000 + 16
 ld de, smiley_sprite
 ld b, 16
 copy_loop:
  ld a, [de]
  inc de
  ld [hl+], a
  dec b
  jp nz, copy_loop
 
 # Set the first byte of the background tile map
 ld hl, 0x9800
 ld [hl], 1
 
 # Set the background palette
 ld hl, 0xff47
 ld [hl], 0b1110_0100
 
 # Set LCDC to turn on the display
 ld hl, 0xff40
 ld [hl], 0b1000_0001
 
 # Stop execution
 end: jp end
smiley_sprite: db 0x00, 0x00, 0x00, 0x00, 0x24, 0x24, 0x00, 0x00, 0x81, 0x81, 0x7e, 0x7e, 0x00, 0x00, 0x00, 0x00
[org(0x7fff)] pad: db 0x00
```

Assemble it and open the binary in a Gameboy emulator, and you should see a happy smiley face in the top left corner!

_Our “game” running in a Gameboy emulator_

Depending on the emulator, you might also see some weird symbols on the screen, and this is because we didn’t clear the memory before use, so it might not be filled with zero’s. That will be an exercise for the reader. If you see that, try writing a loop that sets all the background tile memory to zero in the beginning of the program.

# Success!

That’s all for this time, but there’s still many things to learn about the Gameboy. I hope to cover more topics in another article in the future.
