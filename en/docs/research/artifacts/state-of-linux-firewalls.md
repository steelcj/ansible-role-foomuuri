# state-of-linux-firewalls

- **The Compatibility Shim (`iptables-nft`):** On modern Ubuntu releases (including Ubuntu 20.10, 24.04 LTS, and 26.04 LTS), the `iptables` command is actually `iptables-nft` by default. When UFW generates and executes traditional `iptables` rules, this compatibility layer translates and pushes them into the kernel's `nftables` (`nf_tables`) subsystem behind the scenes. [[1](https://serverside.com/blog/iptables-vs-nftables), [2](https://github.com/uburuntu/meridian/issues/60)]
- **No Native UFW Backend:** UFW itself has not been rewritten to speak native `nftables` syntax directly; it still operates via its `iptables` backend implementation. [[1](https://bugs.launchpad.net/bugs/2114851)]
- **End Result:** While your active firewall rules end up residing inside the kernel's `nftables` framework rather than the legacy `ip_tables` kernel modules, UFW is not using a native `nftables` code path or syntax structure. [[1](https://serverside.com/blog/iptables-vs-nftables)]

Yes, **firewalld** is the most widely available, mature alternative to UFW that uses a native `nftables` backend out of the box. Several other specialized tools also exist depending on whether you prefer command-line interfaces, declarative files, or web GUIs. [[1](https://medium.com/@mangaldeepdey/everything-about-linux-firewalls-netfilter-iptables-nftables-ufw-firewalld-faaf6a2647d8)]

Top Native nftables Frontends

- **firewalld****Best Overall Alternative:** Available in Ubuntu's standard repositories (`sudo apt install firewalld`) and serves as the default firewall on RHEL, Fedora, and openSUSE.**Native Backend:** Generates native `nftables` tables and rules directly without relying on `iptables` translation layers.**Key Features:** Uses intuitive network zones (such as `public`, `home`, and `trusted`) and named services (`firewall-cmd --add-service=http`), supports dynamic runtime updates without dropping active connections, and offers a desktop GUI (`firewall-config`). [[1](https://www.akamai.com/docs/guides/comparing-network-firewall-solutions/), [2](https://www.facebook.com/fb-answers/linux-firewall-alternatives-to-ufw/), [3](https://medium.com/@mangaldeepdey/everything-about-linux-firewalls-netfilter-iptables-nftables-ufw-firewalld-faaf6a2647d8)]
- **Foomuuri****Declarative Zone Manager:** A modern, multizone bidirectional `nftables` firewall manager written in Python.**Key Features:** Configured via clean, human-readable text files that compile straight into native `nftables` rulesets, supporting dynamic IP lists, DNS-based rules, and D-Bus integration. [[1](https://www.linuxlinks.com/best-free-open-source-firewall-tools/)]
- **nftfw****File-Based Simplicity:** Designed specifically for Debian and Ubuntu systems to make `nftables` management as straightforward as creating empty files in designated folders.**Key Features:** Rules are controlled via a simple directory layout (`incoming.d`, `outgoing.d`, `blacklist.d`), where adding a file named `80` or an IP address automatically compiles and loads the corresponding `nftables` rules. [[1](https://nftfw.uk/)]
- **Webmin****Web-Based GUI:** Includes a dedicated native `nftables` module (listed under Networking > Linux Firewall).**Key Features:** Lets you manage tables, chains, sets, and quick allow/block rules through a browser without touching the command line. [[1](https://infosecmonkey.com/nftables-firewall-the-modern-linux-packet-filter-and-every-gui-worth-using-to-manage-it/)]

Quick Comparison with UFW

| Feature                      | UFW                                  | firewalld                                    | Foomuuri                  |
| ---------------------------- | ------------------------------------ | -------------------------------------------- | ------------------------- |
| **Backend**                  | `iptables-nft` shim                  | Native `nftables`                            | Native `nftables`         |
| **Configuration Style**      | Simple CLI commands (`ufw allow 22`) | CLI (`firewall-cmd`) + Zones                 | Declarative config files  |
| **Ubuntu Repo Availability** | Pre-installed                        | Standard `apt` repository                    | Via PPA / GitHub package  |
| **Dynamic Reloads**          | Reloads full ruleset                 | Runtime changes without breaking connections | Atomic `nftables` updates |

*Note: If you switch to **firewalld** or another manager on Ubuntu, disable UFW first (`sudo ufw disable`) so the two tools do not conflict over kernel netfilter tables.*

Are you configuring a **headless server** or a **desktop workstation**, and do you prefer **CLI commands** or **declarative config files**? I can walk you through setting up the best fit.