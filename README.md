# strudel-release-action

A GitHub Action that builds, signs, notarizes, and packages a **macOS** [strudel](https://github.com/octavore/strudel) app into a DMG.

## Prerequsites

- a paid Apple Developer Program membership
- App Store Connect API key
- **Developer ID Application** certificate

See below for instructions for obtaining the API key and certificate.

## Usage

Example of full workflow that builds a DMG on every `v*` tag push and uploads it as a release asset:

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: macos-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build and sign app
        id: strudel
        uses: octavore/strudel-release-action@v1
        with:
          apple-team-id: ${{ vars.APPLE_TEAM_ID }}
          apple-api-issuer: ${{ vars.APPLE_API_ISSUER }}
          apple-api-key: ${{ vars.APPLE_API_KEY }}
          apple-api-key-contents: ${{ secrets.APPLE_API_KEY_CONTENTS }}
          apple-certificate: ${{ secrets.APPLE_CERTIFICATE }}
          apple-certificate-password: ${{ secrets.APPLE_CERTIFICATE_PASSWORD }}

      - name: Upload DMG to release
        uses: softprops/action-gh-release@v2
        with:
          files: ${{ steps.strudel.outputs.dmg-path }}
```

`steps.strudel.outputs.dmg-path` holds the path to the built DMG, so you can also pass it to `actions/upload-artifact` or any other step that needs the file.

## Configuration

Below are the recommended conventions for repository variables and secrets. Note these need to be explicitly passed to the `strudel-release-action` like in the example above.

### Repository variables

Set these under `https://github.com/{user}/{repo}/settings/variables/actions/new`.

| Variable           | Description                      |
| ------------------ | -------------------------------- |
| `APPLE_TEAM_ID`    | Your Apple Developer Team ID.    |
| `APPLE_API_ISSUER` | App Store Connect API issuer ID. |
| `APPLE_API_KEY`    | App Store Connect API key ID.    |

To find or generate these, go to [App Store Connect > Integrations > API Keys](https://appstoreconnect.apple.com/access/integrations/api). The issuer ID is shown at the top of the page. Generate a new key (or view an existing one) to get its key ID.

The downloaded API key is stored verbatim as a secret (see `APPLE_API_KEY_CONTENTS` below).

### Repository secrets

Set these under `https://github.com/{user}/{repo}/settings/secrets/actions/new`.

| Secret                       | Description                                                                                                                      |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `APPLE_API_KEY_CONTENTS`     | Contents of the App Store Connect API key (`.p8` file), copied as-is from the file you downloaded when generating the key above. |
| `APPLE_CERTIFICATE`          | Base64-encoded Apple signing certificate (`.p12`).                                                                               |
| `APPLE_CERTIFICATE_PASSWORD` | Password used when exporting the `.p12` certificate.                                                                             |

If you don't already have a `Developer ID Application` certificate, generate one first:

1. In Keychain Access, go to **Keychain Access > Certificate Assistant > Request a Certificate from a Certificate Authority**. Enter your email, leave "CA Email Address" blank, select **Saved to disk**, and save the `.csr` file.
2. In [Certificates, Identifiers & Profiles](https://developer.apple.com/account/resources/certificates/list), click **+**, choose **Developer ID Application** (under Software), and upload the CSR from step 1. This requires an Apple Developer Program membership and Account Holder/Admin access.
3. Download the resulting `.cer` file and double-click it to install it into your login Keychain.

To get the signing certificate:

1. In Keychain Access, locate your `Developer ID Application` certificate.
2. Right-click it and select **Export**, saving it as a `.p12` file. Set a password when prompted (this is the value for `APPLE_CERTIFICATE_PASSWORD`).
3. Base64-encode the exported file and copy it to your clipboard:

   ```sh
   base64 -i /path/to/certificate.p12 | pbcopy
   ```

4. Paste the result as the `APPLE_CERTIFICATE` secret.

## Outputs

| Output     | Description            |
| ---------- | ---------------------- |
| `dmg-path` | Path to the built DMG. |

## Releasing a new version of this action

Consumers reference this action by a major version tag (e.g. `@v1`), so a release needs both a specific version tag and an updated major tag pointing at it.

1. Commit your changes to `main`.
2. Tag the commit with the new semver version and push the tag:

   ```sh
   git tag v1.2.0
   git push origin v1.2.0
   ```

3. Move the major version tag to point at the same commit and force-push it:

   ```sh
   git tag -f v1 v1.2.0
   git push origin v1 --force
   ```

4. (optional) Create a GitHub release for the new tag.
