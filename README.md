# 🔎 What Is Your PC Connecting To? Spotting Hidden Connections and Remote Access Trojans

![Windows 10 | 11](https://img.shields.io/badge/Windows-10%20%7C%2011-0078D4?logo=windows&logoColor=white)
![Type: security guide](https://img.shields.io/badge/Type-security%20guide-d73a49)
![Tools: ipconfig, netstat, TCPView](https://img.shields.io/badge/Tools-ipconfig%20%7C%20netstat%20%7C%20TCPView-5c2d91)
![Checked November 2026](https://img.shields.io/badge/Checked-Nov%202026-2ea44f)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey)](LICENSE)

**📄 [Download the PDF](PC-Network-Activity-and-RAT-Check.pdf)**

## 📖 Overview
Modern malware rarely announces itself. There are no pop-ups or error messages, just a program quietly talking to a server somewhere. The most reliable clue is **network activity**: what your PC looks up, and what it connects to.

This guide shows how to check both with tools already in Windows, plus Microsoft's free TCPView. It explains how remote access trojans (RATs) hide behind normal Windows programs, how to look up an address you don't recognize, and how to make script-based attacks harder.

It's built on two videos from **The PC Security Channel**, with supplemental research where the videos leave gaps or need correcting:
- [Your computer is logging this!](https://www.youtube.com/watch?v=IX4q8dX3jbU) (April 2025)
- [How to know if your PC is hacked? Modern RATs](https://www.youtube.com/watch?v=B8ayJPq26iU) (May 2025)

## ⚠️ Important Notes
* **A connection isn't proof of infection.** Most of what you'll see is ordinary: updates, sync, analytics, and Windows itself. Use what you find as a reason to look closer, not to delete things.
* **Don't end Windows processes or delete files based on this alone.** Malware often runs inside real Windows programs, so stopping one can break your PC without removing the problem.
* **Don't flush the DNS cache until you've looked at it.** Flushing erases the evidence you're trying to read.
* **If you think you're infected,** disconnect from the internet and follow the [Windows Malware Removal Guide](https://github.com/ImmuneMoon/Windows-Malware-Removal-Guide).

## 🚀 How to Use
Download the PDF above, or read on below. Start with **The Short Version**, then work through Sections 2 to 4. Section 5 explains what a RAT looks like, and Section 7 covers hardening.

This guide works alongside:
* [Windows Malware Removal Guide](https://github.com/ImmuneMoon/Windows-Malware-Removal-Guide) - What to do if you find something.
* [DNS Rebinding and the NVIDIA App's Local MCP Server](https://github.com/ImmuneMoon/NVIDIA-App-MCP-DNS-Rebinding-Writeup) - Why local services listening on `127.0.0.1` can still matter.
* [Windows 10 Community Support Guide](https://github.com/ImmuneMoon/Windows-10-Community-Support-Guide#autoruns) - How to use Autoruns to see what starts with Windows.

---

<!-- guide:start -->

*Checked against Microsoft's documentation in November 2026. Each section says whether it comes from the videos or from added research.*

**Sources:** [Your computer is logging this!](https://www.youtube.com/watch?v=IX4q8dX3jbU) and [How to know if your PC is hacked? Modern RATs](https://www.youtube.com/watch?v=B8ayJPq26iU), both by The PC Security Channel. Sections marked **Added** come from other research, listed in Sources.

## The Short Version

- **What your PC looked up:** run `ipconfig /displaydns` to see recently resolved web addresses (Section 2).
- **What it's connected to right now, and which program:** use `netstat -ano`, PowerShell, Resource Monitor or TCPView (Section 3).
- **An address you don't recognize:** look it up on VirusTotal and with a whois search before drawing conclusions (Section 4).
- **What a RAT looks like:** a normal Windows program, such as PowerShell, running a scrambled command and talking to an unfamiliar server (Section 5).
- **Hardening:** Microsoft's attack surface reduction rules block the script tricks RATs rely on. PowerShell's execution policy doesn't (Section 7).

## Contents

1. [Why Network Activity Is the Best Clue](#1-why-network-activity-is-the-best-clue)
2. [Check What Your PC Has Looked Up](#2-check-what-your-pc-has-looked-up)
3. [See Live Connections and Which Program Owns Them](#3-see-live-connections-and-which-program-owns-them)
4. [Look Up an Address You Don't Recognize](#4-look-up-an-address-you-dont-recognize)
5. [How a Modern RAT Hides](#5-how-a-modern-rat-hides)
6. [Signs Worth a Closer Look](#6-signs-worth-a-closer-look)
7. [Make Script Attacks Harder](#7-make-script-attacks-harder)
8. [If You Find Something](#8-if-you-find-something)
9. [Sources](#9-sources)

---

## 1. Why Network Activity Is the Best Clue

*From the "Modern RATs" video.*

A **remote access trojan (RAT)** gives an attacker control of your PC. Typical abilities include recording keystrokes, searching your files, using your microphone and camera, and sending what it collects back to the attacker. That last step is called **exfiltration**.

To do any of this, the RAT has to talk to the attacker's server, called a **command and control (C2)** server. Malware can disguise its files and processes, but it can't avoid making that connection. So the first thing to check is where your PC is connecting.

---

## 2. Check What Your PC Has Looked Up

*From the "Your computer is logging this!" video, with added limits from Microsoft's documentation.*

Before your PC connects to a website or server, it looks up the address in **DNS**. Windows keeps recent answers in a **DNS cache**, and you can read it.

### 2.1 Show the cache

Open the Start menu, type `cmd`, and open **Command Prompt**. Then run:

```cmd
ipconfig /displaydns
```

The list is long. To save it to a file on your desktop instead:

```cmd
ipconfig /displaydns > "%USERPROFILE%\Desktop\dns-cache.txt"
```

In PowerShell, this gives a shorter list with one line per name:

```powershell
Get-DnsClientCache | Sort-Object Entry -Unique | Format-Table Entry, Data
```

### 2.2 What you'll see

In the video, the list was mostly Microsoft (Windows, Office, OneDrive, Teams and Windows Update), plus Google and Apple services. There were game launchers, and update checks from hardware makers such as Dell and NVIDIA. Some surprises were:

- **Analytics services,** such as one used by installed software to report usage.
- **Software the presenter hadn't opened in a long time,** still checking in on its own.
- **Updaters installed alongside hardware,** like a monitor's companion software.

That's the useful part: the list shows which software is quietly using the internet. Most of it is harmless, such as updates and sync, but it's a quick way to spot software you didn't know was installed. It can also catch an **infostealer**, a type of malware that sends your saved passwords to an attacker.

### 2.3 What the cache can't tell you

*Added research.*

- **It's a snapshot.** Entries expire after a short time, so the cache only shows recent lookups.
- **It doesn't say which program made a lookup.** Section 3 covers that.
- **It includes your hosts file.** Microsoft notes the cache also holds entries loaded from the local `hosts` file, so not every line is a real lookup.
- **Browsers with secure DNS may skip it.** The video says your browser and private mode don't matter. That's true when the browser uses Windows' own lookup. But Chrome turns on **secure DNS** in automatic mode by default, and it can then do encrypted lookups itself, with its own cache. Those may not appear here. VPNs can also change this.

### 2.4 Clearing it

The video shows that `ipconfig /flushdns` empties the cache. Microsoft describes it as a troubleshooting step. Only use it **after** you've looked, because it removes exactly what you're trying to check.

---

## 3. See Live Connections and Which Program Owns Them

*TCPView is from the "Modern RATs" video. The other methods are added research.*

The DNS cache shows names. These tools show **actual connections**, and the program behind each one.

### 3.1 Command Prompt: netstat

```cmd
netstat -ano
```

| Option | What it adds |
|---|---|
| `-a` | All connections, and ports your PC is listening on |
| `-n` | Numbers instead of names (faster) |
| `-o` | The **PID** (process ID) of the program that owns each connection |

To find the program for a PID, open Task Manager (`Ctrl + Shift + Esc`), go to the **Details** tab, and look it up in the **PID** column.

### 3.2 PowerShell: connections with program names

This lists established connections with the program's name filled in:

```powershell
Get-NetTCPConnection -State Established |
  Select-Object RemoteAddress, RemotePort,
    @{n='Process'; e={(Get-Process -Id $_.OwningProcess).ProcessName}} |
  Sort-Object Process | Format-Table -AutoSize
```

Change `Established` to `Listen` to see which programs are waiting for incoming connections.

### 3.3 Resource Monitor

Press `Windows Key + R`, type `resmon`, and press **Enter**. On the **Network** tab, **TCP Connections** lists each program, the remote address, and the port, and it updates live.

### 3.4 TCPView

TCPView is a free Microsoft tool from the **Sysinternals** suite, and it's the one the video uses.

1. Download [TCPView](https://learn.microsoft.com/en-us/sysinternals/downloads/tcpview) from Microsoft and extract the zip file.
2. Open `tcpview64.exe`.
3. Each row shows the program, its local and remote addresses, and the connection's state. New connections are highlighted as they appear.

In the video, right-clicking a row gives options to copy the address and to run a **whois** lookup on it.

> **Seeing connections to `127.0.0.1` or `::1`?** Those stay on your own PC, between programs on it. They're normal, but local services can still be a security concern. See [DNS Rebinding and the NVIDIA App's Local MCP Server](https://github.com/ImmuneMoon/NVIDIA-App-MCP-DNS-Rebinding-Writeup).

---

## 4. Look Up an Address You Don't Recognize

*From the "Modern RATs" video, with added notes.*

1. **Copy the remote address,** an IP address or domain name, from TCPView or the list above.
2. **Search it on [VirusTotal](https://www.virustotal.com/).** Paste it into the search box to see whether security vendors have flagged it.
3. **Check who owns it.** A whois search, from TCPView's right-click menu or any whois website, shows the organization behind a domain or IP range. As the video notes, it often won't give a clear answer.
4. **Search the domain name** in a search engine, as the video does with an analytics domain. That's often the quickest way to learn what a service is.

How to read the results:

- **Big cloud and content-delivery networks host thousands of sites on shared addresses.** An Amazon, Microsoft or Cloudflare address tells you little on its own.
- **A clean VirusTotal result doesn't prove an address is safe.** New attacker servers often aren't flagged yet.
- **Judge the program and the address together.** An unknown address used by your game launcher is far less worrying than one used by PowerShell.

---

## 5. How a Modern RAT Hides

*From the "Modern RATs" video, with added Microsoft details.*

The video runs a real RAT in a safe test environment. What it shows matters more than any single tool:

1. **No strange program names.** The process list showed only Windows' own tools: PowerShell, Command Prompt, and `wscript.exe` (Windows Script Host).
2. **A scrambled command.** PowerShell was running a long string of letters and numbers. It was **Base64**, a way of encoding text that hides it from a quick look.
3. **Decoding it showed the plan.** The command downloaded a JavaScript file into the public Downloads folder and started it, and that file was the RAT.
4. **The network gave it away.** The RAT connected to a C2 server, and the connection showed the attacker's IP address and port.

This is called **living off the land**: attackers use Windows' own tools so nothing looks out of place. As the video stresses, the program's name alone tells you nothing. The same system program can be doing normal work in one case and fetching malware in another.

### 5.1 See what a program is actually running

*Added research.*

Task Manager can show each program's full command:

1. Open Task Manager and go to the **Details** tab.
2. Right-click a column header and choose **Select columns**.
3. Tick **Command line** and click **OK**.

A PowerShell process with `-EncodedCommand` (or `-enc`, `-e`) followed by a long string deserves a closer look. Microsoft documents that this option takes a Base64 string of **UTF-16LE** text.

### 5.2 Decode it safely

*Added research.*

The video pastes the string into an online decoder. You can decode it in PowerShell instead, without running it:

```powershell
$text = 'PASTE-THE-BASE64-HERE'
[Text.Encoding]::Unicode.GetString([Convert]::FromBase64String($text))
```

This only turns the string back into readable text. It never runs the command. **Don't paste the decoded command into PowerShell.**

---

## 6. Signs Worth a Closer Look

*Added research, based on the videos' examples.*

| What you see | Why it matters | What to do |
|---|---|---|
| PowerShell, `wscript.exe` or `cmd.exe` connecting to the internet | Common tools for running malware scripts | Check its command line (5.1) and the address (Section 4) |
| A long Base64 string in a command line | Hiding what the command does | Decode it (5.2) |
| A program you don't recognize with an internet connection | It could be unwanted software, or malware | Search its name and file location |
| Programs running from `Downloads`, `Public`, `AppData\Local\Temp` or similar folders | Malware often runs from folders anyone can write to | Check the file with Windows Security or VirusTotal |
| Constant connections to one unfamiliar address | Could be a C2 server checking in | Look up the address (Section 4) |
| Something you don't recognize starting with Windows | Malware needs to survive restarts | Check it with [Autoruns](https://github.com/ImmuneMoon/Windows-10-Community-Support-Guide#autoruns) |

---

## 7. Make Script Attacks Harder

*The video's advice, corrected and expanded with Microsoft's documentation.*

The video suggests disabling JavaScript, PowerShell scripts and Windows Script Host, since most people never use them. The idea is sound, but one common method doesn't work the way people think.

### 7.1 PowerShell's execution policy isn't a defense

Microsoft says plainly that the execution policy **isn't a security boundary**. It's a safety feature that stops you running scripts by accident. Malware gets around it easily, for example by passing commands with `-EncodedCommand` as in Section 5. Don't rely on it.

### 7.2 Attack surface reduction rules

These are part of **Microsoft Defender Antivirus**, and Microsoft says they work on any edition of Windows that includes it, including Home. Two of them block exactly what the video's RAT did:

| Rule | GUID |
|---|---|
| Block execution of potentially obfuscated scripts | `5beb7efe-fd9a-4556-801d-275e5ffc04cc` |
| Block JavaScript or VBScript from launching downloaded executable content | `d3e037e1-3eb8-44c8-a917-57927947596d` |

Turn them on from PowerShell, opened as administrator. Start in **audit mode**, which only logs what would have been blocked, so you can check nothing you need breaks:

```powershell
$obfuscated = '5beb7efe-fd9a-4556-801d-275e5ffc04cc'
$jsvbs = 'd3e037e1-3eb8-44c8-a917-57927947596d'
Add-MpPreference -AttackSurfaceReductionRules_Ids $obfuscated, $jsvbs `
  -AttackSurfaceReductionRules_Actions AuditMode, AuditMode
```

When you're happy, run the same commands with `Enabled, Enabled` in place of `AuditMode, AuditMode`. To check which rules are set:

```powershell
$p = Get-MpPreference
$p.AttackSurfaceReductionRules_Ids
$p.AttackSurfaceReductionRules_Actions
```

Notes from Microsoft:
- **Use `Add-MpPreference`, not `Set-MpPreference`.** `Set-MpPreference` replaces any rules you already have.
- **The obfuscated-scripts rule needs cloud-delivered protection** turned on in Windows Security.
- **Work PCs:** if your organization manages the PC, its settings override yours.

### 7.3 Smart App Control (Windows 11)

Smart App Control blocks apps it believes are malicious or untrusted, including unsigned ones. It's in the **Windows Security** app. Microsoft's FAQ says recent updates let you turn it back on without reinstalling Windows. But it also says a reset or reinstall may still be needed if you had optional diagnostic data turned off. There's no way to allow one blocked app as an exception.

---

## 8. If You Find Something

*Added research.*

1. **Disconnect from the internet.** That cuts the attacker off and stops more data leaving.
2. **Don't wipe the PC yet,** and don't flush the DNS cache. Note down what you found.
3. **Follow the [Windows Malware Removal Guide](https://github.com/ImmuneMoon/Windows-Malware-Removal-Guide).**
4. **Change your important passwords from a different, clean device,** not the affected PC. That guide covers what else to do.

---

## 9. Sources

Checked in November 2026.

**Videos (The PC Security Channel)**
- [Your computer is logging this!](https://www.youtube.com/watch?v=IX4q8dX3jbU)
- [How to know if your PC is hacked? Modern RATs](https://www.youtube.com/watch?v=B8ayJPq26iU)

**Microsoft**
- [ipconfig](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ipconfig)
- [Get-DnsClientCache](https://learn.microsoft.com/en-us/powershell/module/dnsclient/get-dnsclientcache)
- [netstat](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netstat)
- [Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [TCPView](https://learn.microsoft.com/en-us/sysinternals/downloads/tcpview)
- [About execution policies](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies)
- [About pwsh (`-EncodedCommand`)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh)
- [Attack surface reduction rules reference](https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-rules-reference)
- [Configure attack surface reduction rules](https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-rules-configure)
- [Smart App Control FAQ](https://support.microsoft.com/en-us/windows/smart-app-control-frequently-asked-questions-285ea03d-fa88-4d56-882e-6698afdb7003)

**Other**
- [Google Chrome: Use secure DNS](https://support.google.com/chrome/answer/10468685?co=GENIE.Platform%3DDesktop)
- [VirusTotal](https://www.virustotal.com/)

<!-- guide:end -->

## ☕ Support the Project

If you find this project helpful and want to support further development by Fulllion Creative Works, consider leaving a tip!

* [Donate via PayPal](https://www.paypal.com/donate/?hosted_button_id=LCDZX75HR4CLC)
* [Support on Ko-fi](https://ko-fi.com/fulllion)

---
© 2026 Fulllion Creative Works. All rights reserved.
