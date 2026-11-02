# Your requirements

How each requirement in your IoT9 multi-lock list is met, with the calls involved and the page where the flow is drawn in full.

| # | Requirement | Status |
|---|---|---|
| [R-1](#r-1) | Add multiple locks to a property | <span class="chg chg-new">supported</span> |
| [R-2](#r-2) | Add multiple gateways to a property | <span class="chg chg-new">supported</span> |
| [R-3](#r-3) | One gateway serves several locks | <span class="chg chg-unchanged">already supported</span> |
| [R-4.1](#r-41) | A user's passcode can differ on each lock | <span class="chg chg-new">supported</span> |
| [R-4.2](#r-42) | A user can keep the same passcode on several locks | <span class="chg chg-new">supported</span> |
| [R-4.3](#r-43) | A user's RFID card can differ on each lock | <span class="chg chg-new">supported</span> |
| [R-4.4](#r-44) | A user can use the same RFID card on several locks | <span class="chg chg-new">supported</span> |
| [R-4.5](#r-45) | A scheduled user's passcode differs on each lock | <span class="chg chg-changed">not possible</span> |
| [R-4.6](#r-46) | A user can have a different schedule on each lock | <span class="chg chg-new">supported</span> |
| [R-4.7](#r-47) | A one time passcode works only on its own lock | <span class="chg chg-unchanged">already supported</span> |
| [R-5](#r-5) | Disarm the anti-theft alarm with any valid access | Lock firmware |

## R-1. Multiple locks in a property { #r-1 }

**Supported.** Add each lock exactly as you add the first one today: `addLock` creates the access point with

```
POST /infrastructureManagement/internal/v2/networks/{networkId}/accessPoints
```

What changes is what happens once the access point exists: the owner's credentials are **bound** to the new lock, and every lock after the first gets its **own passcode credential**. See [Lock Onboarding, step 4](lock-onboarding.md#4-customise-the-lock).

## R-2. Multiple gateways in a property { #r-2 }

**Supported.** A gateway belongs to a **network**, and each site has one network. For every additional gateway:

1. Create another site in the same organisation. It comes with its own network:
    ```
    POST /infrastructureManagement/internal/v1/organisations/{orgId}/sites
    ```
2. Wait for that site's and network's `resource_alive`, as in [Lock Onboarding, step 1](lock-onboarding.md#1-choose-a-property).
3. Add the gateway to that network as you do today.
4. Create the locks this gateway serves in **that** network (`POST .../networks/{networkId}/accessPoints`).

Your property then groups several sites: keep, for each lock, the `networkId` it was created in. People and their credentials belong to the organisation, so one person can be bound to locks in any of these sites.

## R-3. One gateway, several locks { #r-3 }

**Already supported**, up to **6 locks per gateway**. Create each lock in the gateway's network with the same `POST .../networks/{networkId}/accessPoints`. Plan a further gateway (R-2) beyond six.

## R-4. Several locks, one gateway

### R-4.1. A different passcode on each lock { #r-41 }

**Supported.** Each passcode belongs to its own **passcode credential**, and a credential is bound to the locks it may open. Give the person a separate passcode credential for each lock:

- on their first lock, the passcode credential issued with their accessor
- on every later lock, a new one from `POST …/accessors/{accessorId}/credential/create` with `credentialType: passcode`

Bind it to that lock, then set its passcode there. A passcode is never copied between locks, so changing one never changes another. See [Lock Onboarding, step 4](lock-onboarding.md#4-customise-the-lock) and [User Management](user-management.md#how-a-method-on-a-lock-maps-to-spintly).

### R-4.2. The same passcode on several locks, by choice { #r-42 }

**Supported.** Keep one passcode credential per lock, as in R-4.1, and let the person set the **same digits** on each lock. They remain separate passcodes: if the person later changes the passcode on one lock, the others keep theirs. See [Lock Settings, My Access](lock-settings.md).

We recommend this over binding one passcode credential to several locks: separate credentials let each lock's passcode be changed on its own later, with nothing to untangle.

### R-4.3. A different RFID card on each lock { #r-43 }

**Supported.** A card is scanned and assigned **at a lock**, and is bound to that lock only:

```
assignCardToAccessorWithPermission(serialNumber, orgId, accessorId, accessPointId,
    PermissionMode.CREDENTIAL_PERMISSION, true, callback)
```

Assign a different card at each lock. See [User Management, step 5](user-management.md).

### R-4.4. The same RFID card on several locks { #r-44 }

**Supported.** Scan the same card at each further lock with the same call, passing that lock's `serialNumber` and `accessPointId`. The card must be assigned to the **same person** on every lock: a card that already belongs to someone else fails with `7/54`.

### R-4.5. A scheduled user's passcode on each lock { #r-45 }

**Not possible.** A scheduled user's passcode is generated by Spintly (`passcode/generate`), and a scheduled user has **one** generated passcode, valid on every lock they are scheduled on. Their schedules can still differ from lock to lock (R-4.6). See [User Management, Scheduled](user-management.md#scheduled).

### R-4.6. A different schedule on each lock { #r-46 }

**Supported.** One `schedules/user/assign` request can carry a different schedule for each lock, one entry per schedule:

```json
{
  "assign": [
    { "scheduleId": 11, "accessorIds": [107968], "accessPointIds": [4567] },
    { "scheduleId": 12, "accessorIds": [107968], "accessPointIds": [4568] }
  ]
}
```

The person must have a credential bound on each lock first. A person can also have **several schedules on one lock**, set with `schedules/user/composite` and combined by `mergePolicy`, and each schedule can have up to 5 groups of days with up to 4 times each. See [Spintly API changes](api-changes.md#schedulesuserassign).

### R-4.7. One time passcodes are lock specific { #r-47 }

**Already supported, unchanged.** A one time passcode is activated at one lock when it is generated (`otp/generate`, then `otp/activate` for that lock), and works on that lock only. See [User Management, One time](user-management.md#one-time).

## R-5. Disarming the anti-theft alarm { #r-5 }

**Provided in the lock firmware.** Any valid access (passcode, fingerprint, RFID card, app over BLE, or NFC) turns off the anti-theft alarm raised by the door magnet, with the door open or closed. Nothing changes in the app or in the Spintly calls.
