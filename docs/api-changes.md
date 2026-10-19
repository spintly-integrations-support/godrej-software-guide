# Spintly API changes

Every Spintly endpoint and Kafka message that is new or behaves differently, with its request and response. Endpoints not listed here are unchanged.

All Permission Management paths are under `/permissionManagementV3`, all Credential Management paths under `/credentialManagementV3`, and both use your integrator token as today.

## Summary

| | Endpoint or message | Used in |
|---|---|---|
| <span class="chg chg-changed">changed</span> | Kafka `resource_alive`, now for sites, networks and access points too | Lock Onboarding 1, 4 |
| <span class="chg chg-new">new</span> | Kafka `accessor_ready` | Lock Onboarding 1, User Management 3 |
| <span class="chg chg-changed">changed</span> | `POST /credentialManagementV3/v1/accessors` and `POST …/organisations/{orgId}/accessors/{accessorId}`: credentials returned, and bound to `accessPointsToAdd` | Lock Onboarding 1, 4, User Management 3 |
| <span class="chg chg-new">new</span> | `GET …/accessors/{accessorId}/credential` | Wherever a `credentialId` is needed again |
| <span class="chg chg-new">new</span> | `POST …/accessors/{accessorId}/credential/create` | Lock Onboarding 4, User Management, Lock Settings |
| <span class="chg chg-new">new</span> | `PATCH …/accessors/{accessorId}/credentials/{credentialId}/permissions` | Lock Onboarding 8, Lock Settings |
| <span class="chg chg-new">new</span> | `GET …/accessors/{accessorId}/credentials/{credentialId}/permissions` | Lock Settings |
| <span class="chg chg-new">new</span> | `PATCH …/accessPoints/{accessPointId}/credential-permissions` | Lock Onboarding 4, User Management, Lock Settings |
| <span class="chg chg-new">new</span> | `GET …/accessPoints/{accessPointId}/credential-permissions` | User Management, Lock Settings |
| <span class="chg chg-changed">changed</span> | `PATCH …/accessor/{accessorId}/accessSettings`: `dualAuth` and `adminRole` | Lock Onboarding 1, User Management 6 |
| <span class="chg chg-changed">changed</span> | `POST …/passcode/generate`: `credentialId` | User Management 1 |
| <span class="chg chg-changed">changed</span> | `POST …/schedules/user/assign`: needs credentials on the lock, one schedule per lock | User Management 1, 6 |
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

## Credentials

### Creating an accessor, or adding one to an organisation

`POST /credentialManagementV3/v1/accessors` and `POST /credentialManagementV3/v1/organisations/{orgId}/accessors/{accessorId}` are the calls you make today. Two things change in what they do:

- **They return the person's credentials**, one per access type enabled in the request's access settings. Store every `credentialId`.
- **Every credential issued in the call is bound to each lock in `accessPointsToAdd`.** That is how a person gets their first lock in a property: app access, passcode, and fingerprint if enabled, in one call.

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

A person can hold several credentials of one type, so key on `credentialId`, never on `type`. Use this to read a credential created when a type was enabled on access settings, which that call does not return. For example, a person whose accessor was created without fingerprint gets their fingerprint credential only when `fingerprintAccess` is first enabled for them: read it here with `?credentialType=fingerprint`, then bind it.

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

Use it for a passcode that works only on one lock. It does not create app or fingerprint credentials: a person has exactly one of each, issued with the accessor or when the type is first enabled. Cards are not created here, they are assigned at the lock by the Config SDK.

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

The credential type is worked out from the `credentialId`. A `400` means the lock does not support that type, for example a passcode on a lock with no keypad.

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
      { "accessPointId": 4567, "scheduleId": null, "status": "applied" },
      { "accessPointId": 4568, "scheduleId": 88,   "status": "pending" }
    ]
  }
}
```

Use it to show "this passcode also opens…" before a passcode is changed.

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
      { "accessorId": 107968, "credentialId": 900000011, "credentialType": "mobile_ble_nfc", "scheduleId": null, "status": "applied" },
      { "accessorId": 107968, "credentialId": 700000001, "credentialType": "passcode",       "scheduleId": null, "status": "pending" }
    ]
  }
}
```

