# Spintly API changes

Every Spintly endpoint and Kafka message that is new or behaves differently, with its request and response. Endpoints not listed here are unchanged.

All Permission Management paths are under `/permissionManagementV3`, all Credential Management paths under `/credentialManagementV3`, and both use your integrator token as today.

## Summary

| | Endpoint or message | Used in |
|---|---|---|
| <span class="chg chg-changed">changed</span> | Kafka `resource_alive`, now for sites, networks and access points too | Lock Onboarding 1, 4 |
| <span class="chg chg-new">new</span> | Kafka `accessor_ready` | Lock Onboarding 1, User Management 3 |
| <span class="chg chg-new">new</span> | Kafka `resource_unalive`: a deleted resource is gone, for every resource | Lock Settings 8, Profile and Account 4 |
| <span class="chg chg-changed">changed</span> | `PATCH …/organisations/{orgId}/accessSettings`: set once per organisation, now with `dualAuth` and `adminRole` | Lock Onboarding 1 |
| <span class="chg chg-changed">changed</span> | `POST /credentialManagementV3/v1/accessors` and `POST …/organisations/{orgId}/accessors/{accessorId}`: credentials returned, and bound to `accessPointsToAdd` | Lock Onboarding 1, 4, User Management 3 |
| <span class="chg chg-new">new</span> | `GET …/accessors/{accessorId}/credential` | Wherever a `credentialId` is needed again |
| <span class="chg chg-new">new</span> | `POST …/accessors/{accessorId}/credential/create` | Lock Onboarding 4, User Management, Lock Settings |
| <span class="chg chg-new">new</span> | `PATCH …/accessors/{accessorId}/credentials/{credentialId}/permissions` | Lock Onboarding 8, Lock Settings |
| <span class="chg chg-new">new</span> | `GET …/accessors/{accessorId}/credentials/{credentialId}/permissions` | Lock Settings |
| <span class="chg chg-new">new</span> | `PATCH …/accessPoints/{accessPointId}/credential-permissions` | Lock Onboarding 4, User Management, Lock Settings |
| <span class="chg chg-new">new</span> | `GET …/accessPoints/{accessPointId}/credential-permissions` | User Management, Lock Settings |
| <span class="chg chg-changed">changed</span> | `PATCH …/accessor/{accessorId}/accessSettings`: `dualAuth` and `adminRole`, and returns the credentials of the types it enables | Lock Onboarding 1 and 8, User Management 1 and 6 |
| <span class="chg chg-changed">changed</span> | `POST …/passcode/generate`: request unchanged; the person's passcode credential must be bound to the lock first | User Management 1 |
| <span class="chg chg-changed">changed</span> | `POST …/schedules/user/device` and `PATCH …/schedules/user/device/{scheduleId}`: schedule type, up to 5 sections with up to 4 times each | User Management 1, 6 |
| <span class="chg chg-changed">changed</span> | `POST …/schedules/user/assign`: needs credentials on the lock; one schedule per person per lock, a new one replaces it | User Management 1, 6 |
| <span class="chg chg-new">new</span> | `POST …/schedules/user/composite`: several schedules for one person on a lock, combined by `mergePolicy` | User Management 1, 6 |
| <span class="chg chg-changed">changed</span> | `PATCH …/accessors/{accessorId}/permissions` and `PATCH …/accessPoints/{accessPointId}/permissions`: **refused** | Replaced everywhere |

## Kafka

### `resource_alive`

Sent once per resource when Spintly has finished creating it. Mark **each** resource synced on its own message.

```json
{
  "messageVersion": 1,
  "messageData": {
    "requestId": "ahds-akjshak-dsbdk-skjdskd",
    "msgType": "resource_alive",
    "dataVersion": 1,
    "data": { "resourceId": 4567, "resourceName": "access_points" }
  }
}
```

| `resourceName` | `resourceId` is | Gates |
|---|---|---|
| `organisations` | `organisationId` | Creating the owner's accessor |
| `sites` | `siteId` | Adding a lock to the property, together with the network |
| `networks` | `networkId` | Adding a lock to the property, together with the site |
| `gateways` | the gateway's id | |
| `access_points` | `accessPointId` | Binding credentials to the lock |

