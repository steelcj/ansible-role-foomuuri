# Firewall role: support, coverage, edge cases and estimate

Date: 2026-11-15

Author: Christopher Steel

## Summary

Making this role capable of complex firewalls across distributions is about 20 to 31 focused working days, roughly 4 to 6 weeks for one experienced person. A useful first release (defect fixes plus a richer rule model on ufw and firewalld) is 6 to 9 days.

The recommended shape is a fork with three layers:

- **Native backends keep the allow rules.** ufw on Debian-family hosts and firewalld on Red Hat and SUSE-family hosts, driven by a richer common rule model.
- **An nftables backend replaces the iptables template.** It serves Alpine, and any host where you choose to run nftables alone.
- **An nftables overlay table carries the extras.** Address sets, rate limits, NAT and extra drops load beside ufw or firewalld without editing their rules.

The alternative is a new nftables-only role with no ufw or firewalld support, at about 8 to 12 days. It is cheaper and more expressive, but it replaces each distribution's default firewall manager.

All estimates here are the analyst's judgement from reading the role, not measured figures. This analysis is based on commit `1f32061` of robertdebock/ansible-role-firewall <a id="cite-debock-2026"></a>[(de Bock, 2026)](#ref-debock-2026).

## Current state

The role can only express "open this port and protocol to everyone". Each entry in `firewall_services` carries `name`, `protocol`, `state`, and on ufw only, `rule`.

### What each backend receives

| Backend | Selected for | Mechanism | Keys used |
| --- | --- | --- | --- |
| ufw | Debian, Ubuntu, and any unmapped distribution | `community.general.ufw` | `rule`, `name`, `protocol` |
| firewalld | RHEL and CentOS 7 to 9, Fedora, Oracle Linux, openSUSE Leap | `ansible.posix.firewalld` | `name`, `protocol`, `state` |
| iptables | Alpine only | `templates/iptables.j2` | `name`, `protocol`, `state` |

### Defects found by reading the code

These were found by inspection and have not been reproduced on a host.

- **Service and package maps disagree.** Packages are chosen by OS family, the service by exact distribution name. Rocky, AlmaLinux, EL 10, SLES and openSUSE Tumbleweed get firewalld installed but `ufw` as the service.
- **The template is fixed.** `src: iptables.j2` is hard-coded, and the rule file path is defined for Alpine only.
- **The template ignores two defaults.** It hard-codes `tcp` and always accepts, so `firewall_default_protocol` and `firewall_default_rule` have no effect there.
- **`state: absent` on ufw is not idempotent.** The open task has no state filter, so the rule is added and then deleted on every run.
- **The assert rejects valid ufw actions.** Only `allow` and `deny` pass, so `reject`, `limit` and `block` all fail.
- **Backend choice is locked in `vars/`.** Only role parameters or extra vars can override `firewall_service` and the package lists.
- **The test suite does not test the firewall.** Molecule runs in Docker, and every firewall task carries the guard `ansible_connection not in [ "container", "docker", "community.docker.docker" ]`, so all of them are skipped. The guard matches connection names, not whether the target has its own network stack.

## Support required

The work falls into four groups, in the order they should be built. Most of the first two groups is passing extra optional keys to modules that already support them.

### Rule model

A new `firewall_rules` list sits beside `firewall_services`, so existing playbooks keep working.

| Capability | ufw | firewalld | nftables |
| --- | --- | --- | --- |
| Source and destination address | Module keys `from_ip`, `to_ip` | Rich rule, or source bound to a zone | Native |
| Interface in and out | Module keys `interface_in`, `interface_out` | Interface bound to a zone | Native |
| Port ranges and port lists | Native | Native for ranges, one entry per port for lists | Native |
| Action: accept, drop, reject | Native | Rich rule or zone target | Native |
| Rate limit | `limit` only, fixed threshold | Rich rule `limit` | Native, tunable |
| Per-rule logging | Module key `log` | Rich rule `log` | Native |
| Direction: in, out, forward | `direction` and `route` keys | Policies between zones | Native |
| Explicit ordering | `insert` key | Rich rule `priority` | File order |
| Comment | Module key `comment` | Not on plain ports | Native |
| ICMP control | Not in the command line, needs `before.rules` | ICMP blocks per zone | Native |

### Policy and state

- **Default policies** for incoming, outgoing and routed traffic, validated the same way on every backend.
- **Idempotent removal** of rules marked absent.
- **Optional purge** of rules the role did not declare, off by default.
- **Lockout protection:** a management-source SSH rule applied before any policy tightens, plus a timed rollback.
- **Input validation** through `meta/argument_specs.yml`, replacing the hand-written asserts.

