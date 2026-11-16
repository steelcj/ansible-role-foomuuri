---
name: firewall-manager-choices-analysis
org: universal-cake
lang: en
type: analysis
version: 0.1.0
date: 2026-11-16
---

# Firewall manager choices and the Foomuuri bridge

Version: 0.1.0

Status: Draft

Author: Christopher Steel

An analysis of the firewall managers considered for a fleet that spans several Linux families (ufw, firewalld, hand-written nftables, a hybrid of these, and Foomuuri), and of how far Foomuuri on nftables closes the gaps between distributions. It consolidates the research in this directory and re-checks its load-bearing facts against live sources.

## Summary

The decision to run Foomuuri on nftables everywhere holds up. No other option gives one declarative rule language, validation before install, an atomic single-table reload and removal by omission on both the Debian and Red Hat families.

- **The gap is in the managers, not the kernel.** Every current distribution filters packets with the same `nf_tables` subsystem. What differs is the default manager, its rule model, and how much of nftables it can reach.
- **Foomuuri bridges the manager gap where it is packaged.** That is Debian 13, Ubuntu 24.04 and later, Fedora, and EL 9 and 10 through EPEL. One rule string means the same thing on all of them.
- **It does not bridge everything.** Alpine and other non-systemd hosts, SUSE, EL 8 and Ubuntu 22.04 are out of reach. Across the hosts it does reach, the packaged version runs from 0.22 to 0.34, so a second, smaller gap opens between Foomuuri versions.
- **firewalld everywhere is the credible runner-up.** It is packaged and vendor-maintained on more families than Foomuuri. It loses on fit: its model is imperative, inbound-centred, and needs a second language for anything beyond a port.

Three findings from the re-check change what the research says:

1. **Ubuntu 24.04's fix for the Foomuuri D-Bus flaws is only in Ubuntu Pro.** The public `universe` package, which Linux Mint 22 also uses, is still 0.22 without the fix. The research's reason for preferring distribution packages, that security updates arrive with the system, is not true on two of the three first-release targets.
2. **The D-Bus handler starts with the firewall by default, and the role did not turn it off.** Masking `foomuuri-dbus.service` on hosts with fixed interfaces closes this on every version. The role now does so by default, through `foomuuri_dbus: false`. That change has not been tested on a host.
3. **The version table in the research is stale.** `trixie-backports`, EPEL 9 and the AUR are all at 0.34 now, and EPEL 10 is at 0.33.

## Scope and method

| Item | Value |
| --- | --- |
| Research read | Every file in `en/docs/research/`, the design note on pinning in `en/docs/design/`, and `uc-raw-node-contract.md` |
| Role read | `ansible-role-foomuuri` 0.1.0 as it stands in this repository, untested |
| Re-checked on 2026-11-16 | Package versions, the security notices, Foomuuri's systemd units and command line at 0.22 and on `main`, the ufw nftables bug, firewalld's nftables design note, Docker's nftables page |
| Not done | Nothing was run on a host. No ufw, firewalld or Foomuuri configuration was tested for this document. |

Each claim carries one of four bases:

- **Checked:** read from a live source on 2026-11-16, cited.
- **Research:** taken from the research documents and not re-checked.
- **Analyst:** the analyst's own knowledge or judgement, not sourced.
- **To verify:** open, with the check that would settle it.

### Research documents

| Document | What it contributes |
| --- | --- |
| [state-of-linux-firewalls.md](artifacts/state-of-linux-firewalls.md) | ufw runs through the `iptables-nft` shim. Survey of native nftables front ends. |
| [foomuuri-is-a-free-open-source-multizone-firewall-generator.md](artifacts/foomuuri-is-a-free-open-source-multizone-firewall-generator.md) | What Foomuuri is, its security record, links to other Foomuuri roles |
| [firewall-role-analysis.md](artifacts/firewall-role-analysis.md) | First plan: fork `robertdebock.firewall` into ufw, firewalld and nftables backends with an overlay table. 20 to 31 days. |
| [you-may-well-be-right-and-my-short-answer-no-was-too-blunt.md](artifacts/you-may-well-be-right-and-my-short-answer-no-was-too-blunt.md) | What Robert's role can do with small edits, and why firewalld stays the hard backend |
| [firewall-role-analysis-v0.1.2.md](artifacts/firewall-role-analysis-v0.1.2.md) | The same plan with testing moved to Incus |
| [ansible-role-foomuuri-analysis-v0.1.0.md](artifacts/ansible-role-foomuuri-analysis-v0.1.0.md) | Pre-build analysis: Robert's role and Tubby1981's Foomuuri role reviewed, six configurations run through `foomuuri check` |
| [firewall-role-analysis-v0.1.3.md](artifacts/firewall-role-analysis-v0.1.3.md) | The decision: Foomuuri everywhere, fragments, purge, Incus profile. 15 to 23 days. |

