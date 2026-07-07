---
title: "From Kali VM to Incus: Building a Pentest Lab That Doesn't Fight Back - Part 1"
date: 2026-06-26
tags: ["linux", "containers", "infrastructure"]
categories: ["infra"]
summary: "Why I moved my pentest environment through four iterations — VM, systemd-nspawn, Exegol, and finally Incus — and what each step taught me about Linux internals. - Part 1"
---

I was running the [GOAD](https://orange-cyberdefense.github.io/GOAD/) lab on my
laptop — 32 GB of RAM, which on paper should be plenty for most things — and
watching the memory usage climb past what I was comfortable with. GOAD on its
own was already eating most of it. If I had also wanted to run a full Kali VM
on top of that to actually attack the lab, I simply wouldn't have had the RAM
left to do it. Not a budget problem in terms of money — a budget problem in
terms of memory.

That was the moment a requirement became concrete for me, rather than just a
vague preference: whatever I used to run my attack environment, it could not
allocate a full, fixed chunk of memory and a second kernel for every single
instance, the way a VM does. That's simply how virtualization works, and
there's no configuration that gets around it.

There's a second, separate reason VMs didn't fit my workflow, and it has
nothing to do with RAM: GPU sharing. I wanted GPU acceleration both on my host
system and inside the attack container at the same time — partly for GUI tools
like Wireshark or ZAP, but more importantly for password cracking with hashcat,
where GPU acceleration is the difference between a wordlist attack finishing in
minutes versus not finishing at all. On its own, losing GPU acceleration would
have been annoying rather than disqualifying, but offloading hashcat to the
host wasn't an appealing alternative either: it would have meant installing and
maintaining my own set of wordlists on the host, on top of the ones that
already come with Kali and already take up disk space there. I didn't want to
duplicate that. So I didn't want to live without GPU acceleration in the
container if I didn't have to — and a VM with passthrough would have meant
handing the entire GPU over to one VM exclusively, leaving the host without it
for as long as the VM was running. That's a different limitation than the
memory one, but it pointed in the same direction: I needed something where
the line between "host" and "attack environment" was thinner than a full
virtual machine.

It had to be containers, sharing the host kernel and host GPU rather than each
one bringing its own. But getting from "it has to be containers" to a setup I
was actually happy with took a few wrong turns first — and looking back, I
think the wrong turns taught me more than the final answer did. So let me back
up a bit.

## My linux journey prior to cybersecurity

I started using Linux casually in high school, mostly out of curiosity rather
than necessity — distro-hopping a bit, breaking things, fixing them, the usual
entry point. It stayed a casual thing for a while: comfortable enough on the
command line to get around, but without much understanding of what was actually
happening underneath.

That changed when I switched to Arch Linux as my daily driver. Arch doesn't
hold your hand, and that's exactly what made it useful as a learning tool.
Installing it the first few times meant actually understanding what a
bootloader does, rather than clicking through an installer that handles it for
you. From there it kept going: picking a window manager and a display manager
and understanding what each one is actually responsible for, figuring out the
difference between Xorg and Wayland and why that distinction matters for things
like screen sharing or GPU drivers, setting up a sound server and dealing with
the inevitable PulseAudio vs. PipeWire questions, and generally getting
comfortable reading and writing config files instead of treating them as
something to copy-paste and hope for the best. None of this was
cybersecurity-related at the time. It was just normal daily-driver maintenance.
But it built the kind of intuition for how a Linux system fits together that
I'd later rely on constantly.

The more deliberate, security-focused learning came later, once I started
preparing for certifications — first RHCSA, then more security-specific
material. That's also when I set up my first proper home lab using QEMU/KVM,
mostly to have disposable, snapshot-able machines to practice on without
risking my daily system. It worked well enough for a while, and it's where this
whole story about containers really starts.

There's one more thing containers gave me that I didn't fully appreciate until
later: the ability to spin up several isolated environments at once without the
memory cost multiplying every time. I haven't built this out properly yet, but
the idea of running a handful of containers side by side — each with its own
network namespace, each able to have different firewall rules — for testing how
something behaves across a small simulated network, is the kind of thing that's
realistic with containers and basically isn't with VMs, simply because of how
much memory five VMs would cost compared to five containers sharing one kernel.
It's still more of a "this is clearly possible and I want to explore it
properly" item than something I've fully exploited, but it was part of the
appeal early on.

With the VM question settled, the next decision was which kind of container.
And the first answer that comes to mind for almost anyone these days is Docker.

## Docker

Docker was the obvious first thing to try, since it's what almost everyone
reaches for when they need an isolated, reproducible environment. And to be
fair, it does what it's designed to do well. But the way it does it ended up
being exactly the part I didn't like, for reasons that took me a little while
to put into words properly.

Docker containers are built from OCI images, which are made up of a stack of
immutable layers, mounted together at runtime using overlayfs. Each layer is
its own thing — content-addressed, named by a hash, sitting somewhere under
Docker's own directory structure — and the overlay filesystem stitches them
together into what looks, from inside the container, like a single normal
filesystem. It's a genuinely clever design, and there are good reasons it works
this way: layers can be shared between images, cached, distributed
independently, and so on.

But from the outside, on the host, it's not transparent at all. If I want to
look at a container's root filesystem directly — browse it, grep through it,
copy a file out of it without going through `docker cp` — what I actually find on
disk is a pile of hash-named directories scattered around Docker's storage
backend, not something I can just cd into and recognize. Even when the
underlying storage is a btrfs snapshot rather than plain overlayfs, the same
problem holds: the container's filesystem isn't something that presents itself
to me as a directory I'd recognize.

That mattered to me for a very practical reason. After finishing a box or
wrapping up an assignment, I like to go back and pull out my command history,
tmux logs, notes, anything useful I left lying around in the container, and
back it up properly before I either reset the container or move on. With a
container whose root filesystem just lives in a directory on the host, that's
trivial — I go to the directory, I copy what I need. With Docker, it's not
impossible, but it's a layer of indirection I didn't want to deal with every
single time. Given how often I wanted to do this, the convenience of "it's just
a directory" outweighed everything Docker's layering model gets you in return,
at least for my use case.

This single preference — wanting a container's filesystem to be visibly,
directly a directory rather than something abstracted behind a storage driver —
turned out to be the thread that ran through most of what I tried next. It's
also, as it turned out later, not as simple a preference to satisfy as I
initially thought.

## chroot

Before getting into the more complicated container tooling, I want to mention
chroot, mostly because I was already familiar with it from a completely
different context. I'd used chroots plenty of times before, for installing Arch
Linux, or for setting up Debian or Alpine chroots to experiment in. The setup
is about as minimal as it gets: a root filesystem somewhere on disk, the usual
device nodes, `/sys` and `/proc` mounted in, maybe a working directory
bind-mounted in for saving output — and that's pretty much the whole list.

Isolation-wise, chroot gives you very little. There's no separate kernel, no
separate process namespace by default, nothing stopping a process inside from
seeing or affecting things outside, beyond the filesystem root being different.
In my case, that specific gap didn't bother me much, since I'm not in the habit
of running untrusted payloads inside these environments in the first place —
the threat model where chroot's weak isolation would actually matter just
doesn't come up for how I use it.

What did bother me was the lack of network isolation, for much more mundane
reasons. Without a separate network namespace, anything listening on a port
inside the chroot is really listening on the host's network stack, which means
collisions. BloodHound wants Neo4j and sometimes Postgres on their usual ports;
SMB wants its usual ports; and if I'm being a bit lazy and just spin up `python
-m http.server` on port 8000 to serve a payload, there's a decent chance that
collides with some host service I forgot was already using it. None of this is
a hard blocker — you can always reconfigure ports — but it's the kind of small
friction that adds up and made me want something with proper network namespace
isolation, even if I didn't need much else.

So: easy to set up, genuinely useful for quick experimentation, but not quite
enough on its own. The natural next step, since I was already comfortable in
the systemd ecosystem, was systemd-nspawn.

## systemd-nspawn

`systemd-nspawn` felt like a natural next step, since I was already deep in the
systemd ecosystem and nspawn is, in a sense, chroot's much more capable sibling
— proper namespace isolation, its own network interface if you want one, and
`machinectl`/`systemd-run` integration so a container can be managed like any
other systemd unit rather than a process you just leave running in a terminal.

The version I actually used day to day ended up looking like this, trimmed down
to the relevant parts:

```bash
#!/usr/bin/env bash

CONT=kali

SERVICE_ARGS=(
  --unit="$CONT"
  --service=notify
)

NSPAWN_ARGS=(
  --quiet
  --keep-unit
  --boot
  --notify-ready=yes
  --directory="$PWD/kali"
  --machine="$CONT"
  --hostname="$CONT"
  --network-bridge=br0
  --bind-ro=/tmp/.X11-unix
  --bind-ro=/run/user/1000/pipewire-0:/tmp/runtime/pipewire-0
  --bind-ro=/run/user/1000/pulse:/tmp/runtime/pulse
  --bind=/dev/dri
  --bind=/dev/shm
  --bind-ro=/lib/modules
  --bind-ro=/sys/module/compression
  --bind-ro=/usr/lib/x86_64-linux-gnu/nvidia
  $(ls -1d /etc/OpenCL /bin/nvidia* /dev/nvidia* /sys/module/nvidia* /usr/share/nvidia /etc/nvidia 2>/dev/null \
    | sed -e 's/^/--bind-ro=/' -e '/\/$/d')
  --bind="$WORKSPACE":/workspace
)

sudo systemd-run "${SERVICE_ARGS[@]}" systemd-nspawn "${NSPAWN_ARGS[@]}"

```

A few things worth pointing out here. The container gets the host's X11 socket
and PipeWire/PulseAudio sockets bind-mounted in, so GUI tools run with proper
audio and a real display, not some headless or VNC workaround. It gets `/dev/dri`
and a pile of nvidia-related paths bind-mounted in too, which is what makes GPU
acceleration available inside the container for GUI apps and hashcat alike. And
the whole thing runs through `systemd-run`, wrapping the nspawn invocation as a
proper transient systemd service rather than a bare process — `--service=notify`
and `--notify-ready=yes` mean systemd actually knows when the container is up,
instead of just assuming it worked.

That `ls | sed` line generating the nvidia bind-mounts was, in hindsight, the
naive way to do it — I later set up nvidia-container-toolkit when I moved to
Incus, and realized its output (a properly generated list of every
nvidia-related file and device node a container needs) would have made this
line a lot less error-prone. It wouldn't have changed what this setup could or
couldn't do, just made getting there less painful.

I built the argument list as an array on purpose, rather than writing the
command inline or using backslash line continuations. With an array, commenting
out a single flag during debugging is just prefixing that line with `#`. With
inline arguments or backslash continuations, removing or disabling one flag
without breaking the line structure is a lot more awkward. It's a small thing,
but it made iterating on the config noticeably less annoying.

The actual recurring pain was networking. `--network-bridge=br0` means nspawn
wants to manage that bridge and the container's virtual interface itself, and
the natural way to configure that on the systemd side is with systemd-networkd.
The problem is that I run NetworkManager on the host, and the two were not
willing to share. Every time I started the container, I'd end up with both of
them trying to configure the same interface, and the routing table would end up
in some half-broken state until I sorted out which one had won that round. The
actual fix — just let NetworkManager handle it instead of systemd-networkd —
works, but NetworkManager's own configuration for virtual bridges and
container-facing interfaces isn't exactly straightforward either, so "the fix"
was its own small project.

This setup did the job, and it's what I used for a good while. But at some
point I asked myself a question that exposed its real ceiling: what if I wanted
to run this exact setup on a machine that didn't use systemd at all? The whole
thing — systemd-run, systemd-networkd or NetworkManager fighting over an
interface, machinectl — only makes sense on a systemd-based host. That's more
of a hypothetical problem for me specifically, but it bothered me enough as a
design constraint that I went looking for something with less of that built-in
assumption. Which led to a more ambitious attempt at solving the same problem,
still with nspawn, before I gave up on nspawn entirely.
