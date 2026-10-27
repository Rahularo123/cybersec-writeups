# Networking & Recon: Key Concepts and Gotchas (tcpdump, Nmap, and Tools)

These are my personal notes and key takeaways while learning networking and reconnaissance on TryHackMe. Instead of listing every basic command, this covers practical concepts, things i learnt, common misconceptions, and mistakes I ran into.

---

## 1. Port Numbers are Just Defaults, Not Fixed Rules

In textbooks, you learn that HTTP is port 80 and HTTPS is port 443. But in reality, **any service can run on any port from 1 to 65535**.

- **Why it matters:** In labs and real environments, administrators often run web servers on alternative ports like `8008`, `8080`, or `5000` for development or internal panels.
- **Key Takeaway:** If ports 80 and 443 are closed, that doesn't mean there is no website. If you see an open port like 8008 running HTTP, you access it in your browser with:
  `http://<target-ip>:8008`
- Always look at the service name or run version detection (`nmap -sV`) instead of guessing by port number alone.

---

## 2. Common Misconception: "Dirb Exploits Hidden Paths"

When running tools like `dirb`, `gobuster`, or `ffuf`:
- **What they actually do:** They **only discover** paths by guessing names from a wordlist (Reconnaissance). 
- **What they don't do:** They **do not exploit** anything. If `dirb` finds `/secret_backup.zip` or `/admin/login.php`, its job is done. 
- **The Exploit is what you do next:** You manually download the zip to look for leaked passwords, or use Hydra/SQL injection against the login form.
- **Directories vs Files:** In URLs, anything after the domain is a path. Tools can search for folders (`/admin/`) or append extensions (`-X .zip,.php`) to look for specific files.

---

## 3. Tool Mental Map: Hydra vs. Hashcat vs. Dirb

| Tool | Where It Runs | What It Attacks | Input Needed |
| :--- | :--- | :--- | :--- |
| **Hydra** | Over the network (Online) | Live logins (SSH, FTP, Web login forms) | Target IP + Protocol + Wordlist |
| **Hashcat** | Locally on your GPU/CPU (Offline) | Stolen password hashes (from database dumps) | Hash string + Wordlist |
| **Dirb / Gobuster** | Over the web (Online) | Finding hidden website folders and files | URL + Wordlist |

---

## 4. Why Nmap Sends Weird Packets During Ping Scans

When running a scan as root (`sudo nmap -sn <target>`), Nmap doesn't just send a normal `ping` (ICMP Echo). It sends a combination of 4 probes:
1. **ICMP Echo Request** (Standard ping)
2. **ICMP Timestamp Request** (Asks for system clock time; helps if standard ping is blocked)
3. **TCP SYN to Port 443** (HTTPS; standard connection attempt to the most commonly open port)
4. **TCP ACK to Port 80** (HTTP)

### Why send an ACK out of nowhere to Port 80?
- Normally, an `ACK` packet only happens in the *middle* of an ongoing connection.
- Many simple firewalls let incoming ACK packets through because they assume you are replying to something the server requested.
- When the target server receives an unexpected ACK, it gets confused and replies with a **TCP RST (Reset)** packet.
- When Nmap gets that `RST`, it knows for sure: **the host is alive behind the firewall.**

---

## 5. Why Does Nmap Tear Down Open Ports with RST-ACK?

When Nmap finds an open port (like port 22), it completes the 3-way handshake to verify the port is open, and then immediately sends a `[RST, ACK]` packet.
- **Why?** Nmap's job is just scanning, not logging in. 
- Instead of doing a slow, graceful 4-step teardown (`FIN`), an instant `RST` kills the connection immediately so it doesn't waste memory or lock up sockets on the target machine.

---

## 6. What is a Packet Header and BPF?

- **Payload vs Header:** If your network data is a letter, the **payload** is the text inside, and the **header** is the envelope (source address, destination address, ports, and flags).
- **BPF (Berkeley Packet Filter):** An in-kernel filter engine. Instead of copying every single packet to your terminal, BPF discards unwanted packets inside the kernel immediately, saving system resources.
- **Reading TCP Flags in tcpdump:**
  Instead of doing complex math on byte offsets, use built-in aliases:
  ```bash
  # Matches packets where ONLY the RST flag is set:
  tcpdump -r traffic.pcap "tcp[tcpflags] == tcp-rst"

  # Matches packets where the SYN flag is set (even if ACK is also set):
  tcpdump -r traffic.pcap "tcp[tcpflags] & tcp-syn != 0"

## 7. How Traceroute Actually Works

When you see "ICMP Time-to-Live (TTL) Exceeded" packets:

traceroute intentionally starts sending packets with TTL = 1, then TTL = 2, etc.
Each router on the path decrements the TTL by 1.
When TTL hits 0, that router drops the packet and returns an ICMP Time Exceeded message.
traceroute reads the IP of whoever sent that message to map out every hop along the route.

## 8. MAC Address Structure: OUI vs. Device ID

A MAC address has 6 bytes (XX:XX:XX:YY:YY:YY):

First 3 bytes (OUI): Identifies the vendor/manufacturer (e.g., 52:54:00 belongs to QEMU/KVM virtual machines).
Last 3 bytes: Unique identifier for that specific network card (like a serial number).
Two virtual machines on the same cloud network will have the exact same first 3 bytes, but different last 3 bytes.

## 9. Helpful CLI Gotchas to Remember
tcpdump -n vs -nn:
-n prevents resolving IP addresses to domain names.
-nn prevents resolving both IPs and port numbers (so port 22 stays 22 instead of turning into the word ssh).
Always use -r for files: Running tcpdump without -r <file.pcap> listens to your live terminal interface instead of reading the capture file.
Piping to wc:
wc prints: lines  words  bytes.
Use wc -l if you only want the line count (number of packets).
Save scans with -oA: Running nmap -oA scan_results <target> saves output in normal (.nmap), XML (.xml), and grepable (.gnmap) formats all at once.

