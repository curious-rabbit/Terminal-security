# Terminal-security
Handling dangerous sequences in cli applications


# Terminal escape sequence injection

A guide for developers building terminal applications that display untrusted content.


## What this document covers

If your program displays text in a terminal — filenames, file previews, log output, network data — and that text comes from a
source you don't fully control, you have a security problem. The text can contain invisible instructions that the terminal exe
cutes silently. This document explains what those instructions are, why they're dangerous, and how to filter them.


## Background: how terminals process text

A terminal emulator (xterm, iTerm2, Windows Terminal, etc.) does not just display the bytes you send it. It interprets certain
 byte sequences as commands. These commands can change text color, move the cursor, clear the screen, and more. They are calle
d **escape sequences** because most of them begin with the ESC byte (0x1B, also written as `\033` or `\x1b`).

When your program writes the string `\x1b[31m` to the terminal, no visible characters appear. Instead, the terminal switches t
o red text. Everything printed after that is red until a reset sequence (`\x1b[0m`) is sent. This is how programs like `ls --c
olor`, `bat`, and `grep --color` produce colored output.

The problem: if an attacker controls the bytes your program displays, they control what commands the terminal executes.


## The threat model

An attacker can embed escape sequences in any data your program reads and displays:

- **Filenames.** On Linux, a filename can contain any byte except `/` and the null byte. A file named `\x1b]52;c;Y3VybCBldmlsL
mNvbS9zaGVsbC5zaHxzaA==\x07` is a valid filename. When a file manager displays it, the terminal sees a clipboard-write command
.

- **File content.** A text file, source code file, or config file can contain escape sequences. When a previewer or pager read
s and displays the file, the sequences reach the terminal.

- **Data from external programs.** A syntax highlighter or previewer reads a file, adds color sequences, and writes the result
 to stdout. If the input file contains malicious sequences, they may pass through and reach the terminal.

- **Log files.** Web servers, application logs, and system logs may contain user-supplied strings (usernames, request paths, f
orm fields). Viewing logs in a terminal can trigger embedded sequences.

- **Network data.** Chat messages, email headers, DNS records, HTTP responses — any data received over a network and displayed
 in a terminal is a potential vector.

- **Symlink targets.** A symlink can point to a path containing escape sequences. Displaying the target path sends those seque
nces to the terminal.

- **Environment variables and command output.** Any string your program receives from an external source and prints without fi
ltering.

The attacker does not need shell access. They only need to get their content displayed in the victim's terminal — a shared fil
e, a downloaded archive, a git repository, a log entry, or a chat message.


## Control characters

Before we get to escape sequences, there is a simpler category: **control characters.** These are bytes in the range 0x00 thro
ugh 0x1F, plus 0x7F (DEL). They were originally designed to control teletypes and are not printable. Most are harmless today,
but several are still interpreted by terminal emulators.

Some notable control characters:

| Byte | Name | What the terminal does |
|------|------|----------------------|
| 0x07 | BEL | Produces an audible bell or visual flash |
| 0x08 | BS | Moves the cursor one position left (backspace) |
| 0x09 | TAB | Moves to the next tab stop |
| 0x0A | LF | Moves to the next line |
| 0x0D | CR | Moves the cursor to the start of the line |
| 0x0E | SO (Shift Out) | Switches to an alternate character set |
| 0x0F | SI (Shift In) | Switches back to the normal character set |
| 0x1B | ESC | Starts an escape sequence (see below) |
| 0x7F | DEL | Sometimes interpreted as backspace |

**CR (0x0D)** is dangerous because it returns the cursor to the start of the line without advancing to the next one. Text prin
ted after a CR overwrites what was already on the line. An attacker can make a string appear to say one thing while actually b
eing something else. For example, a log entry could show "login successful" on screen while the actual data says "login failed
" — the real text was overwritten by CR followed by the fake text.

**SO (0x0E)** switches the terminal to the "G1" character set, which typically maps ASCII characters to line-drawing symbols.
If a program sends SO without a matching SI, all subsequent output is garbled — letters appear as box-drawing characters. Some
 terminal libraries use SO/SI internally for drawing borders, and an unexpected SO from untrusted content corrupts their state
. This affects the entire terminal session until it is manually reset.

**ESC (0x1B)** is the most dangerous because it introduces escape sequences that can do almost anything.


## Escape sequences

An escape sequence starts with ESC (0x1B) followed by one or more bytes that tell the terminal what to do. There are several f
amilies.

        if byte[j] == 'm' or byte[j] == 'K':
            keep the sequence from i through j
        else:
            drop the sequence
        advance i to j
        break
    j++
