---
title: "Subvert the Grid: Forge Your Own Identity Blackhole"
date: "2025-05-21"
header:
  teaser: /assets/images/anonymity/anonymity.jpg
  teaser_home_page: true
tags: anonymity,evasion,concealment
categories: anonymity
toc: true
toc_sticky: false
toc_label: "Table of Contents"
---
 
In a world of ever-increasing digital surveillance, protecting your privacy and anonymity isn't just a luxury it’s a necessary act of self-defense. Whether you're a security researcher, red team operative, geopolitically oppressed civilian, or a privacy advocate, mastering advanced techniques to obscure your network identity is not just beneficial it’s essential.

This in-depth guide combines the use of **Tor**, **Stem**, **iptables**, and **Wireshark** to build a hardened, anonymized routing setup. We’ll explore Python-based Tor circuit control, datagram analysis, firewall configuration, and real-time packet inspection using Wireshark.

---

## What is Tor?

**Tor (The Onion Router)** anonymizes Internet traffic by routing it through a multi-node, encrypted circuit:

* **Guard Node**: Knows your IP address but not your destination.
* **Middle Node**: Adds further obfuscation without knowing source or destination.
* **Exit Node**: Connects to the target destination without knowing the origin.

Each Tor connection involves encapsulated **cells** that are decrypted layer by layer, preserving anonymity through layered encryption.

<img src="/assets/images/anonymity/onion-encryption.gif">

---

## Tor Cell Structure & Datagram Anatomy

Tor traffic uses uniform, fixed-length **512-byte cells**, which eliminates size-based fingerprinting:

* **Relay Cells**: Carry application data.
* **Control Cells**: Manage circuit states.
* **Padding Cells**: Obfuscate patterns.

Each TCP stream multiplexes circuits, and each circuit contains multiple streams. This nesting structure increases obfuscation.

<img src="/assets/images/anonymity/tor_cell_structure.png">
---

## Setting Up Tor + Python Requests

### Step 1: Install Dependencies

```bash
sudo apt install tor
pip install requests[socks] stem
```

### Step 2: Basic Tor IP Check

```python
import requests

def get_tor_ip():
    session = requests.Session()
    session.proxies = {
        'http': 'socks5h://127.0.0.1:9050',
        'https': 'socks5h://127.0.0.1:9050'
    }
    ip = session.get('https://httpbin.org/ip').json()
    print("[+] IP via Tor:", ip['origin'])

if __name__ == '__main__':
    get_tor_ip()
```

> Ensure your Tor daemon is running: `sudo service tor start`

---

## Advanced Circuit Control with Stem

**Stem** allows fine-grained control of Tor circuits, including on-demand identity rotation.

### Step 1: Enable Tor ControlPort

Modify `/etc/tor/torrc`:

```
ControlPort 9051
CookieAuthentication 0
HashedControlPassword <your_hashed_password>
```

Restart Tor:

```bash
sudo systemctl restart tor
```

Generate password hash:

```bash
tor --hash-password your_password
```

### Step 2: Rotating Circuits in Python

```python
from stem import Signal
from stem.control import Controller
import time

def rotate_identity():
    with Controller.from_port(port=9051) as controller:
        controller.authenticate(password='your_password')
        controller.signal(Signal.NEWNYM)
        print("[+] Requested new identity")

if __name__ == '__main__':
    while True:
        rotate_identity()
        time.sleep(600)  # rotate every 10 minutes
```

> This script periodically signals Tor to build a new circuit, rotating your exit IP.

---

## Enforcing Tor-Only Routing via iptables

Block all outbound connections unless routed through Tor to prevent leaks.

### Example Setup:

```bash
# Flush existing rules
iptables -F
iptables -t nat -F

# Allow loopback
iptables -A OUTPUT -o lo -j ACCEPT

# Allow Tor process to connect anywhere
iptables -A OUTPUT -m owner --uid-owner debian-tor -j ACCEPT

# Redirect other traffic to Tor
iptables -t nat -A OUTPUT -p tcp --syn -j REDIRECT --to-ports 9040

# Block everything else
iptables -A OUTPUT -j REJECT

# Block Torrent providers
iptables -A FORWARD -m string --string "BitTorrent" --algo bm --to 65535 -j DROP
iptables -A FORWARD -m string --string "BitTorrent protocol" --algo bm --to 65535 -j DROP
iptables -A FORWARD -m string --string "peer_id=" --algo bm --to 65535 -j DROP
iptables -A FORWARD -m string --string ".torrent" --algo bm --to 65535 -j DROP
iptables -A FORWARD -m string --string "announce.php?passkey=" --algo bm --to 65535 -j DROP
iptables -A FORWARD -m string --string "torrent" --algo bm --to 65535 -j DROP
iptables -A FORWARD -m string --string "announce" --algo bm --to 65535 -j DROP
iptables -A FORWARD -m string --string "info_hash" --algo bm --to 65535 -j DROP
```

