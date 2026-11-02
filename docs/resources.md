# Resources and permissions

Every Spintly resource your backend creates, what it is ready on, the rules that apply to it, and how it is removed. The flow pages show these calls in context; this page is the one place they are listed together.

## How your objects map to Spintly

| Your app | Spintly | Created |
|---|---|---|
| The account's first property | **Organisation**, with its first **site** and **network** | `createSite`, first branch ([Lock Onboarding, step 1](lock-onboarding.md#1-choose-a-property)) |
| Every further property | A **site** and its **network** in the same organisation | `createSite`, second branch |
| Wi-Fi gateway | **Gateway**, in a network | `createGateway` ([Lock Settings, The Wi-Fi Gateway](lock-settings.md#the-wi-fi-gateway)) |
| Lock | **Access point**, in a network | `addLock` ([Lock Onboarding, step 4](lock-onboarding.md#4-customise-the-lock)) |
| Person | **Accessor** | Accepting an invite ([User Management](user-management.md)) |
| App access, passcode, fingerprint, card | **Credential** of that accessor | Issued with the accessor, by access settings, by `credential/create`, or assigned at the lock (card) |
| "This person can open this lock with this method" | **Binding**: a credential on an access point | The two binding calls ([Spintly API changes](api-changes.md#binding-credentials-to-locks)) |

## Readiness and removal signals

Spintly confirms each resource on Kafka. Act on a resource only once its signal has arrived, and handle every signal idempotently: the same message can arrive more than once.

| Resource | Ready | Removed |
|---|---|---|
| Organisation | `resource_alive`, `resourceName: "organisations"` | `resource_unalive`, `resourceName: "organisations"` |
| Site | `resource_alive`, `resourceName: "sites"` | `resource_unalive`, `resourceName: "sites"` |
| Network | `resource_alive`, `resourceName: "networks"` | `resource_unalive`, `resourceName: "networks"`, when its site is deleted |
| Gateway | `resource_alive`, `resourceName: "gateways"` | `resource_unalive`, `resourceName: "gateways"` |
| Access point | `resource_alive`, `resourceName: "access_points"` | `resource_unalive`, `resourceName: "access_points"` |
| Accessor | `accessor_ready`, `{ accessorId, orgId }` | The `DELETE` response, see [Person](#person) |

`resource_unalive` uses the same envelope as `resource_alive`, and is sent once a deleted resource is gone from every Spintly service. Remove the resource on your side only then:

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

## Organisation

**Create.** With the account's first property, `POST /infrastructureManagement/internal/v1/organisations`. Wait for its `resource_alive` before creating the owner's accessor.

**Set what the organisation allows**, once it is ready:

```
PATCH /permissionManagementV3/v1/organisations/{orgId}/accessSettings
```

```json
{
  "mobile": true,
  "passcodeAccess": true,
  "fingerprint": true,
  "card": true,
  "tapToAccess": true,
  "dualAuth": true,
  "adminRole": true
}
```

An access type turned **off** here is off for **everyone** in the organisation, whatever their own access settings say. `GET` on the same path reads the current values. Turn on every type your app offers, and `dualAuth` and `adminRole` too: each of those works for a person only when it is on both here and on that person (see [Admin role and dual authentication](#admin-role-and-dual-authentication)).

**Delete.** `DELETE /infrastructureManagement/internal/v1/organisations/{orgId}`, only once everything in it is gone, in this order. Wait for each resource's `resource_unalive` before deleting the next level:

1. People: remove every accessor from the organisation
2. Locks: factory reset and delete every access point
3. Gateways: reset and delete every gateway
4. Sites: delete every site (each takes its network with it)
5. The organisation, confirmed by its own `resource_unalive`

## Site and network

**Create.** `POST /infrastructureManagement/internal/v1/organisations/{orgId}/sites`. The site comes with **its own network**; there is no separate network call. It is ready when **both** the site's and the network's `resource_alive` have arrived, and no lock can be added before that.

**One gateway per site.** A gateway lives in a network, and each site has one. For a second gateway, create a second site; see [Your requirements, R-2](requirements.md#r-2).

**Delete.** `DELETE /infrastructureManagement/internal/v1/sites/{siteId}`, only once its locks and gateways are deleted. Its network is deleted with it. Both send `resource_unalive`. Your app already refuses to delete a property that still holds a lock or a gateway ([Profile and Account, Manage Properties](profile-and-account.md#4-manage-properties)).

## Gateway

**Create.** `createGateway`, after handing the gateway its Wi-Fi network over BLE with `configureWifiNetwork`. One gateway serves **up to 6 locks** in its network.

**Delete.** Reset it first, with `resetGatewayOverBle`, or `resetGateway` through the server if Bluetooth fails, then `DELETE /infrastructureManagement/internal/v1/gateways/{serialNumber}`, confirmed by `resource_unalive`. Delete the gateway before its site.

## Access point (lock)

**Create.** `POST /infrastructureManagement/internal/v2/networks/{networkId}/accessPoints`, once the site and network are ready. Wait for the access point's `resource_alive`.

**Configure it before relying on it.** The lock is configured with the Config SDK, with the phone next to it, so that it reaches the network's gateway ([Lock Onboarding](lock-onboarding.md), provisioning). You can bind credentials as soon as the access point's `resource_alive` arrives, but a binding reaches the lock only once the lock is configured and online: until then it stays `add_pending`. Wait for `in_sync` before any Config SDK write that depends on it, such as the master passcode.

**Delete.** Always reset the lock first. A reset wipes everything on it, every person and credential included:

1. `resetDevice(serialNumber, callback)` over BLE, with the phone next to the lock, or, if the lock cannot be reached over Bluetooth, `remoteResetDevice(serialNumber, callback)` through its gateway
2. `DELETE /infrastructureManagement/internal/v1/accessPoints/{accessPointId}`. Every binding on the lock is removed with it
3. `resource_unalive` for the access point confirms it is gone

See [Lock Settings, step 8](lock-settings.md#8-factory-reset-and-remove-device).

## Person { #person }

**Create.**

| The person is | Call |
|---|---|
| New to Spintly | `POST /credentialManagementV3/v1/accessors` |
| Already a Spintly accessor, new to this organisation | `POST /credentialManagementV3/v1/organisations/{orgId}/accessors/{accessorId}` |

Both take the person's **access settings**, and issue **one credential for each access type turned on**. With `accessPointsToAdd`, every credential issued in that call is also bound to each of those locks. The `accessorId` identifies the person from then on. Wait for `accessor_ready` before binding anything else for them.

**Change what they may use.** `PATCH /permissionManagementV3/v1/organisations/{orgId}/accessor/{accessorId}/accessSettings`. Turning a type on for the first time issues its credential and returns it.

**Read their credentials.** `GET /credentialManagementV3/v1/organisations/{orgId}/accessors/{accessorId}/credential`.

**Remove.** From one lock: `credentialsToRemove: [{ accessorId }]` on the lock's binding call. From the organisation: `DELETE /credentialManagementV3/v1/organisations/{orgId}/accessors/{accessorId}`. This removes **everything** the person has in the organisation: every binding on every lock, and every credential. If they come back later, they get new credentials, and new passcodes have to be set. When it was the last organisation they were in, the accessor itself is deleted and the response says `isAccessorDeleted: true`.

## Credentials

| Type | How many per person | Where it comes from |
|---|---|---|
| App access | Exactly one | Issued with the accessor |
| Fingerprint | Exactly one, holding any number of fingers | Issued when fingerprint is turned on |
| Passcode | Any number: one per lock | The first with the accessor, then `credential/create` for each further lock |
| Card | Any number | Assigned at a lock with the Config SDK, never created by your backend |

Scheduled users are the exception for passcodes: they keep one passcode credential for all their locks ([User Management, Scheduled](user-management.md#scheduled)).

## Permissions: bindings

A person opens a lock with a method only when **that method's credential is bound to that lock**. There is one binding per credential per lock, and nothing is granted to the person as a whole. Bind the app credential to every lock the person should see in the app.

**An access type works on a lock only when it is allowed at all three levels:**

| Level | Where it is set | If it is off |
|---|---|---|
| Organisation | `PATCH .../organisations/{orgId}/accessSettings` | Off for everyone in the organisation |
| Person | `PATCH .../accessor/{accessorId}/accessSettings`, or the accessor create call | The person has no credential of that type to bind |
| Lock | The lock hardware: what it supports | The binding call answers `400`, for example a passcode on a lock with no keypad |

Bindings are made with the two calls in [Spintly API changes](api-changes.md#binding-credentials-to-locks), and each binding has a status, `add_pending`, `delete_pending` or `in_sync`, readable with the two binding `GET`s. A binding is on the lock once it is `in_sync`.

## Passcodes

**The master passcode** replaces the lock's factory passcode, with `generateMasterPasscode` ([Lock Onboarding](lock-onboarding.md)). Before calling it:

- passcode is allowed for the organisation and for the person
- the person has a passcode credential **bound to this lock**, and the binding is `in_sync`

**A user passcode** is set with `addUserPasscode` under the same two conditions, and changed with the update call ([Lock Settings, My Access](lock-settings.md#my-access-passcode)).

**A passcode must be unique on its lock.** Two people cannot have the same passcode on one lock. The same digits can be used again on a different lock, so a person can keep the same passcode on all their locks by choice ([Your requirements, R-4.2](requirements.md#r-42)).

## Admin role and dual authentication

Both are set **per person, for the whole organisation**, with `adminRole` and `dualAuth` in the person's access settings. Each takes effect only when it is also on in the **organisation's** access settings.

Once on, it applies to **every credential the person has on every lock**, and to every credential bound to a lock later. Unbinding a credential takes it off for that credential on that lock. You do not repeat it per lock or per credential.
