# Stack Buffer Overflow Exploitation — CSCI 4250/6250
(Still transfering Project over)
> A full exploitation chain from manual shellcode injection to Return-Oriented Programming (ROP), demonstrating both offensive techniques and the countermeasures that motivate them.

---

## Overview

This project explores x86-64 stack-based buffer overflow vulnerabilities in a controlled, academic environment. It covers the full attack progression — from basic shellcode injection through advanced ROP chain construction — as well as automation via `pwntools`.

**Key skills demonstrated:**
- Binary exploitation on x86-64 Linux
- GDB-based memory analysis and exploit development
- NOP sled construction and shellcode injection
- Return-to-libc and ROP chain development (bypassing DEP/NX)
- Exploit automation with `pwntools`
- Custom shellcode authoring (x86-64 assembly)

---

## Environment

| Component | Details |
|---|---|
| Architecture | x86-64 Linux |
| ASLR | Disabled (`randomize_va_space = 0`) |
| Target 1 | Executable stack (`-z execstack`) |
| Target 2 | Non-executable stack (DEP/NX enabled) |
| Debugger | GDB with PEDA / pwndbg |
| ROP tooling | `ROPgadget` |
| Automation | `pwntools` |

---

## Repository Structure

```
.
├── README.md
├── part1_shellcode/
│   ├── exploit.py          # Manual exploit script (shellcode injection)
│   ├── shellcode.asm       # Raw shellcode (25-byte /bin/sh)
│   └── notes.md            # GDB analysis, offsets, NOP sled strategy
├── part2_rop/
│   ├── exploit.py          # Return-to-libc / ROP chain exploit
│   ├── rop_chain.md        # Chain construction walkthrough
│   └── notes.md            # libc mapping, gadget discovery
├── part3_automation/
│   └── auto_exploit.py     # pwntools brute-force automation
├── part4_root_shell/
│   ├── shellcode.asm       # Custom shellcode with setreuid(0,0)
│   └── notes.md            # Privilege escalation notes
└── part5_rop_shellcode/
    ├── exploit.py          # ROP-based execve syscall chain
    └── notes.md            # Full ROP chain construction
```

---

## Part 1 — Shellcode Injection (Stack Executable)

**Concept:** Overflow the buffer to overwrite the return address, redirecting execution into attacker-controlled shellcode placed on the stack.

**Technique highlights:**
- Used GDB + De Bruijn pattern (`cyclic`) to pinpoint the exact offset to `$rip`
- Prepended a NOP sled (`\x90 * N`) to increase landing reliability
- Injected a 25-byte `/bin/sh` shellcode payload
- Verified shell execution with `whoami`

**Shellcode used:**
```
\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53
\x31\xc0\x99\x31\xf6\x54\x5f\xb0\x3b\x0f\x05
```

**Stack layout at exploitation:**
```
[ NOP sled (\x90 * N) ][ shellcode (25B) ][ padding ][ &NOP sled → overwrite RIP ]
```

> See [`part1_shellcode/notes.md`](part1_shellcode/notes.md) for the full GDB session walkthrough and offset calculation.

---

## Part 2 — Return-to-libc / ROP Attack (DEP/NX Enabled)

**Concept:** With a non-executable stack, shellcode injection fails. Instead, chain together existing executable code ("gadgets") from `libc` to call `system("/bin/sh")`.

**Technique highlights:**
- Located `libc` base address via `info proc mappings` in GDB
- Used `ROPgadget` to find a `pop rdi ; ret` gadget for argument setup
- Located `/bin/sh` string within `libc` using `ROPgadget --string`
- Constructed ROP chain to call `system("/bin/sh")` with `exit()` as cleanup

**ROP chain layout:**
```
Low Memory
──────────────────────
| pop rdi ; ret       |  ← load RDI with "/bin/sh" address
| &"/bin/sh"          |  ← argument to system()
| system()            |  ← overwritten return address
| exit()              |  ← prevent crash after shell exits
──────────────────────
High Memory
```

**Key commands used:**
```bash
# Find libc base
(gdb) info proc mappings

# Find useful gadgets
ROPgadget --binary /lib/x86_64-linux-gnu/libc.so.6 --offset <base_addr> --nojop

# Find /bin/sh string
ROPgadget --binary /lib/x86_64-linux-gnu/libc.so.6 --offset <base_addr> --string "/bin/sh"
```

> See [`part2_rop/rop_chain.md`](part2_rop/rop_chain.md) for the full chain construction and debugging process.

---

## Part 3 — Exploit Automation (pwntools)

**Concept:** Automate offset discovery and payload delivery to reduce manual iteration.

- Scripted payload generation and process interaction with `pwntools`
- Automated retry loop to handle address variability during development
- Clean abstraction for reuse across different offset guesses

```python
from pwn import *

# Example structure
p = process(['./bin/hw1_target'])
payload = b'\x90' * NOP_SIZE + shellcode + b'A' * PADDING + p64(RET_ADDR)
p.sendline(payload)
p.interactive()
```

> Full script: [`part3_automation/auto_exploit.py`](part3_automation/auto_exploit.py)

---

## Part 4 — Root Shell via Custom Shellcode

**Concept:** The default shell spawned by Part 1 drops privileges. To retain root, the shellcode must explicitly call `setreuid(0, 0)` before `execve`.

- Wrote custom x86-64 assembly shellcode
- Used `setreuid` syscall (syscall number `0x71`) to set both real and effective UID to 0
- Verified root privilege with `id` after exploitation

> Assembly source: [`part4_root_shell/shellcode.asm`](part4_root_shell/shellcode.asm)

---

## Part 5 — ROP-Based execve Syscall

**Concept:** Rather than calling the libc `system()` wrapper, invoke the `execve` syscall directly using a ROP chain — no libc function calls required.

- Constructed a full ROP chain to set `rax`, `rdi`, `rsi`, `rdx` for the `execve` syscall
- Located a `syscall ; ret` gadget in libc
- Placed `/bin/sh` string on the stack and passed its address via `rdi`

> Chain walkthrough: [`part5_rop_shellcode/notes.md`](part5_rop_shellcode/notes.md)

---

## Key Takeaways

| Attack | Countermeasure Bypassed | Core Technique |
|---|---|---|
| Part 1 | — | Shellcode injection + NOP sled |
| Part 2 | DEP / NX (non-executable stack) | Return-to-libc + ROP chain |
| Part 3 | — | Exploit automation (pwntools) |
| Part 4 | Privilege dropping on setuid | Custom shellcode with setreuid |
| Part 5 | libc function hooking / detection | Direct syscall via ROP |

---

## Disclaimer

This project was completed as part of a university course (CSCI 4250/6250 — Computer Security) in a controlled VM environment. All techniques were applied only to intentionally vulnerable binaries provided for educational purposes. **Do not use these techniques against systems you do not own or have explicit permission to test.**

