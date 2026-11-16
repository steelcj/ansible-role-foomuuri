# Firewall role: design, coverage, edge cases and estimate

Date: 2026-11-15

Author: Christopher Steel

Version: 0.1.3

## Summary

The role is `steelcj/ansible-role-foomuuri` <a id="cite-steel-2026"></a>[(Steel, 2026)](#ref-steel-2026). It manages nftables on every supported distribution through Foomuuri, a firewall manager that compiles one configuration language into a single nftables table <a id="cite-foobaroy-2026a"></a>[(FoobarOy, 2026a)](#ref-foobaroy-2026a). ufw and firewalld are switched off, not driven. Robert de Bock's `ansible-role-firewall` remains the model for structure and testing, but none of its code is reused.

The design goal is a firewall that is highly stable and easy to maintain:

- **One table, replaced atomically.** Foomuuri owns `inet foomuuri` and swaps it whole on every reload. Tables owned by Incus or anything else are never touched.
- **Nothing unchecked is installed.** Every change is rendered to a staging directory, checked with `foomuuri check` (which includes a real nftables syntax check), and only then deployed.
- **One rule language everywhere.** Rules are written in Foomuuri's own syntax and passed through as strings, so the role never maintains a second rule language in YAML.
- **Additive passes.** A play can add rules (SSH first, then HTTP later) without restating earlier ones, through named fragments that Foomuuri merges.
- **Purge on managed hosts.** Anything in the Foomuuri configuration the role did not write is removed.

The first release targets Debian 13, Ubuntu Server 24.04 and Linux Mint 22.3 on single-homed hosts, and is about 15 to 23 focused working days, roughly 3 to 5 weeks for one person. Red Hat-family support is a later, separate step of about 2 to 4 days, and does not need to be dropped.

All estimates are the analyst's judgement, not measured figures. Facts about Foomuuri were checked against its source at commit `e5ac100` <a id="cite-foobaroy-2026b"></a>[(FoobarOy, 2026b)](#ref-foobaroy-2026b), and facts about Robert's role against commit `1f32061` <a id="cite-debock-2026"></a>[(de Bock, 2026)](#ref-debock-2026).

## Decisions

These answers to the open questions in version 0.1.2 set the scope of this version.

| Question | Answer | Effect on the design |
| --- | --- | --- |
| Fork or upstream? | Own repository, `steelcj/ansible-role-foomuuri` | No upstream review delays. Robert's role is a model, not a code base. |
| First-release distributions | Debian 13, Ubuntu Server 24.04, Linux Mint 22.3 (installed over a minimal Ubuntu Server 24.04) | All three are Debian-family. Mint needs its own lookup entry. |
| Host roles | Single-homed: VPS, metal, Incus containers and VMs | NAT and port forwarding are out of scope. Incus hosts still need forwarding for their bridges, see Incus hosts. |
| Red Hat family | nftables if a stable interface is possible, older versions not needed | Foomuuri is that interface. Fedora and EL 9 and 10 through EPEL, later. EL 8 dropped. |
| Docker, libvirt, Kubernetes on targets | None. Incus is used, and does not use libvirt. | Docker and Kubernetes leave scope. Incus coexistence stays. |
| Purge of undeclared rules | Wanted on Ansible-managed hosts | Purge is on by default, scoped to the role's own configuration directory. |
| Incus storage | NVMe on the more powerful laptop, short term | Enough for the first-release matrix. |
| Metal installer | Debian 13, Ubuntu 24 and 25 today, more on request | Covers every first-release distribution on metal. |

## Why Foomuuri for stable nftables

Writing nftables rules from a hand-made Jinja template would give full control, but the role would then own a small compiler: zones, ordering, IPv4 and IPv6 handling, sets and validation. Foomuuri already does that work, is packaged in Debian, Ubuntu, Fedora and EPEL <a id="cite-foobaroy-2026c"></a>[(FoobarOy, 2026c)](#ref-foobaroy-2026c), and was built for the properties this role needs.

### Atomic, self-contained ruleset

Every generated ruleset starts by declaring and deleting `table inet foomuuri`, then defining it again, so a reload replaces only that table in one transaction [(FoobarOy, 2026b)](#ref-foobaroy-2026b). It never issues `flush ruleset`. Tables owned by Incus, libvirt or Docker survive every reload. This is the same safety rule version 0.1.2 set for its nftables overlay, provided by the tool itself.

### Validation and recovery

`foomuuri check` parses the configuration, builds the ruleset and runs nftables' own syntax check on it. A configuration directory other than the live one can be checked by passing `--set=etc_dir=DIR`, which is what makes staged validation possible. At boot, `foomuuri-boot.service` loads the last ruleset that loaded successfully before the configuration is read again, so a bad file never leaves a host unfiltered after a restart.

### One language, any number of files

Foomuuri reads every `*.conf` file under `/etc/foomuuri`, including subdirectories, in alphabetical order, and includes any `*.nft` file as raw nftables [(FoobarOy, 2026a)](#ref-foobaroy-2026a). When the same section, such as `public-localhost`, appears in more than one file, its rules are appended in file order rather than replaced. This was confirmed in the parser [(FoobarOy, 2026b)](#ref-foobaroy-2026b), and it is what makes additive passes possible.

### Version skew

The packaged versions differ by distribution, from the package archives as read for the earlier Foomuuri analysis:

| Distribution | Package source | Version |
| --- | --- | --- |
| Debian 13 | `main`, newer in `trixie-backports` | 0.27, 0.31 in backports |
| Ubuntu 24.04 | `universe` | 0.22, to verify with `apt-cache policy foomuuri` |
| Linux Mint 22.3 | Ubuntu 24.04's `universe` | Same as Ubuntu 24.04 |
| Upstream | Source release | 0.34 |

Two ways to handle this:

- **Distribution packages.** Security updates arrive with the system, and the role's own defaults must use only syntax that 0.22 accepts. Newer features come only through the user's own rule strings, guarded by a version fact.
- **Pinned source install everywhere.** One version on every host, but the role takes over updates.

The recommendation is distribution packages, with the installed version recorded as a fact so assertions can refuse settings an older version does not support. This keeps maintenance with the distributions.

## Lessons from Robert's role

Robert's role was reviewed in version 0.1.2. Its defects become design rules here.

| Defect in `ansible-role-firewall` | Rule in `ansible-role-foomuuri` |
| --- | --- |
| Service and package maps keyed differently, breaking Rocky, AlmaLinux, SLES | One lookup key for everything: `distribution-major_version`, then `distribution`, then `os_family`, then `default` |
| Backend choice locked in `vars/` | Every public setting in `defaults/`. Private lookup tables stay in `vars/`. |
| Hard-coded template and ignored defaults | No templated rule language. Rules are Foomuuri strings. |
| `state: absent` not idempotent on ufw | Each fragment file is rendered whole on every run, so removing a rule from the variables removes it from the host |
| Hand-written assert rejects valid actions | `meta/argument_specs.yml` for types, `assert.yml` only for checks across variables |
| Every firewall task skipped in containers, so nothing is tested | Skip list is a variable. Incus system containers run the full role. |

## Design

### Platform resolution

The lookup tables follow Robert's pattern. Linux Mint needs its own entry, because Ansible is expected to report it as distribution `Linux Mint`, major version `22`, family `Debian`. This should be confirmed with `ansible -m setup` on the workstation before the table is written.

| Lookup key | Install method | Package source |
| --- | --- | --- |
| `Debian-13` | `package` | `main` |
| `Ubuntu-24` | `package` | `universe` |
| `Linux Mint-22` | `package` | Ubuntu 24.04's `universe` |
| `default` | Assertion failure naming the supported platforms | None |

An unsupported platform fails loudly. Robert's role silently falls back to ufw on unknown distributions, which is how the Rocky and SLES defects stayed hidden.

### Conflicting firewalls

Foomuuri must be the only firewall manager on the host. Before the first start, the role stops, disables and masks every conflicting service that exists: `ufw`, `firewalld`, `nftables`, `netfilter-persistent`, `iptables` and `ip6tables`. Packages stay installed, because other software may need the command line tools.

ufw needs one extra step. ufw is installed on Ubuntu and Mint, and stopping its service does not remove the rules it loaded into the kernel. Because a drop in any table is final, leftover ufw rules could still block traffic that Foomuuri accepts. The role therefore runs `ufw disable`, which removes ufw's rules, before masking the service. firewalld removes its own table when stopped.

The distribution's `nftables.service` is masked because its configuration file usually starts with `flush ruleset`, which would wipe the Foomuuri table and every other table on the next boot.

### Variable model

The variable model is the one proposed in the Foomuuri pre-build analysis, kept short here:

| Variable | Foomuuri section | Content |
| --- | --- | --- |
| `foomuuri_config` | `foomuuri { }` | Global options, keys as Foomuuri spells them |
| `foomuuri_zones` | `zone { }` | Zone name to list of interfaces |
| `foomuuri_macros` | `macro { }` | Name to Foomuuri string |
| `foomuuri_rules` | Any rule section | Section name, such as `public-localhost`, to list of rule strings |
| `foomuuri_iplists` | `iplist { }` | Named address sets, static or fetched |
| `foomuuri_files` | Raw files | Extra `.conf` or `.nft` files owned by the role |

`foomuuri_rules` uses the section name as the key, so every Foomuuri section works without a role change. The defaults match Foomuuri's own host firewall example: SSH, DHCP and ping in, everything out.

### Additive passes and fragments

The question this answers: can one pass allow SSH, a later pass add HTTP, and so on, without each pass restating everything before it?

A role that writes one configuration file cannot do this. With purge on, pass two would declare only HTTP and the role would remove SSH. The answer is to let each pass own one named fragment.

- **A fragment is one file.** Each run of the role writes `/etc/foomuuri/ansible/<fragment>.conf`, named by `foomuuri_fragment` (default `10-base`). The file starts with a marker comment that identifies it as managed by the role.
- **Fragments merge.** Because Foomuuri appends rules for a section that appears in several files, two fragments that both contain `public-localhost` produce one section holding both sets of rules, in fragment order.
- **A fragment is declarative.** Each run renders its fragment whole, so removing a rule from that pass's variables removes it from the host. Other fragments are left alone.
- **A fragment can be removed.** `foomuuri_fragment_state: absent` deletes it.

Two passes, as separate plays:

```yaml
# Pass one, run by the baseline playbook
- name: Baseline firewall
  hosts: all
  roles:
    - role: steelcj.foomuuri
      vars:
        foomuuri_fragment: 10-base
        foomuuri_zones:
          public: ["{{ ansible_facts['default_ipv4']['interface'] }}"]
        foomuuri_rules:
          public-localhost:
            - ssh
            - ping
          localhost-public:
            - accept

# Pass two, run later by the web server playbook
- name: Web server firewall
  hosts: webservers
  roles:
    - role: steelcj.foomuuri
      vars:
        foomuuri_fragment: 20-web
        foomuuri_rules:
          public-localhost:
            - http
            - https
```

The resulting `public-localhost` section holds `ssh`, `ping`, `http` and `https`, followed by Foomuuri's automatic final `reject log`.

Three rules keep fragments safe:

- **No terminal rules in fragments.** A bare `accept`, `drop` or `reject` at the end of fragment `10-base` would come before the rules of `20-web`, and those rules would never be reached. The assertion refuses terminal rules in any fragment except one named for them, by convention `99-final`. Without it, Foomuuri adds its own final rule, `drop log`, or `reject log` for sections leaving `localhost` <a id="cite-foobaroy-2026d"></a>[(FoobarOy, 2026d)](#ref-foobaroy-2026d), so most hosts need no final fragment.
- **Zones and global options belong to the base fragment.** Later fragments only add rules, macros and address sets. Defining a zone in two fragments would merge its interfaces, which is rarely intended.
- **Every pass checks the whole firewall.** The staging step copies all existing fragments and adds this pass's new version before running `foomuuri check`. The SSH assertion also runs on the merged configuration, so a later pass cannot be the one that locks the host out, and an earlier fragment with a typo blocks later passes until it is fixed.

### Purge

Purge removes anything in the Foomuuri configuration that the role did not write. It never touches nftables tables other than `inet foomuuri`, so Incus's rules are safe.

| `foomuuri_purge` | Removes | Use when |
| --- | --- | --- |
| `unmanaged` (default) | Every `.conf` or `.nft` file under `/etc/foomuuri` without the role's marker, such as a hand-written file | Ansible-managed hosts using additive passes |
| `fragments` | Also every managed fragment not declared in this run | One play declares the whole firewall, for example through inventory layers |
| `none` | Nothing | Hosts where people also edit the firewall by hand |

Rules added by hand with `foomuuri` commands at runtime, such as a temporary block on an address set, live only until the next reload, because Foomuuri rebuilds the table from configuration each time.

`fragments` and additive passes do not mix: pass two with `fragments` would delete pass one's fragment. The assertion refuses `foomuuri_purge: fragments` unless `foomuuri_fragment` is the only fragment declared for the host, or the run explicitly lists the fragments to keep.

### Inventory layers as an alternative

When one play can see the whole firewall, inventory layers are simpler than fragments. Variables named `foomuuri_rules__<layer>` are merged in name order from `group_vars` and `host_vars`, and rendered into one fragment:

```yaml
# group_vars/all.yml
foomuuri_rules__10_base:
  public-localhost: [ssh, ping]

# group_vars/webservers.yml
foomuuri_rules__20_web:
  public-localhost: [http, https]
```

Use layers when one playbook owns the firewall, with `foomuuri_purge: fragments` if wanted. Use fragments when separate playbooks or roles each add their own ports at different times. The role supports both.

### Validate, then apply

Each run follows the same sequence:

1. Render this pass's fragment into a temporary staging directory, next to copies of every other file in `/etc/foomuuri`.
2. Run `foomuuri --set=etc_dir=<stage>/etc --set=state_dir=<stage>/state check`.
3. Run the cross-variable assertions on the merged configuration, including the SSH check.
4. Deploy the fragment, and remove files according to the purge setting.
5. Reload, never restart. `systemctl restart` runs `foomuuri stop`, which removes the ruleset and leaves a short unfiltered window. A reload swaps the table atomically.

The check runs in every test tier, including containers.

### Lockout protection

- **SSH assertion.** When `foomuuri_require_ssh` is true (the default), the merged configuration must contain an SSH rule in a section that reaches `localhost`.
- **Timed rollback.** Before a reload, the role keeps a copy of the previous configuration and schedules a one-shot `systemd-run` timer that restores it and reloads after two minutes. A task after the reload cancels the timer only if Ansible can still reach the host over a new SSH connection. This gives Foomuuri's interactive `try-reload` behaviour without its prompt.

### Incus hosts

The Mint workstation is both a first-release target and an Incus host, so it is a design case, not an edge case.

Incus adds its own nftables table for its bridges, which Foomuuri leaves alone. Foomuuri's own forward and input chains still apply to bridge traffic, though, and a drop there is final. With only the default host firewall, containers and VMs would lose DHCP, DNS and their outbound connection.

The role therefore needs a small, optional Incus profile, applied as its own fragment:

- An `incus` zone holding `incusbr0` and any other managed bridges.
- `incus-localhost` rules for DHCP and DNS served by Incus on the host, over IPv4 and IPv6.
- `incus-public` accepting outbound traffic. Return traffic is accepted by connection tracking, and Incus does its own masquerading.
- `localhost-incus` accepting, so the host can reach its instances.

This is forwarding without NAT, so it stays inside the single-homed scope. The exact macros should be confirmed against the Foomuuri version on Ubuntu 24.04.

## Distribution coverage

| Family | First release | Later | Notes |
| --- | --- | --- | --- |
| Debian 13 | Yes | | Foomuuri in `main` |
| Ubuntu Server 24.04 | Yes | | Foomuuri 0.22 in `universe`, to verify |
| Linux Mint 22.3 | Yes | | Built on Ubuntu 24.04. Own lookup entry. |
| Debian 12 | | Optional | Foomuuri from `bookworm-backports` |
| Ubuntu 25.04 and later | | Yes | Newer Foomuuri in `universe` |
| Fedora | | Yes | Fedora repository |
| EL 9 and 10 (RHEL, Rocky, AlmaLinux, CentOS Stream) | | Yes | EPEL, run before this role. firewalld masked. |
| EL 8 | | No | No Foomuuri package, Python too old |
| Alpine | | No | Foomuuri ships systemd units only |
| Arch | | Optional | AUR package, to check that it is current |

Red Hat-family support stays possible because Foomuuri gives the same configuration language there. The extra work is EPEL ordering, firewalld masking and two more images in the test matrix.

## Edge cases

- **Leftover ufw rules.** Handled by `ufw disable` before masking. The role should also report, not fix, a host that still has rules in both legacy and nft-backed iptables.
- **Docker on future targets.** Docker writes its own rules and bypasses host firewalls for published ports. It is out of scope while no target runs it. If one does later, Foomuuri's documentation and a fragment for the Docker bridge are the place to start.
- **IPv6.** Foomuuri's `inet` table covers both families from one rule.
- **Raw nftables.** A fragment may carry `.nft` content through `foomuuri_files`. Raw rules are tied to nftables syntax, not Foomuuri's, so they are kept to cases Foomuuri cannot express.
- **Reboot persistence.** `foomuuri-boot.service` loads the last good ruleset early in boot, and `foomuuri.service` is ordered before the network comes up.
- **Containers as targets.** The skip list becomes `foomuuri_skip_connections`, defaulting to Docker connection names. Incus system containers have their own network namespace and run the role fully.
- **Kernel modules in containers.** A container cannot load modules. The netfilter modules must be loaded on the Incus host or listed in the instance's `linux.kernel_modules` option <a id="cite-linuxcontainers-nd"></a>[(Linux Containers, n.d.)](#ref-linuxcontainers-nd).

## Testing

Testing uses Incus throughout, in three tiers. Docker is not used.

| Tier | When it runs | What it proves |
| --- | --- | --- |
| Incus system containers | Every change | Install, conflicts, staged check, fragments, purge, port probes over IPv4 and IPv6 |
| Incus VMs | Before each release | Real distribution kernel, reboot persistence, lockout rollback, Incus host profile |
| Metal | Release candidates | Real hardware, unreliable links, console recovery |

### Test harness

- **Molecule driver.** Molecule's default (delegated) driver, with `create.yml` and `destroy.yml` playbooks that call `incus launch` and `incus delete`. A scenario variable adds `--vm`.
- **Connection.** `community.general.incus`, which runs tasks through the Incus command line with no SSH setup <a id="cite-ansible-nd"></a>[(Ansible Project, n.d.)](#ref-ansible-nd).
- **Dedicated Incus project.** Test instances live in their own project, for example `fw-test`, and are destroyed after each scenario. Long-lived instances such as `deb13`, `ubu2404` and `rky9` are never targets.
- **Probe host.** Each scenario has a target and a probe on the same bridge, so tests send real traffic.
- **Lockout scenarios use SSH.** The Incus connection does not use the instance's network, so lockout tests must connect over SSH.
- **Mint image.** If the Incus image server has no Mint 22.3 image, the scenario builds one from the Ubuntu 24.04 image by adding Mint's repositories, the same way the metal install layers Mint over Ubuntu Server.

### Scenarios

| Scenario | Checks |
| --- | --- |
| Defaults only | SSH, DHCP and ping accepted, all outbound accepted, a second run reports no changes |
| Two passes | Pass one allows SSH, pass two adds HTTP and HTTPS, the probe reaches all three, pass one is not changed by pass two |
| Fragment removal | Pass two's fragment set to absent, HTTP closed again, SSH still open |
| Purge `unmanaged` | A hand-written file is removed, fragments survive |
| Purge `fragments` | An undeclared fragment is removed in a single-play run, and the assertion refuses the mixed case |
| Terminal rule refused | A `drop` at the end of `10-base` fails the assertion before anything is deployed |
| Bad configuration | A syntax error fails the staged check and the live configuration is unchanged |
| Conflicts | ufw active with rules before the run, no ufw rules left after it |
| Lockout | Over SSH, a configuration without SSH is refused, and the rollback fires when the confirmation task is skipped |
| Persistence | Reboot, then repeat the probe checks (VM tier) |
| Incus host | Incus installed inside a VM, the Incus profile applied, a nested container gets an address and reaches the outside (VM tier) |

The first-release matrix is three images in both Incus tiers: Debian 13, Ubuntu 24.04 and Linux Mint 22.3.

### Storage

The NVMe disk on the more powerful laptop is enough for this matrix: three images, containers for most runs, and VM scenarios run one or two at a time. Instances are destroyed after every scenario, including failed ones, and unused cached images are removed. The Red Hat family adds two or three more images later.

### On metal

The metal host is a laptop installed by the raw-node installer and handed to Ansible. The installer already produces Debian 13 and Ubuntu 24, and Mint 22.3 is layered over Ubuntu 24.04, so every first-release distribution can be tested on metal by reinstalling the same laptop.

- **Install-to-firewall handoff.** The firewall is in place on first boot after the installer's handoff, with no manual steps.
- **Lockout recovery** with console access.
- **Low-bandwidth or intermittent links**, simulated with `tc netem`, to check that a dropped Ansible session never leaves a half-applied firewall.
- **The workstation case.** Mint 22.3 with Incus, to confirm the Incus profile on real hardware.

## Time estimate

Estimates assume one person experienced with Ansible and Linux firewalls, about six productive hours a day.

| Phase | Work | Days |
| --- | --- | --- |
| Foundations | Repository in Robert's style, lookup tables, argument specs, skip variable, Incus test harness | 2 to 3 |
| Install and conflicts | Packages for three distributions, version fact, `ufw disable`, service masking | 1 to 2 |
| Configuration model | Zones, rules, macros, address sets, raw files, staged check, reload handler | 3 to 4 |
| Fragments and purge | Fragment files, marker, merge order, terminal rule assertion, three purge modes, inventory layers | 2 to 3 |
| Safety | SSH assertion on merged configuration, timed rollback, Incus host profile | 2 to 3 |
| Test matrix | Scenarios above on three images in both tiers, metal passes | 3 to 5 |
| Documentation and release | README, examples for additive passes and Incus hosts, changelog, Galaxy release | 2 to 3 |
| Total | | 15 to 23 |

### Staged delivery

- **First release, 10 to 15 days:** Foundations, Install and conflicts, Configuration model, Fragments and purge, and the container tier. This is usable on the VPS and other single-homed hosts.
- **Hardening, 5 to 8 more days:** Safety, the VM and metal tiers, and documentation. Needed before the workstation and other Incus hosts move to the role.

### What moves the estimate

- **Down from version 0.1.2 (20 to 31 days):** NAT, the ufw and firewalld backends, the overlay table, Alpine and Docker coexistence are all out of scope, and Foomuuri replaces a hand-written nftables template.
- **Up:** Red Hat-family support adds about 2 to 4 days, mostly testing.
- **Up:** a pinned source install instead of distribution packages adds about 1 to 2 days, plus ongoing update work.
- **Up:** if Ubuntu 24.04's Foomuuri 0.22 rejects syntax the role needs, working around it adds about 1 day.
- **Outside the estimate:** extending the raw-node installer to more distributions.

## Open questions

- Distribution packages (0.22 on Ubuntu 24.04 and Mint, to verify) or a pinned source install for one Foomuuri version everywhere?
- Should the timed rollback be on by default for remote hosts?
- Which fragment names should be reserved by convention, for example `10-base`, `50-incus` and `99-final`?
- License: Robert uses Apache-2.0. Which licence for this role?
- Publish as a standalone role on Galaxy under which namespace, or inside a collection?

## References

<a id="ref-ansible-nd"></a>
[Return to citation](#cite-ansible-nd)
Ansible Project. (n.d.). *community.general.incus connection: Run tasks in Incus instances using the Incus CLI*. Ansible Documentation. Retrieved November 15, 2026, from https://docs.ansible.com/ansible/devel/collections/community/general/incus_connection.html

<a id="ref-debock-2026"></a>
[Return to citation](#cite-debock-2026)
de Bock, R. (2026). *ansible-role-firewall* (Commit 1f32061) [Computer software]. GitHub. https://github.com/robertdebock/ansible-role-firewall

<a id="ref-foobaroy-2026a"></a>
[Return to citation](#cite-foobaroy-2026a)
FoobarOy. (2026a). *Basics*. Foomuuri documentation. Retrieved November 15, 2026, from https://foomuuri.foobar.fi/latest/config/basic/

<a id="ref-foobaroy-2026b"></a>
[Return to citation](#cite-foobaroy-2026b)
FoobarOy. (2026b). *foomuuri* (Commit e5ac100) [Computer software]. GitHub. https://github.com/FoobarOy/foomuuri

<a id="ref-foobaroy-2026c"></a>
[Return to citation](#cite-foobaroy-2026c)
FoobarOy. (2026c). *Installation*. Foomuuri documentation. Retrieved November 15, 2026, from https://foomuuri.foobar.fi/latest/install/

<a id="ref-foobaroy-2026d"></a>
[Return to citation](#cite-foobaroy-2026d)
FoobarOy. (2026d). *zone-zone*. Foomuuri documentation. Retrieved November 15, 2026, from https://foomuuri.foobar.fi/latest/config/section/zonezone/

<a id="ref-linuxcontainers-nd"></a>
[Return to citation](#cite-linuxcontainers-nd)
Linux Containers. (n.d.). *Instance options*. Incus documentation. Retrieved November 15, 2026, from https://linuxcontainers.org/incus/docs/main/reference/instance_options/

<a id="ref-steel-2026"></a>
[Return to citation](#cite-steel-2026)
Steel, C. (2026). *ansible-role-foomuuri* [Computer software]. GitHub. https://github.com/steelcj/ansible-role-foomuuri

## Changelog

| Version | Notes |
| --- | --- |
| 0.1.3 | Rebuilt around Foomuuri and nftables after the owner's answers: decisions recorded, first release set to Debian 13, Ubuntu 24.04 and Linux Mint 22.3, additive passes through fragments, purge modes, Incus host profile, new test scenarios, re-estimated at 15 to 23 days |
| 0.1.2 | Testing moved to Incus containers, Incus VMs and a metal laptop. Docker removed from testing. |
