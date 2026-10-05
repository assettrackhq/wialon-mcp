# Wialon through FleetAI — reference

Everything here is checked against what the tools do today. When the two
disagree, the tool is right.

## The nineteen operations behind `wialon_api`

Call `wialon_api` with `operation` and no `args` to be handed that operation's
own schema.

**The account**

- `wialon_check_connection` — is the session alive
- `wialon_search_units` — find units by name (`*` is a wildcard)
- `wialon_list_groups` — unit groups, with the ids of their members
- `wialon_list_resources` — resources, with their report templates and notifications
- `wialon_get_account_snapshot` — units, groups and resources in one call

**One unit, in depth**

- `wialon_get_unit_briefing` — position, counters, sensors, fault codes and findings in one call
- `wialon_get_sensors_now` — the current calculated sensor values
- `wialon_get_last_messages` — the raw messages behind those values
- `wialon_get_messages_interval` — raw messages over a period

**Geofences and alerts**

- `wialon_list_geofences` — pass `query` to narrow by name; accounts hold thousands
- `wialon_list_notifications`

**Reports**

- `wialon_list_saved_reports` — the account's own templates, with their ids
- `wialon_run_saved_report` — run one of them
- `wialon_run_temporary_report` — build one the account never saved
- `wialon_get_report_catalog` — the table names an ad-hoc report can use
- `wialon_fetch_more_report_rows` — the next page of a result
- `wialon_export_report` — run and export in one call; answers with the document attached

**Audits**

- `wialon_audit_notifications` — alerts that are switched off or cover no unit
- `wialon_audit_unit_settings` — one unit's sensors, trip detector and last report

Unit-level operations take `unitId` or `unitName`. Dates are `dateFrom` and
`dateTo` as `YYYY-MM-DD` on the account's clock; a bare `dateTo` means the whole
of that day.

## Report exports

Formats: `pdf` (the default), `xlsx`, `csv`. The document is rendered by Wialon
itself, so it looks exactly like the account's own export.

- Give `templateId` for a saved report, or `tableName` for one the account has
  not saved (`wialon_get_report_catalog` lists the names, e.g. `unit_trips`,
  `unit_speedings`, `unit_stays`).
- `resourceId` and `objectId` are required: the resource that owns the template
  and the unit or group to run it over.
- `landscape: true` for a PDF with many columns; `fileName` names the file in
  the user's language.
- An empty interval answers `rows: 0` and attaches no file — say so rather than
  handing over an empty document.

## Traps

**Fault codes depend on the tracker.** The unit briefing carries `faultCodes`
only when the tracker sends a parameter named for them (`can_dtc`, `obd_dtc`,
`dtc_count`, …). A basic GPS tracker sends none — say the vehicle does not
report them, not that it has no faults.

**Eco-driving needs its criteria.** A score exists only where the account set
up eco-driving criteria for the unit; without them `rank` is null.

**One report result per session.** Wialon holds a single result. Running a
second report discards the first, which is why running and exporting is one
operation and never two.

**Raw messages are expensive.** A fleet-week of them is megabytes. Ask for one
unit and a bounded interval, and prefer `wialon_get_sensors_now` when the
question is about now. Raw parameters are named by the tracker's firmware
(`io_239`, `pwr_ext`); a sensor is the account's own name for one of them, so
quote sensors to the user and parameters only when asked.

**A trip endpoint may carry no address.** Coordinates are not an address. Say
"59.2182, 15.1377" as coordinates rather than guessing a place.

**A driver's binding is not reliable.** Many accounts hold bindings that name no
unit the token can see. Report the vehicle name a tool resolved, and say "not
assigned" when nothing resolves.

**A service limit of zero is switched off.** `get_maintenance_status` leaves
such a limit out, and a service never marked as done has no days count. An
interval can still be overdue by tens of thousands of kilometres when nobody
has recorded a service since the counter started — say that, rather than
calling the vehicle neglected.

**A dropped session is retried once.** Wialon ends a session after a spell of
inactivity; the server signs in again with the token and retries on its own. If
it still reports the connection to Wialon dropped, ask the user to check that
the token is still valid — retrying the same call will not help.

## Regions

A token is minted at one data centre and is meaningless at the others. A
sign-in through the client carries the server the user picked. A token sent as
it is goes where `?region=com|eu|us` on the URL says, and the server refuses a
value it does not know; if every call fails at once with an authentication
error, the region is the first thing to check.

## Privacy

The token is the identity. The server holds it only for the request it came
with, keeps the session it signs in to for a few idle minutes, and sends both
only to Wialon's own hosts. It never appears in an answer — if one ever seems
to contain it, that is a bug worth reporting.
