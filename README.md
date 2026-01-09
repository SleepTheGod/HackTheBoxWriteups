# 🧠 Hack The Box Writeups

> **Author:** ClumsyLulz  
> **Platform:** Hack The Box  
> **Repo:** SleepTheGod/HackTheBoxWriteups  
> **Purpose:** High‑quality technical writeups for retired HTB challenges and machines

---

## 📌 About

This repository contains technical walkthroughs and deep‑dive explanations for challenges and machines from **Hack The Box**.

Each writeup focuses on **understanding the vulnerability**, exploitation methodology, and allocator / protocol behavior — not just reaching a shell.

Content includes
- Binary exploitation
- Heap exploitation
- glibc internals
- Reverse engineering
- Web & misc challenges

---

## 📁 Repository Structure

```
HackTheBoxWriteups/
├── Blinded_Heap_Exploit_Writeup.md
├── Heapify.md
├── Guardian.md
├── SatelliteHijack.md
└── ...
```

Each `.md` file corresponds to a single Hack The Box challenge or machine writeup.

---

## 🧪 Tooling

Common tools used throughout the writeups:

- pwntools
- GDB + pwndbg
- objdump / readelf
- custom glibc & ld loaders
- tmux + gdb automation

Example:
```bash
LD_PRELOAD=./libc.so.6 ./binary
python3 solve.py
```

---

## 🎯 Contribution

Want to contribute?

- Follow existing markdown structure
- Explain *why* the exploit works
- Include allocator diagrams or pseudocode if helpful
- Do **not** include active HTB content

Pull requests are welcome.

---

## ⚠️ Disclaimer

All material is provided for **educational purposes only**.  
Only test techniques on systems you own or have permission to assess.

---

## 🧑‍💻 Author

**ClumsyLulz**

GitHub: https://github.com/SleepTheGod
