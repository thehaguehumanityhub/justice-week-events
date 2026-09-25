# justice-week-events

Programme listing for The Hague Justice Week 2026, embedded on
`humanityhub.org/the-hague-justice-week/`.

Owner: Giovanni, Communications and Marketing Manager, The Hague Humanity Hub.
He works from two MacBooks. This repo is the only shared state between them.
Nothing lives outside git except the filled-in `.xlsx` submissions, which are
inputs and are deliberately never committed.

## What is in here

| File | Role |
|---|---|
| `events-embed.html` | The widget. Self-contained HTML fragment, scoped CSS, no dependencies. Changes rarely. |
| `events.json` | The programme. Changes constantly. This is the file you almost always want. |
| `template_to_events.py` | Turns filled-in Excel templates into `events.json`. The normal path. |
| `gen_events.py` | Turns a Google Form `.xlsx` export into `events.json`. Legacy, kept in case the form is used again. |
| `PROGRAMME.md` | Human-facing notes: the Shortcoder snippet and the update ritual. |
| `CHANGELOG.md` | What changed in each version of the widget, script and template. |

Sibling repo `thehaguehumanityhub/thjw` holds the co-hosting organisations logo
grid, a separate widget on the same page. Same delivery method, different data.

## How it reaches the website

GitHub, then jsDelivr, then the Shortcoder WordPress plugin. No iframes, and no
file uploads into WordPress, because WordPress strips `<script>` from page
content and rejects `.html` uploads. Shortcoder stores its snippets outside page
content, so scripts survive.

The Shortcoder snippet (named `jw-programme`) fetches `events-embed.html` from
jsDelivr, injects it, and re-creates its `<script>` tags so they execute.
`innerHTML` alone does not run injected scripts. The widget then fetches
`events.json` itself.

**The repo must stay public.** jsDelivr will not serve a private repo.

## Architecture, and the one thing to not undo

Code and data are separate files on purpose. The Hub's embeddable-dashboards
playbook says to bake data into the fragment, and that is right for a dashboard
whose data is final on publication. It is wrong here: the programme changes most
days until November, and baking it in would mean regenerating and committing a
20 KB code file to correct one start time, with no readable diff.

The cost of the split is accepted knowingly: a second point of failure if
`events.json` fails to load (handled with a graceful message rather than a blank
widget), and two purges instead of one.

`events-embed.html` contains a `ROOT` constant holding the jsDelivr base URL.
**If the repo is ever renamed or moved, that constant must change too.** Missing
it produces a widget that loads and renders its chrome but stays empty, because
it is still looking for `events.json` at the old address. This has already caught
us once.

## The update loop

```bash
cd ~/Developer/justice-week-events
git pull                                          # someone may have edited on github.com
python3 template_to_events.py ~/Downloads/thjw/*.xlsx
git diff --stat                                   # only events.json should differ
git add events.json && git commit -m "Update programme" && git push
open "https://purge.jsdelivr.net/gh/thehaguehumanityhub/justice-week-events@main/events.json"
```

Then hard-refresh the page.

`template_to_events.py` writes `events.json` into the current working directory,
so run it from the repo root. It accepts several files at once and merges them,
sorted by date. It skips rows still holding the template's grey example row. It
prints which activities lack a date and which lack a registration link: that
output is the chase list, and is worth surfacing to Giovanni rather than
swallowing.

It rebuilds the programme from scratch on every run, so it must be given every
file, not just the newest. As a guard, it refuses to write a programme with
fewer activities than the current `events.json` and lists what would vanish.
`--allow-fewer` overrides that when activities are being removed on purpose.

**The purge is not optional.** jsDelivr holds a branch URL for twelve hours at
the edge and seven days in the visitor's browser. Without the purge an update
simply does not appear, and the page looks broken rather than stale. This is the
single most common way an update appears to have failed.

## Data contract for events.json

```json
{ "updated": "2026-09-16",
  "events": [ { "id": "e01", "title": "...", "host": "...", "date": "2026-11-17", "end_date": "",
                "start": "15:00", "end": "16:30", "venue": "", "format": "Online",
                "tags": ["..."], "audience": ["..."], "description": "...",
                "register": "https://...", "contact": "info@example.org" } ] }
```

Only `id` and `title` are required. Everything else degrades deliberately:

- No `date`: renders "Date to follow" and sorts to the bottom **in both
  directions**, rather than floating to the top when the sort reverses.
- No `end_date`, or one not after `date`: a single-day activity. A later
  `end_date` makes it a run, shown as "16–20 Nov" with the weekdays beneath, and
  sorted after that first day's single events. The template's optional
  "End date" column feeds it.
