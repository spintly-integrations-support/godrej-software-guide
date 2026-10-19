# Still to be confirmed

Everything on this site that may still change, and where it appears. Each is being confirmed on Spintly's side, and this page will be updated as each one closes.

| Item | What is open | Pages |
|---|---|---|
| Config SDK v3 names | Members the SDK guide marks planned or as an open point: the lock-only scan, `deleteFingerprintTemplateOverBle`, `getBatteryStatus`, the gateway software calls, `remoteResetDevice`, `resetGatewayOverBle`, and the firmware rename to `updateToFirmwareVersion` (next major version) | [Lock Onboarding](lock-onboarding.md), [User Management](user-management.md), [Lock Settings](lock-settings.md), [Config SDK v3](config-sdk-v3.md) |
| iOS signatures | The iOS members are expected to take the same arguments in the same order as Android | The same |
| Fingerprint enrolment and access | Whether a finger can be enrolled before the fingerprint credential is bound to the lock. Until confirmed: **bind first, then enrol** | [Lock Onboarding, step 8](lock-onboarding.md#8-fingerprint-and-rfid), [Lock Settings](lock-settings.md#my-access-fingerprints) |
| Access SDK | That no Access SDK member changes | [User Onboarding](user-onboarding.md), [Home](home.md), [Lock Control Panel](lock-control-panel.md), [Notifications](notifications.md) |
| `addAllAccessPoints` | That `addAllAccessPoints: true` on the accessor call binds the issued credentials to every lock in the property, as `accessPointsToAdd` does for the locks it lists | [Spintly API changes](api-changes.md#creating-an-accessor-or-adding-one-to-an-organisation) |
| New response fields | Final field names of the two new `GET` calls, and of the credentials returned by the access settings call | [Spintly API changes](api-changes.md#reading-the-locks-a-credential-is-on) |
| Moving existing properties | The steps and timing for moving properties created before the change | [Profile and Account](profile-and-account.md#properties-that-already-exist) |
