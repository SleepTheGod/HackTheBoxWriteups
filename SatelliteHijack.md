---

Satellite Hijack - HTB CTF Challenge Writeup
Satellite Hijack - HTB CTF Challenge Writeup
Challenge Name: Satellite Hijack
 Category: Reverse Engineering / Binary Exploitation
 Platform: Debian 12
Writeup By : Taylor Christian Newsome | ClumsyLulz

---

Challenge Overview
The challenge scenario is straight out of a dystopian pre-war setting:
The crew has located a dilapidated pre-war bunker. Deep within, a dusty control panel reveals that it was once used for communication with a low-orbit observation satellite. During the war, actors on all sides infiltrated and hacked each other's systems, inserting backdoors to cripple or take control of critical machinery. The control panel has been tampered with to prevent the transmission of control codes - your task is to recover the codes and regain control of the satellite to locate enemy factions.
At first glance, the challenge seems like a simple binary execution, but deeper inspection shows that hidden code paths and tampered libraries are blocking normal operations.

---

Reconnaissance
I started by inspecting the files provided
root@clumsy:~/htb/ctf/rev_satellitehijack# ls
satellite  library.so 
The satellite binary is the main executable, while library.so appears to be a tampered shared library. The supplied script I madetest.sh automates some of the exploitation steps.
#!/bin/bash
# SatelliteHijack Automation Script for Debian 12
# Challenge Author: clubby789
# Exploit Author: ClumsyLulz | Taylor Christian Newsome
# Usage: ./satellite_hijack.sh

set -euo pipefail

# Check dependencies
for cmd in ldd nm python3; do
    if ! command -v $cmd &>/dev/null; then
        echo "[!] Missing dependency: $cmd"
        echo "Install with: sudo apt install $cmd -y"
        exit 1
    fi
done

BINARY="./satellite"
LIBRARY="./library.so"

# Verify binary and library exist
if [[ ! -f "$BINARY" || ! -f "$LIBRARY" ]]; then
    echo "[!] Binary or library not found in current directory"
    exit 1
fi

echo "[*] Checking linked libraries..."
ldd "$BINARY"

# Set environment variable to enable hidden code path
ENV_NAME=$(python3 - <<'EOF'
s = b"TBU`QSPE`FOWJSPOSPONFOU"
print("".join([chr(x-1) for x in s]))
EOF
)
export "$ENV_NAME"=1
echo "[*] Exported $ENV_NAME=1 to trigger hidden satellite function"

# Launch binary in interactive mode
echo "[*] Starting satellite binary. Type your messages:"
"$BINARY"

# Optional: Decode known XORed flag using Python
decode_flag() {
python3 - <<'EOF'
buf = bytearray(b"l5{0v0Y7fVf?u>|:O!|Lx!o$j,;f")
for i in range(len(buf)):
    buf[i] ^= i
print("Decoded flag:", (b"HTB{" + buf).decode())
EOF
}

read -p "[*] Do you want to decode the embedded flag automatically? (y/n) " yn
if [[ "$yn" =~ ^[Yy]$ ]]; then
    decode_flag
fi

echo "[*] SatelliteHijack setup complete."
Step 1: Dependency Check
The script begins by verifying required dependencies
for cmd in ldd nm python3; do
    command -v $cmd &>/dev/null || echo "Install $cmd"
done
This ensures that ldd (for linked libraries), nm (symbol inspection), and python3 are available.
Step 2: Inspecting Linked Libraries
We use ldd to check which shared libraries are loaded by the binary
[*] Checking linked libraries...
linux-vdso.so.1
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
./library.so
Notice that library.so is injected alongside standard system libraries. This is a strong hint that the binary's behavior is modified by the custom library.
Step 3: Triggering Hidden Functionality
The script contains a clever Python snippet that decodes an environment variable name
s = b"TBU`QSPE`FOWJSPOSPONFOU"
print("".join([chr(x-1) for x in s]))
This decodes to SAT_PROD_ENVIRONRONMENT
By exporting this variable before running the binary, we enable a hidden satellite control function
export SAT_PROD_ENVIRONRONMENT=1
This is a classic technique in CTF reverse engineering backdoors are often gated behind environment variables or command-line flags.
Step 4: Running the Binary
Once the environment variable is set, the binary launches an interactive console simulating satellite communication
| READY TO TRANSMIT |
> hello from clumsy
Sending `hello from clumsy`
We can send messages, but the real goal is to extract the embedded flag.
Step 5: Decoding the Flag
The challenge provided a byte sequence that is XORed with its index. We can decode it using Python
buf = bytearray(b"l5{0v0Y7fVf?u>|:O!|Lx!o$j,;f")
for i in range(len(buf)):
    buf[i] ^= i
print("Decoded flag:", (b"HTB{" + buf).decode())
Result
