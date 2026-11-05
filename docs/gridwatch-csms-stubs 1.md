# GridWatch — Vendor CSMS Interface & Stubs

This is everything you need to integrate Part C against the vendor CSMS **without a running vendor service.** It restates the interface contract and gives you canned payloads for the normal, error, and quirk cases. Wire your client to these shapes. If a local fake helps you test, build a trivial one — but you do not need one to complete the assignment, and we are not providing one.

> Read the field notes at the bottom of `pilot-sample-data.json` first. The quirks below are not bugs in these stubs — they are how the real feed behaves.

---

## 1. Poll API

```
GET /v1/chargers?site_id={site_id}&since={iso8601}&cursor={opaque}
```

- **Minimum poll interval: 30 seconds per site.** Polling faster returns `429`.
- Paginated via an opaque `cursor`. Absence of `next_cursor` means you've drained the page set.
- `since` is your high-water mark; the server returns events at or after it. The server does **not** guarantee you won't see an event twice across calls — design your cursor handling accordingly.

### 1a. Success — first page (has more)

```json
{
  "site_id": "S-IN-0007",
  "server_time": "2026-06-09T14:32:25+05:30",
  "next_cursor": "eyJvZmZzZXQiOjUwfQ==",
  "events": [
    { "type": "status", "charger_id": "C-IN-0007-A", "connector_id": "1", "status": "Charging", "ts": "2026-06-09T14:32:11+05:30" },
    { "type": "meter_value", "charger_id": "C-IN-0007-A", "connector_id": "1", "energy_register_wh": 4821553, "power_w": 41200, "ts": "2026-06-09T14:32:11+05:30" },
    { "type": "meter_value", "charger_id": "C-IN-0007-A", "connector_id": "1", "energy_register_wh": 4821667, "power_w": 41050, "ts": "2026-06-09T14:32:21+05:30" },
    { "type": "status", "charger_id": "C-IN-0007-B", "connector_id": "1", "status": "Available", "ts": "2026-06-09T14:32:09+05:30" }
  ]
}
```

### 1b. Success — final page (no `next_cursor`)

```json
{
  "site_id": "S-IN-0007",
  "server_time": "2026-06-09T14:32:25+05:30",
  "events": [
    { "type": "fault", "charger_id": "C-IN-0007-A", "connector_id": "2", "code": "OverTemperature", "severity": "warning", "ts": "2026-06-09T14:30:55+05:30" }
  ]
}
```

### 1c. Rate limited (`HTTP 429`)

```json
{ "error": "rate_limited", "retry_after_seconds": 30 }
```
Sent with a `Retry-After: 30` header when you poll a site more than once per 30 s.

### 1d. Server error (`HTTP 500`)

```json
{ "error": "internal", "request_id": "req-7f3a9c" }
```
Occurs intermittently under load. Transient — the same request usually succeeds on retry.

### 1e. Malformed page (real, occasional)

A page that is valid JSON but **violates the contract** — a missing field, a null where a number is expected, a truncated event. Your ingester must not crash the whole batch on one of these. Example:

```json
{
  "site_id": "S-DE-0003",
  "server_time": "2026-06-09T23:15:02+02:00",
  "events": [
    { "type": "meter_value", "charger_id": "C-DE-0003-A", "connector_id": "1", "energy_register_wh": null, "power_w": 0, "ts": "2026-06-09T23:14:58+02:00" },
    { "type": "status", "charger_id": "C-DE-0003-A", "ts": "2026-06-09T23:14:58+02:00" }
  ]
}
```
(First event: `energy_register_wh` is null. Second: `connector_id` and `status` are missing.)

---

## 2. Webhook (push)

You register a URL; the CSMS POSTs event batches to it.

```
POST {your_registered_url}
Header: X-CSMS-Signature: {hmac-sha256 of body, shared secret}
```

- Delivery is **best-effort, at-least-once.** Expect duplicates.
- Events may arrive **out of order** and may be **batched up to 60 seconds.**
- You must respond `2xx` quickly. If you are slow or error, the CSMS **redelivers the whole batch.**

### 2a. Normal batch

```json
{
  "delivery_id": "dlv-001",
  "events": [
    { "type": "session", "event": "session.start", "charger_id": "C-DE-0003-A", "connector_id": "1", "session_id": "sess-de-99x", "start_meter_wh": 9930120, "ts": "2026-06-09T22:41:03+02:00" },
    { "type": "session", "event": "session.stop", "charger_id": "C-DE-0003-A", "connector_id": "1", "session_id": "sess-de-99x", "start_meter_wh": 9930120, "stop_meter_wh": 9971540, "ts": "2026-06-09T23:14:58+02:00" }
  ]
}
```

### 2b. Duplicate delivery (at-least-once)

A redelivery of the **same** `session.stop` as above, with a different `delivery_id`. Your settlement path must not double-bill on this.

```json
{
  "delivery_id": "dlv-002",
  "events": [
    { "type": "session", "event": "session.stop", "charger_id": "C-DE-0003-A", "connector_id": "1", "session_id": "sess-de-99x", "start_meter_wh": 9930120, "stop_meter_wh": 9971540, "ts": "2026-06-09T23:14:58+02:00" }
  ]
}
```

### 2c. Out-of-order batch

The `session.stop` arrives **before** the matching `session.start`, and a stale `status` lands after a newer one. Your state must converge correctly regardless of arrival order.

```json
{
  "delivery_id": "dlv-003",
  "events": [
    { "type": "session", "event": "session.stop", "charger_id": "C-IN-0007-A", "connector_id": "1", "session_id": "sess-in-44a", "start_meter_wh": 4810000, "stop_meter_wh": 4821667, "ts": "2026-06-09T14:32:21+05:30" },
    { "type": "status", "charger_id": "C-IN-0007-A", "connector_id": "1", "status": "Available", "ts": "2026-06-09T14:33:40+05:30" },
    { "type": "status", "charger_id": "C-IN-0007-A", "connector_id": "1", "status": "Charging", "ts": "2026-06-09T14:31:50+05:30" },
    { "type": "session", "event": "session.start", "charger_id": "C-IN-0007-A", "connector_id": "1", "session_id": "sess-in-44a", "start_meter_wh": 4810000, "ts": "2026-06-09T14:25:02+05:30" }
  ]
}
```
(Correct final status is `Available` at 14:33:40, not `Charging` at 14:31:50 — newest wins, not last-received.)

### 2d. Meter reset within a session (correctness hazard)

A connector whose energy register **reset/replaced** mid-session: `stop_meter_wh` < `start_meter_wh`. Naive `stop - start` gives a negative figure. Decide what your billing does.

```json
{
  "delivery_id": "dlv-004",
  "events": [
    { "type": "session", "event": "session.stop", "charger_id": "C-IN-0012-A", "connector_id": "1", "session_id": "sess-in-77c", "start_meter_wh": 12044990, "stop_meter_wh": 31200, "ts": "2026-06-09T18:40:12+05:30" }
  ]
}
```

---

## 3. How to use these

- Build your CSMS client against these payload shapes. Drive your tests with them — especially 1c/1d/1e and 2b/2c/2d, which are the cases that separate a robust ingester from a happy-path one.
- The pilot site/charger catalogue (and more telemetry) is in `pilot-sample-data.json`. The full pilot is 50 sites / ~220 chargers; the principles are identical at that size.
- Nothing here carries coordinates. Resolving `address` → lat/lng is your problem to design (it is for the map).
