Vulnerability Assessment Lab

I am a junior studying cybersecurity at UNC Wilmington. I passed CompTIA Security+ in July 2026, and a lot of what I learned there was conceptual. Vulnerability management, risk assessment, CVSS scoring, hardening legacy systems. I wanted to actually do it instead of just reading about it, (see these acynnyms in real time) so I built a lab and ran a real assessment against a deliberately vulnerable host.

The scan itself was the easy part. Figuring out what the results meant took a lot longer, and that turned out to be the whole point.

The setup
Component	Detail
Scanner	Tenable Nessus Essentials, Basic Network Scan policy
Target	Metasploitable 2, running Ubuntu 8.04
Hypervisor	Oracle VirtualBox
Network	Host-only adapter, isolated from my LAN and the internet
Scan type	Uncredentialed
Duration	19 minutes

Metasploitable ships with services that are genuinely exploitable, not simulated. Putting it on my real network would have exposed every other host to it and given an attacker a foothold. So I isolated it on a host-only adapter with no route out. Network segmentation is one of those Security+ concepts that feels abstract until you have a reason to actually use it.

 Building the lab

Getting the environment working took longer than the scan did.

The VM would not pull an address. Running dhclient showed DHCPDISCOVER
going out with no offers coming back, because a host-only adapter in
VirtualBox has no DHCP server behind it by default. I assigned a static
address manually instead, which is better for a lab anyway since the
target IP will not move between scans.

The first static address I picked also failed. VirtualBox had created a
second host-only adapter on a different subnet than the one I assumed,
and my VM was attached to that one. Once I checked which interface the
VM was actually bound to and matched the subnet, the host could reach
the target and the scan had something to talk to.

Small problems, but they were a reminder that a scanner is only as good
as the connectivity underneath it. An unreachable target returns a clean
report, and a clean report from a broken scan is worse than no report.

What came back
Severity	Count
Critical	10
High	6
Medium	24
Low	9
Informational	137

69 unique vulnerabilities across 186 total findings. Informational results were about 74 percent of the output by volume. My first real takeaway was that most of a scan is noise, and getting past it is a skill in itself.

How I prioritized remediation

Nessus sorts by CVSS. I did not, and here is my reasoning.

1. Bind Shell Backdoor Detection, CVSS 9.8

I put this above two findings that scored a full 10.0. Everything else on the list describes a weakness. This one describes a backdoor that is already listening and already serving shells. No exploit needed, no credentials, no preconditions. On a production host I would not treat this as a patching ticket. I would treat it as an incident and assume the system was already compromised. CVSS scores the flaw itself, not the fact that someone may already be inside.

2. VNC Server Using Default Password, CVSS 10.0

The VNC service accepts the password "password" and hands over an interactive remote session. This is a textbook authentication failure and it takes under a minute to fix, which is why I would do it before the other 10.0 finding.

3. Canonical Ubuntu 8.04 End of Support, CVSS 10.0

This has the highest CVSS score in the report but it describes a condition rather than something directly exploitable. Nobody attacks "end of life." It matters because it is the root cause behind most of the other 68 findings. Upgrading resolves them in bulk. High impact, long timeline. This is a migration project, not a patch.

Where CVSS and real risk disagreed

This was the most interesting thing I found. Comparing severity against exploitation likelihood:

Finding	CVSS	Severity	EPSS
Logjam, Diffie Hellman modulus 1024 bits or less	3.7	Low	0.9986
SSL DROWN Attack	5.9	Medium	0.8211
Samba Badlock	7.5	High	0.3693

The Low severity finding has a near certain chance of being exploited. The High severity finding sits around a third. If I had remediated strictly in CVSS order I would have fixed Badlock first and pushed Logjam to the bottom, and I would have had real world risk exactly backwards.

Tenable's own VPR scoring agrees with that read. Samba Badlock and rlogin both drop from CVSS 7.5 down to VPR 4.9, which says the practical urgency is lower than the raw severity implies.

What I took from this is that severity, exploitability, and exposure are three different questions. A prioritization model that only answers the first one will send remediation effort to the wrong place.

Other findings worth noting
SSLv2 and SSLv3 still enabled, CVSS 9.8. Deprecated protocols with known cryptographic weaknesses still negotiable.
NFS shares world readable, CVSS 7.5. Exported filesystems reachable with no authentication at all.
Unencrypted Telnet server, CVSS 6.5. Credentials crossing the network in cleartext.
rlogin service detected, CVSS 7.5. Legacy trust based remote access with essentially no authentication.
Apache Tomcat, ISC BIND, SSH, and SMB each returned multiple issues, which is what you would expect from an unpatched 2008 era distribution.
What this scan did not tell me

Host authentication came back as Fail, so this was an uncredentialed scan. That means I only saw the network facing attack surface. A credentialed scan would also surface missing patches, local privilege escalation paths, and host configuration weaknesses that are invisible from the outside. The real finding count is almost certainly higher than 69.

Nessus Essentials also leaves out some of the reporting and export features in the paid tiers, so I captured findings manually with screenshots.

What I would do next

Run the same scan with credentials and compare the two result sets. The gap between what an attacker sees from outside and what actually exists on the host is the part I want to understand better.
