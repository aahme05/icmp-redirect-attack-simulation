# ICMP Redirect Attack Simulation

**Environment:** Docker containers on a SEED Ubuntu 20.04 VM (SEED Labs framework by Prof. Wenliang Du, Syracuse University)

This lab is a hands-on implementation of the **ICMP Redirect Attack Lab** from the SEED Labs series ("Internet Security: A Hands-on Approach"), https://seedsecuritylabs.org/. Lab instructions, container topology, and grading rubric are copyright Wenliang Du, licensed CC BY-NC-SA 4.0, this repo documents my own implementation, commands, and observations, not the original lab handout.

## Topology

Four containers simulate a small routed network:

| Host | Role | IP |
|---|---|---|
| Victim | Target of the attack | 10.9.0.5 |
| Attacker | Sends spoofed ICMP redirects | 10.9.0.105 |
| Malicious Router | Attacker-controlled router | 10.9.0.111 |
| Router | Legitimate router to 192.168.60.0/24 | 10.9.0.11 |

Normally, the victim reaches 192.168.60.0/24 via the legitimate Router. The goal is to trick the victim into routing that traffic through the Malicious Router instead, a textbook Man-in-the-Middle setup at the network layer.

## Task 1 — ICMP Redirect Attack

Using Scapy, I crafted a spoofed ICMP redirect message (type 5) claiming to originate from the legitimate router, telling the victim "use 10.9.0.111 for this destination instead." Because Ubuntu 20.04 in this lab config accepts redirects (net.ipv4.conf.all.accept_redirects=1), the victim's routing cache (not the routing table itself) picks up the bogus entry:

    # ip route show cache
    192.168.60.5 via 10.9.0.111 dev eth0
        cache <redirected> expires 296sec

A traceroute/mtr from the victim afterward confirms traffic to 192.168.60.5 now transits the malicious router.

**What I tested beyond the base task:**
- Whether a redirect can point to a remote (off-LAN) gateway, it can't take effect the same way, since the kernel's sanity check ties the redirect to traffic actually sent toward that destination.
- Whether redirecting to a non-existent local host silently breaks connectivity (it does, a real DoS side-effect of this attack).
- The effect of the malicious router's own send_redirects sysctl flags on whether the attack still works when toggled.

## Task 2 — MITM via Sniff-and-Spoof

With the victim now routing through the malicious router, IP forwarding on that router is disabled, instead of passively forwarding packets, a sniff-and-spoof Python script captures the victim's outbound TCP traffic, rewrites the payload in flight (replacing a target string with a same-length substitute to avoid breaking TCP sequencing), and re-injects it toward the real destination. This turns the redirect from a routing curiosity into an active on-path tampering attack against a live netcat session between victim and destination.

## Key Takeaways

- ICMP redirects operate on the route cache, not the persistent routing table, ip route flush cache clears the effect, but a live attacker can keep re-injecting.
- The kernel's redirect-acceptance sanity checks vary by version (Ubuntu 20.04 vs. 22.04+), directly affecting how easy this attack is to pull off in the wild, which is exactly why accept_redirects should be disabled on production hosts.
- Turning a redirect attack into a full MITM requires disabling forwarding on the rogue router and taking over packet relay manually via raw sniffing/spoofing.

## Tools

Docker & Docker Compose, Scapy (Python), ip route / mtr, sysctl, netcat

## Reference

Du, W. SEED Labs — ICMP Redirect Attack Lab. https://seedsecuritylabs.org/ — used under CC BY-NC-SA 4.0. Lab handout text and figures are not reproduced here.
