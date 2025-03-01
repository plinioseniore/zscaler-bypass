# ZScaler

## Introduction

ZScaler is a cloud security product similar to Cisco Umbrella, this is a study to understand how it works.

> DISCLAIMER : This is a study of how ZScaler works and which are the possible bypass, there is no intention to promote the bypass of ZScaler products. If your organization has placed ZScaler, is to protect your devices from cyberthreats. If you are using this information for any goal than is not a study or analysis, you are doing that at your own risk.

The product is composed of two main features called ZIA (ZScaler Internet Access) and ZPA (ZScaler Private Access). The ZIA is a tunneller for traffic to internet rather the ZPA is a tunneller for traffic to local resources not exposed to internet.

## ZScaler Private Access

This module replace the needs of a VPN to access local resources, it use the same approach of a VPN with a tunnelled traffic to the target resource but it doesn't involve a virtual adapter like a standard VPN. Once a request to a local resource is identified, that request only is redirected in the ZPA tunnel.

It could seems an always on VPN with specific enforced routes, but at least at user level you don't see your routing table altered. The whole ZCC (ZScaler Client Connector, that runs ZIA and ZPA on your end device) acts on a lower level than your TCP/IP stack on Windows.

When a VPN is setup the ZPA disable itself automatically, as this overlap the VPN features. This assume that the VPN you are using give you access to same resources accessible via ZPA, even if this is not true in case of VPN to connect to a customer site rather than your employer one.

When ZPA is enabled, the DNS traffic is intercepted. This can be understood because when ZPA is enabled the DNS request timeouts and then get a reply, likely at inner level the reply is modified so that ZPA can takeover and redirect the traffic. An '''nslookup''' to a local resource get resolved as a local address, like 100.1.1.1 that is picked by ZPA Tunnel.

## ZScaler Internet Access

This module is similar to Cisco SWB and has two operating modes called v1 and v2. The Tunnel v1 is an HTTP(S) proxy rather the v2 is a full tunnel that redirect all the traffic to the ZScaler edges. In both the modes the traffic is inspected, so a custom ZScaler certificate is used.

The inspection is only on HTTP(S) traffic, other kind of traffic that use encryption are instead passed trough or rejected based on firewall policy at edge level. This can be understood because a whitelisted SSH tunnel still show the server certificate at connection and not a ZScaler ones.

## Protection Limits and Bypass

The extend of the protection and prevention of bypasses of ZScaler is somehow limited by how far restriction could apply, where the hardned limit is the restriction of local IP address. Unless local IPs are restricted, there are some bypass. Is expected that local IPs are allowed to access local resources like printers or RDP, that otherwise should be redirected via ZPA.

If configured to allow local traffic, a local proxy on the network can be used to bypass the traffic inspection and filter. This is similar to Cisco Umbrella with the difference that in v2 the proxy cannot be emulated locally, because it use DTLS and the ZCC will not trust a device that emulate their edge proxy.

So the only option is to run an HTTP proxy and point application trough it, but even here ZScaler try hard to avoid this, forcing a 'zscaler.cfg' file in all Firefox instances. This setting prevent to setup proxy in Firefox as well is assumed that at system level is not allowed to setup a proxy configuration at Operative System level, the proxy cannot leveraged.
This is true for some release of Firefox but not for all, as example Firefox installed from Microsoft App Center is not affected by ZCC and still is allowed to have a proxy configured.
Anyhow any application that allow the configuration of a proxy will be allowed outside the tunnel.

Another limit is on the behaviour if the ZScaler Edges cannot be reached, is likely that in that case ZCC is configured to be disabled. In that case a network or a device with a rule that stops traffic to ZScaler will trick ZSCC in disable state. The list of IP addresses used by ZScaler is publicly available and the specific one used by the application is even nicely displayed in the ZCC user interface.

### Bypass with Firewall Rule

With the limits stated above, a traffic rule that drops traffic to ZScaler is the easier bypass.

### Traffic established before ZCC connects  

The ZScaler protection starts only once ZCC connects, it rely on a local DNS for the resolution of the IP addresses it use for its edges. Existing connection are not dropped.

So an easy way to bypass ZScaler is to disable the internet connected interface, re-enable and connect to a target service that is not allowed via ZIA. If this happen while ZCC is figuring out to connect to ZScaler ZIA Edges, the connection gets established and survive even after ZScaler is connected.
This works fine with connection that once established are never dropped, like SSH Tunnels and VPNs.

Using an SSH Client like Putty or others, the connection to an SSH Server and a tunnel listening on a local port can allow a path not monitored by ZScaler. The SSH Clients generally runs without installation and without privileges, making this bypass effective on restricted environments where there is no application whitelisting.

One nice thing to observe is that ZCC observe the routing table, so if you have a VPN that establish a connection before ZCC, the ZCC connection to the ZScaler Edge will go though the VPN. If the firewall in the VPN blocks the traffic to the ZScaler Edge, this will completely disable the ZScaler protection.

### Local Proxy

A local proxy used to redirect the traffic can be on the local network or on the device itself, if is allowed to run a Virtual Machine with a bridged interface. The traffic via the bridged interface cannot be intercepted by ZScaler.

### A local interface with a public IP address

This is untested but likely to work, a virtual interface with a public IP address is not redirected via ZPA even if the local traffic is not allowed. This because the block of local traffic is done via an IP rule set in ZPA for RFC 1918 addresses, so a public IP address will not trigger ZPA.

If that traffic is not redirected via ZIA because is seen by the Operating System as local traffic, that interface can be used to run a local proxy even with restriction of local addresses.

## Trusted Network

The definition of Trusted Network can use multiple logics but the most common one is to check the DNS Name Server. In this case having changing the DNS Name in your router will let ZScaler Client define the network as Trusted.

The logics that can be used to define a Trusted Network are provided in the ZScaler manual
https://help.zscaler.com/zscaler-client-connector/configuring-forwarding-profiles-zscaler-client-connector

Based on the network type (Trusted, VPN Trusted, VPN Split Channel and Off-Trusted) different forwarding profiles can be enforced and this can change the ZCC behavior.

The type of VPN (Trusted or Split Channel) depends on the name of the network adpter, if the following words are included the VPN is consider Trusted: Cisco, Juniper, Fortinet, PanGP, or VPN as long a route that force all traffic through that interface is set.

## Conclusion

The overall behavior of ZScaler and so the above statements can be identified from the Architects Guide Universal ZTNA from ZScaler
https://www.zscaler.com/resources/white-papers/architects-guide-universal-ztna.pdf
