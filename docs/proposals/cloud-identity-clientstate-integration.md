# Extend Fleet's Conditional Access framework to Google Cloud Identity

## TL;DR

Fleet's "Conditional Access" today is two distinct mechanisms sharing a name:

- **Microsoft Intune** (`ConditionalAccessMicrosoftIntegration`) — Fleet is a
  Microsoft Compliance Partner and PATCHes per-host compliance into Intune so
  Entra Conditional Access can gate apps. **API-push pattern.**
- **Okta** (`ConditionalAccessIDPAssets` + the Apple SCEP profile in
  `conditional_access_idp.go`) — Fleet issues a per-device certificate that
  Okta consumes as a device-trust signal. **mTLS/cert-presentation pattern**,
  because Okta has no Intune-style compliance-partner API.

This proposal adds a **third provider** that follows the **Microsoft pattern,
not the Okta pattern**: write per-device compliance, health, and management
signals into Google Cloud Identity via the
`devices.deviceUsers.clientStates.patch` API. Once written, **Context-Aware
Access** and **Chrome Enterprise Premium** can gate Workspace, Google-fronted
SaaS, and any IdP-federated app on Fleet's view of device state.

Microsoft Intune ≈ Google Cloud Identity ClientState in structure: a vendor
device registry that accepts compliance signals from a registered partner and
exposes those signals to a vendor-side policy engine (Entra CA / Google CAA).
The Okta SCEP-cert path is unrelated and is not the model here.

This is the interim path explicitly called out in fleetdm/fleet#28476, and it
is a strict superset of what fleetdm/fleet#43583 asks for (CBCM/Chrome
Enterprise Premium attestation): once Fleet is writing into Cloud Identity,
CEP consumes the same signal alongside browser policies.

## Related issues

- fleetdm/fleet#28476 — Google BeyondCorp Alliance Partner (umbrella; this
  proposal is the "interim" path that issue describes, and it does not block
  on partner status)
- fleetdm/fleet#43583 — Integrate with Chrome Browser Cloud Management to
  provide MDM device attestation (this proposal subsumes it: ClientState
  signals are what CEP evaluates)
- fleetdm/fleet#6566 — Device Trust Scoring (broader trust-scoring framework
  this would plug into)
- fleetdm/fleet#42915 — IdP host vitals from Google Workspace (inverse
  direction: pulling from Google rather than pushing to Google)

## Why the partner route is not required

Per the Google reference at
`docs.cloud.google.com/identity/docs/reference/rest/v1/devices.deviceUsers.clientStates/patch`:

> Resource name of the ClientState in format:
> `devices/{device}/deviceUsers/{deviceUser}/clientState/{partner}`, where
> `partner` corresponds to the partner storing the data. For partners belonging
> to the "BeyondCorp Alliance", this is the partner ID specified to you by
> Google. **For all other callers, this is a string of the form
> `{customer}-suffix`**, where `customer` is the organization's customer ID
> (the value after the leading `C` in the Directory API's `customers/my_customer`
> response) and the suffix is an arbitrary string chosen by the caller. This
> suffix is displayed verbatim in the admin console and is the identifier used
> when setting up Custom Access Levels in Context-Aware Access.

So any Workspace customer can call this API with `{C-id}-fleet` (or
`{C-id}-fleet-{team}`) as the partner segment, today, without Google's
involvement. The only prerequisite is a service account with domain-wide
delegation and the `cloud-identity.devices` scope.

If Fleet later joins the BeyondCorp Alliance (fleetdm/fleet#28476), the
integration trivially flips to using Google's assigned partner ID. Three
things improve, the third of which is end-user-visible:

- Fleet shows up in the third-party integrations list in admin.google.com,
  making customer setup one-click.
- Custom access-level setup is one-click rather than requiring the admin
  to type the `{C-id}-fleet` partner segment.
- **CAA Remediator messages render with "Fleet" as the partner name**
  rather than a raw `{C-id}-fleet` token. This is the only end-user-visible
  improvement available on the Google side, and it is the strongest
  practical reason to pursue partner status — see *Rich end-user
  remediation lives in Fleet Desktop, not Google's deny page* below.

The data model, the fields Fleet writes, and the customer-side
configuration do not change between non-partner and partner modes.

## Customer-side prerequisites and edition eligibility

Cloud Identity's third-party partner integrations (the feature group that
includes `devices.deviceUsers.clientStates.patch`) are gated on the
customer holding one of a specific set of editions. Per Google's
*Set up third-party partner integrations* admin help page:

