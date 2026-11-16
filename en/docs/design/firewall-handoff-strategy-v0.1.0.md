---
name: firewall-handoff-strategy
org: universal-cake
lang: en
type: design
version: 0.1.0
date: 2026-11-16
---

# Firewall handoff strategy

Version: 0.1.0

Status: Draft

Author: Christopher Steel

How `ansible-role-foomuuri` takes over the firewall of a node during onboarding, and which gaps that leaves. It replaces the more elaborate ideas in the session note of 2026-11-16 13:08:17 (guard tables, blocking package scripts) with one simple order of work.

## Strategy

The role's primary job is to set up the firewall. It does that in three steps, always in this order:

1. **Create.** Write a Foomuuri firewall that does the same thing as the raw node at handoff, and load it.
2. **Remove.** Switch off every other firewall.
3. **Document.** Any gap this order leaves is written down here, not engineered away inside the role.

Known gaps are handled outside the role, by running everything over WireGuard. The limits of that are in WireGuard as the answer to gaps.

### Why create comes first

Each firewall manager keeps its rules in its own nftables table, and the kernel runs all tables side by side. A packet must be accepted by every table, and a drop in any one is final. While two firewalls overlap, the host enforces the stricter of the two, so it is never more open than either. Creating before removing therefore leaves no unfiltered moment. Removing first does.

### What "the same thing" means

The raw node contract, clause RN-FW-03, defines the firewall at handoff:

| Direction | Contract | Role default, fragment `10-base` |
| --- | --- | --- |
| Inbound | SSH, DHCP and DHCPv6 replies, ping, the ICMP networking needs | `ssh`, `dhcp-client`, `dhcpv6-client`, `ping` in `public-localhost` |
| Outbound | Everything | `accept` in `localhost-public` |
| Forwarded | Nothing | No section, so Foomuuri's own final rule applies |

The role's defaults already match the contract when SSH is on port 22 and open to any source. See gap 4 for the other cases.

## The two paths

| Path | Starting state | What the role does | Unfiltered moment |
| --- | --- | --- | --- |
| Raw node, as the contract promises | Foomuuri is already the manager, with only `00-raw-node.conf` (RN-FWM-01). No other firewall is active (RN-FW-01). | Writes its fragment, removes the raw node file, checks the result in staging, reloads. The reload replaces the table in one step. | None. There is nothing to swap. |
| Adopted host | Another manager is active, such as ufw on an existing Ubuntu or Mint host | Create, then remove, as above | None expected. The role now follows the order, see Changes to the role. |

Most nodes take the first path. The order of work only matters on the second.

## Known gaps

None of these has been reproduced on a host. Each comes from reading the role, the contract and the tools.

| ID | Gap | Applies to | What is exposed | Handling |
| --- | --- | --- | --- | --- |
| 1 | The role removed before it created. `tasks/main.yml` ran the conflicts step, including `ufw disable`, before it started Foomuuri. | Adopted hosts | Every listening service, for the few seconds between the two tasks | Closed in the role on 2026-11-16, untested. See Changes to the role. |
| 2 | Stopping a service that flushes the whole ruleset also deletes Foomuuri's table. The distribution's `nftables.service` is the likely case. To verify what its stop action does on each platform. | Adopted hosts where that service is running | Every listening service, from the stop until the role's next task | Narrowed in the role on 2026-11-16, untested: after the remove step it checks that Foomuuri's table is still loaded, and reloads if not |
| 3 | During the overlap, the old firewall can still drop traffic that Foomuuri accepts | Adopted hosts | Nothing. Traffic is blocked, not exposed. | None needed. SSH is open in both. |
| 4 | The role's base rules use the `ssh` service, which is port 22 from any source. The contract allows another `ssh_port` and a list of `ssh_sources`, and the role has no inputs for them. | Nodes with a non-default SSH port or restricted sources | With another port: SSH is blocked at the first reload, although the role's SSH check passes. With restricted sources: SSH becomes open to everyone. | Until the role takes these as inputs, override `foomuuri_base_rules` on such nodes |
| 5 | Replacing one Foomuuri package with another may stop or restart the service, and stopping Foomuuri deletes its table. Not checked for any package. | Any node whose Foomuuri package is swapped | Every listening service, for the length of the package operation | Raw node producers install the standard package from the start, so onboarding never swaps packages |
| 6 | The D-Bus handler can run between the package install and the role masking it | First install, before Foomuuri 0.31 or an unpatched package | Local users can change zone assignments | Accepted. It lasts for part of one run. |
| 7 | The timed rollback is designed but not built | Every run that changes rules | A valid configuration that still cuts off SSH is not reverted | The SSH check on the merged configuration, and console access |

## WireGuard as the answer to gaps

Running everything over WireGuard is a sound way to handle what the role does not. It covers less than "any gap", and the difference is worth stating.

- **What it covers.** If every service except WireGuard itself listens only on the WireGuard interface, an unfiltered moment exposes nothing. The only public listener is a UDP port that ignores packets it cannot authenticate.
- **What it does not cover.** WireGuard protects traffic that goes through it. It does not filter the host. A service that listens on a public address is reachable during an unfiltered moment, whether or not management runs over WireGuard.
- **It is not there at handoff.** The contract hands a node over with SSH on a physical interface and no WireGuard. The first run of this role, which is where the swap happens, runs before WireGuard exists unless WireGuard is added to the contract or set up earlier in onboarding.
- **It needs the firewall too.** WireGuard's port and interface need their own rules. That is a later fragment, for example `30-wireguard`, and fits the role's additive passes.

So WireGuard closes gaps 1, 2 and 5 only on nodes where nothing else listens publicly. Creating before removing closes gap 1 everywhere.

## Changes to the role

The strategy needed two small changes. Both were made on 2026-11-16. Neither has been run on a host.

1. **Start Foomuuri before the conflicts step.** In `tasks/main.yml`, "Import conflicts.yml" now comes last. Before it, the role starts Foomuuri and flushes handlers, so the configuration written by the run is loaded before ufw's rules are removed. Closes gap 1.
2. **Confirm the table after the conflicts step.** `tasks/conflicts.yml` ends with a task that reads `inet foomuuri` and a task that reloads Foomuuri if the table is gone. Closes most of gap 2.

One side effect to check in testing: with `foomuuri_dbus: true`, `foomuuri-dbus.service` declares a conflict with `firewalld.service`, so starting Foomuuri first makes systemd stop firewalld at that moment, ahead of the conflicts step. With the default, `foomuuri_dbus: false`, the handler is masked and this does not arise.

Out of scope for the role, by the principle above: guard tables, blocking package scripts, installing WireGuard, and building Foomuuri packages.

## Open questions

- Should `ssh_port` and `ssh_sources` become inputs to the role, so that the base fragment follows the contract's shared parameters? This would close gap 4.
- Should WireGuard be part of the raw node contract, or an onboarding step before this role?
- What does stopping `nftables.service` do on Debian 13, Ubuntu 24.04 and Mint 22? It decides how much gap 2 matters.

## Changelog

| Version | Status | Notes |
| --- | --- | --- |
| 0.1.0 | Draft | First version: create, remove, document. Two paths, seven known gaps, the limits of WireGuard as mitigation, two changes to the role. Updated the same day once both changes were made. |
