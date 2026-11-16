# 2026-11-16 13:08:17 Swapping firewalls without an unfiltered window

## Question

> The part I would worry about is the swap itself. A package upgrade or removal usually stops or restarts the service, and stopping Foomuuri deletes its table, so the host is unfiltered for a moment. I have not checked what Debian's packaging does here. With your own package you decide.
>
> * Assumtion, you can not install the foomuuri managed firewall without first removing the old one since most firewalls are interfaces for the same underlying nftables
> * Options:
>   * kill or redirect network while the change takes place

## Answer

The assumption does not hold, which is good news: you can install the Foomuuri firewall while the old one is still in place, so there is no need to kill or redirect the network.

### Why the assumption fails

Different managers are front ends to the same nftables, but each keeps its rules in its own table, and the kernel runs all tables side by side. firewalld has `firewalld`, Foomuuri has `inet foomuuri`, and ufw's rules sit in the iptables-compatibility tables.

A packet has to be accepted by every table, and a drop in any one is final. So while two firewalls overlap, the host enforces the stricter of the two. It is never more open than either.

That allows make-before-break:

1. Load Foomuuri's table while the old firewall is still active.
2. Confirm SSH still works.
3. Remove the old manager's rules.

The risk during the overlap is over-blocking, not exposure.

### The case I was actually worried about

My sentence was about a narrower swap: replacing one Foomuuri package with another. Both use the same table name, so they cannot overlap. But the rules live in the kernel, not in a running process, and replacing files on disk does not touch them. The only thing that opens the firewall is a package script running `foomuuri stop`. There are two ways to prevent that:

- **Stop the scripts from stopping the service.** On Debian-family hosts a temporary `policy-rc.d` blocks service actions during the install; then run `foomuuri reload`, which swaps the table atomically. In your own package you can turn off restart-on-upgrade for good.
- **Load a guard table first.** A small independent table, for example `inet swap_guard`, drops inbound traffic except established connections, loopback and SSH. Because a drop anywhere is final, the host stays filtered even if `inet foomuuri` vanishes for a moment. Delete the guard once the new Foomuuri is loaded.

The guard table is the gentle version of your option. Killing or redirecting the network also works, but over SSH it cuts the connection Ansible is using, and it breaks established sessions for everything else on the host.

Both techniques are from my knowledge of the tools, not tested here.

### Two things to watch

- **Anything that runs `flush ruleset` wipes every table**, including Foomuuri's and a guard. The distribution's `nftables.service` is the usual culprit when it is stopped. I believe Debian's unit does this on stop, but I have not checked. Such services must be stopped before Foomuuri loads, or masked without being stopped.
- **The role currently does break-before-make.** `tasks/main.yml` runs the conflicts step, including `ufw disable`, before "Start and enable foomuuri". On a host with ufw active, that leaves a few seconds with no firewall. Moving the Foomuuri start ahead of the ufw and firewalld steps would close it.

Want me to reorder the role that way?
