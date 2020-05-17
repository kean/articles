---
layout: post
title: Navigation Problem
description: How to solve it without Coordinators and why Coordinators might be not an optimal solution for you
date: 2019-03-24 9:00:00 +0300
category: programming
tags: programming
permalink: /post/navigation-problem
uuid: 11f62862-1a42-4eef-9dcc-f8c15a8a19c0
---

Coordinators, Wireframe, Routers, Flows – all of these concepts try to deal with the "navigation problem". Recently I've been working on an app which heavily used Flows and Wireframes – yes, both at the same time. The result was a huge mess. It was painfully hard to follow the logic, even trivial changes required a lot of work, many of these flows were not testable.

There seems to be a reason why this happens. Coordinators often break separation of concerns.

A Coordinator would typically end up creating new modules (Factory), manipulating data (Model), deciding what to do display based on that data (Controller), and pushing content on screen using UIKit objects (View). Model, View, Controller, Model, View... are you getting this? These are not three separate components, this is one component, and this is the problem.

{% include ad-hor.html %}

## MVC

Any MVC-based architecture already provides you with the tools to manage navigation. In this article I'm going to use MVP as an example. But the same principles can be apply to other patterns like MVC and MVVM.

> I use Factories which construct Model and Presenter objects and which have access to Context for DI. I also skip some protocol declarations and `final` keywords which I would normally use. And I use delegates which could be of course replaced with other mechanisms like closures.

> Please keep in mind that the goal of the article is to describe a solution which would only require these three MVP components – Model, View and Presenter – but would still maintain full separation of concerns and keep the modules unit testable.

We need a domain area for our example. Let's pretend that we live in a world of IoT devices where every door lock is a connected one. Now we need an app.

### Managing Multiple Screens

First, we need to be able to connect to the lock. This is an activity the requires multiple screens to complete:

- We need to locate the device in the network
- Ask the user to enter the credentials to log in
- Show the device details page when we finished

We would also need to pass context – the detected device – from the first step to the last.

Let's see how we would approach this if we limit ourselves to MVP. I would start by creating a Presenter and a View for our flow. The View is going to be a subclass of `UINavigationController` which we would then present modally on the screen.

```swift
class DeviceInstallationViewController: UINavigationController {
    init(presenter: DeviceInstallationPresenter) {
        self.presenter = presenter
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        presenter.onViewDidLoad()
    }

    func showDeviceDetectionScreen(presenter: DeviceDetectionPresenter) {
        let vc = DeviceDetectionViewController(presenter: presenter)
        pushViewController(vc, animated: false) // This is the first screen
    }
}

class DeviceInstallationPresenter {
    func onViewDidLoad() {
        let presenter = DeviceDetectionPresenterFactory.make(delegate: self)
        view.showDeviceDetectionScreen(presenter: presenter)
    }

    // MARK: DeviceDetectionPresenterDelegate

    func deviceDetected(_ device: Device) {
        // TODO: save the device and show Login screen
    }
}
```

So now we already have a place to put the logic which selects which screens to present and in which order –  `DeviceInstallationPresenter`. We have a way to actually display the first step and all of the remaining steps – `DeviceInstallationViewController`. And if we need to manipulate some data we can also create `DeviceInstallationModel`.

Now let's add the remaining steps:

```swift
class DeviceInstallationViewController: UINavigationController {
    func showLoginScreen(presenter: DeviceLoginPresenter) {
        let vc = DeviceLoginViewController(presenter: presenter)
        pushViewController(vc, animated: true)
    }

    func showDeviceDetails(presenter: DeviceDetailsPresenter) {
        let vc = DeviceDetailsViewController(presenter: presenter)

        // Let's say we want to add a "Close" button to the Details screen
        // but only if it is shown within this flow.
        vc.navigationItem.rightBarButtonItem = UIBarButtonItem(title: "Close", target: self.presenter, action: #selector(buttonCloseDeviceDetailsTapped))

        pushViewController(vc, animated: true)
    }
}

class DeviceInstallationPresenter {
    // MARK: DeviceDetectionPresenterDelegate

    func deviceDetected(_ device: Device) {
        model.device = device // Remember which device we detected

        let presenter = DeviceLoginPresenterFactory.make(delegate: self)
        view.showCredentialsScreen(presenter: presenter)
    }

    // MARK: DeviceLoginPresenterDelegate

    func deviceLoginFinished(token: AuthToken, credentials: Credentials) {
        assert(model.device != nil, "The device needs to be decected first")
        guard let device = model.device else { return }

        model.save(credentials: credentials, token: token for: device)
    }

    // MARK: Details Screen Navigation Items

    func buttonCloseDeviceDetailsTapped() {
        assert(model.device != nil, "The device needs to be decected first")
        guard let device = model.device else { return }

        delegate.deviceInstallationPresenter(self, didFinishInstallingDevice: device)
    }
}

```

Great, the flow is completed. Let's see how we would use it:

