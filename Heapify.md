# Heapify – glibc 2.35 Heap Exploitation Writeup

> **Target:** `heapify`  
> **libc:** glibc 2.35  
> **Technique:** Tcache poisoning + smallbin abuse + oracle-based ASLR bypass  
> **Outcome:** Full RCE via GOT overwrite → one_gadget  

---

## 1. Overview

`heapify` is a menu-driven heap challenge exposing multiple subtle allocator behaviors:

- Priority values parsed via `scanf`
- Repeated allocate/free cycles
- User-controlled sizes and metadata adjacency
- No direct output of heap or libc pointers

Despite modern glibc mitigations:

- Safe-linking
- Tcache limits
- ASLR

The challenge is solvable by **turning the program itself into an oracle** that confirms whether a guessed pointer is correct.

This exploit performs:

1. **Oracle-based libc leak (smallbin)**
2. **Oracle-based heap leak (tcache fd)**
3. **Safe-linking bypass**
4. **Tcache poisoning**
5. **GOT overwrite**
6. **one_gadget execution**

---

## 2. Binary & Environment

```bash
Arch: amd64
PIE: No
RELRO: Partial
NX: Enabled
Canary: No
libc: 2.35
```

Local execution uses the provided loader:

```bash
LD_PRELOAD=./libc.so.6 ./heapify
```

---

## 3. Primitive Summary

| Primitive | Description |
|---------|-------------|
| Oracle | Program prints success only if internal pointer is valid |
| Smallbin Leak | Indirect libc pointer recovery |
| Tcache Leak | Safe-linked heap pointer |
| Poison | Controlled fd overwrite |
| Write | Arbitrary GOT overwrite |
| Exec | one_gadget trigger |

---

## 4. Oracle Leak Technique

The core trick:

- The program **compares priority values**
- A correct pointer guess results in:
```
Congratulations, here's the flag
```

This allows nibble-by-nibble brute forcing of addresses.

### Key Insight

Each failed comparison tells us whether the guessed pointer is **above or below** the real value.

We exploit this to reconstruct:

- `main_arena` → libc base
- safe-linked `fd` → heap base

---

## 5. Libc Leak (Smallbin)

### Setup

1. Fill tcache (7 entries)
2. Free one more chunk → fastbin
3. Abuse `scanf` internal allocation
4. Force chunk into **unsorted → smallbin**
5. Oracle-bruteforce the leaked pointer

```python
leak = oracle_libc(io, 0x00007f0000000000)
libc.address = leak - 0x219d10
```

Offset `0x219d10` corresponds to `main_arena` for glibc 2.35.

---

## 6. Heap Leak (Safe-Linking)

Safe-linking encodes pointers as:

```
fd = ptr ^ (heap >> 12)
```

### Steps

1. Prepare controlled tcache state
2. Allocate without overwriting priority → fd leak
3. Oracle-bruteforce encoded fd
4. Deobfuscate pointer

```python
heap_base = deobfuscate(leak) - 0x6d0
```

Offset verified in GDB.

---

## 7. Safe-Linking Bypass

Helper:

```python
def obfuscate(ptr, heap):
    return ptr ^ (heap >> 12)
```

Used when crafting poisoned tcache entries.

---

## 8. Tcache Poisoning – Stage 1

Goal: Gain write to a libc GOT entry.

### Target

```python
__j_rawmemchr@GOT
```

### Method

1. Fake chunk planted via priority overwrite
2. Freed into tcache
3. Poison fd → GOT address
4. Allocate → controlled write

Payload:

```python
fake_chunk = p64(obfuscate(target, heap_base+0x1000))
```

---

## 9. Tcache Poisoning – Stage 2

Second overwrite used to stabilize execution and avoid crashes.

Targets used:

- `strlen@GOT`
- `mempcpy@GOT`
- `strcmp@GOT`

Final jump lands in:

```python
one_gadget = libc.address + 0xebcf8
```

---

## 10. Trigger

Any libc call referencing overwritten GOT entries immediately transfers execution.

Shell obtained.

```bash
$ id
uid=1000 gid=1000 groups=1000
```

---

## 11. Why This Works

Despite glibc 2.35 protections:

- Oracle leaks defeat ASLR
- Safe-linking is reversible
- Tcache poisoning still viable
- Partial RELRO leaves GOT writable

This is a **logic flaw exploitation**, not a raw memory bug.

---

## 12. Files

- `solve.py` – Full exploit
- `heapify` – Challenge binary
- `libc.so.6` – glibc 2.35
- `ld-2.35.so` – Loader

---

## 13. Final Notes

This exploit is:

- Fully deterministic
- GDB-verified offsets
- No brute-force beyond oracle logic
- Works local & remote

Modern heap exploitation is about **state manipulation**, not crashes.

---

🧠 **Exploit author:** root  
