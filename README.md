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

## 🛠️ Tools Used

`netdiscover` · `nmap` · `ffuf` · `ssh2john` · `john` · Base58 decoder · `ssh`

---

##  Findings Summary

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
