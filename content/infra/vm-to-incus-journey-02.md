---
title: "From Kali VM to Incus: Building a Pentest Lab That Doesn't Fight Back - Part 2"
date: 2026-06-28
tags: ["linux", "containers", "infrastructure"]
categories: ["infra"]
summary: "Why I moved my pentest environment through four iterations — VM, systemd-nspawn, Exegol, and finally Incus — and what each step taught me about Linux internals. - Part 2"
---

## systemd-nspawn upgraded

The idea behind the next attempt was simple enough on paper: instead of
maintaining one fixed Kali install that I kept upgrading in place, build a
clean base image once, and spin up disposable or named instances from it as
copy-on-write overlays, so the base stayed untouched and each instance only
ever wrote its own diff on top.

The base image lived as a btrfs subvolume, bootstrapped with debootstrap
directly from Kali's repositories:

```bash
MACHINES=/var/lib/machines
TEMPLATE=kali-base

bootstrap() {
  btrfs subvolume create "$MACHINES/$TEMPLATE"

  debootstrap --components=main,contrib,non-free \
    kali-rolling "$MACHINES/$TEMPLATE" \
    http://http.kali.org/kali

  # install packages, enable services, etc. inside the fresh base
}
```

And instances were created from that base using nspawn's --overlay option,
which takes a lower (read-only) directory and an upper (writable) directory and
presents the combination as a single root filesystem to the container:

```bash
template-container() {
  systemd-run --unit="nspawn-$NAME" --service-type=notify \
    systemd-nspawn \
      --keep-unit --boot \
      --overlay="$MACHINES/$TEMPLATE:$MACHINES/$NAME:/" \
      --machine="$NAME" \
      --hostname="$NAME" \
      "${GPU_ARGS[@]}" "${GUI_ARGS[@]}"
}
```


I also wrapped the whole thing in a small command-line interface using getopt,
so I could do things like `spawn.sh start --name pentest1 --workspace
~/hack/machines` and have it build the right overlay arguments and bind mounts
on the fly, rather than editing the script by hand for every new instance.

This actually worked, and it worked surprisingly well at first. Spinning up a
new named instance was fast, each one kept its own changes cleanly separated
from the base, and disposing of an instance was as simple as deleting its
overlay directory, leaving the base untouched. I originally thought of this
mostly as a way to avoid manually managing btrfs snapshots myself, and as a way
to keep changes isolated into a single diff rather than mixed into the base.

Then it broke, in a way that taught me something I should have seen coming.
Kali is a rolling release, and at some point I upgraded the base image to pick
up newer packages — mainly to save bandwidth, since upgrading once on the base
meant every instance built from it afterward would already be current, instead
of redownloading the same updates separately in every overlay. But I had also
installed and upgraded a few things directly inside one of the running overlay
instances before that base upgrade happened. The result was a library version
mismatch between what the base now provided and what an application inside that
overlay instance expected, and the application just stopped working correctly.

That's when it clicked: this is exactly the problem OCI's layered image model
exists to prevent. Docker doesn't just use one read-only layer and one writable
layer — it uses a stack of independently versioned, content-addressed layers
specifically so that this kind of "the bottom layer moved out from under the
top layer" conflict can't happen the way it just had to me. I had, without
quite intending to, built a simplified, single-layer version of the same idea
Docker already solves properly, hit the exact failure mode that Docker's more
complex design exists to avoid, and arrived at a real appreciation for why that
complexity is there in the first place.

And yet, knowing that didn't change my mind about Docker. The
directory-transparency objection from earlier was still just as true for a
properly layered system as it was for my simplified one — multiple
hash-addressed layers are, if anything, less transparent than my single overlay
diff, not more. I understood why Docker's model exists, and I still didn't want
to use it for this particular job. So instead of going back to Docker, or
trying to build a more correct, multi-layer version of my own overlay scheme
myself, I went looking for something that had already solved this properly
without giving up on the "it's just a directory" property I cared about.


