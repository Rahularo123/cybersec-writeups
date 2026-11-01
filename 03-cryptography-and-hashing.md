# Public Key Cryptography: RSA, Diffie-Hellman, and SSH Authentication

These are my personal notes, key concepts, and practical takeaways from the TryHackMe Public Key Cryptography room.

---

## 1. Symmetric vs. Asymmetric: The Real-World Partnership

- **Symmetric Encryption (e.g., AES):**
  - Uses the **exact same secret key** for encryption and decryption.
  - **Superpower:** Extremely fast; encrypts 100% of actual data traffic (web pages, files, video streams).
  - **Limitation:** Cannot prove *who* created a message (no public authentication) because both parties share the exact same key.

- **Asymmetric Encryption (e.g., RSA, ECC):**
  - Uses a **key pair**: a Public Key (known to everyone) and a Private Key (known only to the owner).
  - **Superpower:** Authenticity, Digital Signatures, and Key Exchange.
  - **Limitation:** Hundreds of times slower than symmetric encryption due to heavy math.

### How they work together (Hybrid Encryption)
In real protocols like HTTPS/TLS:
1. **Asymmetric** steps in for a split second at the start to **authenticate** identity and safely exchange a key.
2. **Symmetric (AES)** takes over for the rest of the session to encrypt all data at high speed.

---

## 2. How RSA Works (The Prime Factorization Trapdoor)

RSA is based on a one-way math trick: **multiplying two large prime numbers is easy, but factoring that big number back into primes is virtually impossible.**

### The Setup:
1. Bob picks two secret prime numbers: $p$ and $q$.
2. Bob multiplies them: $n = p \times q$. (This number $n$ is shared publicly with the world).
3. Bob calculates a secret "clock size": 
   $$\phi(n) = (p - 1) \times (q - 1)$$
   *(Only someone who knows $p$ and $q$ can calculate this number!)*
4. Bob chooses an encryption exponent $e$ that shares no common factors with $\phi(n)$.
5. Bob calculates the matching decryption key $d$ such that:
   $$(e \times d) \pmod{\phi(n)} = 1$$
   *(This ensures $d$ mathematically cancels out / undoes $e$).*

### The Keys:
- **Public Key (given to everyone):** $(n, e)$
- **Private Key (kept secret by Bob):** $(n, d)$

### Encryption & Decryption:
- **Alice Encrypts Message $x$:**
  $$y = x^e \pmod n$$
- **Bob Decrypts Ciphertext $y$:**
  $$x = y^d \pmod n$$
*(The original message $x$ is restored!)*

---

## 3. Diffie-Hellman Key Exchange (Formulas & Concept)

Diffie-Hellman allows two parties to create the **exact same shared symmetric key** across an insecure network **without ever sending the key over the wire**.

### The Formulas:
- Alice and Bob publicly agree on a base number $g$ and a prime modulus $p$.
- Alice picks secret number $a$; Bob picks secret number $b$.

1. **Calculate Public Keys:**
   - Alice computes and sends:
     $$A = g^a \pmod p$$
   - Bob computes and sends:
     $$B = g^b \pmod p$$

2. **Calculate the Shared Secret Key ($K$):**
   - Alice computes:
     $$K = B^a \pmod p$$
   - Bob computes:
     $$K = A^b \pmod p$$

Both calculations arrive at the **exact same number $K$**. Because the secret key never traveled across the network, an eavesdropper recording all traffic cannot steal it.

---

## 4. Digital Signatures & How SSH Authentication Works

