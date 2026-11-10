# Releases (this fork)

This fork is **not** on Google Play. It ships a sideloaded APK through GitHub Releases: pushing a
tag `ensky-vX.Y.Z` makes GitHub Actions (`.github/workflows/android-release.yml`) build and sign
`dictate-<version>-<tag>.apk` and attach it to a GitHub Release. Users install and update it with
[Obtainium](https://github.com/ImranR98/Obtainium) (or by downloading the APK directly).

A pull request or branch push never signs anything — the workflow only compiles the debug variant
and runs the unit tests. A signed release is built on tag pushes and on manual dispatch (the latter
uploads the APK as a workflow artifact instead of publishing it).

## Required repository secrets

| Secret | Meaning |
|---|---|
| `SIGNING_KEYSTORE_BASE64` | The signing keystore (`release.jks`), base64-encoded |
| `SIGNING_STORE_PASSWORD` | The keystore's store password |
| `SIGNING_KEY_ALIAS` | The key alias inside the keystore |
| `SIGNING_KEY_PASSWORD` | The key's own password |

Set them under *Repository settings → Secrets and variables → Actions*.

### Creating the keystore (one time)

`keytool` ships with any JDK. From a directory **outside the repo**:

```bash
keytool -genkeypair -v \
  -keystore release.jks \
  -alias dictate \
  -keyalg RSA -keysize 4096 -validity 10000
```

Then hand it to the workflow:

```bash
base64 -w0 release.jks > release.jks.b64     # macOS: base64 -i release.jks -o release.jks.b64
```

and paste the contents of `release.jks.b64` into `SIGNING_KEYSTORE_BASE64`.

**Keep `release.jks` safe — losing it strands every user.** Android accepts app updates only
when they are signed with the same certificate, so once the fork is on the phones, `release.jks`
(and its passwords) must be kept for the lifetime of the package id `tw.ensky.dictate` — store it
somewhere backed up (a secret manager, not the repo). If a different key is ever shipped, existing
users have to uninstall first and lose their settings. There is no Play Console to re-issue
anything.

This fork signs with its **own** key on purpose: the upstream Play upload key is not available
here, and there is no upstream Play listing behind these distributions. Building locally instead of
in CI works the same way — copy `keystore.properties.template` to `keystore.properties`, fill it
in, and `./gradlew :app:assembleRelease` produces a signed APK.

## Cutting a release

1. Set the release version in `gradle.properties`: bump `projectVersionCode` and set
   `projectVersionName=X.Y.Z` (a `-suffix` may follow, and lands in the APK's versionName but not
   in the file name — the version in the name is what comes before the dash).
2. Tag and push:

   ```bash
   git tag ensky-vX.Y.Z
   git push origin ensky-vX.Y.Z
   ```

3. The workflow runs on the runner-provided Ubuntu image, installs the exact SDK/NDK/CMake
   pinned in `gradle/`, fetches the vendored sherpa-onnx libraries and builds the Bergamot engine —
   no local setup needed — then verifies the APK signature and creates the release at
   `https://github.com/ensky-corp/DictateKeyboard/releases/tag/ensky-vX.Y.Z`.

The version does not have to match the tag (the workflow names the file from `projectVersionName`
whatever the tag says), but keep them in step: tag `ensky-v6.4.0` should be version `6.4.0`.

## Pointing Obtainium at this

In Obtainium: *Add app → Custom source / GitHub*, user `ensky-corp`, repository `DictateKeyboard`,
releases source GitHub. To only ever pick up this fork's releases and not any other tag —
and to give an eventual re-tag somewhere to go — set **Version extraction** → from the release
tag, and filter with:

- include tags matching: `ensky-v`
- APK filter matching: `dictate-v?\d+\.\d+\.\d+` (the file the workflow uploads)

Obtainium checks the package id (`tw.ensky.dictate`) and the signing certificate, so updates flow
in while the fork keeps one keystore.