## Exegol

Before I found that something, I went through one more stop: Exegol. I hadn't
even known about it until I came across it while working through the GOAD lab —
it kept coming up as a recommended way to set up an attack environment for that
lab, so I gave it a try.

And to be honest, it's a genuinely good tool. It's easy to get running, it
works on Windows as well as Linux, which is more than I can say for my own
setup, and the tool selection inside it isn't bloated with everything under the
sun — it's a curated list that happens to match almost exactly what I'd
actually reach for. The logging is well thought out, and the defaults are
sensible enough that I didn't feel like I needed to reconfigure much before
getting to work.

What ultimately moved me away from it had less to do with Exegol itself and
more to do with how I prefer to learn a new tool. My usual approach is to first
understand the underlying concept properly, then get a general sense of what
tools exist for it, and only after that lean on man pages and `--help` output
for the exact syntax, until I've used a command enough times that the common
flags stick in memory without needing to look them up. Man pages and info
pages are the backbone of that last step for me. Exegol's container image
is configured, deliberately, to skip installing documentation — no man
pages, no info pages — which makes sense from Exegol's side, since it keeps
the image smaller and faster to pull, but it broke that part of my workflow
specifically.

I did consider fixing it myself: get inside the container, change the APT
configuration to stop excluding documentation, then update or reinstall the
relevant packages to actually pull the man pages in. But thinking it through,
that's effectively rebuilding the image from inside a running instance — and
for a disposable container, that didn't make sense. If it had been a persistent
base image I was going to reuse for a long time, the one-time cost might
have been worth it. For something I'd spin up, use for one engagement, and
throw away, it wasn't. At that point the reasoning was simple: why go
through the effort of patching someone else's image back into the shape I
wanted, when I could just use Kali directly and have it that way from the
start.


On top of that, a few smaller things added up rather than any single
dealbreaker: the license terms weren't something I was fully comfortable with
for my use case, and the free community edition lagged behind in a way that
bothered me more than it might bother someone else. I genuinely value running
Debian stable on my host system specifically because of how stable it is, but
inside my attack environment I want the opposite trade-off — the latest version
of every tool, an apt update away, without waiting on a community edition's
release cycle to catch up.

None of these were dramatic problems on their own. But together, they were
enough that Exegol didn't become my long-term setup, even though I'd genuinely
recommend it to plenty of people who don't share my specific set of
preferences. And the underlying objection to Docker — the one I'd already
arrived at, and then re-confirmed for myself while building my own overlay
scheme — was still sitting underneath Exegol the whole time, since it's a
Docker-based tool itself. That objection had now survived three different
attempts to get around it. At that point it stopped feeling like a preference I
should keep working around, and started feeling like a real requirement.


## Incus

Before getting to what I actually settled on, here's the whole journey side by side:

| Approach| Isolation | Setup effort | Persistence / visibility | Verdict |
|---------|-----------|--------------|--------------------------|---------|
QEMU/KVM VM|Full (separate kernel)|Low|Full, but fixed memory cost per instance|Too heavy for what I needed|
Docker|Namespace-level (OCI layers)|Low|Full, but hidden behind hash-named layers|Wrong storage model for my workflow|
chroot|Filesystem-level only|Very low|Full, plain directory|Too little isolation (port collisions)|
systemd-nspawn (flat)|Namespace-level|High (manual GPU/network config)|Full, plain directory|Worked, but systemd-only and fragile networking|
systemd-nspawn (overlay)|Namespace-level|High (custom tooling)|Full, plain directory, single CoW layer|Worked until a base upgrade broke an instance|
Exegol|Namespace-level (Docker-based)|Low|Full, but same Docker storage objection|Great tool, wrong fit for my learning workflow|
Incus|Namespace-level|Medium|Full, plain directory|Settled here|

Every row after the VM is really the same underlying question asked a different
way: how do I get proper isolation and a persistent, directly visible
filesystem, without paying for a second kernel per instance. Incus turned out
to be the first answer that didn't ask me to give up something on that list.

