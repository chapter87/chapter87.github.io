---
title: 'My first crackme: the clue was the function that was not there'
description: 'I solved a beginner crackme from crackmes.one on Linux with seven commands and no decompiler. The password was never in the binary as text. The thing that gave it away was a library function the program did not import.'
pubDate: 'Oct 22 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

I have been reading about reverse engineering for months and doing none of it. This week I
changed that with the smallest possible target: a difficulty-1 crackme from
[crackmes.one](https://crackmes.one), a site where people upload small programs built on
purpose to be taken apart. The one I picked is called *thefirsttest*, written in C by
JoeJoeJoe, 12 KB, x86-64 Linux. The description is one line: "Recover the correct password
to win, I know you can do it!"

I did it twice. The first pass was guided, with someone handing me each command. Then I
redid it alone from a clean folder, which is the version written up here. Seven commands,
no Ghidra, no IDA. Just the tools that were already on my Fedora laptop.

What I want to record is not the answer (it is a difficulty 1, the answer is not interesting)
but the order of operations, because that order is the same for a crackme as it is for an
unknown binary someone emails you at work.

## 1. Ask the OS what it is

```
$ file thefirsttest
thefirsttest: ELF 64-bit LSB executable, x86-64, dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2, not stripped
```

Three words in there do real work. *x86-64* tells me which registers I will be reading.
*Dynamically linked* means the program borrows functions like printing and reading from
the system C library at run time, so the list of borrowed functions will describe what it
can do. *Not stripped* means the author left symbol names in. I will see `main` by name
rather than as a bare address. On a beginner crackme that is a gift. On malware it almost
never happens.

## 2. Let it talk

```
$ strings -n 4 thefirsttest | grep -v '^[._]'
Welcome to your first test!
Please input the password:
WARNING: Please input password twice
Well done, you are a master hacker!
Wrong password, try agian!
```

(There is compiler noise above and below that, trimmed.)

Two things. The password is not sitting there in plain text, so the program either computes
it or compares against something that is not readable text. And I now have the exact
success string, which means I can find where it is printed and walk backwards to the
decision that leads there.

That "input password twice" line bothered me. I made a note and moved on. More on it below.

## 3. List what it borrows

```
$ nm -D thefirsttest | grep ' U '
                 U __isoc23_scanf@GLIBC_2.38
                 U __libc_start_main@GLIBC_2.34
                 U puts@GLIBC_2.2.5
```

`U` means undefined here, resolved from a shared library when the program starts. Ignore
`__libc_start_main`, it is just the C runtime calling `main`. That leaves `puts` (print a
line) and `scanf` (read formatted input). Nothing else.

This is the line that solved the crackme for me, and it did so by absence. A C program that
compares a text password needs `strcmp` or `memcmp`. Neither is imported. So whatever
check `main` does, it cannot be comparing strings. The only thing left is comparing a
value, which means the "password" is a number. I had not read a single instruction yet.

I did not fully trust that inference, and I think not trusting it was correct. An author
could inline a byte-by-byte comparison loop and never call `strcmp`. So the next step was
to confirm it in the code rather than stop here.

## 4. Read main

```
$ objdump -d -M intel --no-show-raw-insn thefirsttest | sed -n '/<main>:/,/ret/p'
```

The whole function is 25 instructions. The part that matters:

```
  40049c:    lea    rax,[rbp-0x4]
  4004a0:    mov    rsi,rax
  4004a3:    mov    edi,0x401235
  4004ad:    call   400380 <__isoc23_scanf@plt>
  4004b2:    mov    eax,DWORD PTR [rbp-0x4]
  4004b5:    cmp    eax,0x72abb3c9
  4004ba:    jne    4004c8 <main+0x52>
  4004bc:    mov    edi,0x401240
  4004c1:    call   400370 <puts@plt>
  4004c6:    jmp    4004d2 <main+0x5c>
  4004c8:    mov    edi,0x401264
  4004cd:    call   400370 <puts@plt>
```

Reading it as a sentence: take the address of a 4-byte slot on the stack, hand it to
`scanf` along with a format string at `0x401235`, load whatever `scanf` wrote into `eax`,
compare it with the constant `0x72abb3c9`, and if they differ jump to the block that prints
the string at `0x401264`. Otherwise fall through and print `0x401240`.

Four bytes is the size of a C `int`. A single `cmp` against a constant. So step 3 was right.

And `scanf` is called once. The warning about entering the password twice was a bluff, which
I enjoyed more than I should have. It is a tiny example of something real: the strings a
program prints are not evidence of what it does. The code is.

## 5. Resolve the two unknowns

The format string and the constant.

```
$ r2 -q -e scr.color=0 -c 'psz @ 0x401235' thefirsttest
%d
$ printf '%d\n' 0x72abb3c9
1923855305
```

`%d` means `scanf` reads a decimal integer. So the input is the number 1923855305, typed as
digits.

## 6. Prove it, and prove the proof

```
$ echo 1923855305 | ./thefirsttest
Well done, you are a master hacker!
$ echo 72abb3c9 | ./thefirsttest
Wrong password, try agian!
$ echo 12345 | ./thefirsttest
Wrong password, try agian!
```

The second test is the one I care about. If typing the hex digits as text had also worked,
my reading of the format string would have been wrong and the success on the first line
would have been luck. A deliberate failure that matches the model is worth as much as the
pass.

## 7. Watch the decision happen

Everything above is static: I read the program without running it. The last step is
dynamic. I put a breakpoint on the `cmp` instruction itself, fed the program a wrong
answer, and looked at the two sides of the comparison while it was paused.

```
(gdb) break *0x4004b5
Breakpoint 1 at 0x4004b5
(gdb) run < wrong.txt
Breakpoint 1, 0x00000000004004b5 in main ()
(gdb) p/d $eax
$1 = 12345
(gdb) p/d 0x72abb3c9
$2 = 1923855305
(gdb) set $eax = 0x72abb3c9
(gdb) continue
Well done, you are a master hacker!
```

My wrong input, 12345, sitting in the register one instruction before judgement. Then I
overwrote the register and let it continue, and the program congratulated me without ever
receiving the real password. That is the move that matters beyond crackmes. On a sample
that refuses to run outside its target environment, this is how you force it down the
path it is hiding.

## What the seven steps add up to

Steps 1 to 5 are triage: identify, strings, imports, disassemble, resolve. Steps 6 and 7
are verification, static then dynamic. I have seen this sequence described in a dozen
places and never felt it until I ran it with my own hands on a binary I had not seen the
source of.

The thing I will remember is step 3. The absent `strcmp` told me more than any string the
author put in. I expect that pattern, learning from what a program does *not* do, shows up
a lot further up the difficulty scale.

After finishing I installed Ghidra and opened the same file. Its decompiler turned those 25
instructions into six lines of C with the `if (local_c == 0x72abb3c9)` right there. Good
to have. Also a bit deflating, and I am glad I did it the long way first, because now I know
what the decompiler is summarising.

Next one is a difficulty 2 with a text password, which should bring in the `strcmp` that
was missing here, or a loop that does its job by hand.