```swift
class DeviceTabViewController {
    func showDeviceInstallation(_ presenter: DeviceInstallationPresenter) {
        let vc = DeviceInstallationViewController(presenter: presenter)
        present(vc, animated: true)
    }
}

class DeviceTabPresenter {
    func buttonInstallDeviceTapped() {
        let presenter = DeviceInstallationPresenterFactory.make(delegate: self)
        view.showDeviceInstallation(presenter)
    }
}
```

As you probably noticed the usage is identical to how we present the steps of the flow itself.

The good thing about this approach is that, unlike Coordinators, it already has memory management sorted out. Views hold strong references to Presenters. Views also hold strong references to child Views (e.g. child view controllers). When we show `DeviceInstallationViewController` it is automatically retained by UIKIt and it also retains its presenter.

> In the example, `DeviceInstallationViewController` is a navigation controller, but it could also be the initial view controller of the flow which would could the show the way we wanted.

We were able to use class MVP components to implement the entire flow which means that we have a clear separation of concerns and also a clear way to write unit tests for the flow – we can fully test `DeviceInstallationModel` and we can fully test `DeviceInstallationPresenter`.

Now that we finished with the flow, let's take a brief look at the other ways we can use MVP. It's clear that we can use the same approach to show one Screen from another Screen – we already did when we used `DeviceInstallationPresenter`.

### MVP for Table Cells

MVP can scale from managing multiple screens down to managing individual table cells. Let's see how it can be done on another example.

The installed devices are displayed in a list. Tapping on an item in the list opens the Device Details screen.

> This is a relatively simple example where MVP might seem like overkill but remember that this is just an example. Your cells might be more complex, they might be reused between screens, etc.

```swift
class DeviceCell: UIViewController {
    // "Late binding"
    // We can't inject `Presenter` in `init` because of the cell reuse
    func display(presenter: DeviceCellPresenterProtocol) {
        titleLabel.text = presenter.title
    }
}

class DeviceCellPresenter {
    let title: String

    private let device: Device
    private weak var delegate: DeviceCellDelegate?

    init(device: Device, delegate: DeviceCellDelegate) {
        self.device = device
        self.title = device.name
        self.delegate = delegate
    }

    func deviceCellTapped() {
        self.delegate?.deviceCellTapped(device: device)
    }
}
```

Cell selection in UIKit is handled by `UITableViewDelegate` so we need to send the "cell tapped" events from the class that manages the list:

```swift
class DeviceListViewController: UIViewController, UITableViewDelegate {
    func display(devices: [DeviceCellPresenter]) {
        self.devices = devices
        tableView.reloadData()
    }

    //  MARK: Table View

    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) {
        cell.display(presenter: devices[indexPath.row])
    }

    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        devices[indexPath.row].deviceCellTapped()
    }
}

class DeviceListViewPresenter: DeviceCellPresenterDelegate {

    private func didFetchDevices(_ devices: [Device]) {
        view.display(devices: devices.map {
            DeviceCellPresenterFactory.make(device: $0, delegate: self)
        })
    }

    // MARK: DeviceCellPresenterDelegate

    func deviceCellTapped(device: Device) {
        // TODO: handle cell taps
    }
}
```

Now that we wired everything together let's implement the actual Details screen presentation which we delegate from `DeviceCell` to `DeviceCellListViewController` which has access to `navigationController`.

```swift
class DeviceListViewController: UIViewController, UITableViewDelegate {
    // MARK: Device Details

    func showDeviceDetails(presenter: DeviceDetailsPresenter) {
        let vc = DeviceDetailsPresenter(presenter: presenter)
        push(vc)

        // We use the same screen that we used in the installation flow
        // but here we are OK with just having a default back button,
        // we don't add a "Close" button.
    }
}

class DeviceListViewPresenter {

    // MARK: DeviceCellPresenterDelegate

    func deviceCellTapped(device: Device) {
        let presenter = DeviceDetailsPresenterFactory.make(device: device)
        view.showDeviceDetails(presenter)
    }
}
```

## Alternatives

A more advanced design might include a separate component which would know about transitions between the screens - something like native segues or exactly native segues. This design would be a bit more complex but it does have a few pros:

- Screens no longer need to know about other Screens
- Transitions can be reused between modules
- Transitions can be used by Views which can't show new screens by themselves

Another potential change that you could make is to actually use the term `Flow` for modules which don't have their own UI and only manager navigation between the screens. So `DeviceInstallationPresenter` would than be called `DeviceInstallationFlow` but it would still have it's own model if needed – `DeviceInstallationModel`. This might make the separation a bit more clear but I personally prefer the term `Presenter`.

## Final Thoughts

MVP and other similar patterns like MVC, MVVM scale very well from managing individual views like table cells to managing flows consisting of multiple screens. It maintains a clear separation of concerns and makes code testable. It is elegant and it has memory management sorted out. I would encourage you to give it a try before reaching out for other tools.
