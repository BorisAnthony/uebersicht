# Übersicht: macOS 27 Golden Gate Spaces thumbnail fix

Investigation and local validation: October 6, 2026.

On macOS 27, Übersicht widgets appeared in the desktop thumbnails at the top of
Mission Control. Switching widget windows from `Stationary` to `Transient` on
macOS 27 and later resolved the reported issue. Widgets hide while Mission
Control is open and return afterward. Earlier macOS versions retain the existing
window behavior.

The fix is committed as `b0721e04` (`Hide widget windows from Mission Control on
macOS 27`) on branch `macos27-spaces-fix`.

## The reported problem

The user observed widget content inside the Spaces thumbnails after upgrading
to macOS 27 (Golden Gate). This had not occurred on their previous macOS version.
The user does not use interactive widgets and accepted a change to widget
visibility during Mission Control.

These thumbnails are composed by macOS from native windows. The relevant control
is the app's `NSWindow` configuration, rather than the React rendering code,
widget CSS, or Node server.

## Code governing the behavior

[Uebersicht/UBWindow.m](Uebersicht/UBWindow.m) configures every widget window with
three collection behaviors. Before the fix, they were:

```objc
NSWindowCollectionBehaviorStationary |
NSWindowCollectionBehaviorCanJoinAllSpaces |
NSWindowCollectionBehaviorIgnoresCycle
```

| Behavior | Purpose |
| --- | --- |
| `Stationary` | Keeps the window visible and stationary during Mission Control, like the desktop. |
| `CanJoinAllSpaces` | Allows the window to appear across desktops. |
| `IgnoresCycle` | Excludes the window from keyboard window cycling. |

Apple's description of `Stationary` does not guarantee exclusion from Spaces
thumbnails. Our working explanation is that macOS 27 changed how those previews
include stationary windows. We did not isolate the exact internal WindowServer
change; the confirmed result is that changing the window policy fixes the
reported symptom.

The repository history explains the original choice. Commit `fb34bb8`, dated
September 17, 2020, replaced `Transient` with `Stationary` specifically to keep
windows in place during Exposé.

[Uebersicht/UBWindowGroup.m](Uebersicht/UBWindowGroup.m) creates the window layers
for each display. With interaction enabled, there are foreground and background
windows; with interaction disabled, there is one agnostic window. All use
`UBWindow`, so the fix covers each mode.

`UBWindow` also controls their levels: foreground windows sit just below normal
application windows, while background and agnostic windows sit at desktop level.
These levels remain unchanged.

The existing `disableSnapshotRestoration` call is unrelated to Mission Control
capture. It disables snapshots used for restoring saved application state.

## The source change

The initializer now selects the Mission Control policy according to the running
macOS version:

```objc
NSWindowCollectionBehavior missionControlBehavior =
    NSWindowCollectionBehaviorStationary;
if (@available(macOS 27.0, *)) {
    missionControlBehavior = NSWindowCollectionBehaviorTransient;
}
[self setCollectionBehavior:(
    missionControlBehavior |
    NSWindowCollectionBehaviorCanJoinAllSpaces |
    NSWindowCollectionBehaviorIgnoresCycle
)];
```

`Transient` is documented to hide a window during Mission Control. It replaces
`Stationary`; the two behaviors are mutually exclusive and must not be combined.

This intentionally gives up widget visibility on the full-size desktop while
Mission Control is open. It preserves the settings for appearing across Spaces
and avoiding window cycling. Interaction was not removed or disabled, even
though the user does not rely on it.

Only `UBWindow.m` and its explanatory comment changed in the fix commit. Widget
rendering, server code, preferences, and release build settings were untouched.

## Local build and launch issues

Producing a test app exposed separate build and signing problems. These were
independent of the Mission Control policy.

### Dependencies and build environment

The checkout initially lacked the JavaScript build tools. We installed the
dependencies in `server/` and `server/release/`, then built the application with
Xcode using local ad hoc signing.

