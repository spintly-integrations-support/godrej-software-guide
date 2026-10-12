# Reading these pages

Every flow page here is built the way yours are: numbered steps, one diagram per step, read top to bottom. The participants, their colours and the platform tabs are the same as on your site, so a diagram here can be laid next to the one it replaces.

## The participants

The same participants as your site, with the same colour in every diagram. Every arrow is drawn in the colour of the participant it leaves from.

| Participant | What it is |
|---|---|
| <span class="p-key p-user"></span>User | The person holding the phone. Named by role where the role matters, such as **Owner or primary** and **Invited user** |
| <span class="p-key p-app"></span>App | The iOS or Android app |
| <span class="p-key p-backend"></span>Binaryveda's backend | Your backend. The only participant that calls Spintly's servers |
| <span class="p-key p-spintly"></span>Spintly's servers | Spintly's REST APIs |
| <span class="p-key p-kafka"></span>Kafka | Spintly's message bus. Spintly publishes to it and your backend reads from it |
| <span class="p-key p-oauth"></span>OAuth SDK | Trades the Keycloak token for a Spintly session token. Unchanged |
| <span class="p-key p-access"></span>Access SDK | Holds the app credential and the lock permissions, and performs unlocks. Unchanged |
| <span class="p-key p-config"></span>Config SDK | Writes to the lock hardware. **Moves to v3** |
| <span class="p-key p-hardware"></span>Lock hardware | The lock, an accessory, or a gateway, reached over BLE |

## The badges

Each step heading carries one badge.

| Badge | Meaning |
|---|---|
| <span class="chg chg-new">new</span> | A step, call or message that does not exist in your flow today |
| <span class="chg chg-changed">changed</span> | The step exists, but a call, a parameter, an order or a rule in it changes |
| <span class="chg chg-unchanged">unchanged</span> | Nothing to do. Still drawn in full, so the page reads end to end |

Inside a diagram, an arrow whose label begins with **NEW** or **CHANGED** is the part of the step that differs from your diagram. Every other arrow is as it is today.

## Credentials and binding

Two words carry most of the change.

**Credential.** Something a person opens a door with: app access, a passcode, a fingerprint or a card. Each has a `credentialId`. A person holds exactly **one app credential**, and any number of the others.

**Binding.** A credential is *bound* to a lock when it may open that lock. Binding is what replaces today's "give the user permissions on the lock". You bind with one of two calls, depending on which side you start from:

| Starting from | Call |
|---|---|
| One credential, several locks | `PATCH .../accessors/{accessorId}/credentials/{credentialId}/permissions` |
| One lock, several credentials or people | `PATCH .../accessPoints/{accessPointId}/credential-permissions` |

Both are described in [Spintly API changes](api-changes.md).

!!! info "App access follows every binding"

    Bind the app credential to every lock the person should open from the app. Binding any other credential to a lock also lets the person open that lock from the app. App access to a lock ends only when the person has no credential left on it.

## Platform tabs

Diagrams that differ between platforms are split into **iOS** and **Android** tabs, linked across the site, as on yours. Where the change is the same on both, there is one diagram and no tabs.

Config SDK v3 signatures are shown in their Android form. The iOS members are expected to take the same arguments in the same order, and are listed in [Still to be confirmed](pending.md) until they are published.

## Where the request bodies are

The arrows into Spintly's servers carry the method and the path. The request and response bodies of every new or changed endpoint are in [Spintly API changes](api-changes.md). Endpoints that do not change are not repeated.
