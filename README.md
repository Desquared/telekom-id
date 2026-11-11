# Telekom ID (formerly COSMOTE id) SDK for tvOS

Version 4.0.0. A Swift package of prebuilt binaries: add it to your tvOS app and
`import CosmoteID`.

Telekom ID is the one account a user has across Telekom Greece's apps. On tvOS this SDK draws two
flows in your app: **sign-in** (by phone number or e-mail, with a one-time code or password, or
with a QR code the user scans with their phone) and **account activation** from an invitation
code. It keeps the session and hands your app the signed-in user. Every other account feature
(profile, addresses, cards) is done on a phone.

The module and type names (`CosmoteID`, `CosmoteIDManager`, `CosmoteIDSignIn`) keep the old
name, so code written against the iOS pod reads the same.

## Requirements

- tvOS 17 or later
- Xcode 27.2 or later
- An Apple silicon Mac. The simulator slice is arm64 only; there is no Intel slice.
- A login for `libraries.thinkdesquared.com` in `~/.netrc`, as in the next section

## Credentials

The binaries are downloaded from `libraries.thinkdesquared.com`, which needs a login. If you
already install the Cosmote ID (3.x iOS) pod, your `~/.netrc` has it and there is nothing to do.

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
choose version 4.0.0, and add the `CosmoteID` library to your tvOS target.

Or in a `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/Desquared/telekom-id", exact: "4.0.0")
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
    environment: .production
)
```

Use the channel and environment (`.production` or `.uat`) you were given. Call `configure()`
once per launch; calling it again starts the SDK over and signs the user out.

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

A user who is already signed in is signed back in by `start` without a screen: the SDK keeps the
session across launches.

## Activating an account

A user who has an invitation code activates their account here, and is signed in at the end:

```swift
CosmoteIDActivation.start(forViewController: self, code: invitationCode, completionBlock: { user in
    // user is the signed-in UserProfile; nil if no session was made
}, cancelBlock: {
    // the user left without activating
})
```

`code:` is optional: pass it when your app already has the code, and the first screen is filled
in. SwiftUI apps use `CosmoteIDActivation.makeViewController(code:completionBlock:)`.

## Trying it

Run your app on a concrete Apple TV simulator (see above) or an Apple TV, present the sign-in,
and sign in with a test account in the environment you were given. The simulator is driven with
the arrow keys and Return, or with the Apple TV Remote window (Window › Show Apple TV Remote).

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
    environment: .production,
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

**Language.** `CosmoteIDSignIn.start` and `makeViewController` take `language:` (`.el` or `.en`).
The flag switch on the login screen shows when `configure` has `supportsMultipleLanguages: true`
and `languageResponsible: .sdk`; with `.host`, the default, it is hidden.
`SdkText.setLanguage(_:)` sets the language for every screen.

**Text.** The SDK's text is not customizable on tvOS in this version.

**The QR code** is always black on white, so that a phone camera can read it.

**Every property.** In Xcode, Jump to Definition on `Theme` shows all of them. Autocomplete on
a `Theme` value lists them too.

## Coming from the 3.x iOS pod

If your iOS app already builds a `Theme` for the Cosmote ID pod, start from that code. This is
version 4 of the same SDK; the `Theme` is the same class with the same purpose.

- **Colours carry over.** The 3.x colour properties (`navigationBarTextColor`,
  `primaryButtonStartColor`, …) still exist and still work, marked deprecated, except the
  secondary button's text, border and gradient colours and `formBackgroundColor` /
  `formBorderColor`, which no token reproduces and are fully supported. The properties marked
  *new in 4.0* in the table below are supported too, with no warning. Every colour is now
  `UIColor?`, default `nil`. Only the ones in the table below are used on tvOS; `configure()`
  prints the ones you set that have no effect, and you can delete them. The shorter route is the
  `id_sdk_*` tokens above: a few tokens restyle every screen.
- **Fonts change.** The 3.x `UIFont` properties are gone (compile errors). Set the font names
  instead (`regularFontName`, `mediumFontName`, `boldFontName`, and `italicFontName`,
  `boldItalicFontName`, `lightFontName` if you bundle those faces) and the sizes by role
  (`captionFontSize`, `labelFontSize`, `bodyFontSize`, `inputFontSize`, `headingFontSize`). The
  font files must be listed in your app's `UIAppFonts`.
