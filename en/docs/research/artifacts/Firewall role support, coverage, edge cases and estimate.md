# Firewall role: support, coverage, edge cases and estimate

13 Nov 2026 · @sirhc

## Summary

Making this role capable of complex firewalls across distributions is about 20 to 31 focused working days, roughly 4 to 6 weeks for one experienced person. A useful first release (defect fixes plus a richer rule model on ufw and firewalld) is 6 to 9 days.

The recommended shape is a fork with three layers:

- **Native backends keep the allow rules.** ufw on Debian-family hosts and firewalld on Red Hat and SUSE-family hosts, driven by a richer common rule model.
- **An nftables backend replaces the iptables template.** It serves Alpine, and any host where you choose to run nftables alone.
- **An nftables overlay table carries the extras.** Address sets, rate limits, NAT and extra drops load beside ufw or firewalld without editing their rules.

The alternative is a new nftables-only role with no ufw or firewalld support, at about 8 to 12 days. It is cheaper and more expressive, but it replaces each distribution's default firewall manager.

All estimates here are my own judgement from reading the role, not measured figures. This analysis is based on commit `1f32061` of [robertdebock/ansible-role-firewall](https://github.com/robertdebock/ansible-role-firewall) (de Bock, 2026).

## Current state

The role can only express "open this port and protocol to everyone". Each entry in `firewall_services` carries `name`, `protocol`, `state`, and on ufw only, `rule`.

### What each backend receives

| Backend   | Selected for                                                | Mechanism                 | Keys used                   |
| --------- | ----------------------------------------------------------- | ------------------------- | --------------------------- |
| ufw       | Debian, Ubuntu, and any unmapped distribution               | `community.general.ufw`   | `rule`, `name`, `protocol`  |
| firewalld | RHEL and CentOS 7 to 9, Fedora, Oracle Linux, openSUSE Leap | `ansible.posix.firewalld` | `name`, `protocol`, `state` |
| iptables  | Alpine only                                                 | `templates/iptables.j2`   | `name`, `protocol`, `state` |

### Defects found by reading the code

These were found by inspection and have not been reproduced on a host.

- **Service and package maps disagree.** Packages are chosen by OS family, the service by exact distribution name. Rocky, AlmaLinux, EL 10, SLES and openSUSE Tumbleweed get firewalld installed but `ufw` as the service.
- **The template is fixed.** `src: iptables.j2` is hard-coded, and the rule file path is defined for Alpine only.
- **The template ignores two defaults.** It hard-codes `tcp` and always accepts, so `firewall_default_protocol` and `firewall_default_rule` have no effect there.
- **`state: absent` on ufw is not idempotent.** The open task has no state filter, so the rule is added and then deleted on every run.
- **The assert rejects valid ufw actions.** Only `allow` and `deny` pass, so `reject`, `limit` and `block` all fail.
- **Backend choice is locked in `vars/`.** Only role parameters or extra vars can override `firewall_service` and the package lists.
- **The test suite does not test the firewall.** Molecule runs in Docker, and every firewall task is skipped for container connections.

## Support required

The work falls into four groups, in the order they should be built. Most of the first two groups is passing extra optional keys to modules that already support them.

### Rule model

A new `firewall_rules` list sits beside `firewall_services`, so existing playbooks keep working.

| Capability                     | ufw                                           | firewalld                                       | nftables        |
| ------------------------------ | --------------------------------------------- | ----------------------------------------------- | --------------- |
| Source and destination address | Module keys `from_ip`, `to_ip`                | Rich rule, or source bound to a zone            | Native          |
| Interface in and out           | Module keys `interface_in`, `interface_out`   | Interface bound to a zone                       | Native          |
| Port ranges and port lists     | Native                                        | Native for ranges, one entry per port for lists | Native          |
| Action: accept, drop, reject   | Native                                        | Rich rule or zone target                        | Native          |
| Rate limit                     | `limit` only, fixed threshold                 | Rich rule `limit`                               | Native, tunable |
| Per-rule logging               | Module key `log`                              | Rich rule `log`                                 | Native          |
| Direction: in, out, forward    | `direction` and `route` keys                  | Policies between zones                          | Native          |
| Explicit ordering              | `insert` key                                  | Rich rule `priority`                            | File order      |
| Comment                        | Module key `comment`                          | Not on plain ports                              | Native          |
| ICMP control                   | Not in the command line, needs `before.rules` | ICMP blocks per zone                            | Native          |

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

| Family                                      | Native backend    | Today                           | After the work             |
| ------------------------------------------- | ----------------- | ------------------------------- | -------------------------- |
| Debian, Ubuntu, Linux Mint, Raspberry Pi OS | ufw               | Works                           | Full rule model            |
| Rocky Linux, AlmaLinux 8 and 9              | firewalld         | Broken, service resolves to ufw | Fixed by map entries       |
| RHEL, CentOS Stream, Rocky, AlmaLinux 10    | firewalld         | Broken, same cause              | Fixed by map entries       |
| RHEL and CentOS 7 to 9, Oracle Linux        | firewalld         | Works                           | Full rule model            |
| Fedora                                      | firewalld         | Works                           | Full rule model            |
| openSUSE Leap                               | firewalld         | Works                           | Full rule model            |
| openSUSE Tumbleweed, SLES                   | firewalld         | Broken, same cause              | Fixed by map entries       |
| Alpine                                      | iptables template | Open ports only                 | nftables backend           |
| Arch and other unmapped distributions       | ufw fallback      | Untested                        | ufw or nftables, by choice |

### Indirect coverage

- **nftables on any distribution.** The `nft` tool is packaged everywhere listed, so the nftables backend is the universal fallback.
- **ufw on Red Hat-family hosts.** ufw is in EPEL 8, 9 and 10 ([Fedora Project, n.d.](https://packages.fedoraproject.org/pkgs/ufw/ufw/)). It is a community package there, and firewalld remains the vendor default.
- **ufw version skew.** EPEL and Fedora carry ufw 0.35, while Debian and Ubuntu carry 0.36. As far as I know `prepend` arrived in 0.36, so ordering options need a version check.

### What limits coverage

ufw has no native nftables backend. It drives `iptables`, which on current distributions is the nft-backed variant, and a native port is only planned for the medium to long term ([Launchpad, n.d.](https://bugs.launchpad.net/bugs/1880453)). Sets and other nft-only features therefore cannot be expressed through ufw itself.

## Edge cases

The common rule model will never cover everything, so the role needs two escape hatches: an nftables overlay and backend-native extra rules. Each has a hard limit that decides what belongs in it.

### nftables overlay

The overlay is a role-owned table, for example `inet fw_overlay`, loaded beside ufw or firewalld. It works because nftables allows several tables to hook the same point, ordered by priority, and firewalld keeps its own rules inside its own table ([Garver, 2018](https://firewalld.org/2018/07/nftables-backend)).

The limit is that a drop is final but an accept is not. A packet accepted in one chain is still checked by the other chains on that hook, and any of them can drop it (Garver, 2018). So the overlay can tighten, but it cannot open a port the native firewall blocks.

| Rule type                               | Where it goes  | Why                                                          |
| --------------------------------------- | -------------- | ------------------------------------------------------------ |
| Allow a service                         | Native backend | Only the native firewall's accept gets past its own default drop |
| Block list or allow list by address set | Overlay        | A drop in the overlay is final                               |
| Tunable rate limits                     | Overlay        | Excess packets are dropped, which is final                   |
| NAT, port forwarding, masquerade        | Overlay        | NAT chains are separate from filter verdicts, but forwarded traffic still needs a native allow |
| Extra logging and counters              | Overlay        | Does not change the verdict                                  |

Three rules keep the overlay safe:

- **Never use `flush ruleset`.** It would wipe the native firewall's rules. The overlay file declares its table, deletes that one table, then defines it again.
- **Load it from its own systemd unit.** The distribution's `nftables.service` often loads a file that starts with `flush ruleset`, so it must stay disabled or be edited.
- **Set priorities explicitly.** Overlay drops should run before the native chains, at a priority below 0, so the order does not depend on load order.

### Extra rules in the native backend

For rules that must permit traffic, or that the model cannot express, each backend has its own raw form.

| Backend   | Escape hatch                                                 | Caveat                                                       |
| --------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ufw       | Managed blocks in `/etc/ufw/before.rules` and `before6.rules` | iptables-restore syntax, IPv4 and IPv6 kept in separate files |
| firewalld | `firewall_rich_rules` and policies                           | Avoid direct rules, they use iptables and outrank all other firewalld rules (Garver, 2018) |
| nftables  | `firewall_nftables_raw` list, or a replacement template      | None, this is the native language                            |

Raw rules are not portable. A playbook that uses them is tied to that backend, so the variables should be named per backend to make this obvious.

### Other cases to handle

- **Docker.** Docker writes its own firewall rules and bypasses ufw for published ports. Its nftables mode is experimental since Docker 29.0.0, has no `DOCKER-USER` chain, and expects user rules in separate tables ordered by priority ([Docker Inc., n.d.](https://docs.docker.com/engine/network/firewall-nftables/)). The overlay fits that model.
- **libvirt, Incus, fail2ban, Kubernetes.** Each adds its own tables or chains. A purge option must never touch tables the role did not create.
- **Mixed iptables variants.** A host with rules in both legacy and nft-backed iptables filters twice. The role should detect and report this, not fix it.
- **IPv6.** ufw and nftables `inet` tables cover both families from one rule. The current template is IPv4 only, so Alpine hosts have no IPv6 filtering today.
- **Lockout.** Changing default policy over SSH can cut the session. Apply the SSH rule first, then use a timed rollback that reverts unless the play confirms.
- **Reboot persistence.** ufw and firewalld persist by themselves. The nftables backend and the overlay need their units enabled and tested across a reboot.
- **Containers as targets.** Firewall tasks are skipped under container connections, which is correct, but it means container tests prove nothing.

## Testing

VM and metal testing does help: it is the only way to test this role at all, because the existing container tests skip every firewall task. VMs should carry almost all of it, with metal reserved for what VMs cannot show.

### On VMs

Run Molecule against full VMs (Incus VMs, libvirt or Vagrant) with one target and one probe host per scenario.

| Check             | Method                                                       |
| ----------------- | ------------------------------------------------------------ |
| Rules apply       | Converge, then compare `ufw status`, `firewall-cmd --list-all` or `nft list ruleset` to the expected output |
| Idempotence       | Second run reports no changes                                |
| Real behaviour    | Probe host tests allowed and blocked ports over IPv4 and IPv6 |
| Removal and purge | Flip rules to absent, confirm they are gone and unmanaged tables are untouched |
| Lockout safety    | Tighten policy over SSH, confirm the session survives and the rollback fires when unconfirmed |
| Persistence       | Reboot, then repeat the probe checks                         |
| Coexistence       | Repeat with Docker and libvirt installed                     |

A minimum matrix is seven images: Debian 12 and 13, Ubuntu 24.04, Rocky or AlmaLinux 9 and 10, Fedora, and Alpine. openSUSE Leap and Arch are worth adding if you intend to claim them.

### On metal

- **Multiple real interfaces**, VLANs and bridges, for interface and forwarding rules.
- **NAT and routing** between real network segments.
- **Lockout recovery** with console access, to confirm the rollback on real hardware.
- **Low-bandwidth or intermittent links**, to check that a dropped Ansible session never leaves a half-applied firewall.

Metal testing is one or two hosts, not the whole matrix.

## Time estimate

The full scope is 20 to 31 focused working days. These are my estimates, assuming one person experienced with Ansible and Linux firewalls, about six productive hours a day, and no upstream review delays.

| Phase                     | Work                                                         | Days   |
| ------------------------- | ------------------------------------------------------------ | ------ |
| Foundations               | Fork, fix the defects listed above, move settings to defaults, VM test harness | 2 to 3 |
| Rule model                | `firewall_rules` on ufw and firewalld, policies, validation, idempotent removal | 4 to 6 |
| nftables backend          | Template, sets, NAT, atomic load, syntax check, Alpine migration | 4 to 6 |
| Edge cases                | Overlay table and unit, per-backend raw rules, purge, lockout rollback | 3 to 5 |
| Test matrix               | Seven VM images, probe scenarios, reboot and coexistence runs, one or two metal hosts | 5 to 8 |
| Documentation and release | README, examples per backend, changelog, Galaxy release      | 2 to 3 |

### Staged delivery

- **First release, 6 to 9 days:** Foundations and Rule model. This already covers source filtering, interfaces, actions, logging and policies on ufw and firewalld hosts.
- **Second release, 7 to 11 more days:** nftables backend and Edge cases.
- **Hardening, 7 to 11 more days:** full Test matrix and Documentation. Some testing happens inside each earlier phase, this is the cross-distribution pass.

### What moves the estimate

- **Down:** dropping firewalld zones and NAT from scope removes about 3 to 5 days.
- **Up:** each extra distribution family adds about half a day to one day of testing and fixes.
- **Up:** contributing upstream instead of forking adds review time outside your control, and the maintainer may not want this scope.
- **Alternative:** a new nftables-only role is about 8 to 12 days including tests, since nothing in the current tasks would be reused.

## Open questions

These answers change the scope and the estimate.

- Fork under your own namespace, or propose the changes upstream?
- Which distributions must be supported in the first release?
- Are the target hosts single-homed servers, or routers and gateways that need NAT and forwarding?
- Should Red Hat-family hosts stay on firewalld, or move to ufw or nftables for one syntax everywhere?
- Do any targets run Docker, libvirt or Kubernetes?
- Is purge of undeclared rules wanted, and on which hosts?

## References

de Bock, R. (2026). *ansible-role-firewall* (Commit 1f32061) [Computer software]. GitHub. https://github.com/robertdebock/ansible-role-firewall

Docker Inc. (n.d.). *Docker with nftables*. Docker Docs. Retrieved November 13, 2026, from https://docs.docker.com/engine/network/firewall-nftables/

Fedora Project. (n.d.). *ufw: Uncomplicated Firewall*. Fedora Packages. Retrieved November 13, 2026, from https://packages.fedoraproject.org/pkgs/ufw/ufw/

Garver, E. (2018, July 24). *nftables backend*. firewalld. https://firewalld.org/2018/07/nftables-backend

Launchpad. (n.d.). *Bug #1880453: Feasability of a nftables port*. ufw bug tracker. Retrieved November 13, 2026, from https://bugs.launchpad.net/bugs/1880453