> Supported editions for this feature: Frontline Standard and Frontline
> Plus; Enterprise Standard and Enterprise Plus; Education Standard,
> Education Plus, and Endpoint Education Upgrade; Cloud Identity
> Premium.

That is the canonical eligibility list. A customer outside it will see
`403 PERMISSION_DENIED` on the first ClientState PATCH, regardless of
how the service account is set up. Fleet's integration page surfaces
this as "Your Workspace edition does not include Cloud Identity Premium
security management" rather than a generic auth error.

**Specifically excluded** (Fleet detects and explains):

- Workspace Business Starter, Business Standard, Business Plus.
- Workspace Enterprise Essentials (without the `+` — Essentials Plus is
  in scope).
- Education Fundamentals. Worth calling out because the public Education
  product page lists "Cloud Identity Premium" as included with
  Fundamentals — that refers to the Cloud Identity Premium *identity
  service* (SSO, MFA, basic device management for Education users),
  which every Education edition gets. The capability this integration
  depends on is **Cloud Identity Premium security management** (the
  CAA + partner-integrations feature pack), which only starts at
  Education Standard. The admin-help editions comparison reflects this;
  the marketing comparison conflates the two. Treat the admin-help page
  as authoritative.
- Cloud Identity Free.

**Required customer-side setup**, beyond having an eligible edition:

- Endpoint Verification deployed to every device that should be evaluated
  (Chrome extension + native helper on macOS/Windows/Linux). Without EV,
  there is no `deviceUser` to PATCH — see *Endpoint Verification as the
  resolution mechanism* below.
- A super-admin to create the GCP service account, enable domain-wide
  delegation, and authorize the `https://www.googleapis.com/auth/cloud-identity.devices`
  scope in the Workspace admin console.
