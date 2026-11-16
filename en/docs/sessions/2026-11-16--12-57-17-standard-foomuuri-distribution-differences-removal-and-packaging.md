# 2026-11-16 12:57:17 Standard Foomuuri: distribution differences, removal and packaging

## Question

> if it goes nftables provided with kernel and Foomuuri compliles rules based in simplified file then I don't see what the distribution issue is if we can just have a standard foomuuri... The major challenge seems to be what to unistall, and remove and how to deal with distro's file system organizations

## Answer

You are mostly right: for Foomuuri the distribution problem is thin. One correction first, then your two challenges.

**The correction.** The kernel half of nftables is the same everywhere, but the `nft` command Foomuuri calls is a distribution package, and so is Python. Foomuuri needs nftables 1.0 or later with JSON support and Python 3.9 or later. Every platform in your scope meets that, so it is a floor to assert, not a problem to solve.

**What actually differs per distribution:**

| Thing | Differs? |
| --- | --- |
| Foomuuri's rule language and output | No |
| Install paths | Barely. Upstream's `make install` uses `/usr/sbin`, `/usr/lib/systemd/system` and `/usr/share/foomuuri`, which are the same on Debian, Ubuntu, Mint, Fedora and EL. Arch is the exception, with `/usr/bin` |
| Dependency package names | Yes, slightly: `python3-gi` on Debian, `python3-gobject` on Red Hat |
| Package format | Yes: `.deb` and `.rpm` |
| SELinux on the Red Hat family | Unknown, to verify |

### Filesystem organisation

This is a smaller problem than it looks. Upstream's Makefile takes `DESTDIR`, `BINDIR` and the systemd unit directory, so one staged tree is the standard Foomuuri. The difficulty is ownership, not layout: the distribution's package and yours would claim the same paths.

### What to remove

Less than you might expect, if the standard package keeps the name `foomuuri` and carries a higher version:

- **The distribution's Foomuuri:** nothing to uninstall. The package manager treats yours as an upgrade in place, and will not downgrade back.
- **Other firewall managers:** stop, disable and mask, not uninstall. The role already does this, including `ufw disable` so ufw's rules leave the kernel.
- **Stray configuration in `/etc/foomuuri`:** the role's purge already handles it.

The part I would worry about is the swap itself. A package upgrade or removal usually stops or restarts the service, and stopping Foomuuri deletes its table, so the host is unfiltered for a moment. I have not checked what Debian's packaging does here. With your own package you decide.

* Assumtion, you can not install the foomuuri managed firewall without first removing the old one since most firewalls are interfaces for the same underlying nftables
* Options:
  * kill or redirect network while the change takes place

### What this means for build or package

Your point narrows the gap between the two routes I described. Building is one `make install` into a staging directory; a `.deb` and an `.rpm` are thin wrappers around that same tree. I would still use a real package over a role-managed copy of files, for two reasons:

- **The package manager tracks the files.** Upgrades remove what a new release dropped, and removal is clean. A role that copies files has to track that list itself.
- **Your raw node producers need Foomuuri before Ansible arrives.** The contract has the node handed over with Foomuuri already running. An installer can consume a package directly; it cannot run the role.

So the standard Foomuuri is one pinned upstream release, packaged once per format and used by both the producers and the role. Debian's `.deb` is still the fastest way to start on the Debian family, with your own thin package as the route to one definition across both families.

Shall I draft this as a design note, with the swap-without-opening-the-firewall check as its first test?
