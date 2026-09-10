# Koi Security API Collectors

Pulls **Koi Security** data into Cribl Stream on a schedule via the Koi external REST API (`https://api.prod.koi.security/api/external/v2`). The pack ships three REST Collectors, each with its own event breaker, empty pipeline, and route, sharing one API base variable and one group secret:

| Collector | Endpoint | What it collects | Schedule (ships disabled) |
|---|---|---|---|
| `vct_koi_alerts` | `GET /alerts` | Alerts as OCSF finding events, incremental | every 15 min |
| `vct_koi_policies` | `GET /policies` | Full policy list, snapshot | every 6 h |
| `vct_koi_agent_activity_events` | `GET /agent-activity/events` | AI coding-agent activity events, incremental | every 15 min |

## About this Pack

### Alerts (`vct_koi_alerts`)

- Polls `GET /alerts`, which returns alerts as **OCSF** finding events (OCSF v1.3.0). The docs describe them as Detection Findings (`class_uid` 2004), but live tenants currently return Application Security Posture Findings (`class_uid` 2007). One event per alert; the unique alert id is `metadata.notification_event_id`.
- Pages 500 alerts at a time with `page` / `page_size` (the API maximum), using `total_count` to know when to stop. Results are requested oldest first (`sort_direction=asc`).
- Each run asks for alerts with `created_at_gte` set to the last collected alert time (collector state tracking), and the run filter `_time > __collectible.state.latestTime` drops the boundary record, so overlapping runs do not duplicate.
- `_time` comes from the alert `time` field (epoch milliseconds).
- Events are tagged `datatype='koi_alert'`, `sourcetype='koi:alert'`, and `source='koi:alerts'`.

### Policies (`vct_koi_policies`)

- Polls `GET /policies` and emits the **whole policy list** every run (snapshot). There is no time filter on this endpoint and no state tracking; the collector's time filter is disabled so nothing is dropped.
- `_time` is the collection time. `created_at` and `updated_at` stay as fields on each event.
- Events are tagged `datatype='koi_policy'`, `sourcetype='koi:policy'`, and `source='koi:policies'`.

### Agent activity events (`vct_koi_agent_activity_events`)

- Polls `GET /agent-activity/events`. The API **requires** `created_at_gte` and `created_at_lte`, the window may not exceed **24 hours**, and the upper bound may not be in the future.
- `created_at_lte` is the run's `latest`, capped at now. `created_at_gte` is the last collected event time (state tracking) or the run's `earliest`, and is **clamped to 24 hours before `created_at_lte`** so the request is always valid. If the collector is off for more than 24 hours the gap is skipped; backfill it with ad hoc runs of 24 hour windows after clearing the collector state.
- `_time` comes from the event `timestamp` (ISO 8601).
- Events are tagged `datatype='koi_agent_activity_event'`, `sourcetype='koi:agent_activity_event'`, and `source='koi:agent-activity-events'`.
- The related `GET /agent-activity/sessions` endpoint (per-session rollups, 30 day window) is not collected; copy this collector and change the path and breaker if you want it.

### Shared behaviour

- All collectors authenticate with `Authorization: Bearer <API key>`, read from a group text secret at run time with `C.Secret()`.
- Pipelines are intentionally empty. Add flattening or enrichment downstream or in the pipelines as you see fit.
- Koi enforces **30 requests per minute per route** and returns HTTP 429 when exceeded. Every collector retries 429 and 503 with exponential backoff (2 s, doubling, up to 60 s, 5 attempts) and honours `retry-after`. At 500 records per page this is far more than a 15 minute schedule will ever need.

## Deployment

### 1. Create an API key (Koi side)

In the Koi console go to **Settings > API Access > Create New API Key**. You need the `xt-Administrator` role. Keys can expire after up to one year; Koi emails the key owner 30, 7, and 3 days before expiry.

Verify before configuring the pack (needs curl). On Windows Git Bash add `--ssl-no-revoke` if curl reports `CRYPT_E_NO_REVOCATION_CHECK`.

```bash
export KOI_TOKEN='<your API key>'
BASE='https://api.prod.koi.security/api/external/v2'
curl -sS -w '\nHTTP %{http_code}\n' -H "Authorization: Bearer $KOI_TOKEN" -H 'Accept: application/json' "$BASE/alerts?page=1&page_size=1"
curl -sS -w '\nHTTP %{http_code}\n' -H "Authorization: Bearer $KOI_TOKEN" -H 'Accept: application/json' "$BASE/policies?page=1&page_size=1"
SINCE=$(date -u -d '23 hours ago' +%Y-%m-%dT%H:%M:%SZ); NOW=$(date -u +%Y-%m-%dT%H:%M:%SZ)   # macOS: date -u -v-23H ...
curl -sS -w '\nHTTP %{http_code}\n' -H "Authorization: Bearer $KOI_TOKEN" -H 'Accept: application/json' "$BASE/agent-activity/events?created_at_gte=$SINCE&created_at_lte=$NOW&page=1&page_size=1"
```

A 401 means the key is wrong or expired. A 200 with the expected array (`alerts`, `policies`, `data`) and `total_count` means you are ready.

### 2. Create the group secret (Cribl side)

Packs cannot contain secrets. In the Worker Group, create a **text secret** with id `vct-koi-rest-secret` holding the API key. The id must match the `vct_koi_secret_id` pack variable.

### 3. Set the pack variables

| Variable | Value |
|---|---|
| `vct_koi_api_base` | `'https://api.prod.koi.security/api/external/v2'` |
| `vct_koi_secret_id` | `'vct-koi-rest-secret'` (the group text secret holding the API key) |

Variable values are JavaScript expressions, so keep the single quotes around strings.

### 4. Point the routes at a destination

The pack routes `vct_koi_alerts`, `vct_koi_policies`, and `vct_koi_agent_activity_events` match on `datatype` and send to `devnull` by default. Change the output of each to your SIEM or Lake destination.

### 5. Run once, then enable the schedules

Open each collector inside the pack and click **Run** in preview mode. For alerts use a wide time range (for example earliest `-30d`) to backfill; for agent activity keep the range under 24 hours. Confirm events appear with the expected `_time`, then enable the schedules, commit, and deploy.

### Collector settings worth knowing

- `created_at_gte` and `created_at_lte` are inclusive ISO 8601 timestamps. The collectors guard against an undefined `earliest` / `latest` (ad hoc runs from the UI drawer) by falling back to the last hour.
- Add `alert_type` (`system` or `policy`) as an extra request parameter on the alerts collector if you want only one class of alert. The pack collects both.
- The agent-activity collector can be narrowed with the optional `session_id` parameter.

## Upgrades

Upgrading in place keeps your pack variables and route outputs. If you add your own functions to a pack pipeline, copy the pipeline rather than editing it so upgrades do not overwrite your changes.

## References

- Koi API, getting started and authentication: https://docs.koi.ai/api-reference (login required)
- Alerts endpoint: https://docs.koi.ai/api-reference/reference/alerts
- Policies endpoint: https://docs.koi.ai/api-reference/reference/policies
- Agent activity endpoint: https://docs.koi.ai/api-reference/reference/agent-activity

## Authors

- Andrew Hendrix <Andrewh@visicoretech.com>
- VisiCore Tech <CriblPacks@VisiCoreTech.com>

## Release Notes

### Version 0.1.0 - 2026-09-10

- Initial release: alerts, policies, and agent-activity REST collectors, three event breakers, empty pipelines, placeholder routes.
