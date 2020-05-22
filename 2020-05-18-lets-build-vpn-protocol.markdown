---
layout: post
title: Let's Build a VPN Protocol
description: How to implement a custom VPN client and server using Packet Tunner Provider
date: 2020-05-18 9:00:00 -0400
category: programming
tags: programming
permalink: /post/lets-build-vpn-protocol
uuid: 4f4b6cb2-192f-4a1c-bafb-f1e84baee96d
disable_suggested_articles: true
disable_comments: true
---

It's been two years since [Network Extension, Part 1 - Introduction]({{ site.url }}/post/network-extensions-into). You've been asking for Part 2. Here it finally is.

> - [Let's Build a VPN Protocol]({{ site.url }}/post/lets-build-vpn-protocol) (You are here)
> - [VPN, Part 1: VPN Profiles]({{ site.url }}/post/vpn-configuration-manager)
> - VPN, Part 2: Packet Tunnel Provider (**Coming Soon**)
> - VPN, Part 3: Server (**Coming Soon**)
> - VPN, Part 4: Split Tunnel and Custom DNS Protocol (Bonus) (**Coming Soon**)

{% include ad-hor.html %}

## VPN

What is VPN? Originally, a VPN (Virtual Private Network) was designed to establish secure connections between a computer and a protected or private corporate network, using an otherwise insecure public network (such as the Internet). In recent years, VPNs also grew in popularity in the consumer market as a way to stay secure and anonymous when going online, and also enjoying unrestricted network access by hiding the source of traffic.

If that sounds a bit too abstract, then you are in the right place. In this series, I'm going to cover most of the bits and pieces that make VPN work. And what is a better way to do that than implementing a custom VPN protocol?

> **Requirements**.
>
> To run the sample code in this series, you are going to need a physical device and an Apple Developer Account to get proper entitlements.
{:.warning}

## Network Extension Framework

You are going to learn how the iOS manages VPN configuration, how to create custom VPN configurations programatically, how to implement both the client and the server sides of the VPN.

The primary topic of this series is going to be [Network Extension](https://developer.apple.com/documentation/networkextension) framework. Among many of its features, it has extensive support for virtual private networks (VPN). The client side of the custom protocol implementation is implemented as a [Packet Tunnel Provider](https://developer.apple.com/documentation/networkextension/packet_tunnel_provider) extension. The Packet Tunnel Provider extension’s containing app uses `NETunnelProviderManager` to create and manage VPN configurations that use the custom protocol, and to control the VPN connections specified by the configurations.

The solution is going to consist of the three major parts:

1. A [Packet Tunnel Provider](https://developer.apple.com/documentation/networkextension/packet_tunnel_provider) extension for iOS
2. A containing app built in SwiftUI which is also going to be used for creating and managing a VPN configuration
3. A custom VPN server also built in Swift

> The entire app source code is available at [kean/VPN](https://github.com/kean/VPN).
{:.info}

<div class="Any-vertInsets">
<a href="{{ site.url }}/post/vpn-configuration-manager">
  <div class="PrimaryButton">
    Continue Reading »
  </div>
</a>
</div>
