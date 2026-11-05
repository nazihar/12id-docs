# The 12iD credential model

**For:** the product owners and developers of organisations that issue or verify credentials with 12iD.
**Describes:** what a 12iD credential is, how long it is valid, how you revoke or suspend it, what happens when
a user loses their phone, and how to design schemas whose values work in proofs. Read it before you design
your first schema: some of these choices cannot be changed later.

Related: the API guide ([`api.md`](api.md)), data protection ([`data-protection.md`](data-protection.md)) and
the Android SDK guide ([the Android SDK guide](android-sdk.md)).

## What a credential is

A credential is a set of named values that your organisation signs and gives to one person. For example:
name, employee number, department, valid until. The person keeps it in the wallet inside **your** app, on
their phone. Later, a verifier (you or another organisation) asks the wallet for a proof, and the wallet
shows only what that request asks for:
- selected values, such as name and department;
- facts about values that reveal nothing else, such as "employee number above 1000" or "valid today".

What lives where:

| | Where | Notes |
|---|---|---|
| The credential | Only on the user's phone | 12iD keeps no copy and cannot read the wallet |
| The schema (the list of value *names*) and your credential definition (your signing key's public part) | The public Indy ledger | Never any value or personal data |
| Revocation registries (which slots are revoked) | The public Indy ledger | Slot numbers only; never who holds a slot |
| The exchange record (who was offered what, and when) | 12iD's platform, for you | Values are erased after your retention period ([`data-protection.md`](data-protection.md)) |

## Validity: every credential expires

A credential stops being accepted when its `expiresOn` date passes, or earlier when you revoke or suspend it
(next section). **Expiry is always there**, also for verifiers that never read the ledger.

- **Every schema contains `expiresOn`.** The console adds it when you publish a schema.
- **You set it on every offer.** It is the expiry date as `YYYYMMDD`, as a string (`"20271231"` is 31 December 2027), today or later. Dates are UTC.
- **The credential is accepted through the end of that day** (until 23:59:59 UTC), and refused from the next day.
- **Every proof request checks it automatically** (`expiresOn >= today`), and you cannot remove that check. An expired credential fails every proof, and the wallet still keeps it.
- **`expiresOn` is never disclosed.** A proof only shows that the credential is still valid, not its date.
- **Any date up to the year 9999 works.** Expiry is a whole day, not a time of day.

**Choose the validity period for each credential type.** Revocation lets you end a credential early, so the
period is about how often the facts should be confirmed again. As a guide: months for access and membership,
a year or two for qualifications, with re-issue on renewal.

## Revoking and suspending

Every credential definition published since platform 0.12 is **revocable**. For each issued credential you can:

| | What it does | Undo |
|---|---|---|
| **Suspend** | Proof requests stop accepting it, for example while you look into a fraud alert | **Reinstate** |
| **Revoke** | Proof requests stop accepting it for good, for example when the account is closed | None: issue a new credential instead |

- **From your backend:** `POST /v1/credentials/{id}/suspend`, `/reinstate` or `/revoke`, with an optional `reason` ([`api.md`](api.md)).
- **From the console:** the same three actions on the credential's row (Exchanges, or the customer's page), for people whose role includes "Revoke and suspend credentials". When your organisation requires approval, a reinstate waits for a second person; a suspend or revoke never waits.
- **It takes effect within seconds.** The change is written to the ledger; until then the credential's `revocation.status` is `suspending`, `reinstating` or `revoking`. When it is written, you receive the `credential.revocation_changed` webhook, and from then on every proof request refuses the credential.
- **The credential stays in the user's wallet.** The wallet no longer offers it for a proof request. Your app can show its state: the SDK's `credentialStatus()` reads it from the ledger.
- **A verifier cannot tell a suspended credential from a revoked one.** Only you see the difference.
- **Every proof request checks it automatically**, like expiry, and you cannot remove that check.

**What the ledger shows.** Revocation works through *registries*: groups of up to 9,999 credentials of one
definition. The ledger records which slots of a registry are revoked, never who holds a slot. A proof shows
which registry the credential is in, not which credential it is. The first time a user's phone proves a
credential from a registry, it downloads that registry's file once: about 2.4 MB.

**Credentials issued before revocation existed.** A credential definition published before platform 0.12 is
not revocable, and cannot become revocable: the ledger does not allow it. To move:
1. Publish a revocable definition next to the old one. The console offers "Publish a revocable credential definition" on the schema's row; it uses the tag `revocable`.
2. Point your backend's offers at the new definition, and issue your users their credentials again.
3. Proof templates that name the schema accept both definitions. The old credentials stay valid until their `expiresOn` date; from then on, only revocable ones remain.

