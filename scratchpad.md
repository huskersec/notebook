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












