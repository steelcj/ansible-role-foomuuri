You may well be right, and my “short answer: no” was too blunt. I re-read the source at the same commit (1f32061). I don’t know yet which point you disagree with, so here is where I think the earlier analysis undersold the role. This is from reading the code, not from a test run.

What it supports as-is that I underweighted
The iptables path is not really Alpine-only. firewall_service, firewall_iptables_rulefile and both package lists live in vars/main.yml, but role parameters and -e both outrank role vars. On a Red Hat-family host you can pass firewall_service: iptables, firewall_iptables_rulefile: /etc/sysconfig/iptables and the iptables-services package, with no edits to the role.
Port ranges and named services work on two of three backends. name: "8000:8100" passes the assertions as a string and is valid for both ufw’s to_port and the template’s --dport. It fails only on firewalld, where a non-numeric name is treated as a service.
Per-item rule works on ufw. item.rule is passed straight to the module, so deny, reject and limit are usable per service. Only the default is restricted to allow or deny.
What minor edits would add

Each of these is a few lines:

Swappable template: change src: iptables.j2 to a firewall_iptables_template default. A differently named template in your playbook’s templates/ is then picked up, so the full ruleset becomes yours to write.
Raw extra rules: a firewall_extra_rules list looped in the template before the final REJECT.
Richer items: optional source, interface, direction and comment keys, passed to ufw as from_ip, interface, direction and comment with default(omit), and added to the template line.
More rule file and service mappings: Debian needs /etc/iptables/rules.v4 and the netfilter-persistent service, since there is no iptables service there.
Where I’d still hold my position
firewalld is the hard backend. Source or interface restrictions there need rich rules or zones, which is a new task rather than an extra parameter.
Without edits, every rule is still “this port, from anywhere”. The complexity comes from the edits, small as they are.
CI does not exercise any of it. Every firewall task is skipped under container connections, so your VM or bare-metal testing is the first real test.

So my revised verdict is: limited as shipped, but a small patch set, perhaps 30 to 50 lines, gets you templated complex rulesets on iptables and ufw.

Which part did you disagree with? If it’s something other than the above, tell me what you’re seeing and I’ll check it against the source, then update firewall-role-analysis.md to match.