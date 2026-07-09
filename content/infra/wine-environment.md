---
title: "A Wine Container for .NET Reverse Engineering (Without a Windows VM)" 
date: 2026-06-30
tags: ["wine", "dotnet", "reverse-engineering", "incus", "htb"]
categories: ["infra"]
summary: "Building a Debian container running Wine, .NET, ILSpy and ysoserial.net, to avoid spinning up a Windows VM for a single HTB box."
---

This one started with a specific problem on a specific box, not a general itch
to avoid Windows VMs.

I was working through Scrambled on HTB, which involved reverse engineering a
.NET program — and, on top of that, actually running the application and
capturing the network traffic it generated. The obvious first instinct is to
spin up a Windows VM, install the .NET tooling there, and call it done.

The complication with that path is networking: the VPN tunnel interface for the
box lives exclusively inside my Kali container's network namespace, so a
Windows VM would need its traffic routed through the host and into that
namespace to actually reach the target. That sounds like a bigger problem than
it is — in practice it's one or two `iptables` rules on the host and another one
or two inside the Kali container, not some unsolvable routing puzzle. So that
wasn't really the reason I avoided the VM route; if it had been, I'd have just
written the rules.

The actual reason was simpler: I wanted to find out whether I could do this
entirely with containers instead, and treat it as a deliberate exercise rather
than a shortcut. Routing a VM's traffic into a container's network namespace is
a known, well-documented pattern. Getting a Windows-only RE toolchain running
inside the same kind of container I already use for everything else was not
something I'd done before, and that gap — "can I actually pull this off without
falling back to the VM-plus-iptables version" — was the more interesting
problem to me than the box itself.

## Setting up the container

Same pattern as the rest of my attack environment: a disposable Incus
container, this time on Debian rather than Kali, since none of the offensive
tooling in Kali was relevant here — I just needed a Linux box that could run
Wine.

```bash
incus launch images:debian/13 deb-wine
incus config device add deb-wine x11-socket disk source=/tmp/.X11-unix path=/tmp/.X11-unix readonly=true
incus config device add deb-wine dri disk source=/dev/dri path=/dev/dri
incus config set deb-wine raw.lxc 'lxc.namespace.share.ipc=1'
```

The X11 socket and `/dev/dri` passthrough are the same idea as the GUI setup in
the Incus post — ILSpy and the PowerShell terminal both render as native
windows on my host's X server, GPU-accelerated, without anything Windows-shaped
touching my actual display stack. Sharing the IPC namespace is required for
running programs with WINE natively on the host's X server. There are security
implications to this workaround, namely, it is not a reasonable amount of
isolation for actual malware analysis. If you're disassembling or running
anything you suspect is malicious, don't reach for this setup — use a properly
isolated VM, snapshotted and network-isolated, instead.

## Wine, 32-bit and a non-root user

Wine needs the i386 architecture enabled before anything else, since a fair
amount of the Windows ecosystem still ships 32-bit binaries even on a 64-bit
host:

```bash
dpkg --add-architecture i386 && apt update
apt-get install wine wine32 wine64 winbind winetricks zenity
```

I deliberately didn't run any of this as root past the initial setup.
Reverse-engineering tooling means running unknown or semi-trusted binaries by
design, and Wine's prefix is just a directory tree under whatever user runs it
— there's no reason to give that directory tree root's permissions when a
dedicated user does the same job with less blast radius if something inside the
Wine prefix turns out to be hostile.

Worth being explicit about the limits of that reasoning, though: it made sense
here because the target was a client application for a corporate workflow, not
malware. A non-root user inside a container is a reasonable amount of isolation
for that.

```bash
useradd -m -s /bin/bash wineuser
passwd wineuser
```

## Getting .NET actually working

This is where the one real snag showed up. As `wineuser`:

```bash
winetricks dotnet48 dotnet9 dotnetdesktop9 powershell_core powershell corefonts
```

`dotnet48` is in there for ysoserial.net specifically — it's built against the
old .NET Framework 4.8, not anything modern, and Wine's winetricks support for
it is mature at this point. The more interesting constraint was on the other
end: I'd originally wanted .NET 10, since that's current, but Debian 13
stable's packaged winetricks doesn't have a verb for `dotnet10` yet — it knows
`dotnet9` and `dotnetdesktop9` just fine, but support for 10 hadn't landed in
the version stable ships. Rather than chase down a newer winetricks for one
version bump, I settled for .NET 9 and picked an ILSpy release built against it
instead of the newest one available. Same trade-off as running Debian stable on
the host in the first place: not always the latest, but not worth fighting for
one version number when 9 does everything I need here.

Once that was sorted, both runtimes installed cleanly, and `winetricks` quietly
handled the part I least wanted to do by hand: getting a correct .NET 9 runtime
and desktop runtime, plus the legacy 4.8 framework for ysoserial.net,
registered inside a single Wine prefix.

From there, `wine powershell` gets me a PowerShell session without leaving my
terminal, which I ended up preferring over `wineconsole powershell` and its
separate graphical window — fewer things to alt-tab between while I'm already
juggling a debugger and a packet capture.

## ILSpy and ysoserial.net

ILSpy needed to be a version built against .NET 9 rather than the newest
release, to match the runtime I'd actually settled on:

```powershell
iwr https://github.com/icsharpcode/ILSpy/releases/download/v9.1/ILSpy_binaries_9.1.0.7988-x64.zip -o ilspy.zip
expand-archive ilspy.zip
ilspy\ILSpy.exe
```

Et voilà! ILSpy is running from the container:

![ILSpy running from inside the container](/images/blog/wine-environment-ilspy.png)

ysoserial.net followed the same pattern — download, extract, run from inside the prefix:

```powershell
iwr https://github.com/pwntester/ysoserial.net/releases/download/v1.36/ysoserial-1dba9c4416ba6e79b6b262b758fa75e2ee9008e9.zip -o ysoserial.net.zip
expand-archive ysoserial.net.zip
```

And from `ysoserial.net\Release\`:

```powershell
PS Z:\home\wineuser\ysoserial.net\Release> .\ysoserial.exe
```

Output:
```text
Missing arguments. You may need to provide the command parameter even if it is being ignored.
ysoserial.net generates deserialization payloads for a variety of .NET formatters.
== GADGETS ==
       (*) ActivitySurrogateDisableTypeCheck [...]
       (*) ActivitySurrogateSelector [...]
<SNIP>
```

A native Windows binary, listing its full gadget chain menu, running under Wine
inside a Debian container, inside Incus — at that point the setup had done its
job.

## Where this actually got used

This is the environment behind the .NET reverse-engineering and deserialization
work I ran into on HTB's [Scrambled](/writeups/htb-scrambled/) and Pov —
running the target binary, generating payloads with ysoserial.net, and
decompiling with ILSpy, all without a Windows VM anywhere in the loop. The
iptables version would have worked fine and probably taken less time. This one
just answered a question I actually wanted answered: that a Windows-only RE
toolchain can live inside the same kind of disposable container I already use
for everything else, with no VM and no extra routing required at all.