The machine's build tools were Node 26.9.0 and npm 11.19.1. The app's bundled
arm64 runtime reported Node 16.1.0. The main [README](README.md#building-übersicht)
still describes Node 8 support, so its Node setup instructions predate the
bundled runtime in this checkout. We did not replace or relink system Node.

A personal npm `allow-scripts` setting conflicted with the older release build
scripts. We used an isolated npm user configuration (`/dev/null`) for the local
build and a temporary dependency cache. Personal npm settings were unchanged.

The native CocoaPods dependencies were built separately. Their old macOS 10.8
deployment target was overridden to 12.0 for this build, and their libraries were
made available to the app linker. These were command-line build adjustments,
not changes to the checked-in project.

### Signature verification was insufficient

The first test app built successfully and passed `codesign --verify`, but failed
at launch. The crash report showed that hardened-runtime library validation
rejected the bundled Sparkle framework because the process and library did not
have compatible signing identities.

After repairing the app's local signature, the app stayed open but its widget
server could not load `fsevents.node`. The bundled Node runtime hit the same
library-validation restriction.

The [README's Xcode section](README.md#building-in-xcode) already warns about
local signing problems and recommends removing the framework-signing build
script. It also points to an App Transport Security workaround for local HTTP.
The app's plist already allows arbitrary loads; that setting did not cause the
observed startup failure.

For this test bundle, we re-signed Sparkle locally and signed the app and both
bundled Node executables without hardened-runtime flags. The final repair used
the equivalent of the following commands, run from the repository root:

```sh
TEST_APP='build/Uebersicht-Mission-Control-Test.app'

codesign --force --deep --sign - "$TEST_APP/Contents/Frameworks/Sparkle.framework"
codesign --force --sign - --options 0 --entitlements Uebersicht/Uebersicht.entitlements "$TEST_APP/Contents/Resources/node-arm64"
codesign --force --sign - --options 0 --entitlements Uebersicht/Uebersicht.entitlements "$TEST_APP/Contents/Resources/node-x64"
codesign --force --sign - --options 0 --entitlements Uebersicht/Uebersicht.entitlements "$TEST_APP"
codesign --verify --deep --strict "$TEST_APP"
```

These commands describe the local testing workaround, not a release-signing or
distribution procedure. The signing adjustments exist only in the generated
test app and are excluded from the fix commit.

## Validation and outcome

Validation was performed on macOS 27.0, build `26A428`:

- `UBWindow.m` passed a compiler syntax check with ARC, modules, and a macOS 12
  deployment target.
- The full Debug application built successfully in Xcode.
- After the signing repair, the bundled Node runtime successfully loaded
  `fsevents`.
- The test app and its Node server remained running, and the local `/state/`
  endpoint returned HTTP 200.
- The user confirmed that the test build resolved the Mission Control issue.
- `git diff --check` passed before the source fix was committed.

The test app was placed at
`build/Uebersicht-Mission-Control-Test.app`, without replacing the installed app.
The `build/` directory is ignored by Git. The branch was subsequently renamed
from `sol-checkout` to `macos27-spaces-fix`.

## Testing again and remaining limits

1. Quit the installed Übersicht before opening the test copy.
2. Launch `build/Uebersicht-Mission-Control-Test.app`.
3. Invoke Mission Control and inspect the Spaces thumbnails.
4. Exit Mission Control and confirm widgets return.
5. Switch desktops and confirm widgets still appear where configured.

To return to the installed version, quit the test copy and reopen the installed
app. No installation replacement is required.

The exact OS compositing change remains unconfirmed. Earlier macOS releases,
interactive widgets, multiple displays, fullscreen Spaces, and the complete
Show Desktop/Stage Manager behavior matrix were not comprehensively tested.
The version guard preserves the earlier source policy, but does not substitute
for testing those environments.

## AppKit references

- [Stationary window behavior](https://developer.apple.com/documentation/appkit/nswindow/collectionbehavior-swift.struct/stationary?language=objc)
- [Transient window behavior](https://developer.apple.com/documentation/appkit/nswindow/collectionbehavior-swift.struct/transient)
- [Window collection behaviors](https://developer.apple.com/documentation/appkit/nswindow/collectionbehavior-swift.struct)
