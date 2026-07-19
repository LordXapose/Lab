# Penetration Test Report — Empire: LupinOne

**Target:** Empire: LupinOne (VulnHub)
**Assessment type:** Black-box, boot-to-root CTF
**Tester:** Kenil
**Date:** 19 July 2026
**Classification:** Educational / Lab

---

## 1. Executive Summary

Empire: LupinOne is an intentionally vulnerable Debian 11 virtual machine. A full
black-box assessment was performed against it in an isolated host-only lab. Starting
from zero knowledge of the host, a complete compromise path to an interactive user
shell was achieved and a clear privilege-escalation vector to a second user was
identified.

The compromise did not rely on a single exotic vulnerability. It was the result of a
chain of small, realistic mistakes: an SSH private key left exposed on a web server,
"security through obscurity" (a hidden dot-file), a weak passphrase protecting that
key, and an overly permissive `sudo` rule. Any one of these fixed in isolation would
have broken the chain.

| Metric | Result |
|---|---|
| Initial foothold | ✅ SSH shell as `icex64` |
| User flag captured | ✅ `3mp!r3{I_See_That_You_Manage_To_Get_My_Bunny}` |
| Privilege-escalation vector | ✅ Identified (`sudo` to `arsene` via `python3.9 heist.py`) |
| Overall risk of the modelled configuration | **Critical** |

---

## 2. Scope & Environment

| Item | Value |
|---|---|
| Target hostname | `LupinOne` / `LupinOne.fritz.box` |
| Target IP (assessed session) | `192.168.178.44` |
| Attacker host | Kali Linux (`192.168.178.87`) |
| Network mode | VMware **host-only** (isolated, no internet exposure) |
| Rules of engagement | Single host, full boot-to-root, no constraints inside the lab |

Both machines were confirmed to be on an isolated host-only VMware segment before
testing, so no traffic left the lab environment.

![Target VM network configuration](images/01-target-vm-network.png)
![Kali VM network configuration](images/02-kali-vm-network.png)

---

## 3. Methodology

The engagement followed a standard offensive workflow:

1. **Reconnaissance** — host discovery and service enumeration.
2. **Web enumeration** — content discovery and analysis of exposed application content.
3. **Credential access** — recovery and cracking of an exposed secret.
4. **Initial access** — authenticated shell via the recovered credential.
5. **Post-exploitation** — local enumeration, flag capture, and privilege-escalation analysis.

Tooling: `netdiscover`, `nmap`, `ffuf`, `john` (with `ssh2john`), a Base58 decoder,
and `ssh`.

---

## 4. Findings Summary

| # | Finding | Severity |
|---|---|---|
| F-01 | SSH private key exposed on the public web server | **Critical** |
| F-02 | `sudo` misconfiguration — user script runnable as another user | **High** |
| F-03 | Weak passphrase on the SSH private key | **High** |
| F-04 | Sensitive content hidden by obscurity only (dot-file / naming) | **Medium** |
| F-05 | Information disclosure via `robots.txt` and directory naming | **Low** |
| F-06 | Verbose service/version banners | **Informational** |

---

## 5. Attack Narrative (Kill Chain)

### 5.1 Reconnaissance

The target was located on the local segment with `netdiscover`, then port-scanned.

![netdiscover host discovery](images/04-netdiscover.png)

A quick `nmap` scan revealed two open services:

```
nmap 192.168.178.44
22/tcp open  ssh
80/tcp open  http
```

![nmap quick scan](images/05-nmap-quick.png)

A full service/script scan (`nmap -sV -sS -sC`) provided versions and a first lead:

```
22/tcp open  ssh   OpenSSH 8.4p1 Debian 5 (protocol 2.0)
80/tcp open  http  Apache httpd 2.4.48 ((Debian))
|_http-title: Site doesn't have a title (text/html).
| http-robots.txt: 1 disallowed entry
|_/~myfiles
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

![nmap full scan with robots.txt entry](images/06-nmap-full.png)

Key takeaways: a modern OpenSSH and Apache 2.4.48, and a `robots.txt` pointing at
`/~myfiles`. The `~` prefix strongly suggests Apache *userdir*-style paths.

### 5.2 Web Enumeration

The web root served only an "Arsène Lupin — The Gentleman Thief" graphic, with no
links or obvious functionality.

![Web root page](images/07-web-root.png)

`robots.txt` confirmed the single disallowed entry:

```
User-agent: *
Disallow: /~myfiles
```

![robots.txt contents](images/08-robots-txt.png)

`/~myfiles` itself returned **Error 404** — a decoy / rabbit hole rather than a real
path. This is a classic misdirection: the disallowed entry is not the interesting one.

![/~myfiles returns 404](images/09-myfiles-404.png)

Because the `~` pattern was confirmed, content discovery was pointed at
`http://192.168.178.44/~FUZZ`, filtering out `403` noise:

