---
layout: post
title:  "The optimiser vs libc"
date:   2018-03-14 19:43:25 +1100
categories: embedded c++
tags: C++ embedded C
---

I had a nice little problem crop up while working with code in a
bare-metal environment.  Its a bare-metal bootloader plugin that's
built with `-nostdlib`.  It contains a simple parser-component which
requires a stack (as most tree parsers do).  The whole thing runs
without a heap and has been validated on the same CPU architecture
running under linux (and valgrind), so the code is presumed pretty
good.

The first thing that happened is the parse failed because the depth of
the parsing `Stack` was too short for the - not a problem it's defined
as follows:

```cpp
template <size_t N>
class Stack
{
...
private:
    Entry mEntries[N];
    size_t mHead;
};

...

    // at the start of the parse-step
    Stack<4> node_stack;
    const Stack<4> path_stack(path); // initialised from the query parameters
```

So just change `<4>` to `<6>` and...  the entire parser doesn't get
started.  The board crashes to a generic error ISR with a corrupted
looking stack (real stack, not `Stack`).  The only address I get out
of the stack-dump is in the middle of `memset`, which would make
perfect sense if I'd blown the stack into no-man's land.

The bootloader stack runs out of SRAM, so even though our `.data` is
in DRAM we have a limited stack to play with.  So I play around moving
objects off the stack and onto the data segment.  I don't have a heap
to play with so its a bit of a pain moving from C++ stack objects to
static `.data` segment.

Fiddling with the stack didn't get me anywhere, nor did relocating the
stack from SRAM into DRAM and calling the parser from there.

I finally pushed deadlocks through the code to find the point where
the code disappears.

```cpp
    // hang(); // this one was reached
    Stack<6> node_stack;
    hang(); // never gets here
    const Stack<6> path_stack(path);
```

That's funny - right on that `Stack` I bumped the size of.  Change to
4 - OK, 5 or higher - goes missing.  So I push the hang() code into
`memset` itself and here is where I first *really* start scratching my
head.  `memset` was hand-coded by me because we don't have standard
libraries:

``` c
void *memset(void *s, int c, size_t n)
{
    uint8_t* s8 = (uint8_t*)s;
    while (n--)
        *s8++ = (uint8_t)c;
    return s;
}
```

And disassembled:

``` asm
Disassembly of section .text.memset:

00000000 <memset>:

void *memset(void *s, int c, size_t n)
{
   0:   27bdffe8        addiu   sp,sp,-24
   4:   afb00010        sw      s0,16(sp)
   8:   afbf0014        sw      ra,20(sp)
    uint8_t* s8 = (uint8_t*)s;
    while (n--)
   c:   10c00003        beqz    a2,1c <memset+0x1c>
  10:   00808021        move    s0,a0
  14:   0c000000        jal     0 <memset>
  18:   30a500ff        andi    a1,a1,0xff
        *s8++ = (uint8_t)c;
    return s;
}
  1c:   8fbf0014        lw      ra,20(sp)
  20:   02001021        move    v0,s0
  24:   8fb00010        lw      s0,16(sp)
  28:   03e00008        jr      ra
  2c:   27bd0018        addiu   sp,sp,24
```

Now I did **not** expect that: there's 3 variables this should be some
pretty simple register-only operation, yet the first stop is to push
some data onto the stack, then the extraordinary `jal 0 <memset>`.
For the MIPS challenged, that's jump to linked - ie `call`.  This
sucker is *recursing*.

(For the MIPS non-conversant remember that the instruction after the
`b`ranch or `j`ump is executed irrespective. The branch-delay slot.)

I try fiddle around the implementation of memset to see if its
something obvious (like misplaced parentheses turning a cast into a
function-call that hits a macro that calls memset - that old
chestnut), but nothing seems to do it - it's really there:

See [here](https://godbolt.org/g/HxSy6z) to play around with it yourself.

> `memset` really is calling `memset`

The brain starts ticking.  Why did I add `memset` in the first-place?
The parser is really a read-only process - I added `strnlen` and
`strncmp`, for the parser, but `memset` came because:

> changing -O3 to -Os caused `memset` to be required

And tracking cross-references shows that the C++ initialisers that set
their contents to zero user `memset` in `-Os`.  This makes sense from
a space saving perspective, but is surprising that a
compiler/optimiser adds function calls.

So what if...? what if the compiler looked at the implementation of
`memset` and said "that looks an awful lot like" `memset`.

So looking through the `gcc` man-page for `memset` leads us to:

```
 -ftree-loop-distribute-patterns
     Perform loop distribution of patterns that can be code generated with
     calls to a library.  This flag is enabled by default at -O3.

     This pass distributes the initialization loops and generates a call
     to memset zero.  For example, the loop

             DO I = 1, N
               A(I) = 0
               B(I) = A(I) + I
             ENDDO

     is transformed to

             DO I = 1, N
                A(I) = 0
             ENDDO
             DO I = 1, N
                B(I) = A(I) + I
             ENDDO

     and the initialization loop is transformed into a call to memset
     zero.
```

The short of that is:

> the compiler replaces code that looks like a `memset` with `memset`

And the fix is to add:

``` c
#pragma GCC optimize("no-tree-loop-distribute-patterns")
```

which you can [see](https://godbolt.org/g/bZD8cw) turns `memset` into
a big loop-unrolled mess, but no recursion.  Note you can play with
[godbolt](https://godbolt.org/g/HxSy6z) to show that this pattern is
reproduced across all gcc compiler versions and targets.

Once I have a cause it's now a lot easier to Google. See also
[https://sourceware.org/bugzilla/show_bug.cgi?id=15605](https://sourceware.org/bugzilla/show_bug.cgi?id=15605)
here with the fairly telling title:

> gcc-4.8 + tree-loop-distribute-patterns breaks is unsafe for GLIBC

That pretty much says it all.  Don't use the C compiler to build the C
library because the C compiler may attempt to use the C library to
implement the C code.

I seem to remember a setting in crosstool-ng that doesn't let you use
`-O3` when building - I'm starting to see why,...

I wanted to come up with a way to get the phrase _recursive memset
considered harmful_ into this article, but I couldn't think of one.