!!! warning "The two GET calls are new"

    The response shapes above are what is being built. Field names may still be adjusted before release, see [Still to be confirmed](pending.md).

### When a binding reaches the lock

`200` means Spintly has **recorded** the change. It then reaches the lock, and the person's app, in the background. It is usually quick, but do not treat the response as proof the lock has it.

Both `GET` calls above return a `status` for each binding:

| `status` | Meaning |
|---|---|
| `pending` | Recorded, still on its way to the lock |
| `applied` | The lock and the person's app have it |
| `failed` | It could not be applied, for example because the passcode clashes with another one already on that lock. Remove the binding, or bind a different credential |

**Wait for `applied` before you write to the lock for that credential.** The Config SDK checks with Spintly before it writes a passcode or enrols a finger, and refuses while the credential is not yet on the lock (for a fingerprint, `6/46`). So, after binding a passcode or fingerprint credential to a lock:

1. Call `GET …/accessPoints/{accessPointId}/credential-permissions?accessorId={accessorId}`.
2. Find the entry for that `credentialId`. If it is `pending`, call again after a short wait, every 2 seconds as your `getLock` polling does.
3. Once it is `applied`, call the SDK: `generateMasterPasscode`, `addUserPasscode` or `performFPEnrollmentOnDevice`. Call `passcode/generate` only then as well.

A credential that was already on the lock, such as a passcode being changed in My Access, is already `applied`, so there is nothing to wait for.

### App access follows every binding

The app credential behaves slightly differently from the others. Binding **any** credential to a lock also gives the person app access to that lock, and app access ends only when nothing of the person's is left on it. Bind the app credential itself for anyone who should open the lock from the app, even if they have nothing else on it.

## Access settings

```
PATCH /permissionManagementV3/v1/organisations/{orgId}/accessor/{accessorId}/accessSettings
```

```json
{ "adminRole": true, "dualAuth": false }
```

Two fields join the person's access settings. Both apply to the **person across the organisation**, on every lock they open from the app:

| Field | Meaning |
|---|---|
| `adminRole` | Admin role, set on the owner. Today set per lock on the permission call |
| `dualAuth` | Dual authentication. Today set per lock on the permission call |

They can also be sent in the access settings of the accessor create and add-to-organisation calls. The other access settings, such as `passcodeAccess`, `fingerprintAccess`, `cardAccess` and `tapToAccess`, keep their meaning: enabling a type for the first time issues its credential.

## Passcodes and schedules

### `passcode/generate`

```
POST /permissionManagementV3/v1/organisations/{orgId}/passcode/generate
```

```json
{ "accessorId": 107968, "size": 6, "credentialId": 700000002 }
```

`credentialId` is new and required: the passcode credential the generated digits belong to. It must already be bound to the lock. Generating again for the same credential changes its digits on **every** lock it is bound to.

### `schedules/user/assign`

The request is unchanged. Two rules are new:

- **The person must have a credential on the lock first.** Otherwise the call fails with `400 "No credentials are bound to this access point for this user"`.
- **One schedule per person per lock.** Assigning a second schedule to the same person on the same lock replaces the first. Different locks can carry different schedules, or share the same `scheduleId`.

The schedule applies to everything the person has on that lock, app access included, and to anything bound there later. `unassign` removes it, and leaves the person's credentials on the lock **without a time limit**, see [User Management](user-management.md#schedules).

## Refused on the new model

| Endpoint | Instead |
|---|---|
| `PATCH …/accessors/{accessorId}/permissions` | `accessPointsToAdd` on the accessor call, or `PATCH …/accessPoints/{accessPointId}/credential-permissions` |
| `PATCH …/accessPoints/{accessPointId}/permissions` | `PATCH …/accessPoints/{accessPointId}/credential-permissions` |

Both answer `400` for a property on the new model. Properties not yet moved keep using them, see [Profile and Account](profile-and-account.md#properties-that-already-exist).
