# Config SDK v3

Every Config SDK member your pages use today, and what replaces it. The OAuth SDK and the Access SDK do not change.

!!! warning "Provisional"

    The v3 names below are from Spintly's Config SDK guide for the Android release `com.spintly:configurationsdkv3:0.0.0.84-7263534`. Members the guide marks **planned** or as an **open point** are marked the same way here, and may still change. They are tracked in [Still to be confirmed](pending.md). The iOS members take the same arguments in the same order as the Android ones shown.

## What changes across the board

- **The lock is found by its `serialNumber`.** Every call that talks to a device takes it first. The SDK finds the device over BLE by itself, so every explicit connect and close member goes: `scanAndConnectFingerprintDevice`, `scanAndConnectCardDevice`, `fingerprintClose`, `nfcEnrollmentClose`, `nfcEnrollmentStopScan`, `meshConfigurationClose`.
- **Passcode and fingerprint calls take a `credentialId`**, right after the `serialNumber`. It names the credential the passcode or the finger belongs to, which is how a lock can have a passcode of its own.
- **`initiateSdk()` runs once after `setAuthToken`.** It downloads the device feature list the other calls rely on.
- **One call at a time.** A second call, or a call during a scan, fails with `5/26`. Start the next call from inside the previous one's callback.
- **Errors are `domain/code`**, on `SpintlyCFServiceException`. `getRequestId()` gives the id to quote to Spintly support.

## Setup

| Today | v3 |
|---|---|
| `getDefaultInstance(context)`, `setEnvironment`, `setAuthToken` | The same, then **`initiateSdk()`** |

A newer setup, `setConfiguration`, `setTokenRefreshHandler`, `logIn`, `logOut`, `isLoggedIn` and `updateAccessToken`, is planned and not needed yet.

## 2. Lock Onboarding

| Step | Today | v3 | Notes |
|---|---|---|---|
| 3 Scan | `configurableDeviceStartScan`, `configurableDeviceStopScan`, `ConfigurableDeviceListener.onDeviceListUpdated` | `startConfigurableDevicesScan(callback)`, `stopScan(callback)`, `SpintlySdkScanCallback.onConfigurableDeviceListUpdated(Set<ScannedBleDevice>)` | `getSerialNumber()` can be null. A lock-only scan, `startConfigurableDevicesScan(SpintlyDeviceType.LOCK, callback)`, is an open point |
| 5 Provisioning | `startDeviceMeshConfiguration`, plus `meshConfigurationClose` on Android | `configureDevice(serialNumber, SpintlyCompletionAndFirmwareUpdateStatusCallback)` | Updates old firmware first. Already configured is `2/56` |
| 6 Firmware | `getListOfFirmwareForSerialNumber`, `firmwareUpdateToSelectedVersion` | `firmwareUpdate(serialNumber, callback)`, then `updateToFirmwareVersion(updateToVer, FirmwareUpdateCallback)` | `updateToVer` is `getPdSwVer()` from the list. It replaces today's `firmwareUpdateToSelectedVersion`. Or `configureDeviceWithMinimumFirmware(serialNumber, minimumFirmwareVersion, callback)` for steps 5 and 6 together |
| 7 Master passcode | iOS `updateMasterPasscode`, Android `generateMasterPasscode`, `(serial, orgId, accessorId, old, new, callback)` | `generateMasterPasscode(serialNumber, credentialId, orgId, accessorId, oldPasscode, newPasscode, KeypadManagerCallback)` | Duplicate passcode is `14/4` |
| 8 Fingerprint | `fingerprintClose`, `scanAndConnectFingerprintDevice(orgId, accessPointId, 1)`, `performFPEnrollmentOnDevice(orgId, accessPointId, 1, accessorId, name, 60, callback)` | `performFPEnrollmentOnDevice(serialNumber, credentialId, orgId, accessorId, templateName, 19, FingerprintManagerCallback)` | 19 second timeout. Pass the person's fingerprint credential. Bind it first and wait for `in_sync` |
| 8 RFID | `nfcEnrollmentStopScan`, `scanAndConnectCardDevice(orgId, accessPointId, 1)`, `assignCardToAccessorWithPermissionOnDevice(true, orgId, accessorId, 1, 1, true, callback)`, `nfcEnrollmentClose` | `assignCardToAccessorWithPermission(serialNumber, orgId, accessorId, accessPointId, PermissionMode.CREDENTIAL_PERMISSION, true, CardManagerCallback)` | `onCardAssigned(credentialId)`. The card is bound to `accessPointId` |

## 5. User Management

| Step | Today | v3 |
|---|---|---|
| 4 Set passcode | `addUserPasscode(serial, orgId, accessorId, passcode, callback)` | `addUserPasscode(serialNumber, credentialId, orgId, accessorId, passcode, KeypadManagerCallback)` |
| 5 Fingerprint and RFID | As Lock Onboarding | As Lock Onboarding |

