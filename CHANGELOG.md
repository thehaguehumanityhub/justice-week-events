# Changelog

Versions cover the code: the widget (`events-embed.html`), the script
(`template_to_events.py`) and the Excel submission template they read.
Programme updates to `events.json` are not versioned; `git log events.json`
is their history.

Each version is tagged in git (`v1.1.0`), so an older widget can be served
from jsDelivr by tag if a release ever needs rolling back:
`https://cdn.jsdelivr.net/gh/thehaguehumanityhub/justice-week-events@v1.0.0/events-embed.html`

## 1.1.0 — 2026-09-25

Widget
- Multi-day activities. An optional `end_date` shows a run as "16–20 Nov" with
  the weekdays beneath, and "Mon 16 – Fri 20 Nov" in the expanded panel. It sorts
  after that first day's single events. Programmes without `end_date` render as
  before.
- Version shown as `data-version` on the widget's root element, so the live
  page can be checked against this file.

Script
- Reads an optional "End date" column.
- Refuses to write a programme with fewer activities than the current
  `events.json`, listing what would disappear. `--allow-fewer` overrides.
- `--version` prints the version.

Template (`THJW-2026-activity-template-v2.xlsx`)
- New "End date" column after "Date".
- Two new topics: Artificial Intelligence, Anti-Corruption.

## 1.0.0 — 2026-09-16

First release: searchable, filterable, sortable programme widget loading
`events.json` from jsDelivr; `template_to_events.py` for the Excel template;
`gen_events.py` for the Google Form export.