- **Sizes need TV values.** Your phone sizes are too small for a 1920-point TV screen: set
  the dimension tokens and font sizes for TV, as above.
- **Images.** Only `closeButtonImage`, `backButtonImage` and `checkMarkButtonImage` are used on
  tvOS. The SDK no longer ships default images on `Theme`, so a line that recoloured one of them
  does nothing.
- **Renamed or removed:** `enviroment:` is now `environment:`; `theme.applicationTitle` is
  `configure(appName:)`; `showTextFieldLine` is `showFormSeparator`; `successGradientImage` is
  `successCheckmarkImage`; `showLogoInNavigationBar` and `statusBarStyle` are gone. `Theme` is
  `final`, so a subclass becomes a function that builds and returns a `Theme`.
- **Helpers you may have used** from the pod, `UIColor(hex:)`, `UIColor(hexString:)` and
  `UIFont.registerFontWithFilenameString`, are no longer in the SDK: keep your own copy.

### Colours on tvOS, by their 3.x names

Every colour the tvOS screens read, under its iOS `Theme` property and its legacy Android (TV)
resource name, so a theme from either app can be carried over. Each one falls back to the token
in the last column when you leave it unset. A property marked *new in 4.0* had no 3.x iOS
counterpart; it carries the Android TV colour's name, so an Android TV value moves across
unchanged. Any 3.x colour property not in this table is not used on tvOS.