```
ffuf -u http://192.168.178.44/~FUZZ \
     -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt \
     -c -ic -fc 403
```

![ffuf directory fuzzing setup](images/10-ffuf-dirs-setup.png)

This uncovered a live directory:

```
secret   [Status: 301, Size: 318, Words: 20, Lines: 10]
```

![ffuf discovers ~secret](images/11-ffuf-secret-found.png)

### 5.3 Sensitive File Discovery

`/~secret/` contained a note written in-character by the box author, effectively a
hint that an SSH private key was hidden nearby as a dot-file:

> "Hello Friend, Im happy that you found my secret diretory, I created like this to
> share with you my create ssh private key file, Its hided somewhere here, so that
> hackers dont find it and crack my passphrase with fasttrack. I'm smart I know that.
> — Your best friend icex64"

![~secret directory note](images/12-secret-directory.png)

Two things were leaked here: (1) a key is present as a **hidden file** in this
directory, and (2) the author explicitly references cracking with **fasttrack** — a
direct hint at the passphrase wordlist.

A second fuzz was run for hidden files, this time appending a leading dot and
`.txt`/`.html` extensions:

```
ffuf -u http://192.168.178.44/~secret/.FUZZ \
     -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt \
     -c -ic -fc 403 \
     -e .txt,.html
```

![ffuf hidden-file fuzzing setup](images/13-ffuf-files-setup.png)

This revealed the hidden file:

```
mysecret.txt   [Status: 200, Size: 4689, Words: 1, Lines: 2]
```

i.e. `http://192.168.178.44/~secret/.mysecret.txt`

![ffuf discovers .mysecret.txt](images/14-ffuf-mysecret-found.png)

### 5.4 Credential Access

The file contained a single long, high-entropy string with no line breaks —
characteristic of an encoded blob rather than a raw key.

![Encoded blob in .mysecret.txt](images/15-base58-blob.png)

The character set (no `0`, `O`, `I`, `l`; mixed case and digits) matched **Base58**.
Decoding it produced a valid OpenSSH private key (`-----BEGIN/END OPENSSH PRIVATE KEY-----`).

![Base58 decode reveals OpenSSH private key](images/16-base58-decoded-key.png)

The decoded key was saved as `id_rsa`. It was passphrase-protected, so it was
converted to a John-compatible hash and cracked using the author-hinted `fasttrack`
wordlist:

```
ssh2john id_rsa > hash.txt
john --wordlist=/usr/share/wordlists/fasttrack.txt hash.txt
```

![ssh2john conversion](images/17-ssh2john.png)
![john invocation](images/18-john-setup.png)

The first attempt hit `Permission denied` on the wordlist path and was re-run with
`sudo`:

![john permission denied, re-run with sudo](images/19-john-permission.png)

John recovered the passphrase almost instantly (single-hash, tiny wordlist):

```
P@55w0rd!   (id_rsa)
1g 0:00:00:01 DONE ... Session completed.
```

![john cracks the passphrase](images/20-john-cracked.png)

### 5.5 Initial Access

With the key and its passphrase, the note's author username (`icex64`) was the obvious
login target. The first attempt was rejected for **bad key permissions**, corrected
with `chmod 600`, and then succeeded:

```
ssh -i id_rsa icex64@192.168.178.44        # rejected: bad permissions
chmod 600 id_rsa
ssh -i id_rsa icex64@192.168.178.44        # passphrase: P@55w0rd!
```

![ssh key login attempt](images/21-ssh-key-attempt.png)
![chmod 600 then successful SSH login as icex64](images/22-ssh-success.png)

```
Welcome to Empire: Lupin One
icex64@LupinOne:~$
```

Foothold established as **`icex64`** on Debian 5.10.0-8-amd64.

### 5.6 Post-Exploitation

**User flag.** The user flag was read from the home directory:

```
icex64@LupinOne:~$ cat user.txt
3mp!r3{I_See_That_You_Manage_To_Get_My_Bunny}
```

![user.txt flag capture](images/24-user-flag.png)

**Privilege-escalation vector.** `sudo -l` revealed that `icex64` may run a specific
Python script as the user `arsene` with no password:

```
User icex64 may run the following commands on LupinOne:
    (arsene) NOPASSWD: /usr/bin/python3.9 /home/arsene/heist.py
```

![sudo -l privilege escalation vector](images/23-sudo-l.png)

This is a textbook lateral-movement / privilege-escalation path. Because the script
runs under Python and the invocation is fixed, escalation to `arsene` is possible if
`heist.py` (or any module it imports from a writable location) can be influenced — for
example via a hijackable/writable import or an insecure call inside the script.
Escalating to `arsene` is the intended next stage on the path toward `root`.

---

## 6. Detailed Findings & Remediation

### F-01 — SSH private key exposed on the public web server *(Critical)*