### Backend selection

- Move `firewall_service`, package lists and file paths from `vars/` to `defaults/`.
- Fix the distribution maps so service and packages are chosen by the same key.
- Add an nftables backend rendered from a template, checked with `nft -c -f`, loaded atomically.
- Make the template path a variable so a playbook can supply its own.
- Retire the iptables template once nftables covers Alpine.

### Advanced features

- **NAT and port forwarding**, with the matching forwarding sysctl.
- **Named address sets** for allow and block lists.
- **Zones** on firewalld, mapped to interface groups elsewhere.
- **Raw rule escape hatches**, covered under edge cases below.

## Distribution coverage

With the map fixes and an nftables backend, the fork can cover every mainstream family below. The Today column is what the current role is expected to do, from reading its maps.

| Family | Native backend | Today | After the work |
| --- | --- | --- | --- |
| Debian, Ubuntu, Linux Mint, Raspberry Pi OS | ufw | Works | Full rule model |
| Rocky Linux, AlmaLinux 8 and 9 | firewalld | Broken, service resolves to ufw | Fixed by map entries |
| RHEL, CentOS Stream, Rocky, AlmaLinux 10 | firewalld | Broken, same cause | Fixed by map entries |
| RHEL and CentOS 7 to 9, Oracle Linux | firewalld | Works | Full rule model |
| Fedora | firewalld | Works | Full rule model |
| openSUSE Leap | firewalld | Works | Full rule model |
| openSUSE Tumbleweed, SLES | firewalld | Broken, same cause | Fixed by map entries |
| Alpine | iptables template | Open ports only | nftables backend |
| Arch and other unmapped distributions | ufw fallback | Untested | ufw or nftables, by choice |

### Indirect coverage

