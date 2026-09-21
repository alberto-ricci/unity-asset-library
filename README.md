# Unity Asset Logger (Firefox extension)

Detects when you acquire a Unity Asset Store package — free or paid — and
logs it to a local, searchable list inside the extension. No servers, no
webhooks, no third-party services. Everything lives in your browser.

## How it works

- A content script runs on `assetstore.unity.com` pages and watches for the
  page to show an **"Open in Unity"** button. Unity only shows that once a
  package is genuinely yours, so it's a reliable signal for both free
  ("Add to My Assets") and paid ("Buy Now" → checkout) acquisitions — and it
  won't fire just because you're revisiting something you already own.
- When it fires, the asset's name, publisher, and (best-effort) category are
  captured and stored locally via `browser.storage.local`.
- Click the toolbar icon to see everything logged so far, search it, add
  tags, and export it to CSV whenever you want.

## Known limitation — please read

I built and tested the **free-asset path** against Unity's own documented
behavior (button swaps to "Open in Unity" in place, no navigation), so that
part is solid.

The **paid-purchase path** is my best effort, not a verified one — I don't
have an authenticated session on the Asset Store to see exactly what page
Unity redirects you to after a successful payment. The content script is
allowed to run on any page under `assetstore.unity.com` so it has the best
chance of catching the transition wherever it happens, but if a paid
purchase doesn't show up in the log, that's the first thing to check:
tell me what page/URL you land on right after paying, and I'll adjust the
detection logic to match it exactly.

## Installation

This is distributed as a self-signed add-on (via AMO's unlisted channel), not
a store listing, so installation is a couple of manual steps:

**Desktop:** `about:addons` → gear icon → **Install Add-on From File...** →
pick the `.xpi` you were given.

**Android:** Settings → About Firefox → tap the Firefox logo 5× quickly (this
unlocks a hidden option) → back out to Settings → **Install Extension from
File** → pick the same `.xpi`.

After installing, check `about:addons` → this extension → **Permissions**
tab, and make sure access to `assetstore.unity.com` is turned on — it can
default to off after a fresh install, which silently blocks all detection
until enabled.

## Using it

1. Browse the Asset Store as normal. Buy or add whatever you like.
2. The toolbar badge shows your total asset count, and changes color based
   on sync state: **blue** means everything's pushed to GitHub, **amber**
   means something's waiting on the next sync (a new capture, or an edit).
3. Click the icon to see the list. Below the sync status line you'll find
   the same total, a pending count (or "All synced"), and a countdown to
   the next automatic sync.
4. Type a **Category** and **Tags** for anything missing them (free text —
   type whatever fits, no fixed list to fight with). New captures get a
   baseline set of tags derived automatically from their category, but feel
   free to refine them.
5. Use the search box at the top to filter by tag, name, or publisher.
6. Click **Export CSV** any time to get a file with everything logged —
   you can open it in Excel, or just paste its contents into a chat with me
   and I'll fold it into your main catalog with proper Purpose text and
   normalized categories, same as we've been doing.

## GitHub sync (optional)

Captures and edits save locally right away, then get batched: a periodic
check (once a day by default, configurable in Settings) pushes everything
in one go if anything's pending, rather than firing a GitHub request on
every single change. The popup's **Sync now** link forces an immediate push
regardless of the timer.

**Setup:**

1. Create a new GitHub repo (public, or at least keep this one file
   public — see why below). You don't need to create the file inside it
   first; the extension creates it on the first sync.
2. Generate a **fine-grained** personal access token: GitHub → Settings →
   Developer settings → Personal access tokens → Fine-grained tokens →
   Generate new token → under "Repository access" choose "Only select
   repositories" and pick this one → under "Permissions" set
   **Contents: Read and write**. Nothing else.
3. Click the toolbar icon → **Settings** → enter your GitHub username,
   the repo name, the file path (`library.json` by default), the branch
   (`main` by default), and the token → **Save**.
4. Add or edit an asset, or click **Sync now**, to trigger the first push.
   The popup's status bar shows whether it went through.

**Commit messages** follow this format, so the repo's history is a readable
log rather than a wall of identical "Update library" entries:

```
<Auto|Manual> sync: <total> total (<delta>) — last updated <timestamp UTC>
```

`<delta>` is `+N new`, `N removed`, or `no count change` (an edit with no
net change in item count) since the previous successful push.

**Why public:** if you want to hand a URL to Claude in a chat so it can
read your library back with no upload on your end, that URL needs to be
fetchable without credentials — Claude only ever holds the URL, never your
GitHub token. The data involved (asset names, publishers, categories, tags)
isn't sensitive, but it's your call.

**Giving Claude a reliable URL:** the Netlify-hosted JSON turned out to be
unreliable for Claude specifically to read fresh (it kept returning stale
cached content in testing, for reasons that trace back to Claude's own
fetch tooling rather than anything wrong with Netlify). The link that
worked reliably was GitHub's own file view:

`https://github.com/<owner>/<repo>/blob/<branch>/<path>`

Give Claude that link once and it can check the current library in future
conversations without needing anything re-uploaded.

## Importing existing history

Settings has an **Import existing library** section for merging in a JSON
file (a prior export, or a corrected file someone's handed you back):

- Assets not already present (matched by publisher + name) are added.
- Assets already present get any **currently blank** fields — Category,
  Purpose, Tags, Size, Last Updated, Version, URL — filled in from the
  import, without touching anything already typed in by hand.
- Nothing is ever deleted or overwritten with a blank.

This is how a corrected file (say, one where I've written proper Purpose
text for a batch of new captures) can be brought back in safely — it fills
gaps rather than replacing what's already there.

## Files

- `manifest.json` — extension configuration
- `content.js` — runs on the Asset Store, detects acquisitions
- `background.js` — stores captures, updates the toolbar badge, syncs to GitHub
- `popup.html` / `popup.js` — the list, search, export, and sync status UI
- `options.html` / `options.js` — GitHub sync settings and library import/merge