**Description.** A working OpenSSH private key for a real system account was
retrievable over unauthenticated HTTP (Base58-encoded inside
`/~secret/.mysecret.txt`). Anyone who discovers the file obtains reusable credential
material for the host.

**Impact.** Direct path to an interactive shell. Encoding (Base58) provides no
protection — it is trivially reversible.

**Remediation.**
- Never store private keys within a web-served directory tree.
- Move key material outside the document root; serve web content from a dedicated,
  minimal directory.
- Treat encoding as *not* a security control; only encryption with a strong secret
  qualifies.
- Rotate any key that has been web-exposed.

### F-02 — `sudo` misconfiguration to another user *(High)*

**Description.** `icex64` can execute `/usr/bin/python3.9 /home/arsene/heist.py` as
`arsene` with `NOPASSWD`. If the script or its import path is writable/influenceable,
this becomes arbitrary code execution as `arsene`.

**Remediation.**
- Remove the rule unless strictly required.
- If a delegated command is unavoidable, ensure the script and every module it can
  import are owned by the target user and not writable by the invoking user.
- Avoid interpreter-based `sudo` entries (`python`, `perl`, `bash`, `find`, etc.) —
  interpreters are effectively wildcards for arbitrary execution.

### F-03 — Weak passphrase on the private key *(High)*

**Description.** The key's passphrase (`P@55w0rd!`) was recovered in ~1 second using a
small, well-known wordlist (`fasttrack.txt`).

**Remediation.** Enforce long, random passphrases (a generated passphrase or
manager-stored secret). A strong passphrase would have blunted F-01 by keeping the key
unusable even after exposure.

### F-04 — Security through obscurity *(Medium)*

**Description.** The key was "protected" only by being a hidden dot-file in a
non-linked directory. Directory/file brute-forcing defeats this immediately.

**Remediation.** Do not rely on unguessable names. Apply real access controls
(authentication, authorization, file-system permissions).

### F-05 — Information disclosure via `robots.txt` / naming *(Low)*

**Description.** `robots.txt` and the consistent `~`-prefixed naming scheme guided
enumeration. Even the decoy narrowed the search space.

**Remediation.** Keep sensitive paths out of `robots.txt` (it is a public file);
control access rather than advertising and then "disallowing" it.

### F-06 — Verbose service banners *(Informational)*

**Description.** `nmap` recovered exact OpenSSH/Apache versions and OS fingerprint,
aiding targeted research.

**Remediation.** Suppress version tokens where practical (`ServerTokens Prod`,
`ServerSignature Off`); keep services patched so version disclosure carries less risk.

---

## 7. Recommendations (Prioritised)

1. **Remove the web-exposed private key immediately and rotate it.** (F-01)
2. **Delete or tightly constrain the `sudo`/`heist.py` rule; avoid interpreter
   entries.** (F-02)
3. **Enforce strong passphrases and key-management hygiene.** (F-03)
4. **Replace obscurity with real access control on sensitive files.** (F-04)
5. **Minimise information disclosure** in `robots.txt` and banners. (F-05, F-06)

**Defence-in-depth note:** this chain only worked because *every* link held. Fixing
any single item — the exposed key, the weak passphrase, or the `sudo` rule — would
have stopped a full compromise. That is the core lesson of the box.

---

## 8. Appendix A — Commands Used

```bash
# Recon
sudo netdiscover
nmap 192.168.178.44
nmap -sV -sS -sC 192.168.178.44

# Web content discovery
ffuf -u http://192.168.178.44/~FUZZ \
     -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt -c -ic -fc 403

ffuf -u http://192.168.178.44/~secret/.FUZZ \
     -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt -c -ic -fc 403 \
     -e .txt,.html

# Retrieve & decode the key
# GET http://192.168.178.44/~secret/.mysecret.txt  -> Base58 decode -> id_rsa

# Crack the passphrase
ssh2john id_rsa > hash.txt
sudo john --wordlist=/usr/share/wordlists/fasttrack.txt hash.txt   # -> P@55w0rd!

# Initial access
chmod 600 id_rsa
ssh -i id_rsa icex64@192.168.178.44                                # passphrase: P@55w0rd!

# Post-exploitation
cat user.txt
sudo -l
```

## 9. Appendix B — Loot & Artifacts

| Artifact | Value |
|---|---|
| Web path (decoy) | `/~myfiles` (404) |
| Web path (real) | `/~secret/` |
| Exposed key file | `/~secret/.mysecret.txt` (Base58 → OpenSSH key) |
| Key passphrase | `P@55w0rd!` |
| Foothold user | `icex64` |
| User flag | `3mp!r3{I_See_That_You_Manage_To_Get_My_Bunny}` |
| Next target user | `arsene` (via `sudo python3.9 heist.py`) |

---

*Prepared for educational purposes against an intentionally vulnerable VulnHub
machine in an isolated lab. No production systems were involved.*