- **nftables on any distribution.** The `nft` tool is packaged everywhere listed, so the nftables backend is the universal fallback.
- **ufw on Red Hat-family hosts.** ufw is in EPEL 8, 9 and 10 <a id="cite-fedora-nd"></a>[(Fedora Project, n.d.)](#ref-fedora-nd). It is a community package there, and firewalld remains the vendor default.
- **ufw version skew.** EPEL and Fedora carry ufw 0.35, while Debian and Ubuntu carry 0.36. As far as the analyst knows `prepend` arrived in 0.36, so ordering options need a version check.

### What limits coverage

ufw has no native nftables backend. It drives `iptables`, which on current distributions is the nft-backed variant, and a native port is only planned for the medium to long term <a id="cite-launchpad-nd"></a>[(Launchpad, n.d.)](#ref-launchpad-nd). Sets and other nft-only features therefore cannot be expressed through ufw itself.

## Edge cases

The common rule model will never cover everything, so the role needs two escape hatches: an nftables overlay and backend-native extra rules. Each has a hard limit that decides what belongs in it.

### nftables overlay

The overlay is a role-owned table, for example `inet fw_overlay`, loaded beside ufw or firewalld. It works because nftables allows several tables to hook the same point, ordered by priority, and firewalld keeps its own rules inside its own table <a id="cite-garver-2018"></a>[(Garver, 2018)](#ref-garver-2018).

The limit is that a drop is final but an accept is not. A packet accepted in one chain is still checked by the other chains on that hook, and any of them can drop it [(Garver, 2018)](#ref-garver-2018). So the overlay can tighten, but it cannot open a port the native firewall blocks.

| Rule type | Where it goes | Why |
| --- | --- | --- |
| Allow a service | Native backend | Only the native firewall's accept gets past its own default drop |
| Block list or allow list by address set | Overlay | A drop in the overlay is final |
| Tunable rate limits | Overlay | Excess packets are dropped, which is final |
| NAT, port forwarding, masquerade | Overlay | NAT chains are separate from filter verdicts, but forwarded traffic still needs a native allow |
| Extra logging and counters | Overlay | Does not change the verdict |

Three rules keep the overlay safe:

- **Never use `flush ruleset`.** It would wipe the native firewall's rules. The overlay file declares its table, deletes that one table, then defines it again.
- **Load it from its own systemd unit.** The distribution's `nftables.service` often loads a file that starts with `flush ruleset`, so it must stay disabled or be edited.
- **Set priorities explicitly.** Overlay drops should run before the native chains, at a priority below 0, so the order does not depend on load order.

A minimal overlay file following those rules:

```nft
table inet fw_overlay
delete table inet fw_overlay

table inet fw_overlay {
    set blocklist {
        type ipv4_addr
        flags interval
        elements = { 203.0.113.0/24 }
    }

    chain input {
        type filter hook input priority -10; policy accept;
        ip saddr @blocklist drop
    }
}
```

### Extra rules in the native backend

For rules that must permit traffic, or that the model cannot express, each backend has its own raw form.

| Backend | Escape hatch | Caveat |
| --- | --- | --- |
| ufw | Managed blocks in `/etc/ufw/before.rules` and `before6.rules` | iptables-restore syntax, IPv4 and IPv6 kept in separate files |
| firewalld | `firewall_rich_rules` and policies | Avoid direct rules, they use iptables and outrank all other firewalld rules [(Garver, 2018)](#ref-garver-2018) |
| nftables | `firewall_nftables_raw` list, or a replacement template | None, this is the native language |

Raw rules are not portable. A playbook that uses them is tied to that backend, so the variables should be named per backend to make this obvious.

### Other cases to handle

- **Docker on targets.** Docker writes its own firewall rules and bypasses ufw for published ports. Its nftables mode is experimental since Docker 29.0.0, has no `DOCKER-USER` chain, and expects user rules in separate tables ordered by priority <a id="cite-docker-nd"></a>[(Docker Inc., n.d.)](#ref-docker-nd). The overlay fits that model. This only matters if a target host runs Docker, the test harness does not use it.
- **libvirt, Incus, fail2ban, Kubernetes.** Each adds its own tables or chains. A purge option must never touch tables the role did not create. An Incus host is a coexistence case in its own right, since Incus adds firewall rules for its managed bridges.
- **Mixed iptables variants.** A host with rules in both legacy and nft-backed iptables filters twice. The role should detect and report this, not fix it.
- **IPv6.** ufw and nftables `inet` tables cover both families from one rule. The current template is IPv4 only, so Alpine hosts have no IPv6 filtering today.
- **Lockout.** Changing default policy over SSH can cut the session. Apply the SSH rule first, then use a timed rollback that reverts unless the play confirms.
- **Reboot persistence.** ufw and firewalld persist by themselves. The nftables backend and the overlay need their units enabled and tested across a reboot.
- **Containers as targets.** Application containers such as Docker's should still be skipped. Incus system containers have their own network namespace and run systemd, so the role can and should run there. The skip guard should become a variable, see Testing.

## Testing

The existing container tests skip every firewall task, so the role is untested today. Testing moves to Incus, which can run both system containers and VMs from one tool, in three tiers. Docker is not used.

| Tier | When it runs | What it proves |
| --- | --- | --- |
| Incus system containers | Every change, full distribution matrix | Rules apply, idempotence, removal and purge, port probes over IPv4 and IPv6 |
| Incus VMs | Before each release | Real distribution kernel, reboot persistence, lockout rollback, coexistence |
| Metal | Release candidates | Real interfaces, NAT between segments, unreliable links, console recovery |

### Test harness

- **Molecule driver.** Use Molecule's default (delegated) driver with short `create.yml` and `destroy.yml` playbooks that call `incus launch` and `incus delete`. The analyst knows of no maintained Incus driver in molecule-plugins.
- **Connection.** Targets use the `community.general.incus` connection, which runs tasks through the Incus command line with no SSH setup <a id="cite-ansible-nd"></a>[(Ansible Project, n.d.)](#ref-ansible-nd).
- **Containers or VMs.** A scenario variable adds `--vm` to `incus launch`, so the same scenarios serve both tiers.
- **Dedicated Incus project.** Test instances are created fresh in their own Incus project, for example `fw-test`, and destroyed afterwards. Long-lived instances on the host, such as `deb13`, `ubu2404` and `rky9`, are never used as targets, since purge and lockout tests change them. The connection selects the project through `ansible_incus_project` [(Ansible Project, n.d.)](#ref-ansible-nd).
- **Probe host.** Each scenario has one target and one probe instance on the same Incus bridge. The probe tests real traffic.
- **Lockout scenarios use SSH.** The Incus connection reaches the instance without using its network, so a lockout test over it proves nothing. Those scenarios must connect over SSH.
- **Skip guard as a variable.** Replace the hard-coded connection list with `firewall_skip_connections`, defaulting to the Docker connection names. The `community.general.incus` connection is not on the list today, so the tasks already run under Incus, but by accident rather than design.

### On Incus containers

A system container has its own network namespace and runs systemd, so ufw, firewalld and nftables run inside it as they would on a host. Three limits apply, all because the container shares the host's kernel:

- **Kernel modules.** A container cannot load them. The netfilter modules the role needs, such as `nf_tables`, conntrack, the NAT modules and `br_netfilter`, must be loaded on the host or listed in the instance's `linux.kernel_modules` option <a id="cite-linuxcontainers-nd"></a>[(Linux Containers, n.d.)](#ref-linuxcontainers-nd). A shared Incus profile for firewall tests should carry this list.
- **Kernel version.** A Rocky Linux 9 container runs Rocky's userspace on the host's kernel. Anything that depends on the kernel version only counts once tested in a VM.
- **sysctl writes.** ufw writes sysctl values on start, and some of them may be read-only in an unprivileged container. The analyst expects some errors here, but this is untested. Where needed, set the value through `linux.sysctl.*` on the instance [(Linux Containers, n.d.)](#ref-linuxcontainers-nd) rather than running the containers privileged.

### On Incus VMs

| Check | Method |
| --- | --- |
| Rules apply | Converge, then compare `ufw status`, `firewall-cmd --list-all` or `nft list ruleset` to the expected output |
| Idempotence | Second run reports no changes |
| Real behaviour | Probe host tests allowed and blocked ports over IPv4 and IPv6 |
| Removal and purge | Flip rules to absent, confirm they are gone and unmanaged tables are untouched |
| Lockout safety | Tighten policy over SSH, confirm the session survives and the rollback fires when unconfirmed |
| Persistence | Reboot, then repeat the probe checks |
| Coexistence | Repeat with Incus and libvirt installed on the target, and Docker only if a target runs it |

The first four checks also run in the container tier.

A minimum matrix is seven images: Debian 12 and 13, Ubuntu 24.04, Rocky or AlmaLinux 9 and 10, Fedora, and Alpine. openSUSE Leap and Arch are worth adding if you intend to claim them. The same images are used in both Incus tiers.

### Storage

The current Incus host is short of space, which limits the matrix more than time does. Containers are small, and VMs each need a full root disk. To keep the footprint down:

- Run the container tier first, since it fits in the least space.
- Run VM scenarios one or two at a time, and destroy instances at the end of every scenario, including failed ones.
- Remove cached images that the matrix no longer uses.
- Review snapshots on the existing instances. The current host holds four instances with 16 snapshots between them, including a Windows VM, and old snapshots are likely to hold much of the used space.
- Add a storage pool on a separate disk before running the full VM matrix.

Freeing or adding Incus storage is a prerequisite for the VM tier, and is outside the estimate below.

### On metal

The metal host is a physical laptop installed by the existing raw-node installer and handed to Ansible. It currently runs Debian 13, the target operating system of the static multi-site host on a VPS, so the first metal pass is Debian 13 with ufw. Once the role is proven there, the same laptop is reinstalled with other distributions through the installer, giving metal passes for firewalld and nftables hosts. Red Hat-family and Alpine metal passes depend on the installer supporting those distributions. Until it does, the VM tier carries them.

Metal checks:

- **Multiple real interfaces**, VLANs and bridges, for interface and forwarding rules. A laptop usually has only one wired port, so this may need a USB network adapter or a managed switch.
- **NAT and routing** between real network segments.
- **Lockout recovery** with console access, to confirm the rollback on real hardware.
- **Low-bandwidth or intermittent links**, to check that a dropped Ansible session never leaves a half-applied firewall. `tc netem` on the controller or an upstream host can simulate these.
- **Install-to-firewall handoff.** The installer's handoff to Ansible, followed by the firewall role, works on first boot without manual steps.

Metal testing is one host reinstalled in turn, not the whole matrix.

## Time estimate

The full scope is 20 to 31 focused working days. These are estimates, assuming one person experienced with Ansible and Linux firewalls, about six productive hours a day, and no upstream review delays.

| Phase | Work | Days |
| --- | --- | --- |
| Foundations | Fork, fix the defects listed above, move settings to defaults, Incus test harness for containers and VMs | 2 to 3 |
| Rule model | `firewall_rules` on ufw and firewalld, policies, validation, idempotent removal | 4 to 6 |
| nftables backend | Template, sets, NAT, atomic load, syntax check, Alpine migration | 4 to 6 |
| Edge cases | Overlay table and unit, per-backend raw rules, purge, lockout rollback | 3 to 5 |
| Test matrix | Seven images in both Incus tiers, probe scenarios, reboot and coexistence runs, metal passes on one host | 5 to 8 |
| Documentation and release | README, examples per backend, changelog, Galaxy release | 2 to 3 |
| Total | | 20 to 31 |

### Staged delivery

- **First release, 6 to 9 days:** Foundations and Rule model. This already covers source filtering, interfaces, actions, logging and policies on ufw and firewalld hosts.
- **Second release, 7 to 11 more days:** nftables backend and Edge cases.
- **Hardening, 7 to 11 more days:** full Test matrix and Documentation. Some testing happens inside each earlier phase, this is the cross-distribution pass.

### What moves the estimate

- **Down:** dropping firewalld zones and NAT from scope removes about 3 to 5 days.
- **Neutral:** the Incus harness adds about half a day to Foundations, and faster container runs recover it during the Test matrix.
- **Up:** each extra distribution family adds about half a day to one day of testing and fixes.
- **Up:** each metal pass on another distribution adds a reinstall cycle, about half a day if the installer already supports it.
- **Outside the estimate:** freeing or adding Incus storage, and extending the installer to new distributions.
- **Up:** contributing upstream instead of forking adds review time outside your control, and the maintainer may not want this scope.
- **Alternative:** a new nftables-only role is about 8 to 12 days including tests, since nothing in the current tasks would be reused.

## Open questions

These answers change the scope and the estimate.

- **Fork under your own namespace**, or propose the changes upstream?

  - https://github.com/steelcj/ansible-role-foomuuri
- Which distributions must be supported in the first release?

  - debian/13
  - ubuntu server 24.04
  - LinuxMint added over a minimal ubuntu server 24.04

    - ```txt
      DISTRIB_ID=LinuxMint
      DISTRIB_RELEASE=22.3
      DISTRIB_CODENAME=zena
      DISTRIB_DESCRIPTION="Linux Mint 22.3 Zena"
      ```

- Are the target hosts single-homed servers, or routers and gateways that need NAT and forwarding?
  - Single homed / VPS /  Metal / incus containers and VM's
- Should Red Hat-family hosts stay on firewalld, or move to ufw or nftables for one syntax everywhere?
  - For Red Hat families do not bother with older versions, if that helps. nftables if "stable" interface can be created, other wise inform me and we can consider dropping Red Hat family if that would overly complicate our approach
- Do any targets run Docker, libvirt or Kubernetes?
  - I belive Incus uses libvirt
- Is purge of undeclared rules wanted, and on which hosts?
- Where will Incus storage for the full VM matrix come from?
- Which distributions will the raw-node installer support for metal passes, beyond Debian 13?

## References

<a id="ref-ansible-nd"></a>
[Return to citation](#cite-ansible-nd)
Ansible Project. (n.d.). *community.general.incus connection: Run tasks in Incus instances using the Incus CLI*. Ansible Documentation. Retrieved November 15, 2026, from https://docs.ansible.com/ansible/devel/collections/community/general/incus_connection.html

<a id="ref-debock-2026"></a>
[Return to citation](#cite-debock-2026)
de Bock, R. (2026). *ansible-role-firewall* (Commit 1f32061) [Computer software]. GitHub. https://github.com/robertdebock/ansible-role-firewall

<a id="ref-docker-nd"></a>
[Return to citation](#cite-docker-nd)
Docker Inc. (n.d.). *Docker with nftables*. Docker Docs. Retrieved November 13, 2026, from https://docs.docker.com/engine/network/firewall-nftables/

<a id="ref-fedora-nd"></a>
[Return to citation](#cite-fedora-nd)
Fedora Project. (n.d.). *ufw: Uncomplicated Firewall*. Fedora Packages. Retrieved November 13, 2026, from https://packages.fedoraproject.org/pkgs/ufw/ufw/

<a id="ref-garver-2018"></a>
[Return to citation](#cite-garver-2018)
Garver, E. (2018, July 24). *nftables backend*. firewalld. https://firewalld.org/2018/07/nftables-backend

<a id="ref-launchpad-nd"></a>
[Return to citation](#cite-launchpad-nd)
Launchpad. (n.d.). *Bug #1880453: Feasability of a nftables port*. ufw bug tracker. Retrieved November 13, 2026, from https://bugs.launchpad.net/bugs/1880453

<a id="ref-linuxcontainers-nd"></a>
[Return to citation](#cite-linuxcontainers-nd)
Linux Containers. (n.d.). *Instance options*. Incus documentation. Retrieved November 15, 2026, from https://linuxcontainers.org/incus/docs/main/reference/instance_options/
