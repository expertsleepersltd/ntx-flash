# macOS Developer ID signing and notarization

The release workflow builds `ntx-flash` as an Intel/Apple Silicon universal
binary, signs it with Expert Sleepers' Developer ID, installs it into a signed
flat package, submits that package to Apple's notary service, and staples the
resulting ticket before publishing it.

The package installs the executable at:

```text
/usr/local/bin/ntx-flash
```

Installation is system-wide and therefore prompts the user for administrator
authorization. The stable package receipt identifier is
`com.expert-sleepers.ntx-flash`, allowing later releases to be treated as
upgrades.

## Required Apple account access

The setup must be performed for the Expert Sleepers Apple Developer team by an
Account Holder or another team member with permission to create Developer ID
certificates and App Store Connect API keys.

Do not use an unrelated developer's certificates. The Developer ID shown by
Gatekeeper should identify Expert Sleepers as the publisher, and Expert
Sleepers should retain control of certificate rotation and notarization.

## 1. Create the Developer ID certificates

The workflow needs both of these certificates, including their private keys:

- **Developer ID Application** signs the `ntx-flash` Mach-O executable.
- **Developer ID Installer** signs the `.pkg` delivered to users.

These are not the similarly named Mac App Store distribution certificates.

On a trusted Mac:

1. Open **Keychain Access**.
2. Choose **Keychain Access > Certificate Assistant > Request a Certificate
   From a Certificate Authority**.
3. Enter the Apple Developer account email and a recognizable common name such
   as `Expert Sleepers ntx-flash releases`.
4. Select **Saved to disk**, then save the certificate signing request (CSR).
5. Open [Certificates, Identifiers & Profiles][apple-certificates] and press
   **+**.
6. Under **Software**, choose **Developer ID Application**, upload the CSR,
   generate the certificate, and download it.
7. Repeat the process for **Developer ID Installer**. The same CSR may be used.
8. Double-click both downloaded `.cer` files to add them to the login keychain.
9. In Keychain Access, select **login > My Certificates** and confirm each
   certificate expands to show its private key.

Confirm that both identities are usable:

```bash
security find-identity -v | grep -E 'Developer ID (Application|Installer)'
```

The output must contain one `Developer ID Application: ...` identity and one
`Developer ID Installer: ...` identity for the Expert Sleepers team.

## 2. Export the certificates as one PKCS#12 archive

In **Keychain Access > login > My Certificates**:

1. Select the Expert Sleepers **Developer ID Application** and **Developer ID
   Installer** certificate entries.
2. Choose **File > Export Items**.
3. Select **Personal Information Exchange (`.p12`)** as the format.
4. Save the file as `expert-sleepers-developer-id.p12`.
5. Protect it with a new strong export password. This password is needed by CI;
   it is not the Apple ID password.

Encode the archive as one base64 line and copy it to the clipboard:

```bash
base64 -i ~/path/to/expert-sleepers-developer-id.p12 | tr -d '\n' | pbcopy
```

Keep the `.p12` and its password in the team's secure credential storage. Do
not commit the archive to Git.

## 3. Create an App Store Connect API key for notarization

A team API key avoids storing an Apple ID password in GitHub.

1. Sign in to [App Store Connect][app-store-connect].
2. Open **Users and Access > Integrations > App Store Connect API > Team
   Keys**.
3. If API access has not been enabled, the Account Holder must first request or
   enable it.
4. Generate a key named `ntx-flash GitHub notarization` with the least-privileged
   role the team permits for notarization (the Developer role is normally
   sufficient).
5. Record the displayed **Issuer ID** and **Key ID**.
6. Download the `AuthKey_<KEY_ID>.p8` file. Apple permits this private key to be
   downloaded only once.

Encode the downloaded private key and copy it to the clipboard:

```bash
base64 -i ~/Downloads/AuthKey_<KEY_ID>.p8 | tr -d '\n' | pbcopy
```

Keep the original `.p8` in secure credential storage. Do not commit it to Git.

## 4. Add the five GitHub Actions secrets

In the `expertsleepersltd/ntx-flash` repository, open **Settings > Secrets and
variables > Actions > Secrets**, then create these repository secrets:

| Secret | Exact value to enter |
| --- | --- |
| `MACOS_CERTIFICATES_P12_BASE64` | Clipboard output from base64-encoding `expert-sleepers-developer-id.p12` |
| `MACOS_CERTIFICATES_P12_PASSWORD` | Password chosen when exporting the `.p12` |
| `APP_STORE_CONNECT_API_KEY_P8_BASE64` | Clipboard output from base64-encoding `AuthKey_<KEY_ID>.p8` |
| `APP_STORE_CONNECT_API_KEY_ID` | **Key ID** displayed for the App Store Connect API key |
| `APP_STORE_CONNECT_API_ISSUER_ID` | **Issuer ID** displayed on the App Store Connect API page |

Do not add quotes, spaces, or explanatory text around the values. The two
base64 values should each be a single line.

GitHub does not expose repository secrets to workflows running from pull
requests made by forks. Configure the secrets in the upstream repository and
test this workflow after merging it.

## 5. Test before publishing a new version

The workflow runs for version tags and through **Actions > Build and Release >
Run workflow**. A manual run checks out and rebuilds the latest existing tag,
which permits testing the signing pipeline without creating a new tag.

A successful macOS job must complete all of these checks:

1. The imported archive contains both required Developer ID identities.
2. `codesign --verify --strict` accepts the universal executable.
3. `pkgutil --check-signature` accepts the signed installer.
4. Apple's notary service returns `Accepted`.
5. `stapler validate` finds the attached notarization ticket.
6. `spctl --assess --type install` accepts the finished package.

Download the macOS workflow artifact to a separate Mac and perform a clean
installation test:

```bash
sudo installer -pkg ./ntx-flash-<version>-macos.pkg -target /
command -v ntx-flash
test "$(command -v ntx-flash)" = /usr/local/bin/ntx-flash
ntx-flash --help
```

The same package can also be opened in Finder to test the normal Installer UI.

Inspect its signatures with:

```bash
pkgutil --check-signature ./ntx-flash-<version>-macos.pkg
xcrun stapler validate -v ./ntx-flash-<version>-macos.pkg
spctl --assess --type install --verbose=4 ./ntx-flash-<version>-macos.pkg
```

## What the workflow does with the credentials

The macOS matrix job:

1. Builds the `arm64` and `x86_64` executables and combines them with `lipo`.
2. Decodes the `.p12` into the ephemeral runner and imports it into a temporary
   keychain.
3. Discovers the `Developer ID Application` and `Developer ID Installer`
   identity names from that keychain.
4. Signs the executable with hardened runtime and a secure timestamp.
5. Creates a flat package that installs the signed executable to
   `/usr/local/bin/ntx-flash`.
6. Signs the package with the Developer ID Installer identity and a secure
   timestamp.
7. Decodes the App Store Connect `.p8` key and submits the package with
   `xcrun notarytool`.
8. Prints Apple's notarization log when a submission is rejected.
9. Staples and validates the accepted ticket, then asks Gatekeeper to assess
   the package.
10. Deletes the temporary keychain and decoded credentials even if a preceding
    step fails.

The private key material is never uploaded as a workflow artifact.

## Rotation and troubleshooting

### Certificate expiration or replacement

Generate replacement Developer ID Application and Installer certificates,
export a new combined `.p12`, then replace both `MACOS_CERTIFICATES_*` secrets.
Secure timestamps allow already shipped packages to remain valid after the
signing certificate expires.

### `Developer ID ... identity not found`

The `.p12` is missing one certificate or its private key. Reopen Keychain
Access and confirm both entries appear under **My Certificates** and expand to
show a private key before exporting them together again.

### `No identity found for ...`

Confirm the archive contains **Developer ID Installer**, not **3rd Party Mac
Developer Installer** or **Mac Installer Distribution**. Those certificates
are for Mac App Store delivery and are not interchangeable with Developer ID
Installer.

### Notarization authentication failure

Confirm that the API key has not been revoked and that the Key ID and Issuer ID
were copied from the same App Store Connect team key. If an Individual API Key
is used instead, `notarytool` requires different arguments; this workflow is
intentionally configured for a Team API Key.

### Notarization rejection

The workflow retrieves Apple's JSON log whenever a submission ID is available.
Read the reported path and architecture, correct the signature or packaging
problem, and rerun the workflow. Do not bypass notarization or publish an
unstapled replacement under the same release filename.

### Removing a test installation

The package installs one executable. Remove it with:

```bash
sudo rm -f /usr/local/bin/ntx-flash
sudo pkgutil --forget com.expert-sleepers.ntx-flash
```

[apple-certificates]: https://developer.apple.com/account/resources/certificates/list
[app-store-connect]: https://appstoreconnect.apple.com/access/integrations/api