> Replace `debian-tor` with your system's Tor user if different.
---


##  Disabling IPV6 to prevent DNS Leaks

### Example Setup:

```bash
sudo net.ipv6.conf.all.disable_ipv6 = 1
sudo net.ipv6.conf.default.disable_ipv6 = 1
```
---


## Deep Packet Analysis with Wireshark

### Filters for Tor:

```wireshark
tcp.port == 9050 || tcp.port == 9001 || tcp.port == 443
```
* 9050 - Default SOCKS proxy port.

* 9001 - Default OR (Onion Routing) port for Tor relays.

* 443 - Tor can use this port to blend in with HTTPS traffic.

### What You’ll See:

* TLS handshakes to Guard/Exit nodes.
* Repeated 512-byte TCP segments (Tor cells).
* Uniform stream timing (Tor's built-in obfuscation).
* Absence of DNS leaks
* Multiplexed TCP streams

### Recognizing Tor Patterns:

| Feature       | Tor            | HTTPS    |
| ------------- | -------------- | -------- |
| Payload Size  | 512 bytes      | Variable |
| Multiplexing  | Yes (Circuits) | No       |
| Timing Jitter | High           | Normal   |
| Padding       | Consistent     | Rare     |
| DNS Requests  | Internal(Safe) | Exposed  |

Use the **Follow TCP Stream** feature to inspect encrypted payloads. While you can't decrypt it, structure analysis is still possible.

---

## Full Example: IP Rotation + Proxy Request

```python
import requests
from stem import Signal
from stem.control import Controller
import time

session = requests.Session()
session.proxies = {
    'http': 'socks5h://127.0.0.1:9050',
    'https': 'socks5h://127.0.0.1:9050'
}

def rotate_identity():
    with Controller.from_port(port=9051) as c:
        c.authenticate(password='your_password')
        c.signal(Signal.NEWNYM)
        print("[+] Identity rotated.")

for _ in range(3):
    rotate_identity()
    time.sleep(5)
    print(session.get("http://httpbin.org/ip").text)
    time.sleep(10)
```

---

## OPSEC Guidelines

*  Use sandboxed VMs or Tails OS
*  Clear cookies, WebGL, Canvas, etc.
*  Use multiple chains (VPN → Tor or Tor → VPN)
*  Never authenticate personal accounts over Tor

> "Anonymity isn't magic it's meticulous discipline."

---

##  Real-World Applications

* Threat Intelligence Gathering
* Anonymous Reporting & Journalism
* C2 Infrastructure in Red Team Ops
* Research in Hostile Environments
* Research in restricted geopolitical zones

---

##  References

### Tor Protocol & Internals

1. [Tor Specification](https://spec.torproject.org/tor-spec)
2. [Tor TLS Overview](https://spec.torproject.org/tor-spec#tls)
3. [Tor Design Paper](https://svn.torproject.org/svn/design-paper/tor-design.pdf)

### Python Libraries

4. [Python Requests SOCKS Proxy](https://docs.python-requests.org/en/latest/user/advanced/#socks)
5. [Stem Python Library](https://stem.torproject.org/)

### Network Obfuscation

6. [Tor Metrics](https://metrics.torproject.org)
7. [Wireshark Tor Detection](https://wiki.wireshark.org/Tor)
8. [Universidade do Minho Tor Detection](https://repositorium.sdum.uminho.pt/bitstream/1822/80970/1/Bruno%20Rafael%20Lamas%20Corredoura%20Dantas.pdf)

### Firewall & OS Security

9. [iptables Reference](https://man7.org/linux/man-pages/man8/iptables.8.html)
10. [Tor Transparent Proxy Setup](https://trac.torproject.org/projects/tor/wiki/doc/TransparentProxy)
11. [Tails OS](https://tails.net)
12. [OPSEC Guide](https://opsecguide.com)

---
