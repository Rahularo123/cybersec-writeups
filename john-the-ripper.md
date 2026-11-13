# John the Ripper Basics: Hash Cracking, Linux Shadow, Windows NTLM, and *2john

These are my personal notes, key concepts, and practical takeaways from the TryHackMe John the Ripper Basics room.

---

## 1. P vs. NP in Hashing

- **Hashing (forward direction) is in $\text{P}$ (Polynomial time):** Fast and computationally efficient for any computer to calculate.
- **"Un-hashing" (reverse direction) is in $\text{NP}$ / intractable:** Mathematically infeasible to solve directly in a reasonable timeframe on classical computers.
- **Why it matters:** Because there is no polynomial-time shortcut to reverse a hash, tools like John the Ripper must use dictionary and brute-force guessing attacks.

---

## 2. Wordlists: What is SecLists?

- **SecLists** (by Daniel Miessler on GitHub) is the cybersecurity industry's standard collection of wordlists.
- Contains organized lists for:
  - Passwords (including `rockyou.txt`)
  - Usernames
  - Web directories / URL endpoints (for Dirb, Gobuster, ffuf)
  - Fuzzing payloads
- Default location in Kali Linux: `/usr/share/seclists/` (installable via `sudo apt install seclists`).

---

## 3. Windows Passwords, Mimikatz, and Pass-the-Hash

### Windows Password Storage:
- **NTHash / NTLM:** The standard hash format modern Windows systems use for local user and service accounts (named after NT and legacy LAN Manager).
- Stored locally in the **SAM** database, or network-wide across a corporate domain in the Active Directory database: **`NTDS.dit`**.

### Why can't an Administrator just open the SAM file?
- **Exclusive OS Lock:** The Windows kernel actively locks `C:\Windows\System32\config\SAM` while running.
- **Double Encryption:** Hashes inside SAM are encrypted on disk by keys stored in the `SYSTEM` registry hive.
- **Loaded into RAM:** Active hashes live inside the memory of the **`lsass.exe`** process.

### The 3 Ways to Get Windows Hashes:
1. **Registry Backup (Disk):** Take an offline snapshot using built-in Windows commands (`reg save HKLM\sam sam.save` and `reg save HKLM\system system.save`).
2. **Mimikatz (RAM):** Inject directly into the running `lsass.exe` process in memory to extract and decrypt active hashes.
3. **Active Directory Dump:** Dump `NTDS.dit` on a Domain Controller to extract hashes for all domain users.

### What is "Pass-the-Hash" (PtH)?
- Windows network protocols (SMB/RPC on port 445) authenticate users using the **raw NTLM hash**, not the plaintext password.
- An attacker with a stolen hash does not need to waste time cracking it with John; they can pass the raw hash directly into network tools (like `psexec` or `crackmapexec`) to get an instant administrative shell.

---

## 4. Linux Passwords: `/etc/passwd` vs. `/etc/shadow`

| Feature | `/etc/passwd` (7 Fields) | `/etc/shadow` (9 Fields) |
| :--- | :--- | :--- |
| **Permissions** | World-readable (`644`) | Strictly `root` only (`640` / `600`) |
| **Purpose** | User account metadata, UID/GID, shell | Secret password hashes and expiration policies |
| **Password Field** | Placeholder `x` | Salted hash (e.g. `$6$salt$hash...`) |

### What is the GECOS Field?
- The **5th field** in `/etc/passwd` storing personal user information (full name, room number, office phone).

---

## 5. Unshadowing: Why and How

### Why `unshadow` is needed:
- A raw `/etc/shadow` line contains 9 colon-separated columns (password aging numbers, expiry dates) that confuse John's standard parser.
- `unshadow` merges `/etc/passwd` and `/etc/shadow`, replacing the `x` in `passwd` with the real hash from `shadow` into a classic unified 7-field format.
- It brings over the **GECOS field** so John can run targeted intelligence attacks.

### Syntax:
```bash
unshadow /etc/passwd /etc/shadow > unshadowed.txt
john --wordlist=rockyou.txt unshadowed.txt
```

---

## 6. Single Crack Mode (`--single`) and Word Mangling

- **Purpose:** Cracks accounts using fast rules based on personal user info before running massive wordlists.
- **How it works:**
  - Extracts base words from Field 1 (Username) and Field 5 (GECOS).
  - Treats **every token independently** (names, room numbers, and phone digits).
  - Mangles them with rules: capitalization, appending numbers (`123`), leetspeak, reversing (`ecilA`), and combinations (`Alice404`).
- **File Input Options:**
  - Full unshadowed file: `john --single unshadowed.txt` (mangles username + GECOS).
  - Simple formatted file: `john --single hashes.txt` where lines are formatted as `username:hash` (mangles username).

---

## 7. Custom Rules in John

- **What they are:** Custom word-mangling directives (e.g., capitalize first letter, append digits/symbols).
- **Location:** Defined in `/etc/john/john.conf` under `[List.Rules:<RuleName>]`.
- **Usage:**
  ```bash
  john --wordlist=rockyou.txt --rule=<RuleName> hash.txt
  ```

---

## 8. Cracking Encrypted Files: The `*2john` Tool Family

John the Ripper cannot read binary files (ZIP, RAR, PDF, SSH keys). It only compares hashes.

### What `*2john` tools do:
- They do **NOT** crack passwords.
- They extract the **verification checksum, salt, and encryption parameters** from the file's header into a single-line hash string that John can understand.

### The 2-Step Workflow:
1. **Extract Hash:**
   - ZIP: `zip2john file.zip > hash.txt`
   - RAR: `rar2john file.rar > hash.txt`
   - SSH Private Key: `ssh2john id_rsa > hash.txt`
2. **Crack with John:**
   ```bash
   john --wordlist=rockyou.txt hash.txt
   ```

---

## 9. Helpful CLI Gotcha: `grep -iF`
- **`-i`**: Case-insensitive search (matches `md5`, `MD5`).
- **`-F`**: Fixed strings (treats search text as literal characters rather than regular expressions).
- Useful when searching John's format list: `john --list=formats | grep -iF "md5"`
