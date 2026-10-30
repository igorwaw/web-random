---
title: "UPnP IGD, NAT-PMP and PCP - holes in your firewall"
date: 2026-10-27T18:00:00
draft: false
tags: ["networking", "security"]
---

Every router has a checkbox labelled "UPnP" or "NAT traversal", sometimes accompanied by "NAT-PMP" or "PCP". On most consumer routers it's enabled by default. On mine, it's disabled and it's going to stay that way. Here's why.

## Short version

All three protocols let a device on your local network ask the router to open a port to the internet. Automatically, without asking you, without any authentication.

It's convenient - for the few cases that actually need it. It's completely unnecessary in most homes and almost all offices. And it's a huge gaping hole in your security and privacy. 

## Long version

### Why was NAT traversal invented?

IPv4 addresses ran out a long time ago, so a typical home network gets one public address from the ISP and the router hides all the devices behind it, using private addresses (usually 192.168.x.x). That's NAT - Network Address Translation. Outgoing connections work fine: the router remembers who asked for what and forwards the replies back. Incoming connections don't: when a packet arrives from the internet, the router has no idea which of the 20 devices behind it should get it, so it drops it.

As a side effect, NAT works like a strict firewall. Nobody from the outside can reach your devices. Which include not only computers and phones (those are reasonably secure, unless you run an outdated OS), but also printers, cameras, washing machines and other IoT devices that - more often than not - get no security patches at all. It's the only reason why all those poorly designed and unmaintained devices are NOT cracked within minutes of getting online. Give any of them a public IP, and that's exactly what will happen.

But some applications want incoming connections:

- **Online games** - especially peer-to-peer multiplayer, voice chat and hosting a match. Xbox "Strict NAT" and PlayStation "NAT Type 3" warnings are the most visible symptom.
- **P2P file sharing** - BitTorrent works without incoming connections, but you get fewer sources: you can only connect to peers that can accept connections themselves.
- **VoIP and video calls** - older Skype was the classic example, most current apps don't need it.
- **Self-hosted services** - Plex remote access, game servers etc.
- **Mesh VPNs** - Tailscale, for example, uses all three protocols (when available) to establish a direct connection instead of going through a relay.

### Why is that dangerous?

#### Privacy

Public IPs are constantly scanned for open ports. With search engines like Shodan, it only takes seconds to find vulnerable printers or robotic vacuum cleaners. Before you ask "why is that legal" - all that Shodan does is index the data that's already publicly available. If you like, type "what is my IP" in your web browser, then go to shodan.io and paste your IP. Hopefully, you'll see no open ports.

The privacy part worries me most. An internet-connected camera in your living room is already a privacy problem. The same camera with an open port and unpatched firmware is a problem of much larger magnitude.

#### No authentication

All three NAT traversal protocols assume that if something is on your LAN, it's a trusted device and it can open ports. That was already an unreasonable assumption when the protocols were created. Even more now, when we connect devices of questionable quality.

It's not theoretical. Malware often uses UPnP IGD - [Conficker](https://en.wikipedia.org/wiki/Conficker) is a well-known early example, but IoT botnets of today do the same. Sure, there are other ways to connect, but let's not make life easier for malware authors.

#### Router implementations

Then there's the router side. UPnP is complex, and the firmware in home routers is usually low quality. Some of the better known incidents:

- **2013:** Rapid7 found 81 million devices answering UPnP discovery requests *from the internet*, not just from the LAN. Of those, 17 million accepted configuration requests - that means anyone on the internet could create port mappings or learn the internal network layout. 
- **2018:** Akamai described UPnProxy - routers with UPnP exposed on WAN being abused as proxies by attackers. Then EternalSilence, where the same trick was used to open SMB ports of machines *inside* the networks to the internet, making them easy targets for EternalBlue exploits.
- **2020:** CallStranger (CVE-2020-12695) - a flaw in the UPnP specification itself, allowing devices to be used for DDoS amplification and data exfiltration.

To be fair, NAT-PMP and PCP are much simpler and therefore easier to implement correctly. But they don't fix the other problems, plus many routers have no way to selectively enable some NAT traversal technologies and disable others.

#### Lack of visibility

If you set up the port forwarding yourself, you know about it. With UPnP, it happens without your knowledge. The information is somewhere in the router's status pages. Most people will never look.

### When is NAT traversal useful?

Most people don't need it and other techniques exist. But there are a few cases where nothing could replace it:

- **A gaming household with non-technical users.** If nobody in the house knows what port forwarding is, UPnP solves a real problem.
- **P2P and non-technical users.** A similar case. But is getting more sources really worth the risk? Isn't it already dangerous for a non-technical person to go to those corners of the internet? (for now, we'll skip the questions of ethics and legality)
- **CGNAT with an ISP that supports PCP.** Carrier-grade NAT means that your external IP already is from the private range (usually 100.64.0.0/10 specifically assigned for CG-NAT), your home LAN is another private network. You're behind 2 NATs. Manual port forwarding is simply impossible. Some (very few) providers support PCP, meaning that your devices would make a hole in 2 firewalls: your home one and the one that belongs to the ISP. The other 2 NAT traversal technologies are useless.

If you do enable it, a few things reduce the risk:

