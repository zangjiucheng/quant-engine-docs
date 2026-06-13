# Code signing (macOS)

How to sign the dashboard binaries. The build supports two modes:

- **Ad-hoc** (default) — every binary is signed with the system's
  ad-hoc signature. Works for CI, scripting, offline use, and
  for the Touch ID credential gate (which uses LAContext and
  needs no entitlement). This is the no-config path and what you
  want for personal development.
- **Developer ID** — every binary is signed with your Apple
  Developer ID Application certificate. Required for
  redistribution (Gatekeeper acceptance, notarization).

The Touch ID credential gate works on **any** signing
configuration, including ad-hoc. You only need Developer ID
signing if you plan to share the binary with other Macs.

The rest of this doc covers the Developer ID path.

## Quick start

```bash
# 1. Find your identity. Look for "Developer ID Application".
security find-identity -v -p codesigning

# 2. Configure with the identity + team ID.
cmake --preset=release \
  -DQE_CODESIGN_IDENTITY="Developer ID Application: Your Name (XXXXXXXXXX)" \
  -DQE_CODESIGN_TEAM_ID=XXXXXXXXXX

# 3. Build. Every binary and the .app bundle get signed.
cmake --build --preset=release -j

# 4. Verify.
codesign -dv --entitlements - build/release/bin/qe_tests
# Expected: "keychain-access-groups" array containing your team
# ID + "com.jiucheng.quant-engine".
```

## Which identity do I need?

`security find-identity -v -p codesigning` can show several
identity types. Only one works for redistribution without extra
setup:

| Identity prefix | Use case |
|---|---|
| `Developer ID Application: …` | **What you want.** Local builds plus distribution to other Macs. |
| `Apple Development: …` | iOS sideload + Mac testing through Xcode-managed profiles. Killed by AMFI (`-413 No matching profile found`) when launched without an embedded provisioning profile. |
| `Mac Developer: …` | Legacy. Same problem as Apple Development. |
| `3rd Party Mac Developer Application: …` | Mac App Store only — different sandbox model than this project uses. |

If you only have an `Apple Development` cert and try to use it,
the build succeeds but every signed binary crashes at launch
with SIGKILL because AMFI refuses to honor the entitlement
without a matching provisioning profile. This is Apple policy,
not a build-system bug — the `codesign` step ran correctly and
the entitlement is embedded; AMFI rejects the entitlement claim
at process start.

## Getting a Developer ID Application cert

If you have an Apple Developer Program membership but no
`Developer ID Application` cert yet:

1. Go to <https://developer.apple.com/account/resources/certificates/list>.
2. Click `+` → **Developer ID Application** (not Apple
   Development).
3. Generate a CSR: Keychain Access → Certificate Assistant →
   "Request a Certificate from a Certificate Authority…", saved
   to disk.
4. Upload the CSR, download the `.cer` file, double-click to
   install into your login keychain.
5. Confirm: `security find-identity -v -p codesigning` should
   now list a `Developer ID Application: …` row.

The 10-character code in parentheses at the end of the identity
string is your **Team ID**. Use it for `QE_CODESIGN_TEAM_ID`.

## What the entitlement does

`build/release/qe_entitlements.plist` (generated from
`cmake/qe_entitlements.plist.in`) claims one entitlement:

```xml
<key>keychain-access-groups</key>
<array>
    <string>XXXXXXXXXX.com.jiucheng.quant-engine</string>
</array>
```

This tells the OS keychain that binaries claiming
`com.jiucheng.quant-engine` (under your team ID) are allowed to
share keychain items in a single trust group. The dashboard
doesn't need this for the default Touch ID gate (which uses
LAContext instead), but it lets us turn on a future
OS-enforced ACL gate that would survive a debugger attached to
the dashboard process.

We deliberately do NOT claim:

- `com.apple.security.app-sandbox` — would block reads under
  `~/Documents/quant-strategy` and other user folders the
  dashboard actually wants to reach.
- `com.apple.security.cs.allow-jit` — we don't JIT anything.
- `com.apple.security.cs.disable-library-validation` — we
  statically link via vcpkg; no untrusted dylib loading needed.

## Notarization

Notarization is the step that lets a signed `.app` launch on
another Mac without Gatekeeper warnings. The signing pipeline
stops at signing; notarization is a release-time step that
requires an Apple-ID-paired keychain credential (`xcrun
notarytool store-credentials`) and a round trip to Apple's
notarization service.

A future release will add `scripts/notarize.sh` and wire it into
the release workflow. For now, a signed binary works fine on
your own machine; for distribution you'll want to notarize
manually before shipping.

## Troubleshooting

### `Failed to parse entitlements: AMFIUnserializeXML: syntax error`

The entitlements plist must not contain XML comments
(`<!-- ... -->`). The CMake template is comment-free for this
reason. If you copy a plist from somewhere else and add
comments, `codesign` will reject it with this error.

### `codesign failed: -1` / `errSecInternalError`

Usually your identity has expired or been revoked. Re-check with
`security find-identity -v -p codesigning`.

### `errSecMissingEntitlement (-34018)` at runtime

You're running an ad-hoc-signed binary (`QE_CODESIGN_IDENTITY`
empty) while some code path expects the keychain-access-groups
entitlement. Either set `QE_NO_LAUNCH_LOCK=1` to bypass for that
session, or rebuild with `QE_CODESIGN_IDENTITY=...`.

### `Disallowing <name> because no eligible provisioning profiles found`

You're using an `Apple Development` identity instead of a
`Developer ID Application` identity. See the identity table
above; the fix is to get a Developer ID Application cert from
developer.apple.com.

## See also

- [credentials-security.md](credentials-security.md) — what the
  launch gate protects, recovery paths, env-var bypass.
- `cmake/QeCodesign.cmake` — the helper module exposing
  `qe_codesign()` and `qe_codesign_bundle()`.
- `cmake/qe_entitlements.plist.in` — the entitlement template.
