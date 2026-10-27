# Cosmote ID SDK for tvOS

Version 4.0.0-alpha. A Swift package of prebuilt binaries: add it to your tvOS app and
`import CosmoteID`.

## Requirements

- tvOS 17 or later
- Xcode 27.1 or later
- An Apple silicon Mac. The simulator slice is arm64 only; there is no Intel slice.
- A login for `libraries.thinkdesquared.com` in `~/.netrc`, as in the next section

## Credentials

The binaries are downloaded from `libraries.thinkdesquared.com`, which needs a login. If you
already install the Cosmote ID pod, your `~/.netrc` has it and there is nothing to do.

Otherwise, add the entry you were given to `~/.netrc` on every machine that builds your app,
including CI:

```
machine libraries.thinkdesquared.com
login <login you were given>
password <password you were given>
```

Without it, Xcode reports that it could not download `CosmoteID.xcframework.zip`.

## Add it to your project

In Xcode: File → Add Package Dependencies…, enter `https://github.com/Desquared/telekom-id`,
choose version 4.0.0-alpha, and add the `CosmoteID` library to your tvOS target.

Or in a `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/Desquared/telekom-id", exact: "4.0.0-alpha")
],
// in your tvOS target:
dependencies: [
    .product(name: "CosmoteID", package: "telekom-id")
]
```

When you build from the command line, use a concrete simulator such as
`-destination 'platform=tvOS Simulator,name=Apple TV 4K (3rd generation)'`, not
`generic/platform=tvOS Simulator`. The generic destination also builds x86_64, which this
package does not contain.

## Use it

Configure the SDK once, at launch:

```swift
import CosmoteID

CosmoteIDManager.sharedInstance.configure(
    channel: "<your channel>",
    enviroment: .production
)
```

`enviroment` is spelled this way on purpose. Use the channel and environment you were given.

Present the sign-in:

```swift
CosmoteIDSignIn.start(forViewController: self, completionBlock: { user in
    // user is the signed-in UserProfile
}, cancelBlock: {
    // the user left without signing in
})
```

If you pass no `cancelBlock`, leaving without signing in calls `completionBlock` with `nil`.

A SwiftUI app embeds `CosmoteIDSignIn.makeViewController(completionBlock:)` instead, and removes
it in the completion block.

## Customizing

Everything visual is set through one `Theme`, passed to `configure()`. Anything you leave unset
keeps the SDK default.

```swift
let theme = Theme()
theme.id_sdk_backgroundGradientStart = UIColor(red: 0.13, green: 0.23, blue: 0.29, alpha: 1)
theme.id_sdk_backgroundGradientEnd = .black
theme.id_sdk_basicText = .white
theme.id_sdk_basicAccent = UIColor(red: 0.24, green: 0.32, blue: 0.40, alpha: 1)
theme.id_sdk_focusBorder = .white
theme.id_sdk_strokeWidthThick = 4      // the focus ring
theme.id_sdk_cornerRadius = 48
theme.id_sdk_buttonHeight = 96
theme.bodyFontSize = 32
theme.regularFontName = "YourBrand-Regular"
theme.boldFontName = "YourBrand-Bold"

CosmoteIDManager.sharedInstance.configure(
    channel: "<your channel>",
    enviroment: .production,
    theme: theme
)
```

**Tokens, and properties by component.** A token (`id_sdk_*`) sets a value everywhere it is
used. A component property, such as `primaryButtonCornerRadius`, sets one place, and wins over
the token. Start with the tokens and use component properties for exceptions. Some component
properties exist for compatibility with the iOS SDK and change nothing on tvOS. `configure()`
prints to the Xcode console the colour properties you set that have no effect, and any font
name that does not resolve.

**Changing the theme later.** Call `CosmoteIDManager.sharedInstance.reapplyTheme(newTheme)`.
Do not call `configure()` again: it starts the SDK over and signs the user out.

**Sizes are in points, and the SDK's defaults are phone sizes.** A tvOS screen is 1920 points
wide, so set your TV sizes explicitly. If you have an Android TV design in dp, double it.

**Focus is the only cue for where the user is.** Set `id_sdk_focusBorder` and `id_sdk_focusFill`
explicitly rather than letting them follow the accent. `id_sdk_strokeWidthThick` is the width
of the focus ring.

**Dark mode.** Give the theme dynamic colours, for example
`UIColor { $0.userInterfaceStyle == .dark ? .white : .black }`, and they follow the system
appearance. Pass `darkTheme:` only if your dark palette is a separate set of fixed colours.

**Fonts** are PostScript names, as for `UIFont(name:)`, of fonts your app bundles. They are often
different from the file names. `CosmoteIDManager.sharedInstance.availableFontNames` lists the
names your app can use. A name that does not resolve is printed by `configure()`, and the
system font is used.

**Images.** The tvOS screens use your `closeButtonImage`, `backButtonImage` and
`checkMarkButtonImage` when you set them. You can also set these with
`SdkImages.set(.close, image)`, `.back` and `.checkmark`. The other image properties are not
used on tvOS.

**Language.** `CosmoteIDSignIn.start` and `makeViewController` take `language:` (`.el` or `.en`),
and `showsLanguageSwitch:` shows the flag switch on the login screen.
`SdkText.setLanguage(_:)` sets the language for every screen.

**Text.** The SDK's text is not customizable on tvOS in this version.

**The QR code** is always black on white, so that a phone camera can read it.

**Every property.** In Xcode, Jump to Definition on `Theme` shows all of them. Autocomplete on
a `Theme` value lists them too.

## Updating

Each version is a tag in this repository. Change the version in Xcode, or in your
`Package.swift`, and resolve again.

## License

Proprietary. The Cosmote ID SDK is the property of Desquared, and you may use it only with
Desquared's written permission. This repository being public does not grant any right to use
it. See [LICENSE](LICENSE).