- The customer's CAA policy authored in admin.google.com referencing the
  Fleet partner segment (`{C-id}-fleet` for the non-partner path, or the
  Alliance-assigned partner ID once Fleet ships #28476).

**Not required** (common misconceptions worth heading off):

- Fleet does not need its own Workspace tenant or edition.
- Fleet does not need to be a registered BeyondCorp Alliance partner —
  the `{C-id}-fleet` partner segment works on day one. Alliance status
  is an upgrade (see TL;DR and the partner-status notes below), not a
  prerequisite.
- No GCP project of Fleet's needs to exist on the customer side; the
  service account lives in the customer's project.

## What the integration does

For each host that is enrolled in Fleet and has at least one Google Workspace
user signed in on a Cloud-Identity-registered device (i.e., the host has gone
through Endpoint Verification or Google Mobile Management), Fleet `PATCH`es a
ClientState resource that mirrors Fleet's view of the device:

| Cloud Identity field | Fleet source | Notes |
| --- | --- | --- |
| `complianceState` | All policies in scope passing | `COMPLIANT` if every applicable policy on the team is passing, else `NON_COMPLIANT`. Driven by the existing policy engine; no new evaluation logic. |
| `managed` | MDM enrollment status | `MANAGED` if the host appears in `host_mdm` with Fleet as the MDM, else `UNMANAGED`. |
| `healthScore` | Configurable mapping | Default mapping: 100% policies passing → `VERY_GOOD`, ≥80% → `GOOD`, ≥50% → `NEUTRAL`, ≥20% → `POOR`, else `VERY_POOR`. Admin-overridable per team. |
| `scoreReason` | Failing policy names | Comma-joined list of failing policy names, capped at the field's length limit, so the Workspace admin can see *why* a device is non-compliant without leaving the Google admin console. |
| `customId` | Fleet `host.uuid` | Stable identifier the admin can cross-reference back to Fleet via deep-link. |
| `assetTags` | Fleet team name, labels | Lets CAA policies be scoped by team or label (e.g. "block access from kiosk-team devices"). |
| `keyValuePairs` | Selected host vitals | Small, opinionated set: `osquery_version`, `os_version`, `disk_encryption_enabled`, `last_seen`, `fleet_team`, `fleet_url`. Capped well under the 10 KB serialized limit. |

The resource name suffix is configurable per Fleet team, defaulting to
`fleet` (so the partner segment becomes `{C-id}-fleet`). Multi-team
deployments can use `{C-id}-fleet-{team-slug}` if they want CAA rules to
differ per team.

## User-facing surface

### Configuration (GitOps + UI)

```yaml
# org-settings or team yaml
integrations:
  google_cloud_identity:
    enabled: true
    # one of:
    service_account_json: $GOOGLE_CLOUD_IDENTITY_SA_JSON   # env-var secret ref
    # or
    workload_identity:
      audience: //iam.googleapis.com/projects/.../locations/global/workloadIdentityPools/.../providers/...
      service_account_email: fleet-cloud-identity@PROJECT.iam.gserviceaccount.com
    impersonated_admin: admin@example.com
    customer_id: C0xxxxxxx           # validated against my_customer at startup
    partner_suffix: fleet            # final partner = "0xxxxxxx-fleet"
    sync_interval: 5m
    health_score_mapping:            # optional override
      very_good: 100
      good: 80
      neutral: 50
      poor: 20
```

A Settings → Integrations → Google Cloud Identity page mirrors the YAML for
non-GitOps users, with a "Test connection" button that calls
`customers/my_customer` and a "Send test signal" button that writes a
ClientState for one selected host.

### Operator UX

- Host detail page gets a "Google Cloud Identity" row showing
  `last_synced_at`, the partner segment used, and a link straight to the
  device in the Google admin console.
- Fleet activity feed records each compliance-state transition pushed to
  Google, so admins can audit what Fleet told Google and when.
- A new `fleet/google_cloud_identity_sync` cron job exposes standard Fleet
  metrics (success/failure count, latency, last sync time per team).

### End-user UX

None directly — but the practical effect is that Workspace admins can write
CAA policies like:

```cel
device.policy_compliant == true AND
device.vendor.partner_id == "0xxxxxxx-fleet" AND
device.vendor.asset_tag.contains("team:engineering")
```

…to gate Drive, Gmail, or any SAML-federated app on Fleet's view of device
health.

## Architecture sketch

A new package `ee/server/integrations/google_cloud_identity/`:

1. **Auth** — a `tokenSource` that mints a domain-wide-delegated access
   token, either from a JSON key or from workload-identity federation.
   Scope: `https://www.googleapis.com/auth/cloud-identity.devices`. The
   `subject` is the admin email from config.
2. **Discovery** — at startup and on a long interval, call
   `customers/my_customer` and cache `id` to verify it matches configured
   `customer_id`. Mismatch is a hard config error.
3. **Resolution** — Fleet reads the device-local Endpoint Verification (EV)
   state to discover every signed-in Workspace user's
   `deviceUsers/{deviceUserId}` pointer directly, no `lookup`-by-email
   roundtrip required. See *Endpoint Verification as the resolution
   mechanism* below for the file layout, schema, and platform path
   variations. A `devices.deviceUsers.lookup?rawResourceId=...` call
   confirms each `resourceId` and yields the canonical resource name; the
   result is cached and re-validated only when EV state changes.
4. **Sync loop** — on `sync_interval`, compute the desired ClientState for
   each resolved (host, deviceUser) pair, compare to last-written state in
   a new `host_google_client_state` table, and `PATCH` only when changed.
   Use `etag` for optimistic concurrency.
5. **Backoff** — `429` and `5xx` from Google use exponential backoff with
   jitter; persistent failure bubbles up to the activity feed and metrics.

The first iteration is push-only. A future enhancement could subscribe to
Cloud Identity's Pub/Sub `device-events` topic to react to device-side state
changes (e.g., user wiped device → Fleet retires the host).

## Endpoint Verification as the resolution mechanism

Google's API reference for
`devices.deviceUsers.lookup?rawResourceId=…` says the raw resource ID is
saved by Endpoint Verification on every managed device. That ID is what
turns "a host" into "a specific `deviceUsers/{deviceUserId}` resource Fleet
can PATCH a ClientState onto." It also avoids the alternative — looking up
by Workspace email — which is ambiguous on shared devices and depends on
Fleet already knowing every signed-in user's email (a separate
IdP-host-vitals problem, fleetdm/fleet#42915).

**The path in Google's docs is stale on current macOS.** Google's reference
names:

- macOS: `~/.secureConnect/context_aware_config.json`
- Windows: `C:\Users\<user>\.secureConnect\context_aware_config.json`
- Linux: `~/.secureConnect/context_aware_config.json`

Current EV on macOS (verified on a developer machine running EV) actually
writes to:

- `~/Library/Application Support/Google/Endpoint Verification/accounts.json`

with a per-account binary file under
`~/Library/Application Support/Google/Endpoint Verification/accounts/<gaia_user_id>`.

The `accounts.json` file is a JSON map keyed by the user's obfuscated
Google account ID, where each value carries exactly the two pieces Fleet
needs:

```json
{
  "<gaia_user_id>": {
    "device": { "resourceId": "<rawResourceId>", "lastSync": "<rfc3339>" },
    "user":   { "email": "user@example.com" }
  }
}
```

Fleet's osquery layer needs a query that:

1. Reads the **macOS-current path** first, then falls back to the
   Google-documented `~/.secureConnect/context_aware_config.json` for older
   EV versions and for Windows/Linux (path layouts on those platforms need
   confirmation against a current EV install — the doc is likely stale on
   them too).
2. Emits one row per user-on-host: `(uid, username, resource_id, email,
   last_sync, source_path)`.
3. Is run per local user account so the host can have multiple rows when
   multiple Workspace identities are signed in.

**Multi-account-per-host is normal, not an edge case.** A test read on a
developer machine showed seven distinct accounts in `accounts.json`
(personal Gmail plus several Workspace tenants). On a shared kiosk, a
contractor's laptop, or anyone who has ever signed into a side gig's
Workspace, this is the default. Fleet must therefore:

- PATCH a ClientState for **every** EV-resolved user on the host whose
  email matches the customer's configured Workspace domain(s) — not just
  "the primary user."
- Filter out non-matching domains entirely (personal Gmail, other
  tenants); Fleet never emits ClientStates to a Workspace it isn't
  configured for, and never logs unconfigured emails to telemetry.

**EV not installed is a first-class host state.** If `accounts.json`
doesn't exist (and the legacy path is also absent), there are no
deviceUsers to patch and the integration is a no-op for that host. Fleet
surfaces this as `Endpoint Verification not installed` on the host detail
page so admins know *why* no signal is going to Google. The customer's CAA
policy on the Google side already denies access for unenrolled devices, so
the security posture is correct by default.

## Behavioral parity with the Microsoft Intune integration

These behaviors come from Fleet's documented Entra Conditional Access
guide. They are vendor-agnostic Fleet rules and should apply identically
to the Google path; the Microsoft-shaped *prerequisites* in that guide do
not (see next section).

1. **"MDM turned off" overrides policy state.** If a user unenrolls Fleet
   via *System Settings → Device Management → Unenroll* (or the Windows /
   Linux equivalents), Fleet immediately PATCHes
   `managed: UNMANAGED` and `complianceState: NON_COMPLIANT` — even if
   every policy is passing. This is the same rule Fleet applies to Intune
   today, and it is load-bearing: admins build CAA policies assuming Fleet
   tells the truth about MDM enrollment, not just policy state.
2. **Up to one-hour push latency, with a Fleet Desktop "Refetch" button.**
   The scheduled sync cadence is best-effort; the tray-icon refetch
   already exists for Intune and must additionally trigger a Google
   PATCH so users blocked by stale state have an immediate path out. The
   one-hour figure should be considered an upper bound, not a target —
   v1 ships with a configurable `sync_interval` defaulting to 5 minutes.
3. **"Pending activation" → "Active" lifecycle in the Google admin
   console.** The Google admin sees the partner segment
   (`{C-id}-fleet` or the Alliance ID) appear in *Devices → Mobile &
   endpoints → Settings → Third-party integrations* only after the first
   successful ClientState write. Fleet documents this and the integration
   page surfaces "Awaiting first signal" until that flips.
4. **Every PATCH lands in the Fleet activity feed.** One activity entry
   per state transition, with the host, the deviceUser resource name, the
   fields that changed, and the trigger (scheduled sync, manual refetch,
   policy-run, MDM-state-change). This is observability the Entra guide
   doesn't document but admins consistently ask for; ship it on both
   providers in this work.
5. **Explicit disable semantics: Fleet retracts before forgetting.** When
   the integration is disabled (or a host's team has the integration
   removed), Fleet does **not** silently stop pushing. Instead, it PATCHes
   `managed: UNMANAGED, complianceState: NON_COMPLIANT,
   scoreReason: "Fleet integration disabled"` for every previously-synced
   deviceUser, then drops the local resolution cache. Without this, stale
   `COMPLIANT` records keep granting access via CAA forever — a gap the
   Entra doc leaves silent and worth closing on both providers.
6. **End-user remediation path through Fleet Desktop.** See *Rich
   end-user remediation lives in Fleet Desktop* below. Short version: the
   Google deny page is not a customizable surface, so all rich
   remediation (failing-policy list, fix instructions, Refetch button)
   must live in Fleet Desktop and be driven by Fleet's own client, the
   same way the Entra "Check Compliance" flow is.
7. **Stale/offline hosts age out to non-compliant.** A configurable
   threshold (default 7 days without check-in) flips
   `complianceState: NON_COMPLIANT, scoreReason: "Host offline > N days"`.
   Without this, a stolen-and-powered-off laptop keeps its last-known
   COMPLIANT state in Google indefinitely. The Entra doc doesn't address
   this; we should fix it on both providers in this work.

## What is intentionally NOT copied from the Microsoft integration

These are Microsoft-shaped requirements that exist because of how
Entra/Intune is wired. They do not have Cloud Identity analogues and
should not be replicated in the Google path:

- **No required "Fleet conditional access" Workspace group.** Intune's
  Compliance Partner API requires the partner relationship to be scoped to
  a specific Entra security group, so the Entra guide tells admins to
  create one and assign users. Cloud Identity has no such scoping;
  `clientStates.patch` is authorized per-deviceUser via the
  customer-scoped service-account token. Customers may *optionally*
  group-scope their CAA policies (e.g. "this CAA rule applies to group
  X"), but that is a customer-side CAA design choice — not a Fleet
  prerequisite. The integration must not gate on group membership.
- **No Platform-SSO-equivalent configuration profile.** Platform SSO is
  macOS's mechanism to bind the local user to an Entra identity and mint
  a device-bound token Intune can attribute to. The Google analog is
  the user installing Endpoint Verification (Chrome extension + native
  helper) and signing in — there is no Fleet-shipped profile that mints
  this binding. Fleet should *detect* EV state, not deploy a profile to
  create it. Pushing a Workspace-specific config profile would be a
  Microsoft-pattern artifact in a Google integration.
- **No Company Portal-equivalent app.** Microsoft's Company Portal
  completes the device registration handshake on macOS. Google has no
  structurally equivalent app — EV is the whole story on desktop. On
  mobile, Google Device Policy serves a related-but-different role
  (managed-device enrollment, not compliance-partner registration). The
  integration does not need to ship, deploy, or document a Google-side
  companion app.
- **No tenant admin-consent OAuth redirect.** Entra integration starts by
  redirecting the admin to Microsoft's consent screen so Fleet's
  multi-tenant app gains tenant access. Google's path is the customer
  creating their own GCP service account, enabling DWD, and authorizing
  scopes in the Workspace admin console — there is no "click consent for
  Fleet's app" step, and the integration page must not pretend there is.
  The setup-flow shape is fundamentally different and the docs should
  reflect that.

## Rich end-user remediation: what Google exposes, what Fleet owns

The end-user-facing denial experience is shaped by **four** surfaces.
Three of them are read-only, admin-set, or fixed-by-Google; only one is
fully Fleet-controlled. Naming them all out so we don't repeat the
incorrect "Google has no levers here" framing, but also don't repeat the
incorrect "there's an API for this" framing.

**1. CAA Remediator strings** (the "Allow users to unblock apps with
remediation messages" feature). When a `device.vendors[…]` check fails,
Google renders one of a small fixed set of strings, e.g. *"Your device
isn't meeting some requirements, based on information from [PARTNER
NAME]"*. `[PARTNER NAME]` is a token Google substitutes from the
BeyondCorp Alliance partner registry; without partner registration, the
substitution falls back to the raw `{C-id}-fleet` partner segment, which
is ugly. The partner cannot supply a URL, logo, or per-policy detail
through this surface.

**2. `description` on a Custom Access Level**
(`accesscontextmanager.googleapis.com`). Up to 2000 characters, rendered
as part of the denial reason. Set by the customer admin in their CAA
policy, *not* by Fleet — but Fleet's docs should ship recommended text
the admin pastes in.

**3. `api_controls.custom_user_message` (Cloud Identity Policy API).**
Field: `error_text`. Doc copy:

> Customize the message to show users when they can't access an app due
> to access settings.

This is the API representation of the admin-console "Additional custom
message" field at *Security → Context-Aware Access → User Message*.

There is a small **documentation conflict** to resolve at implementation
time. The supported-settings table marks this setting
`Mutate Supported: No`, but the REST reference for `policies.create` /
`policies.patch` does not gate per-setting mutability at the method
level — both methods look generically open to any Setting type. The
likely truth is that the per-setting flag is enforced at runtime and the
table is authoritative, but the only way to confirm is to call
`policies.create` against `settings/api_controls.custom_user_message`
and observe the response. The v1 design assumes the conservative reading
(read-only, admin sets the value in admin.google.com), with an
explicit follow-up to retest empirically:

- During integration setup, Fleet calls `policies.get` on
  `api_controls.custom_user_message` and surfaces a warning on the
  integration page if `error_text` is empty or doesn't contain a
  remediation URL. This actively guides the admin toward configuring it
  rather than leaving the user at Google's default copy.
- Fleet docs ship a recommended `error_text` template the admin copies
  in, including Fleet branding and a deep link to the Fleet Desktop
  remediation page. This is the closest available analogue to "the
  partner controls the deny page" under the conservative reading.
- If empirical testing shows `policies.create` accepts a write to this
  setting (i.e., the table is stale), v1 ships with Fleet writing
  `error_text` directly. No integration changes required beyond a
  feature flag and an updated test in Fleet's integration suite.
- One real limit either way: this setting is **tenant-wide, not
  per-CAA-rule and not per-Fleet-policy**. A single string for the whole
  org. Per-policy substitutions are not in the current schema.

A separate Policy API capability worth flagging as **future** work: the
Policy API supports creating custom policies scoped per-org-unit or
per-group via the `policyQuery` field's CEL clauses
(`entity.org_units.exists(...)` / `entity.groups.exists(...)`). For
mutate-supported settings, Fleet could programmatically scope policies
to a Workspace group whose membership Fleet manages based on compliance
state — opening a richer integration surface than just ClientState. Out
of scope for v1; capture as a potential v2 direction.

**4. Fleet Desktop — the only fully Fleet-controlled surface.** This is
where every piece of rich remediation has to live, because none of the
other three surfaces accept per-policy or per-user content from Fleet:

- **Fleet Desktop tray icon** owns the user-facing remediation UI:
  failing-policy list, plain-language fix instructions per policy, a
  Refetch button that triggers an immediate Cloud Identity PATCH, and a
  deep link to the host detail page for advanced users. Same UX pattern
  Fleet already uses for Intune-blocked users; the Google integration
  reuses it byte-for-byte and the user experience does not depend on
  which IdP is gating.
- **Fleet Desktop notifications** fire proactively the moment a policy
  flips to failing (i.e. before the user hits the deny page), giving the
  user a chance to remediate before being blocked at all. This is the
  only place where Fleet can show per-policy specifics with its own
  copy, logo, and timing.
- **A short, customer-tunable Fleet Desktop URL** in the integration
  config (e.g. `remediation_url: https://help.example.com/fleet`) so the
  customer can route blocked users to their own intranet help page if
  they prefer. Defaults to Fleet's hosted remediation help.

**ClientState fields are never user-facing.** Worth saying explicitly
because it's a common (and wrong) assumption: the following fields Fleet
writes are admin-console-only and CEL-evaluable only — they never
render on the blocked user's screen: `scoreReason`, `assetTags`,
`healthScore`, `customId`, `keyValuePairs`. Writing a useful
`scoreReason` is still worth doing (Workspace admins triaging tickets
see it in the admin console; CAA expressions can branch on it), but
admins should not expect it to reach end users via Google's surfaces.

**Partner status has real end-user value even pre-Alliance-features.**
Becoming a registered BeyondCorp Alliance partner (#28476) changes the
substituted `[PARTNER NAME]` token in CAA Remediator from a raw
`{C-id}-fleet` string to "Fleet" on every blocked user's screen across
every Workspace customer using the integration. That's the strongest
end-user-visible argument for pursuing partner status in parallel with
shipping this work.

## Edge cases Fleet decides explicitly here

The Entra guide is silent on these. Fleet should decide and document them
for both providers in this work.

- **Host transferred between teams.** ClientState is rewritten with the
  new team's `assetTags` and `partner_suffix` (if the destination team
  uses a team-specific suffix); the *old* partner segment's ClientState
  is retracted (per the disable semantics above) so a CAA policy keyed on
  the old `assetTag` no longer matches. No "two truths about one device"
  state ever exists.
- **Host deleted from Fleet.** Same retraction PATCH as integration
  disable (UNMANAGED + NON_COMPLIANT + scoreReason "Host removed from
  Fleet"), then the local resolution row is dropped. The Cloud Identity
  ClientState resource itself is not deleted — Google's API doesn't
  expose a delete on partner-scoped ClientStates — but the retracted
  state ensures it stops granting access.
- **End-user signed in to Fleet but no EV account on the host.** The host
  has no Workspace deviceUser to patch; integration is a no-op for that
  host and the host detail page shows "Endpoint Verification not
  installed or no Workspace account signed in." No error, no failed
  state.
- **End-user signs into an additional Workspace identity later.** The
  next osquery check-in surfaces the new `accounts.json` entry; Fleet's
  resolution loop discovers the new deviceUser and starts emitting
  ClientStates for it on the next scheduled sync — no admin intervention.
- **Linux support.** Endpoint Verification supports Linux; Fleet's osquery
  agent supports Linux. Unlike the Entra integration (which explicitly
  excludes Linux because Intune doesn't have a Linux compliance-partner
  path), the Google integration ships with Linux from v1 — same osquery
  resolution query, same PATCH path.

## Open questions for Fleet product

1. **Platform paths on Windows and Linux.** Google's REST reference names
   `~/.secureConnect/context_aware_config.json` on those platforms, but
   the macOS path is already stale (see *Endpoint Verification as the
   resolution mechanism*). Need to verify against a current EV install on
   each platform before shipping the osquery resolution query.
2. **iOS/Android.** Cloud Identity supports company-owned and BYOD mobile
   devices. Fleet-managed iPads (e.g., kiosks) should get the same
   treatment; iOS BYOD policy compliance is a question mark since Fleet's
   policy engine is osquery-driven. Out of scope for v1; revisit once the
   desktop integration is shipped.
3. **Lower-tier fallback for non-eligible editions.** Customers outside
   the eligibility list above (Business Starter/Standard/Plus,
   Enterprise Essentials non-Plus, Education Fundamentals, Cloud
   Identity Free) cannot use ClientState. The Directory API's
   `customerDevices` patch supports a narrower set of compliance signals
   and may be available to lower tiers. Worth supporting as a degraded
   mode, or scope to the Premium-only path for v1?
4. **Push-vs-pull latency target.** The Entra integration tolerates a
   one-hour ceiling. CAA evaluations are real-time and a stale
   `COMPLIANT` signal is a security gap; the default `sync_interval` of
   5 minutes feels right for v1 but the actual target depends on Fleet
   policy-run cadence. Confirm with product.

## Why this is worth doing

- **Closes #43583 in a more general way.** The customer in that issue
  specifically asked for an MDM-backed trust signal to CEP. CEP consumes
  Cloud Identity ClientStates. Shipping this integration is the answer.
- **Closes the open half of #28476.** That issue notes "in the interim, the
  user could write custom policies or use Fleet's host vitals to build an
  automation using Google's Directory API." This proposal makes that the
  product, instead of homework.
- **Materializes #6566 (Device Trust Scoring).** Fleet's policy engine
  already computes per-host compliance; this is the first integration that
  takes the resulting score off-platform into a place customers care about
  (Workspace access decisions).
- **Strategic positioning vs. Jamf, Crowdstrike, VMware.** All three are
  listed BeyondCorp Alliance partners. The "use a custom partner ID"
  pathway means Fleet can ship an equivalent customer outcome *now* and
  pursue partner status in parallel.

## Out of scope

- Pulling device state *from* Google into Fleet (that's #42915's territory).
- Provisioning Workspace users, groups, or applying CAA policies — Fleet
  writes the trust signal; the customer authors CAA rules in their admin
  console.
- ChromeOS device management (#16884).
- Replacing or front-ending Endpoint Verification — devices must already be
  registered in Cloud Identity (via Endpoint Verification, Google Mobile
  Management, or Chrome Browser Cloud Management); Fleet adds a partner-
  scoped ClientState on top of that registration.
