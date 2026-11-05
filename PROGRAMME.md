# Programme widget (`events-embed.html`)

A searchable, sortable, expandable list of Justice Week activities, embedded on
`humanityhub.org/the-hague-justice-week/`.

Repo: **`thehaguehumanityhub/justice-week-events`** (must be public, jsDelivr will not serve a
private repo). Its sibling `thehaguehumanityhub/thjw` holds the co-hosting organisations grid.

Same embed route as that grid, and as the Hub's embeddable-dashboards playbook: a
self-contained fragment on GitHub, served through jsDelivr, injected by Shortcoder.

## The important difference

**The programme is not baked into the fragment.** It lives in `events.json` at the repo root.
To update the listing, replace that one file and purge it. `events-embed.html` never changes.

## Shortcoder snippet

```html
<div id="thjw-ev-host">Loading the programme…</div>
<script>
(function(){
  var URL = "https://cdn.jsdelivr.net/gh/thehaguehumanityhub/justice-week-events@main/events-embed.html";
  fetch(URL, { cache: "no-cache" }).then(function(r){ return r.text(); }).then(function(html){
    var host = document.getElementById("thjw-ev-host");
    host.innerHTML = html;
    host.querySelectorAll("script").forEach(function(old){
      var s = document.createElement("script");
      if (old.src) s.src = old.src; else s.textContent = old.textContent;
      old.parentNode.replaceChild(s, old);
    });
  }).catch(function(){
    document.getElementById("thjw-ev-host").textContent = "Could not load the programme.";
  });
})();
</script>
```

Name the Shortcoder snippet `jw-programme`, then put this on the page:

```
[sc name="jw-programme"]
```

## Updating the programme

1. Export the Google Form responses as `.xlsx`.
2. Ask Claude to regenerate, or run `python3 gen_events.py <export.xlsx>` yourself.
3. Commit the new `events.json`.
4. Purge, then hard-refresh the page (Cmd+Shift+R):

   ```
   https://purge.jsdelivr.net/gh/thehaguehumanityhub/justice-week-events@main/events.json
   ```

   **This is the step that gets forgotten.** jsDelivr holds a branch URL for 12 hours at the
   edge, so without the purge your change simply will not appear and the page looks broken
   rather than stale. Only purge `events-embed.html` as well if you changed the component
   itself, which should be rare.

## What `events.json` looks like

```json
{
  "updated": "2026-09-16",
  "events": [
    {
      "id": "e01",
      "title": "Justice has no Passport",
      "host": "Baltasar Garzón International Foundation",
      "date": "2026-11-17",
      "end_date": "",
      "start": "15:00",
      "end": "16:30",
      "venue": "",
      "format": "Online",
      "tags": ["International Law & Accountability", "Access to Justice"],
      "audience": ["General Public", "Legal Professionals"],
      "description": "Free text. Line breaks are preserved.",
      "register": "https://wkf.ms/4yw1aqI",
      "contact": "contacto@fibgar.org"
    }
  ]
}
```

Every field is optional except `id` and `title`. Empty values degrade gracefully:
`end_date` is only for activities running over several days, such as an exhibition: set it to the
last day and the row reads "16–20 Nov". Leave it empty for a single-day activity. No `date` shows "Date to follow" and sorts to the bottom in both directions, no `host`
shows "Host to be confirmed", no `register` shows a dashed "Registration opens soon"
button in place of the live one. You can hand-edit this file for one-off corrections.

The filter chips build themselves from whatever `format` and `tags` values are present,
so new topics need no code change.

## What is deliberately NOT in this file

The form collects the activity owner's personal e-mail and phone number. Those are not
published, and `gen_events.py` does not copy them. The only contact that reaches the page
is the **Activity Public-Facing Contact Information** field, which is what hosts filled in
for that purpose.

## Known data issues in the form

- **Name of Organisation** is empty on every response so far. That field drives the host
  column. Until it is filled, `gen_events.py` falls back to Activity Owner Name only when
  that value looks like an organisation rather than a person.
- Two option labels contain typos, corrected on display by `gen_events.py`:
  *Authoritarism* → Authoritarianism, *Policmakers* → Policymakers. Better to fix the form.
- The registration field mixes URLs, e-mail addresses and free text. Only a real URL becomes
  a working button.

# Programme explorer (`explorer-embed.html`)

A second, full-width view of the same programme: by day, timeline, by theme, by organisation,
and a "My programme" list visitors keep in their own browser. It reads the same `events.json`,
so every programme update reaches both views at once. It does not replace `events-embed.html`.

It also reads, from this repo, `host-aliases.json` (other spellings of a co-host's name, so
the right logo appears), and from `thehaguehumanityhub/thjw` the organisations, links and
logos of the co-hosts grid.

## Shortcoder snippet

```html
<div id="thjw-x-host">Loading the programme…</div>
<script>
(function(){
  // Optional: where "Back to website" goes, and a photo behind the header.
  window.THJW_EXPLORER = {
    backUrl: "https://humanityhub.org/the-hague-justice-week/"
    // , heroImage: "https://humanityhub.org/wp-content/uploads/…/photo.jpg"
  };
  var URL = "https://cdn.jsdelivr.net/gh/thehaguehumanityhub/justice-week-events@main/explorer-embed.html";
  fetch(URL, { cache: "no-cache" }).then(function(r){ return r.text(); }).then(function(html){
    var host = document.getElementById("thjw-x-host");
    host.innerHTML = html;
    host.querySelectorAll("script").forEach(function(old){
      var s = document.createElement("script");
      if (old.src) s.src = old.src; else s.textContent = old.textContent;
      old.parentNode.replaceChild(s, old);
    });
  }).catch(function(){
    document.getElementById("thjw-x-host").textContent = "Could not load the programme.";
  });
})();
</script>
```

Name the snippet `jw-explorer`, then put `[sc name="jw-explorer"]` on its own page. The
`cache: "no-cache"` option matters: without it a browser keeps its copy of the explorer for up
to seven days after a purge, so an update seems not to have arrived. The
explorer spreads to the full width of the window, so a page without a sidebar suits it best.

## Updating

- The programme: nothing extra. The usual `events.json` update and purge covers both views.
- A host shows initials instead of its logo although it is in the co-hosts grid: add the
  spelling it uses to `host-aliases.json`, commit, and purge that file.
- After changing `explorer-embed.html` itself, purge it:

  ```
  https://purge.jsdelivr.net/gh/thehaguehumanityhub/justice-week-events@main/explorer-embed.html
  ```

## Testing on this computer

```
python3 -m http.server 8765 --bind 127.0.0.1 --directory ~/Developer
```

then open `http://localhost:8765/justice-week-events/dev/explorer.html`. It loads the local
files, not the CDN. Add `?now=2026-11-18T14:10` to preview the "today" features of the week.
