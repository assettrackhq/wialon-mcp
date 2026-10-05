---
name: wialon
description: Answer questions about a Wialon fleet — where vehicles are, trips and mileage, fuel levels, fills and drains, driving style and speeding, idling, geofence visits, the nearest vehicle to a place, a what-changed digest, drivers, service intervals, saved reports and PDF/XLSX/CSV exports — through the FleetAI MCP server. Use whenever the task involves vehicles, units, drivers, telematics or a Wialon account; not for editing anything in Wialon, which these tools cannot do.
---

# Wialon fleet, through FleetAI

The `fleetai` MCP server reads one Wialon account. Every tool it exposes reads;
none of them changes anything in Wialon, so exploring is free — several tools,
or the same tool twice with different arguments, is the normal way to reach an
answer.

The server is `https://ai.theassettrack.com/mcp`. The user signs in through
their MCP client — a page where they pick their Wialon server, then Wialon's
own login — or the client sends a Wialon access token of theirs as
`Authorization: Bearer <token>`, with `?region=eu` (or `us`) on the URL for an
account not on `hosting.wialon.com`.

If none of the tools below is available, the server is not connected: point the
user to their client's MCP setup rather than improvising an answer. If a call
answers that the sign-in expired or the token is refused, ask them to sign in
again or check the token in that configuration. Never ask them to paste a token
into the chat or into a file in the repository.

## Which tool answers what

| The question                                   | The tool                    |
| ---------------------------------------------- | --------------------------- |
| How is the fleet doing, how many are online    | `get_fleet_status`          |
| Where is one vehicle                           | `get_vehicle_location`      |
| Where is everything                            | `get_all_vehicle_locations` |
| Who is low on fuel                             | `check_fuel_levels`         |
| Who drives what                                | `get_drivers`               |
| Everything about one unit                      | `get_vehicle_info`          |
| Trips, distance and driving time over a period | `analyze_trips`             |
| One vehicle's latest trips, with places        | `get_trip_summary`          |
| The route of one drive, as ordered positions   | `get_trip_track`            |
| Fleet-wide totals right now                    | `get_fleet_analytics`       |
| Service intervals                              | `get_maintenance_status`    |
| How much of the fleet is reporting now         | `get_utilization`           |
| Fuel filled, fuel drained, where it refuelled  | `get_fuel_events`           |
| Who drives worst, eco score, speeding          | `get_driving_behavior`      |
| Idling, engine hours                           | `get_idling`                |
| Who was at a geofence, how long on site        | `get_geofence_visits`       |
| Which vehicle is closest to a place            | `find_nearest_vehicles`     |
| Anything unusual, a briefing, period vs period | `get_fleet_digest`          |
| Anything else in Wialon                        | `wialon_api`                |

The report-backed tools — fuel events, driving, idling, geofence visits and
the digest — take the fleet by default, a `group` by name, or one `vehicleId`.
For the fleet they answer one row per vehicle; for one vehicle, its events with
time and place. A fleet-wide eco-driving report over weeks can take Wialon a
minute or more to compute; that is the report, not a hang.

`wialon_api` is the door to nineteen more operations — raw messages, sensors,
geofences, groups, notifications, saved and ad-hoc reports, report exports and
configuration audits. Call it with `operation` and no `args` and it answers
with that operation's own schema, so there is never a reason to guess arguments.

## Rules that decide whether the answer is right

**A reading is only current if the position behind it is.** Every vehicle
carries `positionAgeMinutes`, and sensors carry `health.asOfMinutesAgo`. Hours
or days old means "this is when it was last heard from" — never "it is doing
this now". A speed of 13 km/h recorded in April is not a moving truck.

**`null` means nobody measured it.** Report it as unknown. It is never zero, and
an unreadable fuel sensor is not an empty tank.

**Give a measurement the unit the data carries.** `fuelUnit` is litres or
percent depending on the account, and they are different facts. Mileage is
kilometres and engine hours are hours, as the account's own counters keep them.

**Relative dates belong to the tool.** `analyze_trips`, `get_trip_track` and the
report-backed tools take a named `period` — `today`, `yesterday`, `this_week`,
`last_week`, `last_7_days`, `last_30_days` — cut in the account's own timezone. For a date
no period names, pass `from` and `to` as `YYYY-MM-DD`; a bare date means the
whole of that day. Compute "last 7 days" yourself and you will report today's
driving under yesterday's name. `days` is a rolling look-back that ends now:
`days: 2` is the last 48 hours, not today and yesterday.

**State is three facts.** `contact` is `online`, `offline` (no message within
`offlineAfterMinutes`) or `no_data` (never reported). `motion` is `moving`,
`idling` (stopped, ignition on) or `parked`, and is `null` unless online.
`driver` is the assigned driver's name or `null`. `approximate` is true when
the position came from cell towers — never give it as an exact place. Say them
in words; never print a raw code, an internal unit id or a Wialon flag to the
user.

**Counting is the tool's job.** Use the `contact`, `motion` and `driver`
counts `get_fleet_status` returns rather than counting rows yourself. Motion
counts only online vehicles, so moving + idling + parked equals online.

**A report runs and exports in one call.** Wialon keeps one report result per
session: `wialon_export_report` executes the report and exports the file in the
same operation. Never run a report and export it as two steps — the export
would carry whatever ran in between.

**A route comes back as points.** `get_trip_track` answers up to 600
positions in order — both ends kept, the rest evenly thinned. Summarise the
drive (where it started, where it ended, how long, top speed) or draw it; do
not read the coordinates out one by one.

