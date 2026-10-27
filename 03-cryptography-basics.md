# Cryptography Basics: Core Concepts, Symmetric vs. Asymmetric, and XOR

These are the essential building blocks and practical takeaways from the TryHackMe Cryptography Basics room.

---

## 1. The Core Terminology

- **Plaintext:** The readable original data before encryption (text, images, files, or binary).
- **Ciphertext:** The scrambled, unreadable result produced by encryption.
- **Cipher:** The mathematical algorithm or formula used to encrypt and decrypt data (e.g., AES, RSA).
- **Key:** The secret sequence of bits fed into the cipher.
- **Kerckhoffs's Principle (Crucial Concept):** In modern cryptography, **the cipher/algorithm is always public knowledge**. Security depends *entirely* on keeping the key secret, never on keeping the algorithm hidden.

---

## 2. Symmetric vs. Asymmetric Encryption

### A. Symmetric Encryption (Shared Secret)
- **Mechanism:** Uses the **exact same key** for both encryption and decryption.
- **Advantage:** Extremely fast; ideal for encrypting large amounts of data (files, hard drives, active network traffic).
- **Major Problem (Key Exchange):** Both parties must securely agree on the secret key first. If you email the key over an insecure channel, an attacker intercepts everything.
- **Common Standards:**
  - **DES:** Deprecated (56-bit key, easily broken today).
  - **3DES:** Deprecated transition standard (effectively ~112-bit security).
  - **AES (Advanced Encryption Standard):** The gold standard today (128, 192, or 256-bit keys).

### B. Asymmetric Encryption (Public / Private Key Pair)
- **Mechanism:** Uses two mathematically linked keys:
  - **Public Key:** Shared openly with anyone; used to **encrypt** data.
  - **Private Key:** Kept strictly secret by the owner; used to **decrypt** data.
- **Advantage:** Solves the key-distribution problem—no shared secret needs to travel over the wire.
- **Trade-off:** Much slower and computationally heavier than symmetric encryption (uses large keys like 2048-bit or 4096-bit).
- **Common Standards:** RSA, Diffie-Hellman, ECC (Elliptic Curve Cryptography).

> **Real-World Reality (Hybrid Encryption):** In practice (like HTTPS/TLS), asymmetric encryption is used *first* just to exchange a temporary symmetric key. Then, both machines switch to AES for fast data transfer.

---

## 3. XOR as a Symmetric Cipher (The CTF Classic)

The bitwise **XOR (`^`)** operation is the foundation of many stream ciphers and One-Time Pads:

| Bit A | Bit B | A XOR B |
| :---: | :---: | :---: |
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

### Why XOR works as symmetric encryption:
XOR is **reversible using the exact same key**:
- **Encryption:** `Plaintext ^ Key = Ciphertext`
- **Decryption:** `Ciphertext ^ Key = Plaintext`

Because the operation is identical in both directions with the same key, it is a pure symmetric cipher. In CTFs and malware analysis, you will constantly see XOR used for basic obfuscation and payload encryption.
