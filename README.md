# Lab 06 — Telnet vs. SSH: A Remote-Access Security Demonstration

**Goal:** prove, with real packet captures, *why* Telnet is insecure and SSH is
secure for remote device management. Telnet sends credentials in **plaintext**
that an attacker on the path can **read and alter**; SSH **encrypts** the session
so they cannot.

**Scope:** all captures are done on **my own lab** — my own Kali VM and my own
Packet Tracer topology. No network I don't own is touched.

**Tools:** Cisco Packet Tracer, Wireshark, Kali Linux.

---

## 1. Background

Remote management lets an administrator control a device (router, switch, server)
over the network instead of plugging in directly. Two protocols do this job:

| | **Telnet** | **SSH** |
|---|---|---|
| Purpose | remote CLI access | remote CLI access (same job) |
| Port | **23** | **22** |
| Encryption | ❌ none (plaintext) | ✅ encrypted |
| Secure? | **No** | **Yes** |

### Why it matters — the CIA triad

Security rests on three properties. Telnet fails two of them; SSH protects all:

- **Confidentiality** — can anyone *read* the data? Telnet: ❌ (password is
  visible). SSH: ✅ (encrypted).
- **Integrity** — can anyone *alter* the data undetected? Telnet: ❌. SSH: ✅
  (integrity checks detect tampering).
- **Authentication** — am I really talking to the right device? SSH verifies the
  server with host keys; Telnet does not.

### How an attacker captures the traffic

An attacker on the same network doesn't need magic. Using **ARP spoofing**, they
trick devices into sending traffic *through* the attacker's machine
(man-in-the-middle). Once in the middle, they capture everything — and if it's
Telnet, they read the password directly. This is exactly why plaintext protocols
are dangerous on any shared network.

---

## 2. Methodology

I demonstrated the difference two ways:

- **Part A (Packet Tracer):** built a topology, configured Telnet then SSH on a
  router, and inspected the packets — to show the setup and the port difference.
- **Part B (real Wireshark on Kali):** captured a live Telnet login and a live
  SSH login, then used **Follow TCP Stream** to show the plaintext password vs.
  the encrypted session.

---

## 3. Part A — Packet Tracer Setup

### Topology

```
PC0 ── Switch ── Router (R1)
```

<!-- SCREENSHOT 1: your Packet Tracer topology -->
![Packet Tracer topology](images/01-topology.png)

### Telnet configuration on R1

```
enable
configure terminal
hostname R1
interface gig0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
exit
line vty 0 4
 password cisco123
 login
 transport input telnet
```

<!-- SCREENSHOT 2: the Telnet config on R1 (CLI) -->
![R1 Telnet configuration](images/02-telnet-config.png)

### Telnet login from PC0

```
telnet 192.168.1.1
```

<!-- SCREENSHOT 3: successful Telnet login from PC0 -->
![Telnet login from PC0](images/03-telnet-login.png)

### SSH configuration on R1

```
configure terminal
ip domain-name lab.local
crypto key generate rsa      (choose 1024)
username admin password admin123
line vty 0 4
 transport input ssh
 login local
```

<!-- SCREENSHOT 4: the SSH config on R1 (CLI) -->
![R1 SSH configuration](images/04-ssh-config.png)

### SSH login from PC0

```
ssh -l admin 192.168.1.1
```

<!-- SCREENSHOT 5: successful SSH login from PC0 -->
![SSH login from PC0](images/05-ssh-login.png)

> **Note on Packet Tracer:** its PDU viewer confirms Telnet runs on **port 23
> unencrypted** and SSH on **port 22 encrypted**, but it does not clearly display
> the typed password. For that proof, I used real Wireshark (Part B).

---

## 4. Part B — Real Capture with Wireshark (Kali)

### Setup — a local Telnet server (my own VM)

```bash
sudo apt update
sudo apt install inetutils-telnetd -y
sudo useradd -m testuser
sudo passwd testuser          # set a throwaway password, e.g. test123
```

### 4.1 Capturing Telnet (insecure)

1. Started Wireshark capture on the **loopback (`lo`)** interface.
2. Ran `telnet localhost` and logged in as `testuser`.
3. Applied the display filter: `telnet`

<!-- SCREENSHOT 6: Wireshark showing captured Telnet packets (filter: telnet) -->
![Captured Telnet packets](images/06-telnet-packets.png)

4. Right-clicked a packet → **Follow → TCP Stream**.

**Result — the username and password are visible in plain text:** ⭐

<!-- SCREENSHOT 7: Follow TCP Stream showing the plaintext password -->
![Telnet plaintext password revealed](images/07-telnet-plaintext.png)

This is the vulnerability: anyone capturing this traffic reads the credentials
directly.

### 4.2 Capturing SSH (secure)

1. Captured again while running `ssh testuser@localhost`.
2. Applied the display filter: `ssh`

<!-- SCREENSHOT 8: Wireshark showing captured SSH packets (filter: ssh) -->
![Captured SSH packets](images/08-ssh-packets.png)

3. Right-clicked a packet → **Follow → TCP Stream**.

**Result — the session is encrypted and unreadable:** ⭐

<!-- SCREENSHOT 9: Follow TCP Stream showing encrypted SSH gibberish -->
![SSH encrypted stream](images/09-ssh-encrypted.png)

### 4.3 Bonus — HTTPS/TLS is encrypted too

Modern web traffic (HTTPS) uses TLS, which encrypts exactly like SSH. A capture
of normal browsing shows only unreadable "Application Data":

<!-- SCREENSHOT 10 (optional): TLS/HTTPS traffic showing encrypted Application Data -->
![TLS encrypted traffic](images/10-tls-encrypted.png)

---

## 5. Analysis

Side by side, the captures tell the whole story:

| | Telnet | SSH |
|---|---|---|
| Follow TCP Stream shows | **readable password** ❌ | **encrypted gibberish** ✅ |
| Confidentiality | broken | protected |
| Integrity | broken | protected |
| Port | 23 | 22 |

An attacker performing ARP spoofing on the same LAN would capture the Telnet
session and read the credentials instantly — then log in as the administrator.
With SSH, the same captured traffic is useless to them.

---

## 6. The Broader Pattern

Telnet is not the only plaintext protocol. The same "insecure → secure" upgrade
exists across the board:

| Insecure (plaintext) | Secure (encrypted) | What it does |
|---|---|---|
| Telnet (23) | **SSH (22)** | remote CLI access |
| HTTP (80) | **HTTPS (443)** | web browsing |
| FTP (21) | **SFTP / FTPS** | file transfer |
| POP3 (110) | **POP3S (995)** | retrieving email |

**The lesson:** if a protocol sends data in plaintext, assume anyone on the path
can read it.

---

## 7. Recommendations

For a secure network, an administrator should:

1. **Disable Telnet entirely** — use **SSH only** (`transport input ssh`).
2. **Use key-based authentication** for SSH (stronger than passwords).
3. **Restrict management access** to specific trusted IP addresses.
4. **Prefer the encrypted version** of every protocol (HTTPS, SFTP, etc.).

---

## 8. Conclusion

The packet captures prove it directly: **Telnet exposes credentials in plaintext,
while SSH encrypts the entire session.** For any network that needs to be
secure — which is every real network — **SSH is the correct choice** and Telnet
should be disabled. The difference isn't theoretical; it's visible in two
screenshots.

---

## Ethics & scope

Every capture here was performed on **my own equipment** — my own Kali VM and my
own Packet Tracer topology — using a throwaway test account. Capturing traffic on
a network you don't own or aren't authorised to test is illegal (and against
Sri Lanka's Computer Crimes Act). The goal here is to **understand and defend**.
