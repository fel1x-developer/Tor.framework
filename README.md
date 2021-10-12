# Tor.framework

[![Carthage Compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage) 
[![Travis CI](https://img.shields.io/travis/iCepa/Tor.framework.svg)](https://travis-ci.org/iCepa/Tor.framework)

Tor.framework is the easiest way to embed Tor in your iOS application. The API is *not* stable yet, and subject to change.

Currently, the framework compiles in static versions of `tor`, `libevent`, `openssl`, and `liblzma`:

|          |         |
|:-------- | -------:|
| tor      | 0.4.6.7  |
| libevent | 2.1.12  |
| OpenSSL  | 1.1.1l  |
| liblzma  | 5.2.5  |

## Requirements

- iOS 8.0 or later
- Xcode 7.0 or later
- `autoconf`,  `automake`,  `libtool` and  `gettext` in your `PATH`

## Installation

Embedded frameworks require a minimum deployment target of iOS 8 or OS X Mavericks (10.9).

If you use `brew`, make sure to install `autoconf`,  `automake`,  `libtool` and  `gettext`:

```
brew install automake autoconf libtool gettext
```

### Initial Setup

```bash
git clone git@github.com:iCepa/Tor.framework

cd Tor.framework

git submodule update --init --recursive

carthage build --no-skip-current
```

### Carthage

To integrate Tor into your Xcode project using Carthage, specify it in your  `Cartfile`:

```ogdl
github "iCepa/Tor.framework" "master"
```

The above method will configure Carthage to fetch and compile Tor.framework from source. 
Alternatively, you may use the following to use binary-compiled versions of Tor.framework that 
correspond to releases in GitHub:

```ogdl
binary "https://icepa.github.io/Tor.framework/Tor.json" == 406.7.2
```

For available precompiled versions, see [docs/Tor.json](docs/Tor.json). Since Tor 0.3.5.2, 
the Tor.framework release version numbers follow the format "ABB.C.X" for tor version "0.A.B.C" 
and Tor.framework release X (for that version of Tor). Note that the "BB" slot is a two-digit number, 
with a leading zero, if necessary. "305.2.1" is the first release from tor 0.3.5.2.

#### Building a Carthage Binary archive

For maintainers/contributors of Tor.framework, a new precompiled release can be generated by 
doing the following:

Ensure that you have committed changes to the submodule trees for tor, libevent, openssl, and xz.

In `Tor/version.sh`, increment the `TOR_BUNDLE_SHORT_VERSION_STRING` version per the 
format described above. Change `TOR_BUNDLE_SHORT_VERSION_DATE` to the current date. 
Commit these changes.

Also update info in `README.md`, `Tor.podspec` and `docs/Tor.json`!

Create a git tag for the version, and then 
[build + archive the framework](https://github.com/Carthage/Carthage/#archive-prebuilt-frameworks-into-one-zip-file):

```bash
carthage build --no-skip-current

carthage archive Tor
```
(This generates a `Tor.framework.zip` file in the repo.)

Then create a [release](https://github.com/iCepa/Tor.framework/releases) in GitHub which corresponds
to the tag, attach the generated `Tor.framework.zip` to the release.

Add a corresponding entry to [docs/Tor.json](docs/Tor.json), commit & push that so that it becomes 
available at https://icepa.github.io/Tor.framework/Tor.json

### Upgrading Tor

To upgrade Tor:

```bash
cd Tor/tor
git fetch
git checkout tor-0.4.6.7 # Find latest versions with git tag -l
rm -r * && git checkout . # Get rid of all autogenerated configuration files, which may not work with the newest version anymore.
git submodule update --init --recursive # Later Tor has submodules.
```

-> Test build by building `Tor-iOS` and `Tor-Mac` targets in Xcode.

Check build output in the Report Navigator. (Last tab in the left pane.)

The `tor.sh` build script will call `make show-libs`, which outputs all libraries which are created by 
the Tor build. This is echoed with a "LIBRARIES: " header. Search for that in the build output and
compare the list against the list of "Frameworks and Libraries" in the `Tor-iOS` and `Tor-Mac`
targets. Add missing ones accordingly.

The typically can be found in `~/Library/Developer/Xcode/DerivedData/Tor-`[random ID]`/Build/Products/Debug[-iphonesimulator]`.

The `project.pbxproj` file may need manual editing to set the references to the built libraries 
in a way, which is independent of your personal setup. Check other entries for how that is done.

### CocoaPods

Directly reference the provided podspec like so:

```ruby
pod 'Tor', podspec: 'https://raw.githubusercontent.com/iCepa/Tor.framework/v405.8.1/Tor.podspec'
```

You could also reference master, to always get the latest version:

```ruby
pod 'Tor', podspec: 'https://raw.githubusercontent.com/iCepa/Tor.framework/master/Tor.podspec'
```


## Usage

Starting an instance of Tor involves using three classes: `TORThread`, `TORConfiguration` and `TORController`.

Here is an example of integrating Tor with `NSURLSession`:

```objc
TORConfiguration *configuration = [TORConfiguration new];
configuration.cookieAuthentication = @(YES);
configuration.dataDirectory = [NSURL URLWithString:NSTemporaryDirectory()];
configuration.controlSocket = [configuration.dataDirectory URLByAppendingPathComponent:@"control_port"];
configuration.arguments = @[@"--ignore-missing-torrc"];

TORThread *thread = [[TORThread alloc] initWithConfiguration:configuration];
[thread start];

NSURL *cookieURL = [configuration.dataDirectory URLByAppendingPathComponent:@"control_auth_cookie"];
NSData *cookie = [NSData dataWithContentsOfURL:cookieURL];
TORController *controller = [[TORController alloc] initWithSocketURL:configuration.controlSocket];
[controller authenticateWithData:cookie completion:^(BOOL success, NSError *error) {
    if (!success)
        return;

    [controller addObserverForCircuitEstablished:^(BOOL established) {
        if (!established)
            return;

        [controller getSessionConfiguration:^(NSURLSessionConfiguration *configuration) {
            NSURLSession *session = [NSURLSession sessionWithConfiguration:configuration];
            ...
        }];
    }];
}];
```

## Known Issues

- Carthage warns about the xcconfigs dependency being seemingly unused.
  It isn't. It's only xcconfig files containing build settings, so nothing actually ends up in the build
  product. Unfortunately Carthage can't be configured to not throw this warning.

## License

Tor.framework is available under the MIT license. See the 
[`LICENSE`](https://github.com/iCepa/Tor.framework/blob/master/LICENSE) file for more info.