### `accessor_ready`

Sent when an accessor exists in an organisation and can have credentials bound.

```json
{
  "messageVersion": 1,
  "messageData": {
    "requestId": "ahds-akjshak-dsbdk-skjdskd",
    "msgType": "accessor_ready",
    "dataVersion": 1,
    "data": { "accessorId": 107968, "orgId": 123 }
  }
}
```

**For both messages:** handle them idempotently, since the same message can arrive twice, and ignore a `msgType` you do not know rather than retrying it.

### `resource_unalive`

The counterpart of `resource_alive`: sent once per resource when it has been deleted and every Spintly service has let go of it. `resourceName` takes the same values: `organisations`, `sites`, `networks`, `gateways`, `access_points`. Same envelope:

```json
{
  "messageVersion": 1,
  "messageData": {
    "msgType": "resource_unalive",
    "dataVersion": 1,
    "data": { "resourceId": 4567, "resourceName": "access_points" }
  }
}
```

Remove the resource on your side when it arrives, and handle it idempotently. See [Resources and permissions](resources.md#readiness-and-removal-signals).

## Credentials

### Creating an accessor, or adding one to an organisation

`POST /credentialManagementV3/v1/accessors` and `POST /credentialManagementV3/v1/organisations/{orgId}/accessors/{accessorId}` are the calls you make today. Two things change in what they do:

- **They return the person's credentials**, one per access type enabled in the request's access settings. Store every `credentialId`.
- **Every credential issued in the call is bound to each lock in `accessPointsToAdd`.** That is how a person gets their first lock in a property: app access, passcode, and fingerprint if enabled, in one call. `addAllAccessPoints: true` does the same for every lock in the organisation.

```json
{
  "type": "success",
  "message": {
    "credentials": [
      {
        "integratorUniqueId": "ff766cf7-4182-4327-8898-bf2b947ec658",
        "accessorId": 107968,
        "integratorId": 123,
        "credentialTypes": [
          { "type": "mobile_ble_nfc", "credentialId": 900000011, "format": "32_bit", "status": 1 },
          { "type": "passcode",       "credentialId": 700000001, "format": "32_bit", "status": 1 },
          { "type": "fingerprint",    "credentialId": 44389500,  "format": "32_bit", "status": 1 }
        ]
      }
    ]
  }
}
```

Credentials issued **later** are not bound to anything until you bind them: a type enabled afterwards on access settings, or a credential from `credential/create`.

### Reading a person's credentials

```
GET /credentialManagementV3/v1/organisations/{orgId}/accessors/{accessorId}/credential
    ?credentialType=passcode      optional, omit for all
```

```json
{
  "type": "success",
  "message": {
    "orgId": 123,
    "accessorId": 107968,
    "credentials": [
      { "type": "passcode",    "credentialId": 700000001, "format": "32_bit", "status": 1 },
      { "type": "passcode",    "credentialId": 700000002, "format": "32_bit", "status": 1 },
      { "type": "fingerprint", "credentialId": 44389500,  "format": "32_bit", "status": 1 }
    ]
  }
}
```

A person can hold several credentials of one type, so key on `credentialId`, never on `type`. Use it whenever you need a `credentialId` you did not store, for example after a failed write on your side.

### Creating another credential

```
POST /permissionManagementV3/v1/organisations/{orgId}/accessors/{accessorId}/credential/create
```

```json
{ "credentialType": "passcode" }
```

```json
{
  "type": "success",
  "message": { "orgId": 123, "accessorId": 107968, "credentialType": "passcode", "credentialId": 700000002 }
}
```

Use it for the passcode of every lock after the person's first: each lock needs its own passcode credential. Scheduled users are the exception: they keep one passcode credential, bound to all their locks. It does not create app or fingerprint credentials: a person has exactly one of each, issued with the accessor or when the type is first enabled. Cards are not created here, they are assigned at the lock by the Config SDK.

There is no limit on how many passcodes a person can hold. Apply any limit you want in your backend, before calling.

## Binding credentials to locks

Two calls do the same thing from different sides. Use whichever matches what you have in hand. Both are idempotent: adding a binding that exists, or removing one that does not, does nothing.

### One credential, several locks

```
PATCH /permissionManagementV3/v1/organisations/{orgId}/accessors/{accessorId}/credentials/{credentialId}/permissions
```

```json
{ "accessPointsToAdd": [4567, 4568], "accessPointsToRemove": [4569] }
```

```json
{ "type": "success", "message": "success" }
```

The credential type is worked out from the `credentialId`. A `400` means the lock does not support that type, for example a passcode on a lock with no keypad. A credential can be bound to several locks, but a **passcode is never copied between locks**: use a fresh passcode credential for each lock, see [Lock Onboarding, step 4](lock-onboarding.md#4-customise-the-lock).

### One lock, several credentials or people

```
PATCH /permissionManagementV3/v1/organisations/{orgId}/accessPoints/{accessPointId}/credential-permissions
```

```json
{
  "credentialsToAdd": [
    { "accessorId": 107968, "credentialId": 900000011 },
    { "accessorId": 107968, "credentialId": 700000002 }
  ],
  "credentialsToRemove": [
    { "accessorId": 107970 }
  ]
}
```

An entry in `credentialsToRemove` with **only an `accessorId`** removes **everything** that person has on this lock: app access, passcodes, fingerprints and cards. Use it to take someone off a lock. An entry with a `credentialId` removes that credential only.

### Reading the locks a credential is on

```
GET /permissionManagementV3/v1/organisations/{orgId}/accessors/{accessorId}/credentials/{credentialId}/permissions
```

```json
{
  "type": "success",
  "message": {
    "credentialId": 700000001,
    "credentialType": "passcode",
    "accessPoints": [
      { "accessPointId": 4567, "status": "in_sync" },
      { "accessPointId": 4568, "status": "add_pending" }
    ]
  }
}
```

Use it to see which locks an app or fingerprint credential is on, and whether each binding is in sync. Passcode credentials are one per lock, so they are on a single lock.

### Reading what is on a lock

```
GET /permissionManagementV3/v1/organisations/{orgId}/accessPoints/{accessPointId}/credential-permissions
    ?accessorId=107968      optional, omit for everyone on the lock
```

```json
{
  "type": "success",
  "message": {
    "accessPointId": 4567,
    "credentials": [
      { "accessorId": 107968, "credentialId": 900000011, "credentialType": "mobile_ble_nfc", "status": "in_sync" },
      { "accessorId": 107968, "credentialId": 700000001, "credentialType": "passcode",       "status": "add_pending" }
    ]
  }
}
```


### When a binding reaches the lock

`200` means Spintly has **recorded** the change. It then reaches the lock, and the person's app, in the background. It is usually quick, but do not treat the response as proof the lock has it.

Both `GET` calls above return a `status` for each binding:

| `status` | Meaning |
|---|---|
| `add_pending` | Added, still on its way to the lock |
| `delete_pending` | Removed, still being taken off the lock. Until it is gone, the credential may still open the lock |
| `in_sync` | The lock and the person's app match what you asked for |

A binding you removed stays in the list as `delete_pending` until the lock has dropped it, and then disappears.

**Wait for `in_sync` before you write to the lock for that credential.** The Config SDK checks with Spintly before it writes a passcode or enrols a finger, and refuses while the credential is not yet on the lock (for a fingerprint, `6/46`). So, after binding a passcode or fingerprint credential to a lock:

1. Call `GET …/accessPoints/{accessPointId}/credential-permissions?accessorId={accessorId}`.
2. Find the entry for that `credentialId`. If it is `add_pending`, call again after a short wait, every 2 seconds as your `getLock` polling does.
3. Once it is `in_sync`, call the SDK: `generateMasterPasscode`, `addUserPasscode` or `performFPEnrollmentOnDevice`. Call `passcode/generate` only then as well.
4. **Stop after 60 seconds.** If it is still `add_pending` by then, tell the user the lock is not ready yet and offer to try again. A binding that takes this long is a fault on Spintly's side, which Spintly retries and is alerted to; there is nothing for your app to fix.

A credential that was already on the lock, such as a passcode being changed in My Access, is already `in_sync`, so there is nothing to wait for.

### App access follows every binding

The app credential behaves slightly differently from the others. Binding **any** credential to a lock also gives the person app access to that lock, and app access ends only when nothing of the person's is left on it. Bind the app credential itself for anyone who should open the lock from the app, even if they have nothing else on it.

## Access settings

### The organisation's

```
PATCH /permissionManagementV3/v1/organisations/{orgId}/accessSettings
```

```json
{ "mobile": true, "passcodeAccess": true, "fingerprint": true, "card": true, "tapToAccess": true, "dualAuth": true, "adminRole": true }
```

What the organisation allows, set once after the organisation's `resource_alive`. A type turned off here is off for everyone in the organisation. `dualAuth` and `adminRole` are new: each works for a person only when it is on both here and on the person. `GET` on the same path reads the values. See [Resources and permissions](resources.md#organisation).

### A person's

```
PATCH /permissionManagementV3/v1/organisations/{orgId}/accessor/{accessorId}/accessSettings
```

```json
{ "adminRole": true, "dualAuth": false }
```

Two fields join the person's access settings. Both apply to the **person across the organisation**, to every credential they have on every lock, and only while the same field is on in the organisation's access settings:

| Field | Meaning |
|---|---|
| `adminRole` | Admin role, set on the owner. Today set per lock on the permission call |
| `dualAuth` | Dual authentication. Today set per lock on the permission call |

They can also be sent in the access settings of the accessor create and add-to-organisation calls. The other access settings, such as `passcodeAccess`, `fingerprintAccess`, `cardAccess` and `tapToAccess`, keep their meaning: enabling a type for the first time issues its credential.

**The response now returns the person's credentials** for every type the request enables, whether the credential was created by this call or already existed. A type that cannot be issued by Spintly, such as a card, is not included. For example, turning on fingerprint for someone whose accessor was created without it:

```json
{ "fingerprintAccess": true }
```

```json
{
  "type": "success",
  "message": {
    "credentials": [
      { "credentialType": "fingerprint", "credentialId": 44389500 }
    ]
  }
}
```

Each entry carries the `credentialType` and the `credentialId`. Store the `credentialId`, then bind it. `credentials` is empty when the request enables no credential type, for example one that only changes `dualAuth`.

## Passcodes and schedules

### `passcode/generate`

```
POST /permissionManagementV3/v1/organisations/{orgId}/passcode/generate
```

```json
{ "accessorId": 107968, "size": 6 }
```

The request is unchanged: no `credentialId`. A scheduled user has **one** passcode credential, and the generated passcode belongs to it. Two rules are new:

- **Bind it first.** The person's passcode credential must be bound to the lock before you generate.
- **One passcode credential only.** Generating for a person who holds no passcode credential, or more than one, fails with `400`. Do not issue a scheduled user a second passcode credential with `credential/create`.

Generate once, on the person's first lock, and bind the same passcode credential to every further lock: the passcode works there without generating again. Generating again changes it on **every** lock it is bound to.

### Creating a schedule: `schedules/user/device`

Schedules can now do more than one date range, one set of days and one time slot.

```
POST /permissionManagementV3/v1/organisations/{orgId}/schedules/user/device
```

```json
{
  "name": "Cleaner",
  "scheduleType": "per_day",
  "dateRange": { "start": "2026-11-01", "end": "2026-12-31" },
  "sections": [
    {
      "repeatType": "weekly",
      "daysOfWeek": [1, 2, 3],
      "triggers": [
        { "state": "allow", "time": "09:00" },
        { "state": "deny",  "time": "13:00" },
        { "state": "allow", "time": "14:00" },
        { "state": "deny",  "time": "18:00" }
      ]
    },
    {
      "repeatType": "weekly",
      "daysOfWeek": [4, 5],
      "triggers": [
        { "state": "allow", "time": "10:00" },
        { "state": "deny",  "time": "16:00" }
      ]
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `scheduleType` | `per_day`: access follows the sections' times. `exclusive`: access **24 hours a day** within `dateRange`; no `sections` |
| `dateRange` | The schedule works only between `start` and `end` |
| `sections` | Up to **5**, in the same format as gateway managed schedules, with `repeatType: "weekly"`. Each covers some days of the week, and a day belongs to at most one section. A day in no section has no access |
| `triggers` | Up to **4** per section, in time order. `allow` opens access from that time, `deny` closes it. Two pairs give two windows in a day |

`daysOfWeek` uses the same day numbers as today. **The times come only from the triggers**: there is no `timeRange` any more. Today's single window is one section with one `allow` and one `deny`.

**Editing** with `PATCH …/schedules/user/device/{scheduleId}` works as for gateway managed schedules: `sectionsToAdd`, `sectionsToUpdate` (each with the section `id` from the `GET`) and `sectionsToDelete` (section ids), plus `name`, `dateRange` and `scheduleType`. An edit applies on every lock and to every person the schedule is assigned to.

**Lock firmware.** Several sections, more than two times a day, and `exclusive` need lock firmware that supports them. A lock on older firmware takes only **one section with one `allow` and one `deny`**, the single window you set today. **Assigning any other schedule to such a lock is refused** with `400`, and nothing is changed on the lock. **Check the lock's `firmwareVersion` before offering more**: the versions that support advanced schedules will be shared with you, see [Still to be confirmed](pending.md).

### `schedules/user/assign`

The request is unchanged. Two rules are new:

- **The person must have a credential on the lock first.** Otherwise the call fails with `400 "No credentials are bound to this access point for this user"`.
- **One schedule per person per lock.** Assigning a schedule to a person on a lock where they already have one **replaces** it, including several set with [`schedules/user/composite`](#several-schedules-on-one-lock-schedulesusercomposite). Different locks can carry different schedules, or share the same `scheduleId`.

The schedule applies to everything the person has on that lock, app access included, and to anything bound there later. `unassign` removes **every** schedule the person has on that lock, and leaves their credentials there **without a time limit**, see [User Management](user-management.md#schedules).

### Several schedules on one lock: `schedules/user/composite`

New. Gives one person **several schedules on the same lock**, which the lock combines.

```
POST /permissionManagementV3/v1/organisations/{orgId}/schedules/user/composite
```

```json
{
  "assign": [
    { "accessorId": 107968, "accessPointIds": [4567], "scheduleIds": [11, 12], "mergePolicy": "or" }
  ]
}
```

| Field | Meaning |
|---|---|
| `accessorId` | **One** person per entry. Send one entry per person |
| `accessPointIds` | The locks this applies to |
| `scheduleIds` | The **complete** list of the person's schedules on those locks. It replaces whatever they had there, one schedule or several |
| `mergePolicy` | How the schedules combine where they overlap. `or`: access whenever **any** of them allows it. `and`: access only when **all** of them allow it. Default `or` |

The same rules as `schedules/user/assign` apply: the person needs a credential on each lock first, and a schedule beyond one section with one `allow` and one `deny` is refused with `400` on a lock with older firmware.

| To | Call |
|---|---|
| Add a schedule to the ones a person has on a lock | `composite` with the existing `scheduleIds` plus the new one |
| Drop one of several | `composite` with the remaining `scheduleIds` |
| Drop them all | `schedules/user/assign` with an `unassign` entry for that person and lock |

## Refused on the new model

| Endpoint | Instead |
|---|---|
| `PATCH …/accessors/{accessorId}/permissions` | `accessPointsToAdd` on the accessor call, or `PATCH …/accessPoints/{accessPointId}/credential-permissions` |
| `PATCH …/accessPoints/{accessPointId}/permissions` | `PATCH …/accessPoints/{accessPointId}/credential-permissions` |

Both answer `400` for a property on the new model. Properties that existed before the change stay on the old model and keep using them, see [Profile and Account](profile-and-account.md#properties-that-already-exist).