A digital signature is mathematically defined as:
$$\text{Digital Signature} = \text{Hash(Data)} \text{ encrypted with the sender's Private Key}$$

### How SSH Client Key Authentication Works Step-by-Step:
1. **Setup:** You place your **Public Key** onto the remote server in `~/.ssh/authorized_keys`.
2. **Challenge:** When you connect (`ssh user@server`), the server generates a random challenge string.
3. **Signing:** Your laptop takes that string, hashes it, encrypts the hash with your secret **Private Key**, and sends it back.
4. **Verification:** The server uses your stored **Public Key** to unlock the signature. If it reveals the original challenge string, your identity is verified.
*(Only the matching private key could have created a signature that your public key unlocks!)*

---

## 5. Important Gotchas, Concepts, and CTF Tricks

- **SSH Server Password vs. SSH Key Passphrase:**
  - **Server Password:** Sent across the network to authenticate with the server.
  - **Key Passphrase:** Used purely on your local laptop to symmetrically encrypt/lock your private key file on disk. It **never** leaves your computer.
- **Strict File Permissions (`chmod 600`):**
  - Linux SSH clients will refuse to connect if your private key file has loose permissions. Run `chmod 600 <key_file>` so only your user account can read it.
- **`ssh -i <file>`:**
  - Used to specify a custom **Private Key** file located outside default paths (e.g., `ssh -i id_rsa user@target`).
- **The "Better Shell" CTF Trick:**
  - Basic reverse shells (e.g. Netcat) are unstable: no arrow keys, no tab completion, and `Ctrl + C` kills the connection.
  - **The upgrade:** Through the reverse shell, append your public key to the target's `~/.ssh/authorized_keys`. Then log in directly via SSH for a fully interactive, permanent, stable terminal.
- **ED25519 vs. RSA:**
  - **RSA:** The traditional standard; requires huge keys (2048 to 4096 bits) and is slower.
  - **ED25519:** Modern Elliptic Curve Cryptography (ECC); uses compact 256-bit keys, is significantly faster, and is the modern standard for SSH.
- **PGP vs. GPG:**
  - **PGP (Pretty Good Privacy):** The original standard for applying public-key cryptography to everyday files and emails.
  - **GPG (GNU Privacy Guard):** The open-source Linux command-line tool (`gpg`) used to encrypt sensitive files and verify digital signatures on software downloads.
# Hashing Basics: Hashes, Linux Passwords, Hashcat Modes, and HMACs

These are my personal notes, key concepts, and practical takeaways from the TryHackMe Hashing Basics room.

---

## 1. The Core Difference: Encoding vs. Encryption vs. Hashing

This is one of the most common points of confusion (and a classic interview question):

| Concept | Purpose | Can It Be Reversed? | Key Needed? | Example |
| :--- | :--- | :--- | :--- | :--- |
| **Encoding** | Change data format for transmission / storage. | **Yes, easily** | No key | Base64, Hex, ASCII |
| **Encryption** | Protect confidentiality (hide secrets). | **Yes** (with the right key) | **Yes** | AES, RSA |
| **Hashing** | Create a unique one-way digital fingerprint (integrity). | **No** (one-way math) | No | SHA-256, MD5 |

- **Encoding** is NOT security. Anyone can decode Base64 back to plaintext instantly.
- **Encryption** is a two-way street: lock with key $\rightarrow$ unlock with key.
- **Hashing** is a one-way street: turn an entire book into a short string; you cannot reconstruct the book from the hash.

---

## 2. How Hashes are Represented (Raw Bytes vs. Hex)

- Hash algorithms actually output **raw binary bytes** under the hood.
- Because raw bytes are unreadable by humans, tools encode them into readable strings—most commonly **hexadecimal**.
- Linux command-line tools (`md5sum`, `sha256sum`, `sha512sum`) output in **hexadecimal**.
- **The 1-to-2 Math Rule:** Every **1 raw byte** is printed as **2 hex characters**.
  - Example: SHA-256 outputs **32 raw bytes** $\rightarrow$ it displays as a **64-character** hex string.

---

## 3. Linux Password Storage: `/etc/passwd` vs. `/etc/shadow`

### Why `/etc/shadow` exists:
- Historically, password hashes were stored in `/etc/passwd`.
- However, many normal programs (and all users) need to read `/etc/passwd` to check usernames and user IDs. Storing hashes there meant *any* local user could copy the hashes and crack them offline.
- Modern Linux moved the password hashes into **`/etc/shadow`**, which can **strictly only be read by `root`**.

### Anatomy of a Linux Shadow Hash Field:
Inside `/etc/shadow`, each line is separated by colons (`:`). The password hash field looks like this:
```text
$id$options$salt$hash
```

- **`$id` (Algorithm ID):** Instantly tells you what hashing algorithm created it:
  - `$1$` = MD5
  - `$5$` = SHA-256
  - `$6$` = SHA-512 (modern Linux default)
  - `$y$` = Yescrypt
- **`$salt`:** A random string added to the password before hashing. It prevents attackers from using precomputed lookup tables (Rainbow Tables) to crack passwords instantly.

---

## 4. Hashcat: What is "Hash Mode" (`-m`)?

Hashcat is a fast, GPU-powered offline password cracker. Because it cannot always guess which algorithm created a scrambled hash, you must tell it the exact algorithm using the **`-m` (mode)** flag:

```bash
hashcat -m <mode_number> hash.txt wordlist.txt
```

### Common Hash Modes:
- **`-m 0`**: Plain MD5
- **`-m 1000`**: Windows NTLM passwords
- **`-m 1800`**: Linux SHA-512 (`$6$`) shadow hashes
- **`-m 9300`**: Cisco-IOS scrypt (`$9$`)

The Hashcat Wiki (`example_hashes` table) is the reference manual used to look up the exact mode number for any hash format.

---

## 5. HMAC (Keyed-Hash Message Authentication Code)

A standard hash only verifies **Integrity** (did the data change?). It does not verify **Authenticity** (who actually sent it?).

### Why not just do `Hash(Key + Message)`?
If you simply glue a secret key to a message and hash it, older algorithms (MD5, SHA-1, SHA-256) are vulnerable to a **Length Extension Attack**. A hacker can append extra malicious data to the end of the message and compute a valid hash *without ever knowing the secret key*.

### The Solution: The Double-Hash Sandwich
HMAC solves this by nesting two rounds of hashing:
$$\text{HMAC}(K, M) = H\Big((K \oplus \text{opad}) \;\;\|\;\; H\big((K \oplus \text{ipad}) \;\;\|\;\; M\big)\Big)$$

1. **Inner Hash:** Mixes the key with an inner constant (`ipad`), glues the message $M$, and hashes it.
2. **Outer Hash:** Mixes the key with an outer constant (`opad`), glues the inner hash result, and hashes it a second time.

### Is HMAC an algorithm on its own?
**No.** HMAC is a **wrapper / framework** that must be paired with an underlying hash function. That is why you always see it named with a partner hash:
- **`HMAC-SHA256`**
- **`HMAC-MD5`**

**Key Takeaway:** HMAC uses a **Secret Key + Hashing Algorithm** to guarantee **both Integrity AND Authenticity** simultaneously. It is heavily used in API security and tokens (e.g., JWTs).
