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

## Current state, as of 24 September 2026

- `events.json` on `main` holds **4 activities**, dated 2026-09-16. These are the
  original sample rows.
- Giovanni generated **11 activities** from colleague submissions on 23 September,
  but that commit never landed: git stopped on an unset author identity, and the
  subsequent push prompted for HTTPS credentials, which GitHub no longer accepts.
  Check `git status` and `git log` on his machine before assuming anything is
  missing upstream. The work is probably staged and uncommitted, not lost.
- Of those 11: three had no date, five had no registration link. Both are
  expected states and publish safely.
- **The widget is not on any page yet.** The `jw-programme` shortcode has not been
  added in WordPress. Only the organisations grid from the sibling repo is live.
  Publishing programme updates achieves nothing visible until that is done.
- Git authentication on at least one MacBook is unresolved. `gh auth login` is
  the recommended fix, since it configures the credential helper once, on each
  machine separately.

## Open questions worth raising

Whether the programme should eventually come from a live Google Sheet rather
than emailed spreadsheets. If it does, an n8n job on Giovanni's Unraid server
could pull, regenerate, commit, push and purge on a schedule, and the page would
stay current with nobody opening a terminal. Worth building only once the data
source stops being "people email me files".

Two typos exist in the intake vocabulary, corrected on display by the scripts:
*Authoritarism* to Authoritarianism, and *Policmakers* to Policymakers. Fixing
them at the source removes the patch.