- Prefer NAT-PMP or PCP over UPnP, if the router supports it.
- Make sure the router only listens on the LAN interface. Test from outside if you can.
- Enable "secure mode" (only allow mappings for the requesting host) and restrict allowed ports and internal address ranges. OpenWrt, pfSense and OPNsense all use miniupnpd, which supports access control lists.
- Put IoT devices on a separate network or VLAN where UPnP is disabled.
- Check the list of active mappings now and then.

### Alternatives to NAT traversal

#### Relays

Most applications and devices these days don't need an open port to handle incoming connections - because technically, they're not incoming, they're replies to outgoing connections. Your video chat app connects to the server, so does your friend's video chat app. When you call each other, at least the initial connection goes through the relay (possibly, the server then does some magic to allow 2 clients to connect directly, but I'll spare you the details). So does any app that connects to the provider's "cloud".

#### Manual port forwarding

A simple concept and you have complete control over what you want to open and to whom. The only downside: it takes some knowledge and effort.

#### IPv6

The next generation IP protocol gives plenty of addresses to anyone who asks. That means all incoming connections work perfectly. But also, all advantages of "NAT as an accidental firewall" are gone.

## Unimportant technical details about NAT traversal protocols

### UPnP IGD

The oldest of all (2001, later updated). UPnP (Universal Plug and Play) is a family of protocols from the UPnP Forum, an industry group started by Microsoft at the end of the 1990s. It's meant for all kinds of device discovery and control: media servers, printers, smart TVs. The part responsible for port forwarding is IGD - Internet Gateway Device.

The way it works feels very much like early 2000s enterprise software. Devices are discovered with SSDP: multicast UDP to 239.255.255.250, port 1900. The router replies with a URL to an XML document describing its services. Then the client sends SOAP requests (XML over HTTP) to call methods like `AddPortMapping`, `DeletePortMapping` or `GetExternalIPAddress`. Lots of XML parsing, done by router firmware written as cheaply as possible. What could possibly go wrong?

IGDv2 from 2010 added IPv6 support and a few improvements, but the underlying architecture (and trust model) is the same. Adoption of v2 has been slow, and some clients refused to talk to v2 routers for years because of a version check bug.

### NAT-PMP

NAT Port Mapping Protocol was designed by Apple (Stuart Cheshire) around 2005 as an alternative to UPnP. It was used by Apple's AirPort routers and Back to My Mac, and was eventually published as RFC 6886 in 2013.

The biggest improvement over UPnP is the simplicity: a tiny binary protocol over UDP port 5351, sent directly to the default gateway. A request fits in 12 bytes. The client asks for a mapping, the router replies with the external port and the lease time, and the client needs to renew it before it expires. That's much less opportunity for the router firmware's authors to implement it wrong.

NAT-PMP only supports IPv4 and assumes the default gateway is the device doing NAT, so it doesn't help behind carrier-grade NAT.

### PCP

Port Control Protocol (RFC 6887, also 2013) is NAT-PMP's successor, designed by IETF with partly the same people. It uses the same port 5351, and the packet format is similar enough that a server can support both (NAT-PMP is version 0, PCP is version 2).

PCP extends the idea in a few ways:

- **IPv6 support**, including opening firewall pinholes when there's no NAT.
- **Carrier-grade NAT** - a client can talk to a PCP server that isn't its default gateway, for example the ISP's CGNAT box.
- **Third-party mappings** - with the THIRD_PARTY option, one device can request mappings on behalf of another. The RFC says it should be disabled by default on simple home routers, and I hope it actually is.
- **Authentication** - defined in a separate RFC 7652, using EAP. A wonderful concept, massively improving security, except for one tiny detail: I haven't seen it implemented in any home router, and I'm not aware of any client that would use it.

### Comparison

| | UPnP IGD | NAT-PMP | PCP |
|---|---|---|---|
| Designed by | UPnP Forum (Microsoft and others) | Apple | IETF |
| First published | 2001 (IGDv1), 2010 (IGDv2) | 2005 Apple only, RFC 6886 in 2013 | RFC 6887 in 2013 |
| Transport | SSDP (UDP 1900, multicast) + SOAP over HTTP | UDP 5351, unicast to gateway | UDP 5351, unicast to gateway or other PCP server |
| Message format | XML | Binary, fixed format | Binary, with options |
| Complexity | High | Very low | Moderate |
| IPv6 | IGDv2 only  | No | Yes |
| Mapping for other hosts | Yes, unless the router restricts it | No | Yes, if enabled |
| Works with CGNAT | No | No | Yes, if the ISP runs a PCP server |
| Mapping expiry | Optional (0 = permanent on many routers) | Mandatory | Mandatory |
| Authentication | None | None | Optional (RFC 7652), practically never used |
| Router support | Nearly universal | Common (Apple, miniupnpd-based routers) | Growing, still less common |
| Typical clients | All apps/devices that need it | Apple devices, BitTorrent, Tailscale, game consoles | similar to NAT-PMP but newer apps/devices only |




## More info

- https://datatracker.ietf.org/doc/html/rfc6886 (NAT-PMP)
- https://datatracker.ietf.org/doc/html/rfc6887 (PCP)
- https://openconnectivity.org/developer/specifications/upnp-resources/upnp/ (UPnP specifications)
- https://miniupnp.tuxfamily.org/ (miniupnpd, the implementation used by most open source routers)
- https://www.akamai.com/blog/security/upnproxy-eternalsilence (UPnProxy and EternalSilence)
- https://callstranger.com/ (CallStranger)
