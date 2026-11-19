# Building saml2aws with the SSO extension patch (macOS, Apple Silicon)

This builds saml2aws v2.36.19 with `saml2aws-sso.patch`. The patch lets the
`Browser` provider use a persistent Chrome profile with extensions enabled.
Jamf/MDM-managed Chrome policies then apply, including the force-installed
Microsoft Single Sign On extension.

This is an unofficial patch, not affiliated with or endorsed by
[Versent/saml2aws](https://github.com/Versent/saml2aws). saml2aws and this
patch are MIT-licensed; see `LICENSE.md`.

## Why this exists

**The goal.** You sign in to AWS through Microsoft Entra ID, which sends AWS
a SAML sign-in response. saml2aws's `Browser` provider opens a browser
window, lets you sign in to Microsoft, catches that response as it goes to
AWS, and turns it into temporary AWS credentials.

**Problem 1: the browser driver download is broken.** saml2aws controls the
browser through Playwright, a browser automation library. On first use it
downloads Playwright's driver from `playwright.azureedge.net`. Microsoft has
shut that server down, so the download fails with a 404 error. Newer
saml2aws versions have the same bug, so updating doesn't help. Step 5 below
installs the driver by hand, using Node.js and the `playwright-core` package
from their official sources.

**Problem 2: no SSO extension in the browser.** On Jamf/MDM-managed Macs,
the Microsoft Single Sign On extension is pushed to Chrome through a
configuration profile. The browser saml2aws opens can't use it, for three
reasons:

1. **Wrong browser.** By default saml2aws opens Playwright's own Chromium, a
   separate app that Chrome's managed settings don't cover.
2. **Extensions switched off.** Playwright starts every browser with
   `--disable-extensions`, which applies even to real Chrome.
3. **Throwaway profile.** saml2aws opens each login in a fresh,
   private-style session with no saved profile, where extensions don't run.

saml2aws has no setting to change points 2 and 3, so the source code has to
change.

**What the patch does.**

- It opens the browser with a saved profile on disk
  (`~/.aws/saml2aws/browser-profile-<browser-type>`) instead of a throwaway
  session.
- It stops Playwright passing the flags that disable extensions or block
  Chrome from downloading the extensions your policy installs:
  `--disable-extensions`,
  `--disable-component-extensions-with-background-pages`,
  `--disable-background-networking`, `--disable-component-update` and
  `--disable-default-apps`.
- Used with `--browser-type=chrome`, it opens your real Google Chrome. Chrome
  then picks up the MDM policy, installs the SSO extension into that
  profile, and keeps it there for later logins.

**Why it matters.**

- Sign-in can use the Mac's Entra device registration and SSO, which is
  usually what Conditional Access rules check for, such as "compliant device
  only".
- The usual alternatives, typing a password or using saml2aws's non-browser
  `AzureAD` provider, can't prove the device is managed and may be blocked.
- Logins also get quicker, because the saved profile keeps your session
  between runs.

**Long-term.** This is a workaround you'd have to maintain yourself. A
cleaner fix for a whole fleet is AWS IAM Identity Center with Entra as the
identity provider. `aws sso login` opens your normal browser, so the SSO
extension works with no patching.

## Building

Prerequisites: `git`, `curl`, and the Xcode Command Line Tools (`xcode-select --install`).
The build uses cgo, so it needs the C compiler from those tools.

## 1. Install Go

saml2aws v2.36.19 is built with Go 1.22.5.

**Option A: Homebrew**

```sh
brew install go
```

**Option B: official tarball, no admin rights needed**

```sh
mkdir -p ~/tools && cd ~/tools
curl -LO https://go.dev/dl/go1.22.5.darwin-arm64.tar.gz
# Compare with the sha256 listed at https://go.dev/dl/
shasum -a 256 go1.22.5.darwin-arm64.tar.gz
tar -xzf go1.22.5.darwin-arm64.tar.gz
export PATH="$HOME/tools/go/bin:$PATH"
```

Check it works with `go version`.

## 2. Download the saml2aws source

```sh
mkdir -p ~/src && cd ~/src
git clone --depth 1 --branch v2.36.19 https://github.com/Versent/saml2aws.git
cd saml2aws
```

## 3. Apply the patch

```sh
git apply --check /path/to/saml2aws-sso.patch   # dry run, prints nothing on success
git apply /path/to/saml2aws-sso.patch
git diff --stat                                  # expect: pkg/provider/browser/browser.go
```

If you don't have git, `patch -p1 < /path/to/saml2aws-sso.patch` works too.

The patch is written against v2.36.19. On a newer tag it may need a small
manual merge. The change is self-contained in `Authenticate()` in
`pkg/provider/browser/browser.go`.

## 4. Build

```sh
go test ./pkg/provider/browser/
go build -trimpath -ldflags "-s -w -X main.Version=2.36.19-sso-patched" \
  -o saml2aws ./cmd/saml2aws
./saml2aws --version     # -> 2.36.19-sso-patched
```

`kIOMasterPortDefault` / `SecKeychain... deprecated` warnings during the
build are harmless. Copy `./saml2aws` somewhere on your `PATH` if you like.

## 5. Install the Playwright driver (one-time)

saml2aws drives the browser through the Playwright 1.47.2 driver. The
built-in `--download-browser-driver` fetches that driver from
`playwright.azureedge.net`, which no longer exists, so it fails with a
404 error. Install the driver by hand instead. It is just Node.js plus the
`playwright-core` npm package:

```sh
D=~/Library/Caches/ms-playwright-go/1.47.2
mkdir -p "$D" && cd "$D"

# Node.js runtime (any recent LTS works; check the hash against SHASUMS256.txt)
curl -LO https://nodejs.org/dist/v20.18.0/node-v20.18.0-darwin-arm64.tar.gz
curl -sL https://nodejs.org/dist/v20.18.0/SHASUMS256.txt | grep darwin-arm64.tar.gz
shasum -a 256 node-v20.18.0-darwin-arm64.tar.gz
tar -xzf node-v20.18.0-darwin-arm64.tar.gz --strip-components=2 node-v20.18.0-darwin-arm64/bin/node

# Playwright driver package (extracts to ./package)
curl -L https://registry.npmjs.org/playwright-core/-/playwright-core-1.47.2.tgz | tar -xz

rm node-v20.18.0-darwin-arm64.tar.gz
ls   # expect: node  package
```

If you only use Google Chrome (`--browser-type=chrome`), Playwright's own
Chromium is never needed. To use it anyway, download it from the current CDN:

```sh
PLAYWRIGHT_DOWNLOAD_HOST=https://cdn.playwright.dev ./saml2aws login --download-browser-driver
```

## 6. Log in

Quit Chrome first, then run:

```sh
./saml2aws login --browser-type=chrome
```

- To make Chrome the default, add `browser_type = chrome` under your account
  section in `~/.saml2aws`.
- The first run creates `~/.aws/saml2aws/browser-profile-chrome`. Chrome
  installs the force-installed SSO extension from policy, which can take a
  few seconds. If the sign-in page loads before the extension is ready,
  close the window and run the command again.
- To check, open `chrome://policy` and `chrome://extensions` in the window
  saml2aws opens. Both should show your Jamf settings and the extension.
- Set `SAML2AWS_BROWSER_PROFILE_DIR` to use a different profile folder. Don't
  point it at your everyday Chrome profile while Chrome is running.
- The "Chrome is being controlled by automated test software" banner is
  expected.