## 7. Lock Settings

| Entry | Today | v3 | Notes |
|---|---|---|---|
| My Access, passcode | `addUserPasscode`, iOS `updateUserPasscode` / Android `updatePasscode`, iOS `updateMasterPasscode` / Android `generateMasterPasscode` | `addUserPasscode`, `updatePasscode`, `generateMasterPasscode`, each with `credentialId` after `serialNumber` | |
| My Access, fingerprints | `getFingerprintPermissions`, over BLE. `deleteFingerprintTemplatePermission`, from the lock | `getFingerprintPermissions(orgId, accessorId)`, from the server. `deleteFingerprintTemplatePermission(orgId, accessorId, templateDataId)`, at the server. **New:** `deleteFingerprintTemplateOverBle(serialNumber, organisationId, targetAccessorId, templateDataId, callback)`, from the lock now and then the server | Use the Bluetooth delete when the phone is next to the lock, the server delete when it is away |
| Sound | `readSoundSettings`, `updateLockSoundSettings` | Same names, `(serialNumber, …)` | 0 to 5 |
| Lights | `readLEDIntensitySettings`, `updateLEDIntensitySettings` | Same names, `(serialNumber, …)` | 1 to 5 |
| Timeout alerts | `readTimeoutAlertSettings`, `updateTamperAlarmSettings`, `updatePrankAlarmSettings`, `updateDOTLSettings` | Same names, `(serialNumber, …)` | 3 to 30 **seconds** |
| Lock audio and door bell | `getCurrentLanguageAndDoorbellTone`, `updateVoicePackage`, `setDoorbellTone` | `getCurrentLanguageAndDoorbellTone`, `updateVoicePackage(serialNumber, …)`, **`updateDoorbellTone`** | 220 KB, MD5 |
| Accessories | `beaconStartScan`, `setBeaconTargetDevice`, `beaconStopScan`, `startBeaconAttach`, `getListOfFirmwareForBeacon`, `resetBeacon` | `startBleBeaconScan`, `stopScan`, `attachBeacon(serialNumberOfDevice, scannedBleDevice, callback)`, `firmwareUpdate(scannedBleDevice, callback)` + `updateToFirmwareVersion`, `resetBeacon(scannedBleDevice, callback)`. Factory-fresh beacons first need the force update: `startDfuDevicesScan`, `firmwareUpdateForDfuBeacon`, `updateToFirmwareVersionForDfuBeacon` | Already paired is `2/36` |
| Firmware update | As Lock Onboarding step 6 | As Lock Onboarding step 6 | |
| Factory reset | `resetDevice` | `resetDevice(serialNumber, callback)`; **new** `remoteResetDevice(serialNumber, callback)` resets from the cloud through the gateway | Try `resetDevice` first, `remoteResetDevice` when Bluetooth cannot reach the lock |
| Gateway Wi-Fi | `getWifiNetworks`, `configureWifiNetwork` | Same names, `(serialNumber, …)` | |
| Gateway software | `getListOfSoftwareForGatewaySerialNumber`, `gatewaySoftwareUpdateToSelectedVersion` | `getListOfSoftwareForGatewaySerialNumber(serialNumber, callback)`, `gatewaySoftwareUpdateToSelectedVersion(serialNumber, updateToVersion, callback)` | Open point in the SDK guide |
| Battery | None | **New:** `getBatteryStatus(serialNumber, callback)`, percent 0 to 100 | Read from the lock over BLE. Depends on the lock's firmware |
| Gateway reset | `resetGateway`, over BLE | **`resetGateway(serialNumber, resetBluetooth, resetWifi, callback)`, through the server.** **New:** `resetGatewayOverBle(serialNumber, resetBluetooth, resetWifi, callback)` resets over BLE, to try first | `resetGateway` now means the server route |

## Error codes worth handling

| Code | Meaning | Where |
|---|---|---|
| `2/56` | The device is already configured | Provisioning |
| `2/36` | The accessory is already paired | Accessories |
| `5/26` | Another call or a scan is already running | Any |
| `6/46` | The person has no access on this lock's door | Fingerprint enrolment |
| `7/55` | `ACCESSOR_PERMISSION` used where the person has no access on the lock | RFID. Use `CREDENTIAL_PERMISSION` |
| `14/4` | The passcode is already in use on the lock | Any passcode write |
| `14/2` | The lock rejected the passcode length. Enforce at least 4 digits | Any passcode write |
| `14/6` | The lock's passcode memory is full | Any passcode write |
| `10/14`, `10/15` | Passcode longer than 12 digits, or not digits only | Any passcode write |
| `6/13` | The person already has the maximum number of fingerprints | Fingerprint enrolment |
| `5/22` | The device was not found within 30 seconds | Any |
