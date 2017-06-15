Rocket iOS Best Practices
==================

_This document captures standard / best practices for any iOS app that is developed at Rocket Insights. Feedback / changes / suggestions are encouraged!_

## Contents

1. [Getting Started](#getting-started)
1. [Architecture](#architecture)
1. [Common Libraries](#common-libraries)
1. [Networking](#networking)
1. [Assets](#assets)
1. [Coding Style](#coding-style)
1. [Security](#security)
1. [Analytics](#analytics)
1. [What else?] (#what-else)

## Getting Started

#### Use Swift

All new developent at done in Swift (vs. Objective-C). It is the language that Apple is investing in and where the platform is headed. Additionally,
* Crash rates will be lowered because of language structure (optionals, try statements etc.)
* All new books / stackoverflow / community examples are in Swift
* The Swift stdlib is constantly being improved

#### Use Storyboards

* Easy to grok the application layout / flow instantly
* Iteration is faster, easier to do autolayout, nice to visually see elements
* Make sure to [use multiple storyboards](http://blog.rocketinsights.com/all-together-now-part-deux/) to avoid merge conflicts when working on an app with multiple developers. Logically break up app at functional boundaries and put each one into its own Storyboard. 
* Use IBDesignable/IBInspectable where appropriate
* Interface Builder gives you a live layout preview for the devices of your choice, including iPad split-screen multitasking.

### (Git) Ignores

Use the [GitHub Swift](https://github.com/github/gitignore/blob/master/Swift.gitignore) one.

### Dependency Management

The swift package manager will ultimately be the way to distribute libraries. Until then use [Carthage](https://github.com/Carthage/Carthage) unless there is a compelling reason not to. 

The reason we use Carthage is because its a simpler model then CocoaPods and thus less likely to break with each Xcode iteration.

A reason *not* to use Carthage is if there is a key library that you need to access and it is not available via Carthage.

When updating carthage libs you should usually use a command of the form:
`carthage update --platform iOS --no-use-binaries`

This will speed up the build (only build framework for iOS) and forces ut to compile from source which helps when we need to switch between versions of Xcode.

### Project Structure

Organize your files at both the Group layer in Xcode and also on the filesystem. The structure for files should be

    ├─ Models
    ├─ Views
    ├─ Controllers
    ├─ Services
    ├─ Helpers
    ├─ Storyboards
    ├─ Supporting Files
        ├─ Bridging Header

First, create them as groups (little yellow "folders") within the group with your project's name in Xcode's Project Navigator. Then, for each of the groups, link them to an actual directory in your project path by opening their File Inspector on the right, hitting the little gray folder icon, and creating a new subfolder with the name of the group in your project directory.

#### Localization

Keep all user strings in localization files right from the beginning. This is good not only for translations, but also for finding user-facing text quickly. You can add a launch argument to your build scheme to launch the app in a certain language, e.g.

    -AppleLanguages (Finnish)

#### Constants

Keep your constants' scope as small as possible. For instance, when you only need it inside a class, it should live in that class. Those constants that need to be truly app-wide should be kept in one place. Define a `Constants.swift` file to group, store and access your app-wide constants in a clean way:

```swift
struct Constants {
    struct Config {
        static let baseURL: NSURL(string: "http://www.example.org/")!
        static let splineReticulatorName = "foobar"
    }
    
    struct Color {
        static let primaryColor = UIColor(red: 0.22, green: 0.58, blue: 0.29, alpha: 1.0)
        static let secondaryColor = UIColor.lightGrayColor()
    }
}
```

### Branching Model

There are two main branches `master` and `development`. Master contains code that is released to the app store. Development contains on-going development work that is not ready for deployment.

When a release is ready, the code is merged from `development` to `master`. The release is then cut from `master`. Each time a build is submitted to Apple a tag should be created.

All feature work should be done on a topic branch. A pull request can then be made from the topic branch to `dev`.

## Common Libraries

Generally speaking, make it a conscious decision to add an external dependency to your project. Here are some common libraries we use
* **AlamoFire**: Networking
* **SwiftyJSON**: JSON parsing / management
* **FLAnimatedImage**: If you don't have animated images in your app you are doing it wrong.
* **DateTools**: Who doesn't love date management?
* **UICKeyChainStore**: Wrapper around accessing the Secure Keychain
* **TTTAttributedLabel**: An oldie but goodie -- still the best for annotated/linkable text

## Architecture

Our apps tend to use the  [Model-View-Controller-Store (MVCS)](http://programmers.stackexchange.com/questions/184396/mvcs-model-view-controller-store) architecture. There are valid alternatives -- notably MVVM is becoming popular. However, we default to MVCS because:

* This is the default Apple architecture (MVC), extended by a Services layer that knows how to interact with the Network layer and interface to the data layer.
* Core business logic for non-model objects can also be put into Services layer (e.g., HealthKitServices)

### Models

Keep your models immutable, and use them to translate the remote API's JSON into the object's properties. Use [SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON) to parse the JSON returned by the API.

Make your models be _structs_.

### Views

With today's wealth of screen sizes in the Apple ecosystem and the advent of split-screen multitasking on iPad, the boundaries between devices and form factors become increasingly blurred. Much like today's websites are expected to adapt to different browser window sizes, your app should handle changes in available screen real estate in a graceful way. This can happen e.g. if the user rotates the device or swipes in a secondary iPad app next to your own.

Always use size classes and auto-layout to ensure that a view is flexible. The system will then calculate the appropriate frames based on these rules, and re-evaluate them when the environment changes.

Apple's [recommended approach][wwdc-autolayout-mysteries] for setting up your layout constraints is to create and activate them once during initialization. If you need to change your constraints dynamically, hold references to them and then deactivate/activate them as required. 

### Controllers

MVC does not stand for massive view controller -- so avoid massive view controllers. Use these patterns to help avoid it:
* Child VCs
* Services layer

A good write-up of different techniques is [here](http://khanlou.com/2014/09/8-patterns-to-help-you-destroy-massive-view-controller/).

## Services

At the "ground level" of a mobile app is usually some kind of model storage, that keeps its data in places such as on disk, in a local database, or on a remote server. The services layer knows how to coordinate between the network and the model layer.

E.g., a UserServices class may be responsible for fetching data from a remote API, call into the model objects to populate their properties from the returned JSON and then persist this to the data store.

## Assets

[Asset catalogs](https://developer.apple.com/library/ios/recipes/xcode_help-image_catalog-1.0/_index.html) are the best way to manage all your project's visual assets. They can hold both universal and device-specific (iPhone 4-inch, iPhone Retina, iPad, etc.) assets and will automatically serve the correct ones for a given name.

### Using Vector Images

Don't use bitmap images... [use vector graphics] (http://martiancraft.com/blog/2014/09/vector-images-xcode6/) where possible. This is because vector images:
* Only need a single asset (PDF)
* Are future proof because if a 4x image is ever needed these can be generated from the PDF automatically by Xcode at build time

## Coding Style

Follow the [Rocket Swift Style Guide](https://github.com/rocketinsights/swift-style-guide).

## Security

Even in an age where we trust our portable devices with the most private data, app security remains an often-overlooked subject. 

### Data Storage

If your app needs to store sensitive data, such as a username and password, an authentication token or some personal user details, you need to keep these in a location where they cannot be accessed from outside the app. Never use `NSUserDefaults`, other plist files on disk or Core Data for this, as they are not encrypted! Use the keychain using the UICKeyChainStore pod.

### Networking

Keep any HTTP traffic to remote servers encrypted with TLS at all times. To avoid man-in-the-middle attacks that intercept your encrypted traffic, you can set up certificate pinning.. Popular networking libraries such as AFNetworking and Alamofire support this out of the box.

The general approach for the networking layer is to do the following:

1. Have an XXXSessionManager class that subclasses URLSessionManager/AFHTTPSessionManager. This can perform all required configuration of the session manager including headers, logging etc. This has class methods to return a sessionManager per host.
2. Have an APIClient class that exposes POST/GET/DELETE/PUT methods. The APIClient caches the XXXSessionManager in a static instance. The POST/GET/DELETE/PUT methods take path, parameters and request completion handler that has statusCode, responseObject and the task itself. The calling of the completion handler is wrapped by another closure. This closure can perform common functions like logging user out, re-enqueueing failed tasks, updating user profile etc. 
3. APIClient+XXX provides specific API calls. E.g., APIClient+User will have loginUser() etc.
4. UserServices will use the APIClient class to call server, write to local storage and then return type safe object.

### Logging

Take extra care to set up proper log levels before releasing your app. Production builds should never log passwords, API tokens and the like, as this can easily cause them to leak to the public. On the other hand, logging the basic control flow can help you pinpoint issues that your users are experiencing.

### Linter

Use a linter! We use [SwiftLint](https://github.com/realm/SwiftLint) for all of our projects.

All warnings should fail the build (set them as errors). The linter should be run as part of every build.

## Analytics

Including some analytics framework in your app is strongly recommended, as it allows you to gain insights on how people actually use it. We recommend MixPanel (super awesome!) or Google Firebase (free!). 

A good practice is to create a slim helper class, e.g. `AnalyticsServices`, that handles the the firing of screen views / events.

This has the advantage of allowing you to swap out the entire Analytics framework behind the scenes if needed, without the rest of the app noticing. Also, you can layer in additional analytic endpoints as needed. E.g., tracking screen views in Fabric.

If using Firebase these best practices are recommended:
* Keep all events as flat as possible. This means the _name_ of the event should contain as much data as possible. Parameter should not be used heavily. This is because funnels in Firebase can not support parameters.
* Logically grouped events should have the same prefix. Specific to all screen views we should track the events as screen_xxx. E.g., screen_home, screen_register

### Crash Logs

Include Fabric as part of every project so you can track crash rates. These should be .5% or less. 

## What else

- Certificate management -- developer, match?
- Add section on Testing
- Target set-up?

