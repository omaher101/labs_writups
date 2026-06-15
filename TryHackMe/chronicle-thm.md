# Chronicle - TryHackMe Writeup

## Overview
- **Platform:** TryHackMe
- **Challenge:** Chronicle
- **Category:** Web, Enumeration, Binary Exploitation
- **Difficulty:** Medium

## Tools Used
- nmap, ffuf
- git, scp
- firefox_decrypt
- GDB + pwndbg
- pwntools, ROPgadget

## What I Learned
1. Fuzz everything — hidden directories can reveal gold
2. `git log` to enumerate commit history for secrets
3. `scp` to transfer files between machines
4. `.mozilla` folders can contain sensitive credentials
5. Buffer overflow → ret2libc exploitation

---

## Step 1 — Enumeration
Found two ports:
- Port **80** — old website
- Port **8081** — new website

The new site had a password retrieval feature but required an API key.

Fuzzing port 80 revealed a hidden **`.git`** directory.

## Step 2 — Git Enumeration
```bash
git log        # view commit history
git show <id>  # reveal hidden API key in old commit
```
Found a hardcoded API key in a previous commit.

## Step 3 — Credential Discovery
Used the username + API key against the endpoint and retrieved credentials for user **tommy**.

Fuzzing for more usernames with ffuf revealed additional users.

## Step 4 — Initial Access
SSH login with tommy's credentials:
```bash
ssh tommy@<ip>
```
Found **user.txt** — first flag!

## Step 5 — Lateral Movement
Found another user **carlJ** with a `.mozilla` folder containing Firefox profile data.

Transferred the Firefox profile to local machine:
```bash
scp -r tommy@<ip>:/home/carlJ/.mozilla ./mozilla
```

Used `firefox_decrypt` to extract credentials:
```bash
python3 firefox_decrypt.py ./mozilla/firefox/<profile>
# Option 2 requires a master password
```

Automated the password cracking with a Python script using rockyou.txt wordlist. Found carlJ's password!

## Step 6 — Privilege Escalation via Buffer Overflow
SSH'd as carlJ and found a SUID binary `smail` in the mailing folder.

### Binary Analysis
