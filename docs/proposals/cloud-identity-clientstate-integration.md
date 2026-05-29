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
integration trivially flips to using Google's assigned partner ID, and the
admin-console UX improves (Fleet shows up in the third-party integrations
list, custom access-level setup is one-click). The data model, the fields
written, and the customer-side configuration do not change.

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
3. **Resolution** — for each Fleet host with a known Google Workspace email
   (from existing IdP host vitals / Google Workspace integration work in
   fleetdm/fleet#42915), call
   `devices.deviceUsers.lookup?email=user@example.com` to get the
   `devices/{deviceId}/deviceUsers/{deviceUserId}` pair. Cache aggressively;
   invalidate on Endpoint Verification reinstall.
4. **Sync loop** — on `sync_interval`, compute the desired ClientState for
   each resolved (host, deviceUser) pair, compare to last-written state in
   a new `host_google_client_state` table, and `PATCH` only when changed.
   Use `etag` for optimistic concurrency.
5. **Backoff** — `429` and `5xx` from Google use exponential backoff with
   jitter; persistent failure bubbles up to the activity feed and metrics.

The first iteration is push-only. A future enhancement could subscribe to
Cloud Identity's Pub/Sub `device-events` topic to react to device-side state
changes (e.g., user wiped device → Fleet retires the host).

## Open questions for Fleet product

1. **Host-to-deviceUser mapping.** Fleet already plans IdP host vitals from
   Google Workspace (fleetdm/fleet#42915). Does that work surface
   `deviceUser` resource names, or only user email? If only email, this
   proposal needs `devices.deviceUsers.lookup` capability.
2. **Multi-user macs.** A shared mac can have multiple
   `deviceUsers`. Default behavior here is to write the same ClientState
   for every deviceUser tied to the device. Confirm that matches product
   intent.
3. **iOS/Android.** Cloud Identity supports company-owned and BYOD mobile
   devices. Fleet-managed iPads (e.g., kiosks) should get the same
   treatment; iOS BYOD policy compliance is a question mark since Fleet's
   policy engine is osquery-driven.
4. **Premium-tier gating.** Patch on ClientState requires Cloud Identity
   Premium / Workspace Enterprise on the customer side, but **not** on
   Fleet's side. Fleet should detect a `403 PERMISSION_DENIED` with the
   Premium signal and surface a clear "Workspace Enterprise required"
   error in the integration page, not a generic auth error.
5. **Free-tier fallback.** For customers without the Premium SKU, the
   Directory API's `customerDevices` patch supports a narrower set of
   compliance signals. Worth supporting as a degraded mode, or scope to
   Premium for v1?

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