The console shows each definition as "Not revocable", or with how many credentials its registries have
issued.

**Erasure does not revoke.** Erasing a person's data and ending their credential are separate requests
([`data-protection.md`](data-protection.md), "What erasure does not do"). A credential stays revocable after
its data is erased.

## Wallet recovery: accept the loss, issue again

A wallet's keys live in the phone's secure hardware and never leave it. **There is no backup or export in
this version.** These all mean a new, empty wallet:
- a lost or replaced phone;
- a reinstalled app;
- cleared app data;
- the wallet files restored onto another device. The SDK reports `WalletKeyLost`, because the key stayed on the old phone.

What happens and what you do:
1. The app gets a new wallet. Your backend creates it with `POST /v1/wallets` (a new installation id gives a new wallet; a person may have several).
2. **You issue the person's credentials again.** Make this one step for the user: keep the facts you issued from in your own systems, and offer the credentials again when the new wallet connects to you.
3. The old credentials stay on the old phone. A proof needs the wallet's key, and that key never left the device. **If the phone was lost or stolen** rather than replaced, revoke its credentials: the wallet key is not tied to the phone's screen lock, so whoever can open your app on it could still prove with them.
4. Revoke the old installation's wallet token with `DELETE /v1/wallets/{id}`, and optionally erase the old connection (`POST /v1/connections/{id}/erase`). 12iD removes a revoked wallet's mediator connection after 30 days.

The SDK guide shows the app side (`reset()`, `WalletKeyLost`, and excluding the wallet from Android
backups, so it is never restored where its key is missing).

## Attribute values: how to encode them

Credentials are built on AnonCreds, which signs **strings**. Getting the encoding right at schema design
time is the most common source of mistakes.

**Rules**
- **Every value is a string.** There are no nested objects, arrays, booleans or number types.
- **Predicates (`>=`, `>`, `<=`, `<`) work only on whole numbers** from `-2147483648` to `2147483647`, written as plain digits, optionally with a leading `-`. Anything else (decimals, numbers above that range, text) is stored as a hash: the value can be disclosed but cannot be compared.
- `"00123"` and `"123"` compare as the same number. Store identifiers that must keep their leading zeros as text, and never compare them.
- An empty string is allowed and disclosed as empty.

**Recipes**

| You want | Store as | Example | Predicate example |
|---|---|---|---|
| A date someone may need to compare (birth date, start date) | Days or a compact integer: `YYYYMMDD` | `"19851103"` | Over 18 on 2026-10-05: `dateOfBirth <= 20081005` |
| A date only to display | ISO 8601 text | `"1985-11-03"` | none (text) |
| An amount | Integer in the smallest unit | `"125000"` (cents) | `salary >= 5000000` |
| Yes/no | `"1"` / `"0"` | `"1"` | `isAdult >= 1` |
| A choice from a list | Short text code | `"ENGINEERING"` | none; disclose it |
| A document | Its hash or a reference | `"sha256:9f86…"` | none |
| A time to compare, to the minute | Minutes since 1970-01-01 UTC (fits until the year 6053) | `"29979360"` (2027-01-01 00:00 UTC) | `validFromMinute <= <now in minutes>` |

`YYYYMMDD` compares correctly as a number (`20081005 > 19851103`) and stays readable. Use it for every date
a verifier may test with a predicate.

**Limits on an offer:** at most 125 attributes, names up to 100 characters, values up to 4 KB, and 64 KB for
all names and values together. Credentials carry claims, not documents.

## Schemas cannot change

- A published schema is permanent on the ledger: its attribute names never change.
- **Adding or renaming a field means a new schema version** (and a new credential definition).
- Deprecate the old version in the console to stop new offers. Proof templates that name the schema accept every version, so credentials issued before the change keep working.
- Plan names carefully: use lower camel case, no personal data in names, and include `expiresOn` (added for you).
- **Do not name an attribute `id`.** Wallets treat `id` as the identifier of the credential's subject and cannot store the credential, so the console and the API refuse it. Use a specific name such as `memberId` or `customerId`.

## Checklist before your first schema

1. List the values; for each, decide "disclose" or "compare". Every "compare" value must be a whole number in range.
2. Pick the validity period, and how you will re-issue (renewal, changed facts, a new phone).
3. Decide when your backend revokes or suspends (an account closed, a fraud alert, a lost phone), and who may do it in the console.
4. Decide how a user gets their credentials again on a new phone, and make it one step.
5. Keep personal data out of `externalId` and out of attribute *names*.
6. Publish in the staging environment first, issue to a test wallet, and run your proof templates against it.