## One kernel, many managers

The research's first finding is the one everything else rests on: the packet filter is already the same everywhere. On current releases even the `iptables` command is the nft-backed variant, so rules written in iptables syntax still end up in `nf_tables` (Research).

```text
Intent        "SSH from the admin network, HTTP from anywhere"
                              |
Manager       ufw   |   firewalld   |   Foomuuri   |   hand-written file
                |             \            |              /
Userspace     iptables-nft shim            nft (native)
                              |
Kernel        nf_tables, the same on every current distribution
```

A multi-distribution fleet therefore has no kernel gap to bridge. It has a manager gap, and it shows up in nine ways.

| ID | Gap | What it looks like in a fleet |
| --- | --- | --- |
| G1 | Different manager per family | ufw on Ubuntu and by habit on Debian, firewalld on Red Hat and SUSE, neither on Alpine |
| G2 | Different rule models | ufw has a flat list of host rules. firewalld has zones, services, rich rules and policies. The only shape they share is "open this port to everyone". |
| G3 | Different reach into nftables | ufw cannot express sets or tunable rate limits, because it speaks iptables. firewalld reaches them through rich rules. |
| G4 | No removal by omission | Both are driven item by item. A rule dropped from the inventory stays on the host until something deletes it by name. |
| G5 | Validation after the fact | A bad rule is found when it is applied, or when the service next starts |
| G6 | Unclear ownership of the ruleset | Leftover rules from one manager silently drop what another accepts. A stock `nftables.service` can flush every table on the host. |
| G7 | Uneven direction support | Inbound is easy everywhere. Outbound and forwarded traffic use a different mechanism in each tool. |
| G8 | Lockout and half-applied changes | A dropped SSH session in the middle of a rule-by-rule change leaves the firewall in between two states |
| G9 | Untestable in the usual CI | Container tests skip every firewall task, so cross-distribution claims are never exercised |

Robert de Bock's role is the worked example of G1 and G2. It maps each family to its native manager, and the price is that it can only express the shape all three share. Its service and package maps also disagree, which leaves Rocky, AlmaLinux, EL 10, SLES and Tumbleweed with firewalld installed and `ufw` as the service name (Research, by reading the code).

## The choices

### ufw

ufw is a command-line front end that writes iptables rules. It is installed on Ubuntu and Mint and is the usual choice on Debian.

- **Strengths.** Simple, well documented, part of Ubuntu's supported archive, with an Ansible module (`community.general.ufw`) that already takes source, interface, direction, logging and comment keys (Research).
- **No native nftables path.** ufw drives `iptables`, and reaches the kernel through the shim. The request for an nftables backend has been open since 2020. The maintainer's last word on it, from May 2020, is that a port is "doable and planned in the medium to long term". On 2026-11-16 the bug is still triaged as a wishlist item with no assignee <a id="cite-launchpad-nd"></a>[(Launchpad, n.d.)](#ref-launchpad-nd) (Checked).
- **Consequences.** No named sets, one fixed rate limit, and ICMP control only by editing `before.rules` in iptables-restore syntax, with IPv4 and IPv6 in separate files (Research).
- **Off its home ground.** On Red Hat-family hosts ufw is a community package in EPEL at 0.35, against 0.36 on Debian and Ubuntu, so even ufw-everywhere has version skew (Research).
- **Leaves rules behind.** Stopping the service does not remove its rules from the kernel. Only `ufw disable` does (Research).

Verdict: a good tool for one host and one administrator. As the base for a fleet with complex rules it fails G3, G4 and G5, and it does not bridge G1 without becoming the foreign manager on half the fleet.

### firewalld

firewalld is a daemon with a D-Bus interface, the default on RHEL, Fedora and openSUSE, and packaged in Ubuntu's standard repositories (Research).

