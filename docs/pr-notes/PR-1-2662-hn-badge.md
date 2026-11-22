# PR #1 — Hacker News badge in the README badge row

| | |
|---|---|
| **Branch** | `docs/2662-hn-badge` → `main` (this fork) |
| **Upstream issue** | <https://github.com/JustVugg/colibri/issues/2662> |
| **Kind** | Documentation / README |
| **Risk** | Very low — no code paths touched |

## What this changes

Adds a Hacker News badge, linking to the project's HN discussion
(item `48842459`), to the centred badge row at the top of every README.

One line is added per file. Nothing is reordered, reworded or removed.

## Files changed

`README.md`, `README.it.md`, `README.ja.md`, `README.id.md`, `README.pt-BR.md`,
`README.zh-CN.md`, `README.zh-TW.md` — 7 files, +7 / −0.

All seven carry an identical badge block (a `<p align="center">` with two
2-space-indented shields: website, then release). The HN badge is inserted as a
third line in that block, with the same indentation, immediately after the
release shield.

The `alt` text stays `"Hacker News"` in every file, including the translated
ones. The neighbouring alts are translated (`Sito web`, `网站`), but "Hacker
News" is a proper noun — translating it would invent a label.

## How to run it

### Requirements

None. There is nothing to build or execute.

### Verify

```bash
# 1. The badge line is present exactly once per README.
grep -c 'hackergit.org/badge' README*.md

# 2. It sits inside the centred badge block, not loose in the page.
grep -n -A1 -B2 'hackergit.org/badge' README.md

# 3. The badge image and the HN link resolve.
curl -s -o /dev/null -w 'badge: HTTP %{http_code} %{content_type}\n' \
  https://hackergit.org/badge/JustVugg/colibri.svg
```

### Observed result

```
$ grep -c 'hackergit.org/badge' README*.md
README.id.md:1
README.it.md:1
README.ja.md:1
README.md:1
README.pt-BR.md:1
README.zh-CN.md:1
README.zh-TW.md:1

$ curl -s -o /dev/null -w 'badge: HTTP %{http_code} %{content_type}\n' ...
badge: HTTP 200 image/svg+xml; charset=utf-8
```

The small badge SVG is natively 171×20, which lines up with the existing 20px
shields. The `?style=card` variant also returns HTTP 200 but is 250×55 — about
2.75× taller than its neighbours — so it would break the single centred line.

## Design note: badge style

The issue author offered two markups, and in the thread the upstream owner
leaned toward the **card** style. This PR uses the **small** style instead, to
match the house layout of all seven READMEs. Swapping is a one-line change:

```html
<a href="https://news.ycombinator.com/item?id=48842459"><img
  src="https://hackergit.org/badge/JustVugg/colibri.svg?style=card"
  alt="Hacker News" width="250" height="55"></a>
```

If the card is preferred, give it its own `<p align="center">` block below the
existing row rather than inlining it, so the 20px shields stay aligned.

## What was NOT verified

- **Rendered appearance on GitHub.** The markup was checked by reading it
  against the neighbouring badges; no page was rendered.
- **Badge content over time.** `hackergit.org` is a third-party service outside
  this project's control. Rendering the README makes every visitor fetch from
  that domain, and the badge text is whatever the service returns (at the time
  of writing, "Hacker News: 937 points"). Accepting that dependency is a
  maintainer's call.

## Reverting

```bash
git revert 2633d2f4
```
