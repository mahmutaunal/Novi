# Novi Security Architecture

Last reviewed: September 8, 2026

## Trust boundary

Novi transfers user-selected notification and continuity data directly between paired Android and macOS devices on the local network. There is no Developer-operated notification relay, account service, analytics backend, or remote content archive.

The security boundary assumes the user controls both paired devices and the local network. A rooted Android device, a compromised macOS account, malicious accessibility/notification software, exposed unlocked screens, or physical access can bypass protections outside Novi’s process boundary.

## Pairing and identity

- QR pairing codes are random, expire after two minutes, and rotate after use.
- Each Android device receives a separate high-entropy pairing secret.
- Android stores paired-device records and secrets with Android Keystore-backed AES-256-GCM.
- macOS stores per-device pairing secrets and TLS private material in the data-protection Keychain as non-synchronizing, device-bound items.
- Existing global macOS pairing secrets are migrated once to per-device Keychain items and deleted from `UserDefaults`.
- Removing a device deletes its secret, queued commands, and replay state.

## Transport protection

- The Mac exposes a TLS local-network listener using a self-signed P-256 identity.
- Android pins the paired certificate’s SHA-256 fingerprint; certificate changes require explicit re-pairing approval.
- Notification payloads use AES-256-GCM in addition to TLS.
- Requests use HMAC-SHA256 over timestamp, nonce, and authenticated content.
- Mac → Android commands use an authenticated, certificate-pinned WebSocket channel. The encrypted durable command queue still requires an application ACK before removal.
- Application-layer AES-GCM nonces are generated with `SecureRandom`; the long-lived pairing key is never paired with general-purpose PRNG output.
- The Mac rejects stale timestamps, reused nonces, invalid device identities, invalid signatures, and disabled devices.
- Cleartext Android traffic is disabled at the application manifest boundary.

## Local storage

- Android application backup and device-transfer extraction are disabled and explicitly exclude every private storage domain.
- Android secure-store ciphertext uses a fresh 96-bit GCM IV and storage-key AAD, preventing ciphertext from being swapped between logical records. Legacy ciphertext is migrated after a successful authenticated read.
- Android notification titles/text and clipboard previews are encrypted field-by-field with Keystore-backed AES-GCM. Existing plaintext history is migrated transactionally, remains inside the private application database, and is never included in backup.
- macOS runs inside App Sandbox with only network client/server and user-selected file access.
- macOS notification history, clipboard previews, and queued notification actions are AES-GCM encrypted with a device-bound data-protection Keychain key. Their owner-only files are excluded from backup, have bounded retention, and support user deletion.
- Transfer staging is private application-container state with expiry and integrity validation. User-selected final exports are controlled by the user.

## Logs and diagnostics

Notification bodies, encrypted request bodies, reply text, clipboard contents, pairing secrets, purchase tokens, and transaction receipts must never be written to logs. Android verbose/debug/info calls are removed from Release by R8. Operational warning/error messages must remain content-free.

## Platform exposure

- Every Android component declares `android:exported` explicitly.
- Internal services and receivers are non-exported.
- The two exported Android system services require their platform signature permissions: notification-listener binding and Quick Settings tile binding.
- All app-created Android `PendingIntent` values are immutable.
- Android Release is non-debuggable, minified, resource-shrunk, and signed through environment-provided release credentials.
- Supported production builds verify Google Play purchase JSON against the configured Play licensing RSA public key before granting Pro. Keyless development builds may restore only `PURCHASED` records returned directly by Play Billing, while the release script rejects a missing key. Ownership is reconciled with Google Play on foreground entry and explicit restore.
- macOS enables App Sandbox and Hardened Runtime. Distribution signing credentials are not stored in Git.

## Dependencies and release supply chain

- Gradle Wrapper is protected by a pinned SHA-256 distribution checksum.
- Android dependency repositories are centrally restricted to Google Maven and Maven Central; project-level repositories and floating/SNAPSHOT versions are rejected.
- Swift packages are revision-pinned in `Package.resolved`.
- Generated packages, signing materials, provisioning profiles, store artifacts, and private keys are forbidden from Git.
- `scripts/verify-production-security.sh` enforces the security contract in both CI workflows.
- `swift-crypto` is pinned to 4.5.1 or newer in the 4.x line, which contains the fix for GHSA-8q93-f6xh-4f6f/CVE-2026-43823. Dependency review is time-bounded and does not guarantee against future disclosures or transitive risk.

## Residual risks

- Client-only billing with Play signature verification and Play Console protection remains less resistant to a fully controlled runtime than Developer API verification on a backend.
- App metadata that is not notification or clipboard content continues to rely on platform sandboxing and device-at-rest protection; Novi does not introduce a separate user passphrase.
- A malicious process operating under the same compromised OS user can access data that the operating system allows that user to access.
- Certificate pinning authenticates the paired Mac identity but does not replace secure handling of the pairing QR code.
- Dependency advisories published after the recorded audit require a new review.

## Incident response

Security reports should be sent to `contact@alpwarestudio.com` without notification content, pairing secrets, private keys, verification codes, or other personal data. A confirmed credential exposure requires rotation, affected-device re-pairing, release artifact review, and a documented disclosure decision.

## Official baselines

- Android security risks: <https://developer.android.com/privacy-and-security/risks>
- Android log disclosure: <https://developer.android.com/privacy-and-security/risks/log-info-disclosure>
- Android backup security: <https://developer.android.com/privacy-and-security/risks/backup-best-practices>
- Apple Keychain accessibility: <https://developer.apple.com/documentation/security/restricting-keychain-item-accessibility>
