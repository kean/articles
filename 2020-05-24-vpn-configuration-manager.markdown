---
layout: post
title: "VPN, Part 1: VPN Profiles"
subtitle: Using Network Extension framework to create and manage VPN profiles
description: Building a SwiftUI app that users NETunnelProviderManager (Network Extension framework) to create and manage VPN configuration profiles
date: 2020-05-24 9:00:00 -0400
category: programming
tags: programming
permalink: /post/vpn-configuration-manager
uuid: 21837a5f-3848-46c6-aa58-7f80410cd077
minutes: 10
---

Every [App Extension](https://developer.apple.com/app-extensions/) requires an app, and Network Extensions are no exception. Before we jump into the nitty-gritty details of the packet tunnel itself, let me walk you through the app. 

<!-- {% include ad-hor.html %} -->

## Introduction

The app is not just a container for an extension. It is also going to serve as a way to install and manage the VPN configuration. Let's call our service **BestVPN**.

There are going to be two primary screens:

- A **Welcome** screen with a button to install a VPN profile
- A **Tunnel Details** screen for managing the installed profile

<img alt="VPN iOS app screenshot welcome page" width="325px" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/vpn-tunnel/vpn-app-02.png">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<img alt="VPN iOS app screenshot details page" width="325px" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/vpn-tunnel/vpn-app-03.png">

I'm going to build the app using SwiftUI and share some of the details regarding Network Extensions.

> The entire app source code is available at [kean/VPN](https://github.com/kean/VPN).
{:.info}


## Entitlements

First, we are going to need an entitlement for our app:

- [**Network Extension: Packet Tunnel**](https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_developer_networking_networkextension) entitlement to tunnel IP packets to a remote network using any custom tunneling protocol using [`NEPacketTunnelProvider`](https://developer.apple.com/documentation/networkextension/nepackettunnelprovider)

The easiest way to add entitlements is via <b>Signing & Capabilities</b> in Xcode.

<img alt="Xcode: add entitlements" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/vpn-tunnel/vpn-app-01.png">

One you go through the steps, you are going to see the following two entitlements appear in your apps' entitlements list:

```
<key>com.apple.developer.networking.networkextension</key>
<array>
	<string>packet-tunnel-provider</string>
</array>
```

> You **don't** need a [**Personal VPN**](https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_developer_networking_vpn_api) entitlement which allows apps to create and control a custom system VPN configuration using [`NEVPNManager`](https://developer.apple.com/documentation/networkextension/nevpnmanager). The Packet Tunnel Provider entitlements are classified as enterprise VPNs and only require Network Extension entitlement.
{:.info}

If start interacting with Network Extension APIs and see the following error, most likely means that the entitlements are misconfigured:

```swift
Error Domain=NEConfigurationErrorDomain
Code=10 "permission denied"
UserInfo={NSLocalizedDescription=permission denied}
```

> You must run the app on the physical device, Network Extensions are not supported in the iOS Simulator.
{:.warning}

## VPN Manager

The primary interface for managing VPN configurations that use a custom VPN protocols is [`NETunnelProviderManager`](https://developer.apple.com/documentation/networkextension/netunnelprovidermanager), which is a sublcass of [`NEVPNManager`](https://developer.apple.com/documentation/networkextension/nevpnmanager).

> You can use [`NEVPNManager`](https://developer.apple.com/documentation/networkextension/nevpnmanager) without the tunnel provider to create and manage [personal VPN](https://developer.apple.com/documentation/networkextension/personal_vpn) configurations that use one of the built-in VPN protocols (IPsec or IKEv2).
{:.info}


The first thing that you want to do when your app starts is read the exiting configuration.

```swift
NETunnelProviderManager.loadAllFromPreferences { managers, error in
    // Where managers: [NETunnelProviderManager]
}
```

This method reads all of the VPN configurations created by the calling app that have previously been saved to the Network Extension preferences. The completion closure gives you the list of "managers", each represents a single saved VPN configuration.

In our case, there is only one configuration that the app manages. When the configuration is loaded, the `RouterView` decides which screen to show.

```swift
struct RouterView: View {
    @ObservedObject var service: VPNConfigurationService = .shared

    var body: some View {
        if !service.isStarted {
            // Splash is where I'm loading the configuration for the first time
            return AnyView(SplashView())
        }

        // When the initial configuration is loaded, I either display a Details
        // screen or a Welcome screen.
        if let tunnel = service.tunnel {
        	let model = TunnelViewModel(tunnel: tunnel)
            return AnyView(TunnelDetailsView(model: model))
        } else {
            return AnyView(WelcomeView())
        }
    }
}
```

## Installing VPN Profile

Creating and saving a VPN configuration is relatively easy. All you need to do is create and configure `NETunnelProviderManager` instance and then save it.

```swift
private func makeManager() -> NETunnelProviderManager {
    let manager = NETunnelProviderManager()
    manager.localizedDescription = "BestVPN"

    // Configure a VPN protocol to use a Packet Tunnel Provider
    let proto = NETunnelProviderProtocol()
    // This must match an app extension bundle identifier
    proto.providerBundleIdentifier = "com.github.kean.vpn-client.vpn-tunnel"
    // Replace with an actual VPN server address
    proto.serverAddress = "127.0.0.1:4009"
    // Pass additional information to the tunnel
    proto.providerConfiguration = [:]

    manager.protocolConfiguration = proto

    // Enable the manager by default
    manager.isEnabled = true

    return manager
}
```

In this sample we created a manager and configured its VPN protocol. Once you created an instance of `NETunnelProviderManager`, you can save it:

```swift
let manager = makeManager()
manager.saveToPreferences { error in
    if error == nil {
        // Success
    }
}
```

> When saving an extension, I would sometimes encounter an issue where I would not be able to start a VPN tunnel right after saving it. I'm not the only one having [this issue](https://forums.developer.apple.com/thread/25928). A workaround seems to be to reload the manager using `loadFromPreferences()` method right after saving it.
{:.warning}

> [Apple Developer Forums](https://forums.developer.apple.com) are one of the best and only sources of information when debugging Network Extensions issues.
{:.info}

<div class="SwiftUIExampleWithScreenshot Any-responsiveCard">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left">
            <h3 class="no_toc">Tunnel Details</h3>
            <p>When the user taps "Install VPN Profile" button, a system alert is going appear and is going to guide the user through the process of installing the VPN profile. Once it's done, the system returns the user back to the app where <code>RouterView</code> presents a "Tunnel Details" screen from earlier.</p>
            <blockquote class="info"><p><code>RouterView</code> is a component responsibly for top-level navigation in the BestVPN app.</p></blockquote>
            <p>In the next section, I'm going to cover the "Tunnel Details" screen and how it interact with the system settings in details.</p>
        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right">
            <img alt="VPN iOS app screenshot: details page" src="{{ site.url }}/images/posts/vpn-tunnel/vpn-app-03.png">
        </div>
    </div>
</div>

## Managing VPN

Now that the profile is installed, the same `NETunnelProviderManager` instance can be used to update it and to manage the connection status.

### Enabling Configuration

Enabled VPN configuration have [`isEnabled`](https://developer.apple.com/documentation/networkextension/nevpnmanager/1406382-isenabled) flag set to `true`.

> <h3 class="no_toc">Configuration Model</h3>
>
> Each [`NETunnelProviderManager`](https://developer.apple.com/documentation/networkextension/netunnelprovidermanager#overview) instance corresponds to a single VPN configuration stored in the Network Extension preferences. Multiple VPN configurations can be created and managed by creating multiple `NETunnelProviderManager` instances.
>
>Each VPN configuration is associated with the app that created it. The app’s view of the Network Extension preferences is limited to include only the configurations that were created by the app.
>
> VPN configurations created using `NETunnelProviderManager` are classified as regular enterprise VPN configurations (as opposed to the Personal VPN configurations created by NEVPNManager). Only one enterprise VPN configuration can be enabled on the system at a time. If both a Personal VPN and an enterprise VPN are active on the system simultaneously, the enterprise VPN takes precedence, meaning that if the routes for the two VPNs conflict then the routes for the enterprise VPN will take precedence. The Personal VPN will remain active and connected while the enterprise VPN is active and connected, and any traffic that is routed to the Personal VPN and is not routed to the enterprise VPN will continue to traverse the Personal VPN.
>
> *From [Apple Developer Documentation](https://developer.apple.com/documentation/networkextension/netunnelprovidermanager#overview)*.


To detect configuration changes, use [`NEVPNConfigurationChange`](https://developer.apple.com/documentation/foundation/nsnotification/name/1406509-nevpnconfigurationchange) notification.

```swift
final class TunnelDetailsViewModel: ObservableObject {
    @Published var isEnabled: Bool

    private var observers = [AnyObject]()

    init(tunnel: NETunnelProviderManager) {
    	self.isEnabled = tunnel.isEnabled

    	observers.append(NotificationCenter.default.addObserver(
            forName: .NEVPNConfigurationChange,
            object: tunnel,
            queue: .main) { [weak self] _ in
                self?.isEnabled = tunnel.isEnabled
        })
    }
}
```

<div class="SwiftUIExampleWithScreenshot Any-responsiveCard">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left">
            <h3 class="no_toc">VPN Settings</h3>
            <p>If you go to <b>Settings / General / VPN</b>, you are going to see your VPN configuration registered there.</p>
            <p>The settings screen allow user to switch between configurations (<code>isEnabled</code>), start/stop VPN, and even remove the configuration. This is why it is important to use <a href="https://developer.apple.com/documentation/foundation/nsnotification/name/1406509-nevpnconfigurationchange"><code>.NEVPNConfigurationChange</code></a> and <a href="https://developer.apple.com/documentation/foundation/nsnotification/name/1406683-nevpnstatusdidchange"><code>.NEVPNStatusDidChange</code></a> notification to update your app UI accordingly.</p>
        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right">
            <img alt="iOS VPN settings page screenshot" src="{{ site.url }}/images/posts/vpn-tunnel/settings-01.jpeg">
        </div>
    </div>
</div>

### Managing Connection

A manager has an associated [`NEVPNConnection`](https://developer.apple.com/documentation/networkextension/nevpnconnection) object that is used to control the VPN tunnel specified by the VPN configuration.

You can use the connection object and [`.NEVPNStatusDidChange`](https://developer.apple.com/documentation/foundation/nsnotification/name/1406683-nevpnstatusdidchange) notification to keep track of the connection status.

### Starting/Stopping Tunnel

The tunnel can be started/stopped programatically.

```swift
func buttonStartTapped() {
    do {
        try tunnel.connection.startVPNTunnel(options: [
            // Don't share with anyone!
            NEVPNConnectionStartOptionUsername: "kean",
            NEVPNConnectionStartOptionPassword: "password"
        ] as [String : NSObject])
    } catch {
        self.showError(
            title: "Failed to start VPN tunnel",
            message: error.localizedDescription
        )
    }
}

func buttonStopTapped() {
    tunnel.connection.stopVPNTunnel()
}
```

### VPN On Demand

[VPN On Demand](https://developer.apple.com/documentation/networkextension/personal_vpn/vpn_on_demand_rules) is one of the options of `NETunnelProviderManager` that allows the system to automatically start or stop a VPN connection based on various criteria. For example, you can use VPN On Demand to configure an iPhone to start a VPN connection when it’s on Wi-Fi and stop the connection when it’s on cellular. Or, you can start the VPN connection when an app tries to connect to a specific service that’s only available via VPN.

Here is an example how to configure a manager to always start a tunnel whenever it is needed:

```swift
let onDemandRule = NEOnDemandRuleConnect()
onDemandRule.interfaceTypeMatch = .any
manager.isOnDemandEnabled = true
manager.onDemandRules = [onDemandRule]
```

## Creating Extension

Now let's quickly create a [Packet Tunnel Provider](https://developer.apple.com/documentation/networkextension/packet_tunnel_provider) extension itself to see if it actually works.

<img alt="Xcode: creating packet tunnel provider extension" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/vpn-tunnel/vpn-tunnel-01.png">

<img alt="Xcode: creating packet tunnel provider extension" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/vpn-tunnel/vpn-tunnel-02.png">

> Xcode automatically creates a bundle identifier for an extensions in a form of `<app-bundle-identifier>.<extension-name>`. An extension's bundle identifier must always be prefixed with an app's bundle identifier. If you change either later, make sure to update both to save yourself hours of debugging.
{:.warning}

When the extension is created, you must always set up entitlement to match the app.

<img alt="Xcode: setting entitlements for packet tunnel provider extension" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/vpn-tunnel/vpn-app-04.png">

By default, Xcode creates an `NEPacketTunnelProvider` subclass automatically for you.

```swift
class PacketTunnelProvider: NEPacketTunnelProvider {

    override func startTunnel(
        options: [String : NSObject]?,
        completionHandler: @escaping (Error?) -> Void)
    {
        NSLog("Starting tunnel with options: \(options ?? [:])")
        // Add code here to start the process of connecting the tunnel.
    }
}
```

> Don't rename the `NEPacketTunnelProvider` class. If you do, make sure to update `Info.plist` file with a new class name, otherwise the extension is going to start crashing on launch.
> 
> <img class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/vpn-tunnel/vpn-tunnel-03.png">
{:.warning}

Now when you run the app on the device, and install the profile, it should automatically start the extension.

Debugging an extension is a bit tricky. When you start the app, Xcode automatically attaches the debugger to it. It is not going to happen with the extension. To attach a debugger to an extension, go to **Debug / Attach to Process by PID or Name...** and add the name of your extension.

<img alt="Xcode: attaching debugger to the running process" width="440px" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/vpn-tunnel/vpn-tunnel-04.png">

If the extension is already running, Xcode will attach the debugger to it. If it isn't running, Xcode is going to wait until it starts running. Now when you open a Debug Navigator, you are going to see two processed instead of one (typical to iOS app development).

<img alt="Xcode: debugging multiple processes at the same time" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/vpn-tunnel/vpn-tunnel-05.png">

I encourage you to test whether you are able to attach the debug to the extension.

> <h3 class="no_toc">Troubleshooting Guide</h3>
>
> App extensions are quite unforgiving and debugging them can become a nightmare. If you are not able to attach the debugger or the extension process crashes without reaching `PacketTunnelProvider` go through the following troubleshooting guide.
>
> <img class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/vpn-tunnel/vpn-tunnel-06.png">
> 
> - Make sure both the app and the app extensions has Packet Tunnel Network Extensions entitlement
> - An extension's bundle identifier must always be prefixed with an app's bundle identifier. If you change either later, make sure to update both to save yourself hours of debugging.
> - An extension's `Info.plist` file must contain correct `NSExtensionPointIdentifier` and `NSExtensionPrincipalClass` values. If you changed the name of the `NEPacketTunnelProvider` subclass, make sure to update `NSExtensionPrincipalClass` to reflect it.
>
{:.error}

Congratulations, if you reached this point, you know how to install a VPN configuration, enable it, start a VPN tunnel, and attach a debugger to it. This a great progress! In the next part, I'm going to cover the Packet Tunnel itself.

<!-- <div class="Any-vertInsets">
<a href="{{ site.url }}/post/packet-tunnel-provider">
  <div class="PrimaryButton">
    Continue Reading »
  </div>
</a>
</div>
 -->