<!-- COLOR-TABLE:BEGIN -->
| used for | iOS `Theme` property | legacy Android (TV) colour | token it falls back to |
|---|---|---|---|
| Android TV | `tvAlertBackgroundColor` (new in 4.0) | `id_sdk_tvAlertBackgroundColor` | `id_sdk_basicBackgroundCard` |
| Android TV | `tvPrimaryHighlightedBackgroundColor` (new in 4.0) | `id_sdk_tvPrimaryHighlightedBackgroundColor` | `id_sdk_focusFill` |
| Android TV | `tvSecondaryHighlightedBackgroundColor` (new in 4.0) | `id_sdk_tvSecondaryHighlightedBackgroundColor` | `id_sdk_transparent` |
| Avatar | `avatarBackgroundColor` | `id_sdk_userAvatarBackgroundColor` | `id_sdk_avatarBackground` |
| Avatar | `userAvatarTextColor` (new in 4.0) | `id_sdk_userAvatarTextColor` | `id_sdk_basicText` |
| Backgrounds (general) | `backgroundGradientEndColor` | `id_sdk_generalBackgroundColorEnd` | `id_sdk_backgroundGradientEnd` |
| Backgrounds (general) | `backgroundGradientStartColor` | `id_sdk_generalBackgroundColorStart` | `id_sdk_backgroundGradientStart` |
| Backgrounds (general) | `lightBackgroundTextColor` (new in 4.0) | `id_sdk_lightBackgroundTextColor` | `id_sdk_basicText` |
| Borders (general) | `borderRoundedColor` (new in 4.0) | `id_sdk_borderRoundedColor` | `id_sdk_focusBorder` |
| Borders (general) | `unFocusedBorderRoundedColor` (new in 4.0) | `id_sdk_unFocusedBorderRoundedColor` | `id_sdk_basicStroke` |
| Destructive button | `destructiveButtonBackgroundColor` (new in 4.0) | `id_sdk_destructiveButtonBackgroundColor` | `id_sdk_transparent` |
| Destructive button | `destructiveButtonBorderColor` (new in 4.0) | `id_sdk_destructiveButtonBorderColor` | `id_sdk_functionalDestructive` |
| Destructive button | `destructiveButtonTextColor` (new in 4.0) | `id_sdk_destructiveButtonTextColor` | `id_sdk_functionalDestructive` |
| Error text | `inputErrorTextColor` | `id_sdk_text_error` | `id_sdk_functionalDestructive` |
| Input fields | `inputPlaceholderTextColor` | `id_sdk_formLabelHintTextColor` | `id_sdk_basicTextRecessive` |
| Input fields | `inputTitleTextColor` | `id_sdk_formLabelTextColor` | `id_sdk_basicTextRecessive` |
| Input fields | `inputTextColor` | `id_sdk_formLabelValueTextColor` | `id_sdk_basicText` |
| Input fields | `formBackgroundColor` | `id_sdk_textInputBackgroundColor` | `id_sdk_basicBackground` |
| Input fields | `formBorderColor` | `id_sdk_textInputBorderColor` | `id_sdk_basicStroke` |
| Primary button | `primaryButtonEndColor` | `id_sdk_primaryButtonBackgroundColorEnd` | `id_sdk_basicAccent` |
| Primary button | `primaryButtonStartColor` | `id_sdk_primaryButtonBackgroundColorStart` | `id_sdk_basicAccent` |
| Primary button | `primaryButtonDisabledEndColor` | `id_sdk_primaryButtonDisabledBackgroundColorEnd` | `id_sdk_stateDisabledAccent` |
| Primary button | `primaryButtonDisabledStartColor` | `id_sdk_primaryButtonDisabledBackgroundColorStart` | `id_sdk_stateDisabledAccent` |
| Primary button | `primaryButtonTextColor` | `id_sdk_primaryButtonTextColor` | `id_sdk_basicTextOnAccent` |
| Primary button | `primaryButtonDisabledTextColor` | `id_sdk_primaryDisabledButtonTextColor` | `id_sdk_stateDisabledTextOnAccent` |
| Radio buttons | `radioButtonRecyclerEntryBackground` (new in 4.0) | `id_sdk_radioButtonRecyclerEntryBackground` | `id_sdk_transparent` |
| Radio buttons | `radioButtonSelectedColor` (new in 4.0) | `id_sdk_radioButtonSelectedColor` | `id_sdk_basicAccent` |
| Radio buttons | `radioButtonUnSelectedColor` (new in 4.0) | `id_sdk_radioButtonUnSelectedColor` | `id_sdk_basicText` |
| Secondary button | `secondaryButtonGradientEndColor` | `id_sdk_secondaryButtonBackgroundColorEnd` | `id_sdk_basicBackground` |
| Secondary button | `secondaryButtonGradientStartColor` | `id_sdk_secondaryButtonBackgroundColorStart` | `id_sdk_basicBackground` |
| Secondary button | `secondaryButtonDisabledGradientEndColor` | `id_sdk_secondaryButtonDisabledBackgroundColorEnd` | `id_sdk_basicBackground` |
| Secondary button | `secondaryButtonDisabledGradientStartColor` | `id_sdk_secondaryButtonDisabledBackgroundColorStart` | `id_sdk_basicBackground` |
| Secondary button | `secondaryButtonTextColor` | `id_sdk_secondaryButtonTextColor` | `id_sdk_basicText` |
| Secondary button | `secondaryButtonDisabledTextColor` | `id_sdk_secondaryDisabledButtonTextColor` | `id_sdk_stateDisabledText` |
| Separators | `lightSeparatorColor` | `id_sdk_separatorColorEnd` | `id_sdk_basicStrokeSubtle` |
| Separators | `alternativeOptionColor` | `id_sdk_separatorColorStart` | `id_sdk_basicStrokeSubtle` |
| Text & theme basics | `textColor` | `id_sdk_baseTextColor` | `id_sdk_basicText` |
| Text / link button | `textButtonColor` | `id_sdk_linkTypeButtonTextColor` | `id_sdk_basicText` |
| Toolbar / navigation bar | `toolbarButtonColor` (new in 4.0) | `id_sdk_toolbarButtonColor` | `id_sdk_basicText` |
| Toolbar / navigation bar | `toolbarProgressColor` (new in 4.0) | `id_sdk_toolbarProgressColor` | `id_sdk_basicText` |
<!-- COLOR-TABLE:END -->

## Updating

Each version is a tag in this repository. Change the version in Xcode, or in your
`Package.swift`, and resolve again.

## License

Proprietary. The Telekom ID (formerly COSMOTE id) SDK is the property of Desquared, and you may use it only with
Desquared's written permission. This repository being public does not grant any right to use
it. See [LICENSE](LICENSE).
