---
layout: post
title: "VPN, Part 2: Packet Tunnel Provider"
subtitle: TBD
description: TBD
date: 2020-06-01 9:00:00 -0400
category: programming
tags: programming
permalink: /post/packet-tunnel-provider
uuid: d3d4d400-70f3-40a1-a8d3-bcfe9df881f8
---

 With most of the groundwork covered in ["How Does VPN Work?"](/post/networking-101), let's go ahead and design a VPN protocol.

 {% include ad-hor.html %}

## Designing a Protocol

I want the protocol to be funcional, but also keep it as simple as possible. The goal is not to design a viable VPN protocol, but to demonstrate [Network Extension](shttps://developer.apple.com/documentation/networkextension) framework.

There are three things that our protocol is going to need to be able to do:

- Establish a secure connection
- Authenticate the user based on login/password
- Provide a way to send encrypted IP packets – the actual network traffic – over the network

### Encryption

The industry standard for secure connections is [TLS](https://tools.ietf.org/html/rfc8446). A typical VPN protocol would use *asymmetric* cryptography to allow the client and the server to establish a secure connection and share a *symmetric* encryption key (*hybrid* encryption). However, to keep things simple, I'm going to use a pre-shared key and simple symmetric encryption.

> **Warning!**
>
> Don't rely on this article for the security advice It uses a simplified solution just for demonstration purposes. If you are implementing a VPN protocol, you might want to use a TLS implementation provided by Apple in [Network framework](https://developer.apple.com/documentation/network). Use this framework when you need direct access to protocols like TLS, TCP, and UDP for your custom application protocols. There is an [example project](https://developer.apple.com/documentation/network/building_a_custom_peer-to-peer_protocol) provided by Apple which demonstrates how to use their TLS implementation.
{:.warning}

### Control Packets

<!-- TODO: client auth, server auth response, use code to distinguish; to simplify parsing I'm going to stick JSON in payload -->

### Data Packets

<!-- TBD: Connection to server (we need to define a protocol for authorization) -->
<!-- TBD: here is how to pass password using App Groups and shared Keychain -->

### Transport

[`NEPacketTunnelProvider`](https://developer.apple.com/documentation/networkextension/nepackettunnelprovider) provides you two ways to send packets inside your private network:

- UDP: [`createUDPSession`](https://developer.apple.com/documentation/networkextension/neprovider/1406004-createudpsession)
- TCP: [`createTCPConnection`](https://developer.apple.com/documentation/networkextension/neprovider/1406529-createtcpconnection)

We could use either of these options. UDP is typically going to be faster, it is easier to use, and it is a [preferred way](https://openvpn.net/faq/why-does-openvpn-use-udp-and-tcp/) in some of the existing VPN protocols, such as OpenVPN. You don't need the reliability guarantees of TCP in case of the VPN protocols – the goal is to send opaque IP packets as quickly as possible over the wire, the reliability is the concern of the protocols encapsulated in the IP packets.

## Implementing a Protocol

<!-- TODO: One of the major advantages of Swift is that, not unlike C, it allwows us to reuse code between a client and a server, so let's take advantage of this from the ground up. In order to implement a protocol, I'm going to create a library, `vpn-protocol`. -->

## Packet Tunnel Provider

<!-- TODO: so how do we not implement the iOS client part? -->

The primary API for implementing your custom tunneling protocols in [`NEPacketTunnelProvider`](https://developer.apple.com/documentation/networkextension/nepackettunnelprovider), that allows you to tunnel traffic on an IP layer. They run as app extensions, they run in the background handling network traffic.

> There is also an [NEAppProxyProvider](https://developer.apple.com/documentation/networkextension/neappproxyprovider) which is another subclass of the abstract `NETunnelProvider` class. It allows you to implement tunneling on a higher, app level. Unlike `NEPacketTunnelProvider`, it requies a device to be [*supervised*](https://support.apple.com/en-us/HT202837) and it only works for [*managed* apps](https://www.apple.com/business/docs/resources/Managing_Devices_and_Corporate_Data_on_iOS.pdf). It operates on a TCP/UDP level instead of IP level.
{:.info}

To configure and control a network provider from your app, you use [`NETunnelProviderManager`](https://developer.apple.com/documentation/networkextension/netunnelprovidermanager) family of APIs, which I covered in the [previous post](/post/vpn-configuration-manager).

So, how does it work? Suppose you have a `NEPacketTunnelProvider` running on the system, connected to a VPN server and is providing a tunnel to some internal network. An app wants to get some resource on the network. The app creates a socket, and TCP/IP connection.

<img alt="WWDC: What's New in NetworkExtension and VPN Screenshot" class="Screenshot Any-responsiveCard" src="/images/posts/vpn-client/tunnel-01.png">

The packets for this TCP/IP connection are routed to a virtual `utun0` network interface. Instead of sending packets over the network, it diverts the packets to `NEPacketTunnelProvider`.

<img alt="WWDC: What's New in NetworkExtension and VPN Screenshot" class="Screenshot Any-responsiveCard" src="/images/posts/vpn-client/tunnel-02.png">

The tunnel provider can then use [`NEPacketTunnelFlow`](https://developer.apple.com/documentation/networkextension/nepackettunnelflow) to read the packets from the virtual interface, encapsulate them in your tunneling protocol, send them over to the tunneling server. The server will decapsulate them, inject the packets into the network to send them to their ultimate destination, and encapsulate the responses in the tunneling protocol and send them back to the client, which decapsulates them, and injects them back into the client networking stack via `utun0` virtual interface. They will be delivered back to the TCP/IP stack, and back to the application.

The `NEPacketTunnelProvider` has a lot of control over the `utun0` interface. Most importantly, it can specify the routes – the destination that will be routed to the `utun0` interface and through the tunnel.

> You can learn more about `NEPacketTunnelProvider` in the [documentation](https://developer.apple.com/documentation/networkextension/nepackettunnelprovider), and in WWDC [**What's New in NetworkExtension and VPN**](https://developer.apple.com/videos/play/wwdc2015/717/) session video.
{:.info} 

## VPN Tunnel

In the packet tunnel provider extension that I created in [the previous post](/post/vpn-configuration-manager) there is one file which has a `NEPacketTunnelProvider` subclass. This is the entry point of the extension. When the extension is started, an instance of this class is initialized and the [`startTunnel`](https://developer.apple.com/documentation/networkextension/nepackettunnelprovider/1406118-starttunnel) method is called.  The `completion` handler is what you need to call when you are done setting up a tunnel.

```swift
class PacketTunnelProvider: NEPacketTunnelProvider {
    override func startTunnel(options: [String: NSObject]?,
                              completionHandler: @escaping (Error?) -> Void) {

    }
}
```

### Establish Connection

The first thing I need to do is establish a connection with a server. As mentioned previously, my protocol is based on UDP. I'm going to use [`createUDPSession(to:from:)`](https://developer.apple.com/documentation/networkextension/neprovider/1406004-createudpsession) method which is going to return [`NWUDPSession`](https://developer.apple.com/documentation/networkextension/nwudpsession) for me.

```swift
class PacketTunnelProvider: NEPacketTunnelProvider {
    private var session: NWUDPSession!
    private var observer: AnyObject?
    private var pendingCompletion: ((Error?) -> Void)?

    override func startTunnel(options: [String: NSObject]?, completionHandler: @escaping (Error?) -> Void) {
        self.pendingCompletion = completionHandler
        self.startUDPSession()
    }

    private func startUDPSession() {
        let endpoint = NWHostEndpoint(hostname: "your-vpn-domain.com", port: "your-port")
        self.session = self.createUDPSession(to: endpoint, from: nil)
        self.observer = session.observe(\.state, options: [.new]) { session, _ in
            if session.state == .ready {
                // The session is ready to exchange UDP datagrams with the server
            }
        }
    }
}
```

> The code that you see in this section is simplified to highlight the key points. You can find the complete implementation at [kean/vpn](https://github.com/kean/VPN).
{:.info}

In the demo, I'm running the server on my local machine and I'm using a domain name advertised using [Bonjour](https://developer.apple.com/documentation/foundation/bonjour). [`NWUDPSession`](https://developer.apple.com/documentation/networkextension/nwudpsession) performs DNS resolution for me.

One the connection is established, I can start exchanging UDP datagrams with a server. The first thing I need to do is authentication, using my custom control packet. I'm using my `VPNProtocol` library to create and serialize a datagram.

```swift
private func sessionDidBecomeReady() {
    guard let tunnelProtocol = protocolConfiguration as? NETunnelProviderProtocol,
        let username = protocolConfiguration.username,
        let passwordReference = protocolConfiguration.passwordReference else {
        // Handle errors
    }

    session.setReadHandler({ [weak self] datagrams, error in
        self?.sessionDidReceiveDatagrams(datagrams)
    }, maxDatagrams: 30)
}

private func authenticate(login: String, password: String) {
    let packet = Packets.ClientAuthRequest(login: login, password: password)
    session.writeDatagram(try packet.datagram()) { error in
        // Handle errors
    }
}

private func sessionDidReceiveDatagrams(_ data: [Data]) {
    // Wait for the authentication response from the server
}
```

Once the authentication request is sent, I'm going to then set up a handle and wait for a UDP datagram from the server acknowledging the authentication.

```swift
private func didReceiveDatagram(datagram: Data) throws {
    let code = try PacketCode(datagram: datagram)
    switch code {
    case .serverAuthResponse:
        let response = try Packets.ServerAuthResponse(datagram: datagram)
        if response.isOK {
            self.didSetupTunnel(address: response.address)
        }
    default:
        break
    }
}

/// - parameter address: The address of the remote endpoint that is providing the tunnel service.
private func didSetupTunnel(address: String) {
    let settings = NEPacketTunnelNetworkSettings(tunnelRemoteAddress: address)
    // Configure DNS/split-tunnel/etc settings if needed

    setTunnelNetworkSettings(settings) { error in
        self.pendingCompletion?(nil)
        self.pendingCompletion = nil
    }
}
```

> [`NEPacketTunnelNetworkSettings`](https://developer.apple.com/documentation/networkextension/nepackettunnelnetworksettings) provides a configuration for a packet tunnel provider’s virtual interface. There are some interesting things that you can do such set custom DNS addresses, configure *split* tunnel, and more. I'm going to cover this in the upcoming *bonus* article.
{:.info}

### Route Traffic

Now, how do you actually route the traffic from the system through the tunnel? A packet tunnel provider has a [`packetFlow`](https://developer.apple.com/documentation/networkextension/nepackettunnelprovider/1406185-packetflow) property returning a [`NEPacketTunnelFlow`](https://developer.apple.com/documentation/networkextension/nepackettunnelflow) object which is used to receive IP packets routed to the tunnel’s virtual interface and inject IP packets into the networking stack via the tunnel’s virtual interface.

```swift
private func didStartTunnel() {
    readPackets()
}

private func readPackets() {
    packetFlow.readPacketObjects { packets in
        let datagrams = packets.map(crypto.encrypt)
        self.udpSession.writeMultipleDatagrams(datagrams) { error in
            // Handle errors
        }
        // Continue reading packets from the virtual interface
        self.readPackets()
    }
}
```

That's what you do to send the packets out. Coming back it's very similar. When I receive new UDP datagram over the connection, decrypt them and write them into the packet flow. Let's extend our UDP session `readPackets` handler to do just that.

```swift
private func didReceiveDatagram(datagram: Data) throws {
    let code = try PacketCode(datagram: datagram)
    switch code {
    case .serverAuthResponse:
        // ...
    case .data:
        let packet = crypto.decrypt(datagram)
        let proto = IPProtocol(packet: packet)
        self.packetFlow.writePackets([packet], withProtocols: [proto.number])
    default:
        break
    }
}
```

<div class="References" markdown="1">

## References

1. Apple Developer Documentation, [**NetworkExtension Framework**](https://developer.apple.com/documentation/networkextension)
2. WWDC 2017, [**Session 707: Advances in Networking, Part 1**](https://developer.apple.com/videos/play/wwdc2017/707)
3. WWDC 2017, [**Session 709: Advances in Networking, Part 2**](https://developer.apple.com/videos/play/wwdc2017/709/)
4. WWDC 2015, [**Session 717: What's New in NetworkExtension and VPN**](https://developer.apple.com/videos/wwdc/2015/?id=717)
5. OpenVPN, [**Why does OpenVPN use UDP and TCP?**)](https://openvpn.net/faq/why-does-openvpn-use-udp-and-tcp/)

</div>

<div class="Any-vertInsets">
<a href="{{ site.url }}/post/vpn-server">
  <div class="PrimaryButton">
    Continue Reading »
  </div>
</a>
</div>