```


### Where to filter

Filter as close to the data source as possible, not at the display layer. This prevents untrusted data from leaking through if the display path changes.

- **Filenames and paths:** Filter before passing to any display function. Every code path that displays a filename needs protection — directory listings, status bars, error messages, delete prompts, rename dialogs.

- **File content for preview:** Filter after reading the file, before storing or displaying the lines. If using an external previewer, filter the previewer's stdout before display.

- **Log entries:** Filter when reading from the log file, not when writing to the screen.

- **Network data:** Filter immediately after receiving, before any processing that might display it.

If filtering at every input boundary is impractical, a display-layer filter (in your print/render function) works as a safety net. But be aware that this mixes concerns — your display function must now understand security policy — and it may strip formatting that was intentionally added by your own program.


## C1 control characters

There is an additional set of control characters in the range U+0080 through U+009F, called C1 controls. In ISO 8859 encodings, these are single bytes (0x80-0x9F) and some terminals interpret them as escape sequence introducers. For example, 0x9B is the 8-bit form of CSI (`ESC [`).

In UTF-8 — which is used by virtually all modern terminals — these codepoints are encoded as two bytes (0xC2 0x80 through 0xC2 0x9F). Modern terminals running in UTF-8 mode do not interpret these as control characters. The terminal library (if any) decodes them as Unicode codepoints and displays replacement characters or nothing.

For most applications, C1 controls are not a practical attack vector. However, if your application might run on a terminal in 8-bit mode, or if you want defense in depth, you can filter them by checking for runes in the U+0080 to U+009F range.


## Common mistakes

**Filtering only CR and LF.** Many applications check for `\r` and `\n` but let everything else through, including ESC. This blocks line manipulation but leaves the application wide open to escape sequence injection.

**Relying on the terminal library to sanitize output.** Most terminal libraries (ncurses, tcell, crossterm, etc.) write cell contents to the terminal but do not filter escape sequences in the strings you give them. If you pass a string containing ESC to a cell-writing function, the ESC reaches the terminal.

**Trusting external program output.** A previewer or syntax highlighter is expected to add color codes, but it may also pass through escape sequences embedded in the file it processes. Filter the output even if you trust the program itself.

**Filtering ESC but not other control characters.** Blocking ESC prevents escape sequences but still allows SO (charset corruption), CR (line overwrite), BS (character erasure), and BEL (audible annoyance). Filter all control characters in the 0x00-0x1F range, not just ESC.

**Using a blocklist instead of an allowlist.** Listing specific dangerous sequences and blocking them is fragile — new sequences are added to terminals over time, and you will miss some. Instead, block everything by default and only allow the specific sequences you need.


## Summary of threats

| Threat | Attack mechanism | Severity |
|--------|-----------------|----------|
| Clipboard write (OSC 52) | Silent code execution via paste | Critical |
| Charset corruption (SO/SI, ESC charset) | All terminal output garbled | High |
| Terminal state query (CSI DSR, DCS DECRQSS) | Input injection via terminal response | High |
| Screen erasure (CSI ED) | Display wiped | Medium |
| Cursor movement (CSI CUP, etc.) | Display manipulation, content spoofing | Medium |
| Cursor hide (CSI DECTCEM) | User can't see cursor | Medium |
| Carriage return (CR) | Line overwrite, display spoofing | Medium |
| Window title change (OSC 0) | Social engineering | Low |
| Bell (BEL) | Audible annoyance | Low |
| Backspace (BS) | Minor display corruption | Low |


## Further reading

- ECMA-48 standard (5th edition) — the specification that defines C0/C1 control characters and escape sequences like CSI, OSC, and DCS: https://ecma-international.org/publications-and-standards/standards/ecma-48/
- XTerm Control Sequences — the most comprehensive reference for escape sequences supported by modern terminal emulators: https://invisible-island.net/xterm/ctlseqs/ctlseqs.html
- "Terminal Escape Injection" by InfosecMatter — practical demonstration of escape sequence attacks across platforms, with proof-of-concept code: https://www.infosecmatter.com/terminal-escape-injection/
- "Don't Trust This Title: Abusing Terminal Emulators with ANSI Escape Characters" by CyberArk — research on terminal emulator vulnerabilities including denial of service and UI spoofing via escape sequences: https://www.cyberark.com/resources/threat-research-blog/dont-trust-this-title-abusing-terminal-emulators-with-ansi-escape-characters
- "Weaponizing ANSI Escape Sequences" by Packet Labs — overview of the DEFCON 31 presentation by STÖK on weaponizing escape sequences in log files and terminal applications: https://www.packetlabs.net/posts/weaponizing-ansi-escape-sequences
- CVE-2021-25743 — ANSI escape character injection in Kubernetes, demonstrating the real-world impact of unsanitized terminal output in widely-used tools
- CVE-2022-30123 