- **Strengths.** Native nftables since 0.6.0. It keeps everything in its own `firewalld` table and flushes only that table on reload <a id="cite-garver-2018"></a>[(Garver, 2018)](#ref-garver-2018) (Checked). It is vendor-supported where it is the default, and other software integrates with it: Docker's nftables mode puts its bridges in a firewalld zone named `docker` and adds its own forwarding policy <a id="cite-docker-nd"></a>[(Docker Inc., n.d.)](#ref-docker-nd) (Checked).
- **Two languages.** A port or service is one command. A source restriction, an action other than accept, a rate limit or a log is a rich rule or a zone design. The research calls firewalld "the hard backend" for this reason: these are new tasks, not extra parameters (Research).
- **Inbound-centred.** Zones describe what may reach the host. Outbound and forwarded traffic are handled by policies between zones, a separate mechanism (Research).
- **Imperative state.** It keeps a runtime and a permanent configuration, and `ansible.posix.firewalld` manages one item per task. Removing a rule means naming it with `state: disabled`. A purge of undeclared rules has no natural form (Analyst).
- **Direct rules are a trap.** They still go through iptables, and they outrank every other firewalld rule [(Garver, 2018)](#ref-garver-2018) (Checked).

Verdict: firewalld is the strongest alternative, and the only other one that could be the single manager on every family. It closes G1 and G6. It leaves G2 half open (two languages inside one tool), and does not close G4 or G7 without templating its XML files by hand. The research never costed firewalld-everywhere as a design of its own, see Open questions.

### Hand-written nftables

A role renders an nftables file from a template, checks it with `nft -c -f`, and loads it atomically.

- **Strengths.** The most universal option: `nft` is packaged on every family in scope, Alpine included (Research). It is native, atomic, has no daemon, and removal by omission is free because the file is the whole truth.
- **The role becomes a compiler.** Zones, rule ordering, service names, address handling for both IP families, sets fed from DNS or URLs, and validation of all of it move into Jinja. The decision document puts it this way: the role "would then own a small compiler" (Research).
- **Sharp edge.** The distribution's own `nftables.service` usually loads a file that begins with `flush ruleset`, which removes every other table on the host (Research).
- **Cost.** The first plan priced an nftables-only role at 8 to 12 days, but for a flat rule model without zones (Research).

Verdict: closes every gap except the cost of building and maintaining the abstraction. It is the right fallback for the hosts Foomuuri cannot reach.

### Native manager per family, with an nftables overlay

This was the first recommendation: keep ufw and firewalld for allow rules, add an nftables backend for Alpine, and load a role-owned overlay table for sets, rate limits and NAT.

- **Strengths.** Each host keeps its vendor default, and nothing is switched off.
- **The overlay can only tighten.** A drop in any table is final, but an accept is not: an accepted packet is still seen by every other chain on the same hook [(Garver, 2018)](#ref-garver-2018) (Checked). The overlay can block, but it can never open a port the native manager closes.
- **Three back ends, one model.** Every capability has to be mapped three times, and the common model can only promise what the weakest back end delivers.
- **Cost.** 20 to 31 days, the highest of the options that were estimated (Research).

Verdict: it preserves G1 and G2 and pays to paper over them. It was dropped once the owner preferred nftables on the Red Hat family as well.

### Foomuuri

Foomuuri is a firewall generator: it reads its own rule language from `/etc/foomuuri` and compiles it into one nftables table. It is GPL-2.0, written in Python, and has been developed since 2023 (Research). Its feature list covers zones, filtering in all three directions, macros and templates, automatic splitting of rules by IP family, SNAT and DNAT, rate limiting, address lists refreshed from DNS or URLs, and raw nftables includes <a id="cite-foobaroy-2026b"></a>[(FoobarOy, 2026b)](#ref-foobaroy-2026b) (Checked).

The properties that matter for a fleet are covered under How Foomuuri bridges the gaps, and its limits under Where the bridge stops.

### Others

| Tool | What it is | Why it is not a candidate |
| --- | --- | --- |
| nftfw | Native nftables, driven by files in `incoming.d`, `outgoing.d` and `blacklist.d` | Built for Debian and Ubuntu only, so it does not bridge G1 (Research) |
| Webmin | Browser interface with an nftables module | An interactive tool, not a configuration format an Ansible role can own (Research) |
| `iptables` with `netfilter-persistent` or `iptables-services` | Saved iptables rule files | iptables syntax on the shim, separate IPv4 and IPv6 files, and Robert's template for it is IPv4 only (Research) |
| ferm, FireHOL, Shorewall | Older rule generators | iptables-era tools (Analyst). Foomuuri's own install steps list them among the services to disable <a id="cite-foobaroy-2026c"></a>[(FoobarOy, 2026c)](#ref-foobaroy-2026c) (Checked). |

## Comparison

| Criterion | ufw | firewalld | Hand-written nftables | Foomuuri |
| --- | --- | --- | --- | --- |
| Path to the kernel | `iptables-nft` shim | Native | Native | Native |
| Rule model | Flat host rules | Zones, services, rich rules, policies | Whatever the template implements | Zone pairs, one rule language |
| Same syntax on Debian and Red Hat families | Only through EPEL, older version | Yes | Yes | Yes, where packaged |
| Reaches Alpine and SUSE | No | SUSE yes, Alpine no | Yes | No |
| Declarative files as the interface | Partly, the CLI is the interface | Partly, runtime and permanent state | Yes | Yes |
| Removal by omission | No | No | Yes | Yes |
| Check before install | Per command, `--dry-run` | `--check-config`, on the installed configuration | `nft -c -f` on any file | `foomuuri check` on any directory |
| Apply | Rule by rule, or full reload | Reload of its own table | Atomic | Atomic, its own table only |
| Leaves other tables alone | Shares the iptables tables | Yes | Only if the file avoids `flush ruleset` | Yes |
| Outbound and forwarded traffic | Yes, separate commands | Through policies | Yes | Yes, same form as inbound |
| Sets, tunable rate limits, NAT | No sets, fixed limit, NAT by hand | Yes, through rich rules and zone options | Yes | Yes |
| Vendor default somewhere | Ubuntu | RHEL, Fedora, SUSE | No | No |
| Ansible interface | `community.general.ufw` | `ansible.posix.firewalld` | Template and validate | Template and staged check |
| Community | Large | Large | Large | Small, one upstream author |

The rows for ufw, firewalld and hand-written nftables are from the research plus the analyst's knowledge of the tools. None of them was tested for this document.

### Cost, from the research

| Design | Estimate | Covers |
| --- | --- | --- |
| Native managers plus overlay | 20 to 31 days | Debian, Red Hat and SUSE families, Alpine |
| nftables-only role, flat rule model | 8 to 12 days | Any distribution, no zones |
| Foomuuri role, first release | 15 to 23 days | Debian 13, Ubuntu 24.04, Linux Mint 22.3, with fragments, purge, rollback and an Incus profile |
| Foomuuri role, Red Hat family added | 2 to 4 days more | Fedora, EL 9 and 10 |

All four are the analyst's judgement in the earlier documents, not measured figures, and they do not cover the same scope. The hybrid buys more families for more days. The Foomuuri role buys fewer families but more behaviour (fragments, purge, rollback) for fewer days. What the Foomuuri figures show is where the work goes: adding the Red Hat family is packaging and testing only, because no rule has to be mapped to a second manager.

## Foomuuri at a glance

A SWOT summary of the chosen option. It adds no evidence: each entry is argued, with its basis, in How Foomuuri bridges the gaps, Where the bridge stops, and The contract is the deeper bridge. Strengths and weaknesses are properties of Foomuuri itself. Opportunities and threats come from the distributions, upstream and the fleet around it.

### Strengths

- **One rule language** on every family where it is packaged
- **Check before install**, on a staging directory, from 0.22 on
- **Atomic reload of one table**, leaving every other table alone
- **Removal by omission**, because configuration files are the whole truth
- **Files merge**, which is what makes additive passes possible
- **Every direction in one form**: inbound, outbound and forwarded traffic

### Weaknesses

- **One upstream author** and a small community
- **Its D-Bus handler starts by default**, and was open to any local user before 0.31
- **systemd only**, so no Alpine
- **Syntax and commands change between releases**, with no long-term support branch

### Opportunities

- **Newer releases already carry fixed versions**: Ubuntu 26.04, EPEL 9 and 10, Fedora
- **The raw node contract is written in nftables terms**, so the manager can be replaced without rewriting the promises
- **The output is plain nftables**, so leaving Foomuuri is cheap
- **The Red Hat family costs days, not weeks**, because no rule needs mapping

### Threats

- **Ubuntu 24.04 delivers Foomuuri fixes only through Ubuntu Pro**, and Mint draws on the same archive
- **Packaged versions run from 0.22 to 0.34** across the fleet
- **No package for SUSE**, EL 8 or Ubuntu 22.04
- **Compliance scanners and other software expect the native manager**
- **Upstream maintenance could stall**

### What the SWOT does not show

A SWOT lists entries side by side as if they weighed the same. They do not. Two outweigh the rest: the D-Bus handler under Weaknesses, and the Ubuntu Pro route under Threats. Together they mean a default Foomuuri install on Ubuntu 24.04 or Mint is exposed and will not be fixed by routine updates. Recommendation 2, now in the role, and recommendation 3 answer them.

It also says nothing about the alternatives. The Comparison table above does that job.

## How Foomuuri bridges the gaps

| Gap | Mechanism | Basis |
| --- | --- | --- |
| G1, one manager | The same package name and the same `/etc/foomuuri` on Debian, Ubuntu, Fedora and EPEL. The role switches the native manager off instead of driving it. | Checked [(FoobarOy, 2026c)](#ref-foobaroy-2026c) |
| G2, one rule model | Rules are Foomuuri strings keyed by section name, such as `public-localhost`. The role passes them through and maintains no rule language of its own. | Research |
| G3, full reach | Compiles to native nftables, with sets, rate limits and NAT in the rule language, and `.nft` includes for anything else | Checked [(FoobarOy, 2026b)](#ref-foobaroy-2026b) |
| G4, removal by omission | Each fragment file is rendered whole. A rule removed from the variables is gone after the next run. | Research, role design |
| G5, validation first | `foomuuri check` builds the ruleset and runs `nft --check` on it. `--set=etc_dir=DIR` points it at a staging directory. Both exist in 0.22, the oldest version in scope. | Checked <a id="cite-foobaroy-2023"></a>[(FoobarOy, 2023)](#ref-foobaroy-2023) |
| G6, clear ownership | One table, `inet foomuuri`, deleted and redefined in one transaction. No `flush ruleset`. Tables owned by Incus survive. | Research, read from source at `e5ac100` |
| G7, every direction | Inbound, outbound and forwarded traffic are all zone-pair sections with the same syntax | Research |
| G8, lockout | Atomic reload, last good ruleset kept for boot. The role adds an SSH assertion and a timed rollback. | Research. The rollback is designed, not built. |
| G9, testing | Not a Foomuuri property. The role tests in Incus system containers and VMs, which have their own network stack. | Research |

Two further properties come from Foomuuri's file handling and are what make the role's additive passes possible. It reads every `*.conf` file under its directory in name order, and when a section appears in more than one file its rules are appended, not replaced (Research, confirmed in the parser). A base playbook and a web server playbook can therefore each own one file and never restate each other.

### The same intent in three tools

For operators arriving from either native manager:

| Intent | ufw | firewalld | Foomuuri, in `public-localhost` |
| --- | --- | --- | --- |
| SSH from anywhere | `ufw allow ssh` | `firewall-cmd --permanent --add-service=ssh` | `ssh` |
| TCP 8080 from one network | `ufw allow from 192.0.2.0/24 to any port 8080 proto tcp` | `firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.0.2.0/24" port port="8080" protocol="tcp" accept'` | `tcp 8080 saddr 192.0.2.0/24` |
| All outbound | Default policy | Default, zones do not filter it | `accept` in `localhost-public` |
| Remove a rule | `ufw delete allow ssh` | `--remove-service=ssh`, then reload | Delete the line, reload |

The second row is the gap in one line: the same rule is a different shape in each native tool, and on firewalld it moves to a different language.

## Where the bridge stops

### Platforms out of reach

| Platform | Status | Basis |
| --- | --- | --- |
| Alpine and other non-systemd hosts | Not packaged, and Foomuuri ships systemd units only | Research. No Alpine package is listed today <a id="cite-repology-nd"></a>[(Repology, n.d.)](#ref-repology-nd) (Checked). |
| openSUSE and SLES | No package listed. SUSE reviewed Foomuuri in late 2025 because a community member asked for it in Tumbleweed. Whether it was accepted was not found. | Checked [(Repology, n.d.)](#ref-repology-nd), <a id="cite-gerstner-2026"></a>[(Gerstner, 2026)](#ref-gerstner-2026). To verify with `zypper info foomuuri`. |
| EL 8 | No package, and Foomuuri needs Python 3.9 or later | Research. The requirement is Checked [(FoobarOy, 2026c)](#ref-foobaroy-2026c). |
| Ubuntu 22.04 | Not packaged, source install only | Research |
| Debian 12 | Backports only, at 0.27 | Checked [(Repology, n.d.)](#ref-repology-nd) |
| Arch | AUR only, a user-built package, now current at 0.34 | Checked [(Repology, n.d.)](#ref-repology-nd) |

The research's first plan covered SUSE and Alpine. The Foomuuri decision gave both up. That is acceptable for the stated fleet, and it should be a recorded trade, not an accident.

### A second gap, between Foomuuri versions

Trading several managers for one manager at several versions is a good trade, but it is still a gap.

| Platform | Packaged version | D-Bus fixes | Basis |
| --- | --- | --- | --- |
| Ubuntu 24.04 | 0.22, released 2023-12-12 | Only in `0.22-1ubuntu0.1~esm1`, which requires Ubuntu Pro | Checked <a id="cite-canonical-2026"></a>[(Canonical, 2026)](#ref-canonical-2026) |
| Linux Mint 22 | Ubuntu 24.04's `universe` | Same public package, not fixed | Research for the package source. To verify whether Pro can be attached on Mint. |
| Debian 13 | 0.27 | Fixed in `0.27-2+deb13u1`, DSA-6095-1 | Checked <a id="cite-debian-nd"></a>[(Debian Security Team, n.d.)](#ref-debian-nd) |
| Ubuntu 25.10 | 0.27 | Fixed in the public archive | Checked [(Canonical, 2026)](#ref-canonical-2026) |
| Ubuntu 26.04 | 0.31 | Fixed upstream | Checked [(Repology, n.d.)](#ref-repology-nd) |
| Fedora 42 to 44 | 0.32 to 0.33 in updates | Fixed upstream | Checked [(Repology, n.d.)](#ref-repology-nd) |
| EPEL 10 | 0.33 | Fixed upstream | Checked [(Repology, n.d.)](#ref-repology-nd) |
| EPEL 9, `trixie-backports`, AUR | 0.34 | Fixed upstream | Checked [(Repology, n.d.)](#ref-repology-nd) |

Behaviour that differs across that range <a id="cite-foobaroy-2026a"></a>[(FoobarOy, 2026a)](#ref-foobaroy-2026a) (Checked):

| Version | Change | Effect on the role |
| --- | --- | --- |
| 0.24 | `start-or-good` removed, `foomuuri-boot.service` added | The boot fallback is a separate unit only from 0.24. On 0.22 it is built into `foomuuri.service`. The role already enables the unit only where it exists. |
| 0.27 | `mark_set` becomes a matcher, `mark_save` and `mark_restore` removed | Keep packet marks out of the role's defaults |
| 0.29 | `try-reload` added | Not available on 0.22 or 0.27, so the role's own timed rollback is needed anyway |
| 0.31 | Polkit authorization on D-Bus methods, interface names checked, units hardened | See the next section |
| 0.33 | `foomuuri list` renamed to `foomuuri ruleset list` | Tests should read the table with `nft list table inet foomuuri`, as `verify.yml` does |
| 0.34 | The `\|missing-ok` iplist filter replaced by `missing_ok=yes`. In 0.35 `overwrite` will default to `no`. | Address list settings need a version guard |

The first-release targets sit at the old end of this range, and the Red Hat family at the new end. Role defaults that pass `foomuuri check` on 0.22 and on 0.34 are the practical definition of portable.

### Security maintenance is uneven

The research records two local flaws in Foomuuri's D-Bus service, CVE-2025-67603 and CVE-2025-67858, and advises keeping automatic updates on. The re-check shows that this is not enough on Ubuntu 24.04.

- **The flaws.** Any local user, including `nobody`, could call the D-Bus methods and move interfaces between zones, which can weaken the firewall. Interface names were not validated. SUSE found no path to code execution, and described the upstream author as prompt and cooperative: confirmed the next day, fixed in 0.31 [(Gerstner, 2026)](#ref-gerstner-2026) (Checked).
- **Ubuntu 24.04.** The fix was published on 27 May 2026, and only through ESM Apps. The notice says a community fix might become public later [(Canonical, 2026)](#ref-canonical-2026) (Checked). A host without Ubuntu Pro still runs the vulnerable 0.22.
- **This is structural.** Foomuuri is in `universe` on Ubuntu, and this notice shows how fixes for it reach an LTS release. Any future flaw should be expected to follow the same route (Analyst).
- **The handler runs by default.** In 0.22 and on `main`, `foomuuri.service` declares `Wants=foomuuri-dbus.service`, so starting the firewall starts the D-Bus handler too <a id="cite-foobaroy-2026d"></a>[(FoobarOy, 2026d)](#ref-foobaroy-2026d) (Checked, from the unit files). When this analysis was first written, the role started `foomuuri` and masked nothing of Foomuuri's own. It now stops and masks the handler before Foomuuri starts, unless `foomuuri_dbus` is true. To verify on a host: `systemctl is-active foomuuri-dbus` on a stock Ubuntu 24.04 install of the package, and again after the role has run.

The D-Bus handler exists so that NetworkManager can move interfaces between zones. A host whose zones are fixed in configuration, which is every first-release target, does not need it.

### What replacing the native manager costs

- **Vendor support and compliance.** On RHEL and SUSE the supported manager is firewalld. Hardening benchmarks and their scanners look for ufw, firewalld or a plain nftables service, and may flag a Foomuuri host that meets the intent (Analyst).
- **Integrations.** Software that configures the native manager gets no answer. Docker's firewalld zone is one example [(Docker Inc., n.d.)](#ref-docker-nd). Foomuuri's firewalld emulation covers NetworkManager's zone calls, not the whole firewalld interface (Research).
- **Docker.** It writes its own rules and bypasses a host firewall for published ports whichever manager is used. Its nftables mode is still marked experimental, and expects user rules in separate tables ordered by priority [(Docker Inc., n.d.)](#ref-docker-nd) (Checked). Out of scope while no target runs Docker.
- **One upstream.** Foomuuri has one principal author and a small community. There are far fewer tutorials and third-party answers than for ufw or firewalld (Research).
- **Drops in other tables still win.** Foomuuri owning one table does not stop leftover ufw rules from dropping what it accepts. The role's `ufw disable` step exists for this (Research).

## The contract is the deeper bridge

The raw node contract already separates what is promised from which tool delivers it. Clauses RN-FW-01 to RN-FW-04 are written in nftables terms: nftables is the only packet filter, one manager owns one table, nothing flushes the whole ruleset, the firewall is up before the network. Only RN-FWM-01 names Foomuuri, and the contract says that replacing it is a major version change while the firewall clauses stay.

That gives the fleet two layers of bridge:

- **Foomuuri** bridges the distributions it is packaged for, cheaply.
- **The nftables clauses** bridge everything else. A host Foomuuri cannot reach can still satisfy RN-FW-01 to RN-FW-04 with a hand-written table.

It also bounds the risk of depending on a small project. Foomuuri's output is an ordinary nftables table, readable with `nft list table inet foomuuri`, and the last ruleset that loaded is kept as a plain nftables file (Research). If Foomuuri were abandoned, the compiled ruleset could be loaded by `nft` directly while a replacement is written. This exit has not been rehearsed (Analyst).

## Corrections to the research

| Document | Says | Found on 2026-11-16 |
| --- | --- | --- |
| state-of-linux-firewalls.md | Foomuuri on Ubuntu comes "via PPA / GitHub package" | It is in `universe` from Ubuntu 23.10 on [(FoobarOy, 2026c)](#ref-foobaroy-2026c) |
| foomuuri-is-a-free-open-source… | Keep automatic security updates enabled on Ubuntu | Not sufficient on 24.04: the fix is in ESM Apps only [(Canonical, 2026)](#ref-canonical-2026) |
| firewall-role-analysis-v0.1.3.md | `trixie-backports` carries 0.31 | It carries 0.34 [(Repology, n.d.)](#ref-repology-nd) |
| firewall-role-analysis-v0.1.3.md | Ubuntu 24.04 has 0.22, to verify | Confirmed by the archive listing. A host check with `apt-cache policy foomuuri` is still worth doing. |
| firewall-role-analysis-v0.1.3.md | `foomuuri-boot.service` loads the last good ruleset at boot | True from 0.24. On 0.22, `foomuuri.service` runs `start-or-good` instead [(FoobarOy, 2023)](#ref-foobaroy-2023). |
| firewall-role-analysis-v0.1.3.md | Distribution packages keep maintenance with the distributions | True on Debian 13. Not true on Ubuntu 24.04 without Pro, or on Mint. |
| ansible-role-foomuuri-analysis-v0.1.0.md | EPEL has 0.28 in stable. The AUR shows 0.28. | EPEL 9 has 0.34 and EPEL 10 has 0.33. The AUR has 0.34. |
| Design note on pinning | Pin the `trixie-backports` package, 0.31 or later | That package is now 0.34, which brings the 0.33 and 0.34 syntax changes with it |

One point in the research is confirmed and worth stating, because the whole staged-check design depends on it: `--set=etc_dir=` and `--set=state_dir=` are parsed by Foomuuri 0.22 [(FoobarOy, 2023)](#ref-foobaroy-2023).

## Recommendations

1. **Keep the decision.** Foomuuri on nftables is the best fit for the stated fleet and the stated needs: additive passes, purge, validation before install.
2. **Turn the D-Bus handler off by default.** Done in the role, pending test. `foomuuri_dbus: false` is now the default, and the role stops and masks `foomuuri-dbus.service` before Foomuuri starts. `molecule/default/verify.yml` asserts that the unit is masked and not running. Because `foomuuri.service` only wants that unit, masking it should not affect the firewall, and it closes the two known flaws on every version, patched or not. Both points are inferences from the unit files. The first Molecule run on Ubuntu 24.04 should confirm them. The role does not check the Foomuuri version when `foomuuri_dbus` is true, because a plain version test would refuse Debian 13's patched 0.27.
3. **State the version policy per platform.** Keep distribution packages as the default. On Ubuntu 24.04 and Mint, record that the archive does not deliver Foomuuri fixes without Pro, and offer the pinned `.deb` from the design note as an opt-in install method for those hosts. Pinning everywhere would take over updates on Debian 13 and EPEL, where the distributions already do the work well.
4. **Hold the role's defaults to 0.22 syntax, and prove it.** Run `foomuuri check` on the rendered defaults against 0.22 and against the newest release in every test run. This is the cheapest guard against the second gap.
5. **Add platforms in order of cost.** Ubuntu 26.04 first, since it carries a fixed 0.31 in the archive. Then EL 9 and 10 through EPEL, then Fedora. Leave Debian 12 and Arch as optional, and say plainly that their packages have no security team behind them.
6. **Do not add ufw or firewalld back ends for the hosts Foomuuri misses.** If Alpine, SUSE or EL 8 ever enter the fleet, meet RN-FW-01 to RN-FW-04 there with a small hand-written nftables table. That keeps one kernel interface and avoids bringing G2 back.
7. **Rehearse the exit once.** On a test VM, stop Foomuuri, load its last good ruleset with `nft -f`, and confirm the host stays filtered. It turns the low lock-in from a claim into a tested procedure.
8. **Refresh the research.** Apply the corrections above to the version tables in v0.1.3 and the pre-build analysis.

## Open questions

- Is Ubuntu Pro in use, or acceptable, on the Ubuntu 24.04 hosts? It changes recommendation 3 for those hosts, and it does not help Mint unless Pro can be attached there.
- Does `foomuuri-dbus.service` actually run on a default Ubuntu 24.04 install, and does masking it leave `foomuuri reload` and the iplist timer working?
- Should firewalld-everywhere be costed as the fallback design? The research compares Foomuuri with a hybrid and with hand-written nftables, but never with firewalld as the single manager. It is the option to reach for if Foomuuri's maintenance stalled.
- Is SUSE ever expected in the fleet? If so, the Tumbleweed packaging outcome is worth checking before the decision is treated as final.
- Will any target need NetworkManager to switch zones, for example a laptop moving between networks? That is the one case where the D-Bus handler has to stay on, with `foomuuri_dbus: true`. It should then run 0.31 or later, or a patched distribution package, which the role leaves to the operator to check.

## References

<a id="ref-canonical-2026"></a>
[Return to citation](#cite-canonical-2026)
Canonical. (2026, May 27). *USN-8326-1: Foomuuri vulnerabilities*. Ubuntu Security Notices. https://ubuntu.com/security/notices/USN-8326-1

<a id="ref-debian-nd"></a>
[Return to citation](#cite-debian-nd)
Debian Security Team. (n.d.). *CVE-2025-67603*. Debian Security Bug Tracker. Retrieved November 16, 2026, from https://security-tracker.debian.org/tracker/CVE-2025-67603

<a id="ref-docker-nd"></a>
[Return to citation](#cite-docker-nd)
Docker Inc. (n.d.). *Docker with nftables*. Docker Docs. Retrieved November 16, 2026, from https://docs.docker.com/engine/network/firewall-nftables/

<a id="ref-foobaroy-2023"></a>
[Return to citation](#cite-foobaroy-2023)
FoobarOy. (2023). *foomuuri* (Version 0.22) [Computer software]. GitHub. https://github.com/FoobarOy/foomuuri/tree/v0.22

<a id="ref-foobaroy-2026a"></a>
[Return to citation](#cite-foobaroy-2026a)
FoobarOy. (2026a). *Changelog*. GitHub. Retrieved November 16, 2026, from https://github.com/FoobarOy/foomuuri/blob/main/CHANGELOG.md

<a id="ref-foobaroy-2026b"></a>
[Return to citation](#cite-foobaroy-2026b)
FoobarOy. (2026b). *Foomuuri*. Foomuuri documentation. Retrieved November 16, 2026, from https://foomuuri.foobar.fi/latest/

<a id="ref-foobaroy-2026c"></a>
[Return to citation](#cite-foobaroy-2026c)
FoobarOy. (2026c). *Installation*. Foomuuri documentation. Retrieved November 16, 2026, from https://foomuuri.foobar.fi/latest/install/

<a id="ref-foobaroy-2026d"></a>
[Return to citation](#cite-foobaroy-2026d)
FoobarOy. (2026d). *foomuuri.service* [systemd unit]. GitHub. Retrieved November 16, 2026, from https://github.com/FoobarOy/foomuuri/blob/main/systemd/foomuuri.service

<a id="ref-garver-2018"></a>
[Return to citation](#cite-garver-2018)
Garver, E. (2018, July 24). *nftables backend*. firewalld. https://firewalld.org/2018/07/nftables-backend

<a id="ref-gerstner-2026"></a>
[Return to citation](#cite-gerstner-2026)
Gerstner, M. (2026, January 7). *[Advisory on missing D-Bus authorization and input validation in Foomuuri]*. SUSE Security Team Blog. https://security.opensuse.org/2026/01/07/foomuuri-lack-of-dbus-authorization.html

<a id="ref-launchpad-nd"></a>
[Return to citation](#cite-launchpad-nd)
Launchpad. (n.d.). *Bug #1880453: Feasability of a nftables port*. ufw bug tracker. Retrieved November 16, 2026, from https://bugs.launchpad.net/bugs/1880453

<a id="ref-repology-nd"></a>
[Return to citation](#cite-repology-nd)
Repology. (n.d.). *foomuuri package versions*. Retrieved November 16, 2026, from https://repology.org/project/foomuuri/versions

## Changelog

| Version | Status | Notes |
| --- | --- | --- |
| 0.1.0 | Draft | First version: the research consolidated into one comparison of ufw, firewalld, hand-written nftables, the hybrid and Foomuuri, the gaps Foomuuri closes and leaves open, package versions and security status re-checked on 2026-11-16, corrections to the research, recommendations. A SWOT summary of Foomuuri was added the same day, and the text was brought in line with the role after `foomuuri_dbus` was added to it. |