- No `host`: renders "Host to be confirmed" in italics.
- No `register`: renders a dashed, non-clickable "Registration opens soon"
  rather than a dead button. Only a real URL becomes a button; an email address
  in that field does not.
- Filter chips for `format` and `tags` are generated from whatever values are
  present, so a new topic needs no code change. The corollary: a typo in a tag
  silently creates a stray filter chip. Check the tag list after regenerating.

## Rules that are not negotiable

**Never publish personal contact details.** The intake collects each host's
personal email and phone number. Neither script copies them and neither belongs
in this repo, which is public twice over, on GitHub and on the CDN. The only
contact that reaches the page is the field hosts filled in as public-facing, and
even there a shared inbox is preferred over an individual's address.

**Never commit the raw submissions.** The `.xlsx` files stay outside the repo,
in `~/Downloads/thjw/`.

**Do not put a clone in iCloud Drive, Dropbox or Google Drive.** Sync clients
corrupt `.git`. The Cowork folder Giovanni connects is in iCloud; this repo
deliberately is not.

## Versioning

The code carries a semantic version: `events-embed.html` (a header comment and
`data-version` on the root element), `VERSION` in `template_to_events.py`
(`--version` prints it) and `CHANGELOG.md`. Keep all three in step. Each
release is a git tag (`v1.1.0`), pushed with `git push --tags`.

Bump the version when the widget, the script or the Excel template changes:
patch for fixes, minor for anything new but backward compatible (1.1.0 added
`end_date`), major if an old `events.json` or an old template would stop
working. Programme updates to `events.json` are never versioned; `git log
events.json` is their history.

After a widget release, purge `events-embed.html` as well as `events.json`. To
check which widget a visitor is getting, inspect `#thjw-events` on the live page
and read its `data-version`.

## Submissions: one master file

The script reads `~/Downloads/thjw/*.xlsx`, and today that folder holds a single
master workbook, `events calendar template.xlsx`, with every activity. Keep it
that way. New submissions arrive on the v2 template
(`~/Downloads/THJW-2026-activity-template-v2.xlsx`); their rows are copied into
the master and the colleague's copy is archived, not left in the folder, where
it would duplicate activities. Small edits arrive by email in plain words and
are made in the master. Never edit `events.json` by hand: the next run
overwrites it.

**Edit the master through Excel, not openpyxl.** The workbook's dropdowns are
stored as an Excel extension that openpyxl cannot read and silently deletes on
save. Driving Excel with `osascript` keeps them; openpyxl is fine for reading.
Back the file up before editing it, since it is not in git.

The master's Topics dropdown still has the original 15 topics. The v2 template
adds Artificial Intelligence and Anti-Corruption; add them to the master's
Lists sheet (and widen the Topic 1 to 3 validation range) before using them there.

## Current state, as of 25 September 2026

- Widget and script are at **1.1.0**. `events.json` holds **11 activities**,
  all dated. The Kunstmuseum exhibition runs 16 to 20 November. NowHere's
  screenings #2 and #3 fall on 21 and 22 November, after Justice Week, and
  that is intended.
- Chase list: five activities still lack a registration link (Advisory
  Committee, ECNL, HiiL, Legal Action Worldwide, Kunstmuseum).
- Giovanni reported the WordPress embed working on 24 September. Confirm the
  `jw-programme` shortcode is on the live Justice Week page, not only a draft,
  before relying on it.
- The MacBook with `~/Developer/justice-week-events` is authenticated through
  `gh` as `thehaguehumanityhub`, with a noreply commit email. The other MacBook
  is not yet: it needs `gh auth login` and `git config --global user.name` and
  `user.email`. Its clone holds a staged 11-activity `events.json` from
  23 September that is now obsolete: discard it
  (`git restore --staged events.json && git checkout events.json`) and
  `git pull`, rather than committing it.

## Open questions worth raising

Whether the programme should eventually come from a live Google Sheet rather
than emailed spreadsheets. If it does, an n8n job on Giovanni's Unraid server
could pull, regenerate, commit, push and purge on a schedule, and the page would
stay current with nobody opening a terminal. Worth building only once the data
source stops being "people email me files".

Anti-Corruption overlaps with Corruption & Rule of Law, and Artificial
Intelligence with Technology & Digital Rights. Hosts will split between them,
and so will the filters. Worth merging or renaming before many submissions use
them.

The legacy Google Form had two typos in its vocabulary, *Authoritarism* and
*Policmakers*, which `gen_events.py` corrects on the way in. The Excel template
spells both correctly, so `template_to_events.py` needs no such patch.
