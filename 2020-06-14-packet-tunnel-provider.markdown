---
layout: post
title: "VPN, Part 2: Packet Tunnel Provider"
subtitle: Using NEPacketTunnelProvider to implement a VPN client
description: A tutorial on NEPacketTunnelProvider usage to implement an iOS client for a custom tunneling protocol
date: 2020-06-14 9:00:00 -0400
category: programming
tags: programming
permalink: /post/packet-tunnel-provider
uuid: d3d4d400-70f3-40a1-a8d3-bcfe9df881f8
---

 With most of the groundwork covered in ["How Does VPN Work?"](/post/networking-101), let's go ahead and design a protocol.

 {% include ad-hor.html %}

## Designing a Protocol

I want the protocol to be functional, but keep it as simple as possible. The goal is not to design a viable VPN protocol, but to demonstrate [Network Extension](shttps://developer.apple.com/documentation/networkextension) framework.

There are three things that our protocol is going to need to be able to do:

- Establish a secure connection
- Authenticate the user based on username and password
- Provide a way to send encrypted IP packets – the actual network traffic – over the network

### Encryption

The industry standard for secure connections is [TLS](https://tools.ietf.org/html/rfc8446). A typical VPN protocol would use *asymmetric* cryptography to allow the client and the server to establish a secure connection and share a *symmetric* encryption key (*hybrid* encryption). However, to keep things simple, I'm going to use a pre-shared key.

> **Warning!**
>
> Don't rely on this article for the security advice. If you are implementing a VPN protocol, you might want to use a TLS implementation provided by Apple in [Network framework](https://developer.apple.com/documentation/network). Use this framework when you need direct access to protocols like TLS, TCP, and UDP for your custom application protocols. There is an [example project](https://developer.apple.com/documentation/network/building_a_custom_peer-to-peer_protocol) provided by Apple which demonstrates how to use their TLS implementation.
{:.warning}

### Control Packets

There are two types of packets the client and the server need to be able to exchange: control packets and data packets. To distinguish between packets, we are going to need a header. I'm going to define my protocol as a Swift library, so let's start adding some code.

```swift
/// The VPN protocol header.
public struct Header {
    public let code: PacketCode

    /// There is only one one-byte fields in the header.
    public static let length = 1
}

public enum PacketCode: UInt8 {
    /// A control packet containing client authentication request (JSON).
    case clientAuthRequest = 0x01
    /// A control packet containing server authentication response (JSON).
    case serverAuthResponse = 0x02
    /// A data packet containing encrypted IP packets (raw bytes).
    case data = 0x03

    /// Initilizes the code code with the given UDP packet contents.
    public init(datagram: Data) throws {
        guard datagram.count > 0 else {
            throw PacketParsingError.notEnoughData
        }
        guard let code = PacketCode(rawValue: datagram[0]) else {
            throw PacketParsingError.invalidPacketCode
        }
        self = code
    }
}
``` 

The body of the packets can be anything you want. In my case, I'm going to stick with simple JSON for control packets.

```swift
public enum Body {
    public struct ClientAuthRequest: Codable {
        public let login: String
        public let password: String

        public init(login: String, password: String) {
            self.login = login
            self.password = password
        }
    }

    /// ...
}
```

### Data Packets

To exchange actual network traffic (IP packets), I'm going to use `PacketCode.data` and put the raw (encrypted) data in body.

### Transport

[`NEPacketTunnelProvider`](https://developer.apple.com/documentation/networkextension/nepackettunnelprovider) provides you two ways to send packets inside your private network:

- UDP: [`createUDPSession`](https://developer.apple.com/documentation/networkextension/neprovider/1406004-createudpsession)
- TCP: [`createTCPConnection`](https://developer.apple.com/documentation/networkextension/neprovider/1406529-createtcpconnection)

We could use either of these options. UDP is typically going to be faster, it is easier to use, and it is a [preferred way](https://openvpn.net/faq/why-does-openvpn-use-udp-and-tcp/) in some of the existing VPN protocols, such as OpenVPN. You don't need the reliability guarantees of TCP in case of the VPN protocols – the goal is to send opaque IP packets as quickly as possible over the wire, the reliability is the concern of the protocols encapsulated in the IP packets.

## Packet Tunnel Provider

The primary API for implementing your custom tunneling protocols in [`NEPacketTunnelProvider`](https://developer.apple.com/documentation/networkextension/nepackettunnelprovider), that allows you to tunnel traffic on an IP layer. They run as app extensions, running in the background handling network traffic.

> There is also an [NEAppProxyProvider](https://developer.apple.com/documentation/networkextension/neappproxyprovider) which is another subclass of the abstract `NETunnelProvider` class. It allows you to implement tunneling on a higher, app level. Unlike `NEPacketTunnelProvider`, it requires a device to be [*supervised*](https://support.apple.com/en-us/HT202837) and it only works for [*managed* apps](https://www.apple.com/business/docs/resources/Managing_Devices_and_Corporate_Data_on_iOS.pdf). It operates on a TCP/UDP level instead of IP level.
{:.info}

To configure and control a network provider from your app, you use [`NETunnelProviderManager`](https://developer.apple.com/documentation/networkextension/netunnelprovidermanager) family of APIs, which I covered in the [previous post](/post/vpn-configuration-manager).

So, how does it work? Suppose you have a `NEPacketTunnelProvider` running on the system, connected to a VPN server and providing a tunnel to some internal network. An app wants to get some resource on the network. The app creates a socket, and a TCP/IP connection.

<img alt="WWDC: What's New in NetworkExtension and VPN Screenshot" class="Screenshot Any-responsiveCard" src="/images/posts/vpn-client/tunnel-01.png">

The packets for this TCP/IP connection are routed to a virtual `utun0` network interface. Instead of sending packets over the network, the system diverts the packets to the active `NEPacketTunnelProvider`.

<img alt="WWDC: What's New in NetworkExtension and VPN Screenshot" class="Screenshot Any-responsiveCard" src="/images/posts/vpn-client/tunnel-02.png">

The tunnel provider uses [`NEPacketTunnelFlow`](https://developer.apple.com/documentation/networkextension/nepackettunnelflow) to read the IP packets ([`readPackets(completionHandler:)`](https://developer.apple.com/documentation/networkextension/nepackettunnelflow/1406903-readpackets)) from the virtual interface. It then encapsulates the packets in your tunneling protocol and sends them over to the tunneling server. The server will decapsulate them, and inject the IP packets into the network to send them to their ultimate destination. The responses are sent back to the client in a similar fashion (encapulate-send-decapsulate), and are injected back into the client networking using [`NEPacketTunnelFlow`](https://developer.apple.com/documentation/networkextension/nepackettunnelflow) and `utun0` virtual interface ([`writePackets(_:withProtocols:)`](https://developer.apple.com/documentation/networkextension/nepackettunnelflow/1406484-writepackets)). They will be delivered back to the TCP/IP stack, and back to the application.

The `NEPacketTunnelProvider` has a lot of control over the `utun0` interface. Most importantly, it can specify the routes – the destination that will be routed to the `utun0` interface and through the tunnel.

> You can learn more about `NEPacketTunnelProvider` in the [documentation](https://developer.apple.com/documentation/networkextension/nepackettunnelprovider), and in WWDC [**What's New in NetworkExtension and VPN**](https://developer.apple.com/videos/play/wwdc2015/717/) session video.
{:.info} 

## VPN Tunnel

The packet tunnel provider extension that I created in [the previous post](/post/vpn-configuration-manager) has a file with a `NEPacketTunnelProvider` subclass. This is the entry point of the extension. When the extension is started, an instance of this class is initialized and the [`startTunnel`](https://developer.apple.com/documentation/networkextension/nepackettunnelprovider/1406118-starttunnel) method is called. 

When the Packet Tunnel Provider executes the `completionHandler` closure with a nil error parameter, it signals to the system that it is ready to begin handling network data. Therefore, the Packet Tunnel Provider should call [`setTunnelNetworkSettings(_:completionHandler:)`](https://developer.apple.com/documentation/networkextension/netunnelprovider/1406539-settunnelnetworksettings) and wait for it to complete before executing the completionHandler block.

> The domain and code of the NSError object passed to the `completionHandler` block are defined by the Packet Tunnel Provider (`NEVPNError`).
{:.info}

```swift
class PacketTunnelProvider: NEPacketTunnelProvider {
    override func startTunnel(options: [String: NSObject]?,
                              completionHandler: @escaping (Error?) -> Void) {

    }
}
```

### Establish Connection

The first thing you need to do is read the configuration set by the container app.

```swift
override func startTunnel(options: [String: NSObject]?, completionHandler: @escaping (Error?) -> Void) {
    os_log(.default, log: log, "Starting tunnel, options: %{private}@", "\(String(describing: options))")

    do {
        guard let proto = protocolConfiguration as? NETunnelProviderProtocol else {
            throw NEVPNError(.configurationInvalid)
        }
        self.configuration = try Configuration(proto: proto)
    } catch {
        completionHandler(error)
    }

    // ...
}
```

[`NETunnelProviderProtocol`](https://developer.apple.com/documentation/networkextension/netunnelproviderprotocol) has some built-in fields, such as [`username`](https://developer.apple.com/documentation/networkextension/nevpnprotocol/1406196-username) and [`passwordReference`](https://developer.apple.com/documentation/networkextension/nevpnprotocol/1406650-passwordreference), but you can also provide custom configuration using [`providerConfiguration`](https://developer.apple.com/documentation/networkextension/netunnelproviderprotocol) field.

```swift
private struct Configuration {
    let username: String
    let password: String
    let address: String

    init(proto: NETunnelProviderProtocol) throws {
        guard let serverAddress = proto.serverAddress else {
            throw NEVPNError(.configurationInvalid)
        }
        self.address = serverAddress

        guard let username = proto.username else {
            throw NEVPNError(.configurationInvalid)
        }
        self.username = username

        guard let password = proto.passwordReference.flatMap({
            Keychain.password(for: username, reference: $0)
        }) else {
            throw NEVPNError(.configurationInvalid)
        }
        self.password = password
    }
}
```

Once you have the configuration, you need to establish a connection to a server. As I mentioned previously, my protocol is based on UDP, so I'm going to use [`createUDPSession(to:from:)`](https://developer.apple.com/documentation/networkextension/neprovider/1406004-createudpsession) method which is going to return [`NWUDPSession`](https://developer.apple.com/documentation/networkextension/nwudpsession) for me.

```swift
class PacketTunnelProvider: NEPacketTunnelProvider {
    private var session: NWUDPSession!
    private var observer: AnyObject?
    private var pendingCompletion: ((Error?) -> Void)?

    override func startTunnel(options: [String: NSObject]?, completionHandler: @escaping (Error?) -> Void) {
        // ... (read configuration)

        self.pendingCompletion = completionHandler
        self.startUDPSession()
    }

    private func startUDPSession() {
        let endpoint = NWHostEndpoint(hostname: configuration.address, port: configuration.port)
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

You can either initiate `NWHostEndpoint` with an IP address or with a domain name. [`NWUDPSession`](https://developer.apple.com/documentation/networkextension/nwudpsession) performs DNS resolution for you.

Once the connection is established, you can start exchanging UDP datagrams with a server. The first thing you need to do is authenticate, using a custom control packet. I'm using my `BestVPN` library to create headers, bodies, and serialize them.

```swift
private func sessionDidBecomeReady() {
    session.setReadHandler({ [weak self] datagrams, error in
        guard let self = self else { return }
        self.queue.async {
            self.didReceiveDatagrams(datagrams: datagrams ?? [], error: error)
        }
    }, maxDatagrams: Int.max)

    do {
        try self.authenticate(username: configuration.username, password: configuration.password)
    } catch {
        // TODO: handle errors
        os_log(.default, log: self.log, "Did fail to authenticate: %{public}@", "\(error)")
    }
}

private func authenticate(login: String, password: String) {
    let datagram = try MessageEncoder.encode(
        header: Header(code: .clientAuthRequest),
        body: Body.ClientAuthRequest(login: username, password: password),
        key: key
    )

    udpSession.writeDatagram(datagram) { error in
        if let error = error {
            // TODO: Handle errors
            os_log(.default, log: self.log, "Failed to write auth request datagram, error: %{public}@", "\(error)")
        }
    }
}

private func sessionDidReceiveDatagrams(_ data: [Data]) {
    // Wait for the authentication response from the server
}
```

> **Warning!**
>
> I'm again warning you that I'm using a hardcoded encryption key in the demo for simplicity. In reality, you are going to use a proper TLS handshake or a similar approach. Don't rely on this article for the security advice!
{:.warning}

Once the authentication request is sent, I'm going to then set up a handle and wait for a UDP datagram from the server acknowledging the authentication.

```swift
private func didReceiveDatagram(datagram: Data) throws {
    let code = try PacketCode(datagram: datagram)

    os_log(.default, log: self.log, "Did receive datagram with code: %{public}@", "\(code)")

    switch code {
    case .serverAuthResponse:
        let response = try MessageDecoder.decode(Body.ServerAuthResponse.self, datagram: datagram, key: key)
        os_log(.default, log: self.log, "Did receive auth response: %{private}@", "\(response)")
        if response.isOK {
            // TODO: In reality, you would pass a resolved IP address, in our
            // case we already provide an IP address in the configuration
            self.didSetupTunnel(address: configuration.hostname)
        } else {
            // TODO: Handle error
        }
        self.timeoutTimer?.invalidate()
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

        self.didStartTunnel()
    }
}
```

> [`NEPacketTunnelNetworkSettings`](https://developer.apple.com/documentation/networkextension/nepackettunnelnetworksettings) provides a configuration for a packet tunnel provider’s virtual interface. There are some interesting things that you can do such set custom DNS addresses, configure *split* tunnel, and more. I'm going to cover this in the upcoming *bonus* article.
{:.info}

### Send Traffic

Now, how do you actually send the traffic from the system through the tunnel? A packet tunnel provider has a [`packetFlow`](https://developer.apple.com/documentation/networkextension/nepackettunnelprovider/1406185-packetflow) property returning a [`NEPacketTunnelFlow`](https://developer.apple.com/documentation/networkextension/nepackettunnelflow) object which is used to receive IP packets routed to the tunnel’s virtual interface and inject IP packets into the networking stack via the tunnel’s virtual interface.

```swift
private func didStartTunnel() {
    readPackets()
}

private func readPackets() {
    packetFlow.readPacketObjects { packets in
        do {
            let datagrams = try packets.map {
                try MessageEncoder.encode(
                    header: Header(code: .data),
                    body: $0.data,
                    key: key
                )
            }
            self.session.writeMultipleDatagrams(datagrams) { error in
                // TODO: Handle errors
            }
        } catch {
            // TODO: Handle errors
        }

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
        let data = try MessageDecoder.decode(Data.self, datagram: datagram, key: key)
        let proto = protocolNumber(for: data)
        self.packetFlow.writePackets([data], withProtocols: [proto])
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
