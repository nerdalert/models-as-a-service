# Upgrade to 3.6

## Recommended downtime

When upgrading from RHOAI **3.5** to **3.6**, schedule a **short maintenance window**.

MaaS rewrites gateway `AuthPolicy` identity fields and per-model `TokenRateLimitPolicy` predicates so rate limiting matches a short subscription rate-limit ID from maas-api. Until maas-api, Authorino’s subscription-info cache, and the controller-managed policies are all on the new contract, rate-limit matching can briefly fail open. A small planned downtime (or a traffic pause through the gateway) avoids that window.

## Token rate-limit counters reset

On upgrade, TRLP limit map keys change from per-subscription names
(`{namespace}-{subscription}-{model}-tokens`) to rate-grouped names
(`tokens-{limit}-per-{window}`), and Limitador counters switch from
`selected_subscription_key` to the short `selected_subscription_id`.

Limitador **enforcement** counters for those limits start fresh: in-window token usage against subscription quotas resets. Users may see a one-time “full budget again” effect after upgrade until the next window elapses.

This does **not** rewrite or invalidate historical **Prometheus** or **Loki** usage data. Telemetry labels continue to use the human-readable subscription name (`auth.identity.selected_subscription` / `X-MaaS-Subscription`), not the rate-limit ID. Pre-upgrade series and logs remain queryable under the same subscription labels; only live Limitador quota state is affected.

## What else to know

- `auth.identity.selected_subscription_key` remains for telemetry and debugging.
- Rate limiting matches `auth.identity.selected_subscription_id` (see [Authentication Internals](../architecture-internals/authentication-internals.md)).
- Response header `X-MaaS-Subscription-Rate-Limit-Id` may be injected when select returns `rateLimitId`. Gateway AuthPolicy rejects inbound client values for that header; also ensure Praxis / identity-header strip configs treat `x-maas-*` (prefix or explicit list) so it cannot be spoofed upstream.

## IPP → Praxis API translation migration

When a tenant flips to Praxis (`maas.opendatahub.io/payload-processing-type=praxis`),
maas-controller inspects the live `payload-processing-plugins` ConfigMap
**before** deleting legacy IPP resources ([RHOAIENG-98846](https://issues.redhat.com/browse/RHOAIENG-98846)):

| RHOAI 3.5 ConfigMap | Behavior |
| --- | --- |
| Exact product default | IPP already translated requests. `spec.apiTranslation.enabled` stays omitted/off for a new Praxis tenant. Cleanup proceeds only when active ExternalModels use OpenAI Chat Completions passthrough. |
| Exact default plus response `api-translation` | IPP translated requests and responses. Before cleanup, maas-controller durably writes `spec.apiTranslation.enabled: true`, unless an operator already set the field. The compatibility check still applies. |
| Unknown or customized ConfigMap | Migration **blocked**; the controller does not infer equivalent Praxis behavior. |

The legacy IPP request translator was a multi-provider catch-all documented for
OpenAI, Anthropic, Azure OpenAI, Vertex AI, and Bedrock Mantle. Praxis currently
qualifies only OpenAI `openai-chat` pass-through and Anthropic `messages`
translation for ExternalModels. The `apiTranslation` permission is not a
tenant-wide translator choice: it permits both the shared **Anthropic Messages
client → OpenAI-compatible backend** flow and the ExternalModel **OpenAI Chat
Completions client → selected Anthropic Messages provider** flow. In the
ExternalModel path, `intelligent_route` selects the provider per request, and
the selected provider's profile remains bound through request and response
translation. OpenAI-compatible candidates remain pass-through.

An active Azure OpenAI, Vertex AI, Bedrock, custom provider, or unqualified
format/auth binding is not claimed as compatible. The AIGC resolver fails that
reconcile before publishing or cleaning serving resources, preserving the
last-known route and overlay; the migration compatibility check likewise must
keep IPP state until that binding is qualified or an operator explicitly
accepts the compatibility risk. A passing shared `/v1/messages` request does
not prove ExternalModel compatibility, and a recognized IPP ConfigMap
fingerprint alone does not prove every historical provider direction.

The pure-default IPP ConfigMap already enabled request-side `api-translation`;
its response chain was empty. Baseline #2 added response-side processing.
Recognizing either fingerprint preserves that migration intent, but does not
prove support for every provider or response mode. The recognized fingerprints
and migration decisions are intentionally strict.

For a recognized ConfigMap baseline, an explicit
`spec.apiTranslation.enabled: false` accepts the compatibility loss for an
unsupported active ExternalModel and lets cleanup proceed. It does not bypass
the fingerprint check for a customized ConfigMap. The force annotation below
can bypass that separate check. Both choices accept compatibility risk; they
are overrides, not parity. Review and test affected model/provider bindings
before using them. MaaS records an accepted risk on `MaasTenantConfig` and keeps
it in the `Degraded` condition after the one-shot cleanup while affected
provider bindings remain active. The warning clears when no active
ExternalModel needs the unsupported legacy transform. Forced cleanup of a
customized ConfigMap retains a warning that the migration does not establish
IPP parity.

While blocked:

- `MaasTenantConfig` stays `Ready=False` with a message naming the force annotation
- Annotation `maas.opendatahub.io/payload-processing-migration=blocked` is set
- Legacy IPP resources remain until an operator acknowledges the risk

Force migration after reviewing release notes. Back up the current tenant plugins
ConfigMap from the gateway namespace first — cleanup deletes it and the controller
does not retain the previous custom data:

```bash
kubectl get configmap <tenant-plugins-configmap> -n <gateway-namespace> -o yaml > plugins-backup.yaml
kubectl annotate maastenantconfig default-tenant -n <tenant-namespace> \
  maas.opendatahub.io/force-payload-processing-migration=true --overwrite
```

Known-good ConfigMaps do **not** need the force annotation when their active
ExternalModels are compatible. Applying force on a customized ConfigMap
replaces it with the Praxis plugins schema; re-apply any still-needed
customizations in the Praxis ConfigMap shape afterward. An override does not
establish parity for Azure, Vertex, Bedrock, OpenAI variants, streaming, or
other provider behavior without matching deployed evidence.
## Legacy metrics usage dashboard is opt-in

`Config.spec.usageMetricsDashboard` defaults to `false`. After upgrade, the operator removes any controller-owned copy of the Prometheus-based Perses dashboard (`dashboard-3-maas-usage-admin`). To keep it:

```bash
kubectl patch configs.maas.opendatahub.io default --type=merge \
  -p '{"spec":{"usageMetricsDashboard":true}}'
```

Prefer the logs-based usage dashboards. The operator deploys them with the usage-log resources. See [Setup](../observability/setup.md).