## Incus details

Incus ended up being the simplest part of this whole story to write about,
mostly because it just does the things I'd been trying to build or work around
for months, without making me think about it.

Creating a container is genuinely easy — `incus launch images:kali/current
kali-main` and a minute later I have a running instance, with the heavy lifting
of getting a current, correct Kali image handled for me. Compared to
hand-rolling a debootstrap setup or maintaining my own overlay tooling, this
alone would have saved me a lot of the time I spent on the nspawn attempts.

The part that matters more to me, though, is how a container behaves once it's
running. It feels like a real, mutable system — closer to a normal Debian
install I can poke at, install things into, and leave in whatever state I left
it in — rather than something that gets rebuilt from a declarative spec the
moment anything changes, the way NixOS or a Dockerfile-based image does. That's
not a knock on declarative, rebuild-on-change systems; there are good reasons
people like that model. It's just not the one I wanted for an environment I'm
actively working in day to day, and it's the same underlying preference that
pushed me away from Docker's layered images and made my own single-overlay
scheme appealing in the first place: I want to open a directory and have it be
the actual, current filesystem of the container, not a derived artifact of some
build process.

GPU support works well, including the part I cared most about after the
GUI/hashcat reasoning from earlier: Xorg socket sharing. GUI tools like ZAP,
Firefox, or Wireshark run inside the container and render natively on the
host's X server, with GPU acceleration, without touching any library version on
my actual Debian install. It feels genuinely native, in a way none of my nspawn
bind-mount lists managed to fully replicate, and I got there with a small
fraction of the manual configuration the nspawn setup needed —
`nvidia-container-toolkit` integration handles the enumeration that I used to do
by hand with `ls` and `sed`.

Networking solved the other recurring headache outright, just by not being
involved in the same way nspawn was: Incus doesn't rely on systemd-networkd as
its own idiosyncratic way of configuring interfaces, so it simply doesn't end
up fighting with NetworkManager over the same bridge or interface the way
nspawn did. The conflict that used to mess up my routing table every time I
started a container just isn't there to begin with.

I'll be honest that I haven't gone deep into everything Incus can do yet. My
CPTS exam was coming up while I was getting this set up, and between studying
and actually using the container day to day, I didn't have the time to properly
explore its networking model beyond what I needed to get a single container
working well — let alone build out the multi-container,
per-segment-firewall-rules lab idea I mentioned earlier. That's deliberately
left for later, not because it isn't possible, but because I haven't gotten to
it yet.

In the end, the setup held up exactly the way I'd hoped it would. I ran this
Incus container, on the same Debian install, through all ten days of the CPTS
exam without a single hiccup, and without ever needing to reboot my laptop in
between. GPU-accelerated GUI tools stayed buttery smooth the entire time, and
not having to think about port collisions — running HTTP servers, database
servers, or anything else on whatever port I needed, whenever I needed it —
removed a whole category of small annoyances I'd otherwise have had to manage
manually. For an already stressful ten days, not having to worry about my own
environment on top of everything else mattered more than I expected it to going
in.

There's still plenty I want to explore properly. Incus's networking model is
the obvious next step, especially the idea of running several containers
together with their own firewall rules to simulate small segmented networks.
Beyond that: resource limits, since I haven't touched CPU or memory quotas at
all yet; snapshots and container copies, to get a fresh, ready-to-go
environment for a new engagement without repeating setup steps each time; and
probably some scripting on top, to automate that setup once I have a
snapshot-based workflow worth automating. None of this is urgent — the current
setup already does what I need it to — but it's the natural direction to keep
pushing in.

For now, though, I have a setup that survives a ten-day certification exam
without complaint, gives me GPU acceleration where I actually need it, and lets
me treat my attack environment like a real, working system instead of something
I have to constantly fight with. That's a pretty reasonable place to land,
after going through a VM, a chroot, two different attempts with systemd-nspawn,
and a Docker-based tool to get here.
