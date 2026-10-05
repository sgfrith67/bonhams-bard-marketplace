---
name: bard-home
description: >
  This skill should be used when a Bonhams user asks for the BARD home page or dashboard:
  "open BARD", "bonhams home", "Bonhams home page", "BARD home page", "BARD dashboard",
  "set up BARD", "/bard-home", the
  jewellery market-share or estimate-performance page, or asks to refresh, rebuild,
  restyle or re-publish that page. Opens the team's shared page when one exists, and
  otherwise builds and publishes one interactive page, with a Specialist view and a
  Management view, that loads live data from the bonhams-bard connector each time it is
  opened.
metadata:
  version: "0.2.1"
---

# BARD home page

The team should share **one** BARD page: one link, one set of figures, updated in one place. The page calls the viewer's own `bonhams-bard` connector (the read-only `bard_home` tool) each time it is opened, so a page someone else published shows live data to anyone with the connector. The page sends no SQL and needs no database access of its own.

So start by looking for an existing page, and build a new one only when there is none or the user asks for it.

Bundled files:

- `assets/template.html`: the page (Bonhams styling, both views, live loader, stale-data banner)
- `assets/snapshot.json`: fallback data, shown only when live data cannot load
- `scripts/build_page.py`: fills the template and writes the HTML
- `references/page-guide.md`: how each figure is calculated

## Workflow

### 0. Look for an existing page first

Call the Artifact tool with `action: "list"` and `scope: "all"` (the user's own pages and pages shared with them). Look for the title **BARD — Bonhams Auction Research Database**.

| What the list shows | User asked to open or see BARD | User asked to rebuild, refresh, restyle or change it |
|---|---|---|
| A page shared by someone else | `action: "open"` with its link; stop. Say whose page it is and that it shows live data through their connector. | It can't be updated by this user. Say who owns it and suggest asking them; build a personal copy (steps 1 to 6, without `url`) only if the user still wants one. |
| The user's own page | `action: "open"` with its link; stop. | Continue with steps 1 to 6, publishing to that page's link. |
| Both | Prefer the shared page, and mention that the user also has their own copy. | Update the user's own page. |
| Neither | Continue with steps 1 to 6 and publish a new page. Afterwards, suggest the user share it with the team so colleagues open it instead of each publishing a copy. | Same. |

If several shared pages match, open the most recently updated and mention the others.

### 1. Check the connector and the data

Find the `bonhams-bard` connector's `bard_home` tool (if tools are deferred, search "bonhams-bard bard_home"). If there is no such connector, still build and publish: the page shows the bundled snapshot. Tell the user live data needs the connector added in claude.ai Settings, Connectors, and skip step 2.

Otherwise call `bard_home` once. If the database was asleep the first call can time out; retry once. Read `generated_at` and `meta` (latest sale and caveat per house). Keep these for the summary. A house whose latest sale is more than 60 days old is flagged on the page automatically.

### 2. Decide whether to refresh the fallback snapshot

The snapshot appears only when the page is opened without working live data. Refresh it when the user asks for fresh data, when they will share the page with colleagues who may lack the connector, or when the build step reports `snapshot_age_days` over 30. Otherwise use the bundled one.

To refresh, write the `bard_home` result, unchanged, to `/home/claude/bard_result.txt` and pass it as `--result` in step 3.

### 3. Build

```bash
python scripts/build_page.py \
  --server "bonhams-bard" \
  --out /mnt/user-data/outputs/bard-home.html
  # add --result /home/claude/bard_result.txt if refreshing the snapshot
  # add --yellow "#RRGGBB" only if the user gives Bonhams' exact yellow
```

Run it from the skill directory. Use the connector's display name for `--server` if it differs from `bonhams-bard`. The script prints JSON with the output path and snapshot age. If it fails, read the message; a malformed `--result` file is the usual cause. Retry without `--result` rather than hand-editing the template.

### 4. Publish, or update the existing page

Publish to the user's own page found in step 0 (pass its link as `url`), or create a new one:

- `file_path`: `/mnt/user-data/outputs/bard-home.html`
- `title`: `BARD — Bonhams Auction Research Database`
- `favicon`: 💎
- `capabilities`: `{"mcp": {"servers": [{"server": "bonhams-bard", "tools": ["bard_home"]}]}}`
- `url`: the user's own existing page's link, if step 0 found one. Never pass a page owned by someone else.

The capabilities declaration is what lets the page call the viewer's connector. Without it the page only ever shows the snapshot. Declare only `bard_home`; the page needs no other tool.

If the Artifact tool is unavailable, present the HTML file and say it will show the snapshot only.

### 5. Switching an older page over

Pages published before version 0.2.0 called a Supabase connector with SQL. Republishing to the same link with the steps above replaces that: the capabilities declaration changes to `bonhams-bard` and the viewer is asked to approve the new connector once.

### 6. Tell the user what they have

Keep it short: the link and whether it is new or updated; for a new page, that sharing it with the team (from the page's share option) lets colleagues open the same page rather than publish their own; that the first open asks permission to use the connector once; that Specialist and Management switch at the top and USD/AUD converts every figure; any data caveats from step 1, especially a house with no recent sales; and whether the snapshot was refreshed and its date.

## Things that trip people up

| What they see | Cause | What to tell them |
|---|---|---|
| "Showing the saved snapshot…" | Page opened outside claude.ai, or connector not added | Open it in claude.ai with the bonhams-bard connector connected |
| "Add the bonhams-bard connector…" | No connector with the expected name | Add it in Settings, Connectors, or rebuild with `--server` set to their connector's name |
| "Reconnect bonhams-bard…" | Sign-in lapsed | Reconnect in Settings, Connectors, then press Try again |
| "Live data is turned off for this page" | They declined the connector prompt | Re-publish and accept the prompt on next open |
| "BARD reported an error" or a long wait | Database asleep or a hiccup | Press Try again; if it persists, run the `data-health-check` skill |
| "Incomplete data" banner | A house has no sale captured for over 60 days | Expected; competitor figures for that house are understated. See `data-health-check` |

## Changing the page

For small changes (wording, sections, colours), edit `assets/template.html`, rebuild and re-publish to the same link. Only the page's owner can republish it; if the shared page belongs to someone else, tell the user to ask the owner. Keep the placeholders `__SNAPSHOT__`, `__SERVER__` and `__YELLOW__`; the build fails loudly if one is missing. If a change needs data the page does not already receive, the data owner must add it to the `bard_home` model; do not add a second tool to the page without declaring it in the capabilities.

Answer questions about how a figure is calculated from `references/page-guide.md`.