**An exported report comes back as a file.** The answer carries the document
itself as an embedded resource (base64, with its MIME type) — up to 5 MB; a
larger one is refused with a note to export a shorter interval or CSV. Save it
or hand it to the user; it is the same document Wialon's own Export button
produces.

**A drain is a suspicion, not a theft.** `get_fuel_events` reports a sudden
drop the fuel sensor saw. Sensor noise, a slope or temperature can cause one.
Give its time and place and suggest checking it; never accuse a driver.

**Money only at the user's price.** Idle hours, litres and drains become money
only when the user states a fuel price; otherwise answer in hours and litres.
Litres burnt idling are an estimate from a rate per hour — say so.

**Compare like with like.** A period still running (`today`, `this_week`)
against a whole previous one is not a comparison. `get_fleet_digest` compares
equal lengths and says `periodStillRunning`; when comparing two calls yourself,
use two whole periods or say what is partial. Give both the absolute and the
percentage change.

**CO₂ is an estimate.** When a tool gives `co2Kg`, Wialon computed it. Otherwise
estimate from fuel used — 2.68 kg per litre of diesel, 2.31 per litre of
petrol — or ask which fuel the fleet runs on, and call the result an estimate.

**A per-vehicle report lists only vehicles with data.** `vehiclesInScope` is
how many were asked about and `vehiclesWithData` how many Wialon had anything
for; the rest sent nothing in the period or lack the sensor. Say so rather than
counting them as zero.

**Say what is missing.** What the account's own access allows is what can be
seen. When something is out of reach, say what is missing and what access would
show it. Never fill a gap with a guess, and never invent a figure no tool gave.

## How to answer

**In the user's language.** Answer in the language of the question; keep
vehicle, driver and geofence names exactly as the account writes them.

**A place is a map.** In a client that supports MCP Apps (Claude, ChatGPT),
`get_vehicle_location`, `get_all_vehicle_locations`, `find_nearest_vehicles`
and `get_trip_track` draw an interactive map beside the answer by themselves.
Do not redraw it; say what it shows.

**A ranking is a table, a trend is a chart.** When the client can render them,
put a per-vehicle ranking in a table and a change over time in a chart. Always
state the period and the account's timezone the tool returned.

## Recipes

**"Which trucks are low on fuel?"** — `check_fuel_levels`. It flags a tank
below 20% where the sensor reads in percent; read `fuelUnit` before calling
anything low, and list units whose sensor is unreadable separately rather than
counting them as empty.

**"How far did each vehicle drive last week?"** — `analyze_trips` with
`period: 'last_week'` and no `vehicleId`. For a document the user keeps,
`wialon_api` → `wialon_export_report` with a template id from
`wialon_list_saved_reports` and `dateFrom`/`dateTo`.

**"Show me Friday's route for Van 023"** — `get_trip_summary` or
`analyze_trips` with `from` and `to` set to that date, to find the drive, then
`get_trip_track` for the same window.

**"Which services are due?"** — `get_maintenance_status`. Each interval says
how far it has run against its limit, in kilometres or days, and whether it is
overdue or due soon.

**"How much did we fill last week, and was any fuel stolen?"** —
`get_fuel_events` with `period: 'last_week'`. For the vehicles with drains,
call again with `vehicleId` for each event's time, place and tank levels.

**"Who are my worst drivers this month?"** — `get_driving_behavior` with
`period: 'last_30_days'`. Rank by `rank` (0–10, higher is better) and
`violationsPer100`; for the worst few, call again with `vehicleId` for their
worst events with places. `speedings` stay 0 where no speed limit is set in the
unit's settings, even when eco-driving counts speeding.

**"How much are we losing to idling?"** — `get_idling`. Report idle hours and
share per vehicle; turn it into litres or money only with a rate and a price
the user gives.

**"Who was at the Örebro depot yesterday, and for how long?"** —
`get_geofence_visits` with `geofence: 'Örebro depot'` and `period: 'yesterday'`.
If several geofences match, the answer lists them with ids — ask which is meant.

**"Which truck is closest to Kungsgatan 1?"** — `find_nearest_vehicles` with
`address`. Distances are straight lines, not roads; say so, and mention the
position's age. Positions over a day old are left out unless `includeStale`.

**"Anything unusual?" / a morning briefing** — `get_fleet_digest` (yesterday
against the day before, by default). Lead with what changed most, drains,
vehicles gone silent and service overdue; skip sections with nothing in them.
For a briefing every morning, set up the client's scheduled task (Claude
scheduled tasks, ChatGPT Tasks) to ask for it — this server sends nothing on
its own.

**"This week against last week"** — `get_fleet_digest` with
`period: 'last_week'` compares it with the week before. For two arbitrary
periods, call `analyze_trips` twice with `from`/`to`.

**"What was our CO₂ last month?"** — `get_fleet_digest` with `from` and `to`
for the month, then `co2Kg` if Wialon computed it, else fuel used × the factor
for the fleet's fuel, labelled as an estimate.

**"Is the account set up properly?"** — `wialon_api` →
`wialon_audit_notifications` and `wialon_audit_unit_settings`. They report
alerts that are switched off or cover nothing, units with no fuel sensor or no
trip detector, and units that stopped reporting.

**A question that spans several units and a period** — run the fleet-wide tool
first and only drill into single units for the ones that matter. Ninety-six
per-unit calls is a slow answer and a large bill.

More detail — the operation catalogue, report exports and the traps worth
knowing — is in `reference.md` beside this file.
