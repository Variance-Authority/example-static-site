# Visual regression for a built static site

Point Variance Authority at a folder of built HTML and it watches every page in
it, telling you when one starts rendering differently than the version you
approved. No test framework, no server, no service — the exit code is the whole
CI integration.

Two pages, one shared stylesheet, four commands.

## 1. Take the first look

```bash
npm install
npx playwright install chromium
npx variance run
```

```
2 subject(s) observed, durable run at 2026-11-08T03:04:46.726Z
rendered by playwright-chromium (chromium@156.0.8078.4, darwin/arm64, 1x)
2 changed

[changed] checkout: … the baseline and this run were painted under different identities (platform linux/x64 → darwin/arm64, …); 2,251 pixels differ across 3 regions; …
[changed] pricing: … the baseline and this run were painted under different identities (platform linux/x64 → darwin/arm64, …); 1,871 pixels differ across 3 regions; …
```

Exit code `1`. The baselines in this repository were painted by CI on Linux,
and your laptop rasterises text differently, so the run compares against them
and names the platform it moved from as the cause rather than blaming the page.

## 2. Approve what you see

```bash
npx variance accept checkout pricing
npx variance run
```

Exit code `0`. Named subjects rather than `accept --all`, because `--all`
cannot tell a baseline nobody has ever reviewed from one that just changed.

## 3. Break it

Both pages render the same `.card`. Widen its corner in one place —
`border-radius: 6px` to `14px` in [site/style.css](site/style.css) — and run
again:

```
2 changed

[changed] checkout: 110 pixel(s) differ across 8 region(s)
[changed] pricing: 110 pixel(s) differ across 8 region(s)
```

Exit code `1`. One line of CSS, two pages. The run names each page it affected
and counts the regions that moved, so you know how far the change reached
before you open a single image.

Accept it and the new renderings become the baselines; revert the stylesheet
and the next run is green with nothing to approve.

## Which baselines are in this repository

A baseline records the renderer that painted it, in the file beside the image.
The baselines under [baselines/](baselines) were painted by CI, in the
Playwright image its workflow pins, and committed through git-LFS. A run on
another machine compares against them and names what moved — the platform, the
engine — as the cause. Accepting on your laptop rewrites them with your
machine's pixels, so leave those out of your commits.

## On a pull request

[`.github/workflows/variance.yml`](.github/workflows/variance.yml) renders
both pages on every pull request and posts what changed as a comment. To accept
a change, run the workflow from the Actions tab on your branch with **accept**
ticked. That run accepts every changed page and commits the new images to your
branch, so they arrive in the diff you
approve and merge with the code that changed them.

A commit pushed with the workflow's own token starts no workflow. Set a
`VARIANCE_PUSH_TOKEN` secret with `contents: write`, and the accept commit gets
a check of its own.

Every week the variance-authority repository opens a pull request here that
changes a price on `pricing.html`. It checks that the check goes red, accepts
with a dispatch on its branch, checks the commit that comes back, and closes the pull request
without merging.

## What is where

| file | what it does |
|---|---|
| [variance.config.json](variance.config.json) | the whole configuration: viewport, browser, where baselines land |
| [variance/routes.mjs](variance/routes.mjs) | turns every `.html` under `./site` into a subject, narrowed to its `<main>` |
| [site/](site) | stands in for your build output — replace it with yours |
| [baselines/](baselines) | the accepted images, stored through git-LFS |

## What this path does not do

It compares whole pages. It does not name the component that caused a change,
because built HTML no longer records which source file produced which element
and nothing in the browser can recover that afterwards. When the cause matters
more than the diff, the run needs source provenance — which is what the
Storybook and React paths carry.
